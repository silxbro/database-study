# 04. Array Processing 활용

Array Processing 기능을 활용하면 한 번의 SQL 수행으로 다량의 로우를 동시에 insert/update/delete 할 수 있다.
이는 네트워크를 통한 데이터베이스 Call을 감소시켜주고, 궁극적으로 SQL 수행시간과 CPU 사용량을 획기적으로 줄여준다.

앞 절에서 '납입방법별_월요금집계' 테이블을 가공하는 사례를 보았는데, 이를 Java 프로그램에서 Array Processing을 이용하는 방식으로 바꾸어 보자.

```
public class JavaArrayProcessing{
  public static void insertData( Connection con
                               , PreparedStatement st
                               , String param1
                               , String param2
                               , String param3
                               , long param4) throws Exception(
    st.setString(1, param1);
    st.setString(2, param2);
    st.setString(3, param3);
    st.setLong(4, param4);
    st.addBatch();
  }

  public static void execute(Connection con, String input_month)
  throws Exception {
    long rows = 0;
    String SQLStmt1 = "SELECT 고객번호, 납입월"
                    + ", 지로, 자동이체, 신용카드, 핸드폰, 인터넷 "
                    + "FROM  월요금납부실적 "
                    + "WHERE 납입월 = ?";
    String SQLStmt2 = "INSERT INTO 납입방법별_월요금집계 "
            + "(고객번호, 납입월, 납입방법코드, 납입금액) "
            + "VALUES(?, ?, ?, ?)";

    con.setAutoCommit(false);

    PreparedStatement stmt1 = con.prepareStatement(SQLStmt1);
    PreparedStatement stmt2 = con.prepareStatement(SQLStmt2);
    stmt1.setFetchSize(1000);
    stmt1.setString(1, input_month);
    ResultSet rs = stmt1.executeQuery();
    while (rs.next()) {
      String 고객번호 = rs.getString(1);
      String 납입월 = rs.getString(2);
      long 지로 = rs.getLong(3);
      long 자동이체 = rs.getLong(4);
      long 신용카드 = rs.getLong(5);
      long 핸드폰 = rs.getLong(6);
      long 인터넷 = rs.getLong(7);

      if (지로 > 0)
          insertData (con, stmt2, 고객번호, 납입월, "A", 지로);

      if (자동이체 > 0)
          insertData (con, stmt2, 고객번호, 납입월, "B", 자동이체);

      if (신용카드 > 0)
          insertData (con, stmt2, 고객번호, 납입월, "C", 신용카드);

      if (핸드폰 > 0)
          insertData (con, stmt2, 고객번호, 납입월, "D", 핸드폰);

      if (인터넷 > 0)
          insertData (con, stmt2, 고객번호, 납입월, "E", 인터넷);

      if(++rows%1000 == 0) stmt2.executeBatch();

    }

    rs.close();
    stmt1.close();

    stmt2.executeBatch();
    stmt2.close();

    con.commit();
    con.setAutoCommit(true);

  }

  static Connection getConnection() throws Exception { ...... }
  static void releaseConnection(Connection con) throws Exception { ...... }

  public static void main(String[] args) throws Exception(
    Connection con = getConnection();
    execute(con, "200903");
    releaseConnection(con);
  }
}
```

필자의 로컬 PC에서 테스트해 본 결과, 150,000(=30,000✕5)건을 insert 하는데 단 1.21초 만에 수행을 완료하였다.
아래는 SQL 트레이스 결과인데, insert문에 대한 Execute Call이 30회만 발생한 것에 주목하자.

```
SELECT 고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷
FROM   월요금납부실적 WHERE 납입월 = :1

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute      1   0.01       0.00       0       71          0       0
Fetch       31   0.00       0.04       0      169          0   30000
------- ------ ------ ---------- ------- -------- ---------- -------
total       33   0.01       0.04       0      240          0   30000

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54

Rows   Row Source Operation
-----  ----------------------------------------------------
30000  TABLE ACCESS FULL 월요금납부실적 (cr=169 pr=0 pw=0 time=90083 us)

*************************************************************************

INSERT INTO 납입방법별_월요금집계
(고객번호, 납입월, 납입방법코드, 납입금액)
VALUES (:1, :2, :3, :4)

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute     30   0.18       0.27       2      923       5094  150000
Fetch        0   0.00       0.00       0        0          0       0
------- ------ ------ ---------- ------- -------- ---------- -------
total       31   0.18       0.27       2      923       5094  150000

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54
```

insert된 로우 수가 150,000건이므로 매번 5,000건씩 Array Processing한 것을 알 수 있다.
커서에서 Fetch되는 각 로우마다 5번씩 insert를 수행하는데, 1,000 로우마다 한 번씩 executeBatch를 수행하기 때문에 얻게 된 결과다.

참고로, select 결과를 Fetch 할 때도 1,000개 단위로 Array Fetch 하도록 조정하였다. 30,000건을 읽는데 Fetch Call이 31회만 발생한 것을 확인하기 바란다.
JAVA에서 FetchSize를 조정하지 않으면 기본적으로 10개 단위로 Array Fetch를 수행한다. Fetch Call 최소화 원리는 다음 절에서 다룬다.
<br/><br/><br/>
아래 표는 앞 절에서 수행한 3가지 테스트와 방금 확인한 Array Processing 결과를 쉽게 비교할 수 있도록 정리한 것이다.
네트워크를 경유해 발생하는 데이터베이스 Call이 얼마만큼 심각한 성능부하를 일으키는지 알 수 있다.
그뿐만 아니라 One-SQL로 통합하지 않더라도 Array Processing 만으로 그에 버금가는 성능개선 효과를 얻을 수 있음을 잘 보여준다.

||PL/SQL<br/>(Recursive)|JAVA|JAVA<br/>(Array 처리)|One-SQL|
|:---:|---:|---:|---:|---:|
|Parse Call|5|150,000|1|1|
|Execute Call|150,000|150,000|30|1|
|총 소요시간|7.26초|126.82초|1.21초|0.9초|
- Parse Call과 Execute Call은 insert문에 대한 것만 집계한 것임

Array Processing의 효과를 극대화하려면 연속된 일련의 처리과정이 모두 Array 단위로 진행돼야 한다.
이를테면, 앞선 단계에서 Array 단위로 수천 건씩 아무리 빠르게 Fetch 하더라도 다음 단계에서 수행할 insert가 건건이 처리된다면 그 효과가 크게 반감되며, 반대의 경우도 마찬가지다.
이것은 병렬 프로세싱에서 '병렬로부터 직렬(P→S)' 또는 '직렬로부터 병렬(S→P)'로 처리되는 부분이 병목을 일으키는 것과 같은 이치다.

이해를 돕기 위해 PL/SQL을 이용해 데이터를 Bulk로 1,000건씩 Fetch 해서 Bulk로 insert하는 예제 프로그램을 작성해 보았다.

```
-- 데이터를 Bulk로 읽을 Source 테이블
create table emp
as
select object_id empno, object_name ename, object_type job
     , round(dbms_random.value(1000, 5000), -2) sal
     , owner deptno, created hiredate
from   all_objects
where  rownum <= 10000;

-- 데이터를 Bulk로 넣을 Target 테이블
create table emp2
as
select * from emp where 1 = 2
;

DECLARE
  l_fetch_size NUMBER DEFAULT 1000;  -- 1,000건씩 Array 처리

  CURSOR c IS
    SELECT empno, ename, job, sal, deptno, hiredate
    FROM   emp;

  TYPE array_empno      IS TABLE OF emp.empno%type;
  TYPE array_ename      IS TABLE OF emp.ename%type;
  TYPE array_job        IS TABLE OF emp.job%type;
  TYPE array_sal        IS TABLE OF emp.sal%type;
  TYPE array_deptno     IS TABLE OF emp.deptno%type;
  TYPE array_hiredate   IS TABLE OF emp.hiredate%type;

  l_empno      array_empno     := array_empno   ();
  l_ename      array_ename     := array_ename   ();
  l_job        array_job       := array_job     ();
  l_sal        array_sal       := array_sal     ();
  l_deptno     array_deptno    := array_deptno  ();
  l_hiredate   array_hiredate  := array_hiredate();

  PROCEDURE insert_t( p_empno      IN array_empno
                    , p_ename      IN array_ename
                    , p_job        IN array_job
                    , p_sal        IN array_sal
                    , p_deptno     IN array_deptno
                    , p_hiredate   IN array_hiredate ) IS

  BEGIN
    FORALL i IN p_empno.first..p_empno.last
      INSERT INTO emp2
      VALUES( p_empno   (i)
            , p_ename   (i)
            , p_job     (i)
            , p_sal     (i)
            , p_deptno  (i)
            , p_hiredate(i) );

  EXCEPTION
    WHEN others THEN
      DBMS_OUTPUT.PUT_LINE(SQLERRM);
  END insert_t;

BEGIN

  OPEN c;

  LOOP

    FETCH c BULK COLLECT
    INTO l_empno, l_ename, l_job, l_sal, l_deptno, l_hiredate
    LIMIT l_fetch_size;

    insert_t( l_empno, l_ename, l_job, l_sal, l_deptno, l_hiredate );

    EXIT WHEN c%NOTFOUND;

  END LOOP;

  CLOSE c;

  COMMIT;

EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
END;
/
```

아래 SQL 트레이스 결과를 보면, 10,000건을 처리하는데 select문의 Fetch Call과 insert문의 Execute Call이 각각 10번씩만 발생한 것을 알 수 있다.
(Fetch Call이 1번 더 발생한 것은 데이터가 더 있는지 확인하기 위한 것임)

```
SELECT EMPNO, ENAME, JOB, SAL, DEPTNO, HIREDATE
FROM   EMP

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.01       0.00       0        1          0       0
Execute      1   0.00       0.00       0        0          0       0
Fetch       11   0.01       0.01       0       82          0   10000
------- ------ ------ ---------- ------- -------- ---------- -------
total       13   0.03       0.02       0       83          0   10000

*********************************************************************

INSERT INTO EMP2
VALUE ( :B1, :B2, :B3, :B4, :B5, :B6 )

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        1          0       0
Execute     10   0.03       0.09       0      142        972   10000
Fetch        0   0.00       0.00       0        0          0       0
------- ------ ------ ---------- ------- -------- ---------- -------
total       11   0.03       0.09       0      142        972   10000
```

참고로, EXP, IMP 명령을 통해 데이터를 Export, Import 할 때도 내부적으로 Array processing이 활용되며, 그만큼 대용량 데이터를 처리하는 데 있어 Array Processing은 필수적인 요소다.
Array Processing을 지원하는 인터페이스가 프로그램 언어별로 각기 다르므로 API를 통해 확인하고 이를 잘 활용해서 성능개선 효과가 얼마나 극적인지 직접 확인해 보기 바란다.