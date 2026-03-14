# 03. 데이터베이스 Call이 성능에 미치는 영향

\[그림 5-3\](p.353)에 두 테이블이 있다.
데이터 모델을 보고 쉽게 짐작할 수 있듯이 '월요금납부실적' 테이블은 고객별 납입방법별 납입요금을 컬럼 값으로 입력하고, '납입방법별_월요금집계' 테이블은 납입요금을 납입방법코드별로 하나의 레코드로
입력하도록 하고 있다.

기간계 시스템에서는 요금납부실적을 주로 고객별로 조회하므로 흔히 좌측과 같이 모델링하는 반면(그렇게 하는 게 낫다는 뜻은 아님), 정보계 시스템에서는 다차원 분석을 많이 하므로 우측과 같이 납입방법코드를
PK로 끌어올리는 것이 일반적이다.
이처럼 기간계와 정보계 시스템이 서로 다른 데이터 모델을 사용하기 때문에 발생하는 데이터 가공 요건은 매우 흔하며, 여기서는 '월요금납부실적' 테이블을 이용해 '납입방법별_월요금집계' 테이블 형태로
가공하는 ETL 배치 프로그램이 필요하다고 가정하자.

대용량 데이터를 가공해 본 경험이 많지 않은 개발자들은 위와 같은 업무요건을 만나면 흔히 아래 같은 패턴으로 프로그램을 작성한다.

```
DECLARE
  CURSOR C(INPUT_MONTH VARCHAR2) IS
    SELECT 고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷
    FROM   월요금납부실적
    WHERE  납입월 = INPUT_MONTH;

  REC C%ROWTYPE;
  LTYPE VARCHAR2(1);
BEGIN
  OPEN C('200903');

  LOOP
    FETCH C INTO REC;

    EXIT WHEN C%NOTFOUND;

    IF REC.지로 > 0 THEN
      LTYPE := 'A';
      INSERT INTO 납입방법별_월요금집계 (고객번호, 납입월, 납입방법코드, 납입금액)
      VALUES (REC.고객번호, REC.납입월, LTYPE, REC.지로);
    END IF;

    IF REC.자동이체 > 0 THEN
      LTYPE := 'B';
      INSERT INTO 납입방법별_월요금집계 (고객번호, 납입월, 납입방법코드, 납입금액)
      VALUES (REC.고객번호, REC.납입월, LTYPE, REC.자동이체);
    END IF;

    IF REC.신용카드 > 0 THEN
      LTYPE := 'C';
      INSERT INTO 납입방법별_월요금집계 (고객번호, 납입월, 납입방법코드, 납입금액)
      VALUES (REC.고객번호, REC.납입월, LTYPE, REC.신용카드);

    IF REC.핸드폰 > 0 THEN
      LTYPE := 'D';
      INSERT INTO 납입방법별_월요금집계 (고객번호, 납입월, 납입방법코드, 납입금액)
      VALUES (REC.고객번호, REC.납입월, LTYPE, REC.핸드폰);
    END IF;

    IF REC.인터넷 > 0 THEN
      LTYPE := 'E';
      INSERT INTO 납입방법별_월요금집계 (고객번호, 납입월, 납입방법코드, 납입금액)
      VALUES (REC.고객번호, REC.납입월, LTYPE, REC.인터넷);
    END IF;

  END LOOP;

  COMMIT;

  CLOSE C;

END;
```

개발 초기에는 소량의 테스트용 데이터만으로 프로그램을 수행하므로 무리 없이 돌아가지만 프로젝트가 통합 테스트 단계로 접어들어 실 데이터들이 이관되기 시작하면 갑자기 문제가 심각해진다.
밤새 돌고도 아침까지 끝나지 않는 프로그램들이 부지기수다. 무엇이 문제인가? 문제는 과도한 데이터베이스 Call에 있다.

만약 처리해야 할 월요금납부실적이 100만 건이면 이 테이블에 대한 Fetch Call이 100만 번(Array 단위 Fetch가 작동하지 않을 때), 납입방법별_월요금집계 테이블로의 insert를 위한 Execute
Call이 최대 500만 번, 따라서 최대 600만 번의 데이터베이스 Call이 발생하게 된다.(PL/SQL에서는 커서를 자동으로 캐싱하므로 insert를 위한 Parse Call은 소량만 발생한다.)

그나마 위처럼 PL/SQL문으로 코딩하면 네트워크 트래픽 없는 Recursive Call이므로 제법 빠르게 수행된다. 하지만 C, JAVA, VB, Delphi 등으로 개발된 애플리케이션에서 네트워크를 경유해 수행할 때는
문제가 아주 심각해진다. 아래는 JAVA 프로그램으로 작성한 예제다.

```
public class JavaLoopQuery{
  public static void insertDate( Connection con
                               , String param1
                               , String param2
                               , String param3
                               , long param4) throws Exception(
    String SQLStmt = "INSERT INTO 납입방법별_월요금집계 "
            + "(고객번호, 납입월, 납입방법코드, 납입금액) "
            + "VALUES(?, ?, ?, ?)";
    PreparedStatement st = con.prepareStatement(SQLStmt);
    st.setString(1, param1);
    st.setString(2, param2);
    st.setString(3, param3);
    st.setLong(4, param4);
    st.execute();
    st.close();
  }

  public static void execute(Connection con, String input_month)
  throws Exception {
    String SQLStmt = "SELECT 고객번호, 납입월"
                   + ", 지로, 자동이체, 신용카드, 핸드폰, 인터넷 "
                   + "FROM  월요금납부실적 "
                   + "WHERE 납입월 = ?";
    PreparedStatement stmt = con.prepareStatement(SQLStmt);
    stmt.setString(1, input_month);
    ResultSet rs = stmt.executeQuery();
    while(rs.next()){
      String 고객번호   = rs.getString(1);
      String 납입월     = rs.getString(2);
      long   지로       = rs.getLong(3);
      long   자동이체    = rs.getLong(4);
      long   신용카드    = rs.getLong(5);
      long   핸드폰     = rs.getLong(6);
      long   인터넷     = rs.getLong(7);
      if (지로 > 0)      insertData  (con, 고객번호, 납입월, "A", 지로);
      if (자동이체 > 0)   insertData  (con, 고객번호, 납입월, "B", 자동이체);
      if (신용카드 > 0)   insertData  (con, 고객번호, 납입월, "C", 신용카드);
      if (핸드폰 > 0)     insertData  (con, 고객번호, 납입월, "D", 핸드폰);
      if (인터넷 > 0)     insertData  (con, 고객번호, 납입월, "E", 인터넷);
    }
    rs.close();
    stmt.close();
  }

  static Connection getConnection() throws Exception { ...... }
  static void releaseConnection(Connection con) throws Exception { ...... }

  public static void main(String[] args) throws Exception{
    Connection con = getConnection();
    execute(con, "200903");
    releaseConnection(con);
  }
}
```

아래 CREATE 문으로 월요금납부실적 테이블에 30,000건을 넣고 실제 얼마나 소요되는지 테스트해 보자.

```
CREATE TABLE 월요금납부실적
AS
SELECT TO_CHAR(OBJECT_ID) 고객번호
     , '200903' 납입월
     , round(dbms_random.value(1000, 10000), -2) 지로
     , round(dbms_random.value(1000, 10000), -2) 자동이체
     , round(dbms_random.value(1000, 10000), -2) 신용카드
     , round(dbms_random.value(1000, 10000), -2) 핸드폰
     , round(dbms_random.value(1000, 10000), -2) 인터넷
FROM   ALL_OBJECTS
WHERE  ROWNUM <= 30000;

CREATE TABLE 납입방법별_월요금집계 (
  고객번호        NUMBER
, 납입월         VARCHAR2(6)
, 납입방법코드     VARCHAR2(1)
, 납입금액        NUMBER
) ;
```

필자의 로컬 PC에서 앞서 본 PL/SQL문으로 수행했을 때, 7.26초가 소요되었다.
아래는 그때의 SQL 트레이스 결과이고, select와 insert를 합쳐 총 18만 번가량의 데이터베이스 Call이 발생한 것을 알 수 있다.

```
SELECT 고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷
FROM
  월요금납부실적 WHERE 납입월 = :B1

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute      1   0.00       0.00       0        0          0       0
Fetch    30001   0.23       0.17       0    30004          0   30000
------- ------ ------ ---------- ------- -------- ---------- -------
total    30003   0.23       0.17       0    30004          0   30000

Misses in library cache during parse: 0
Optimizer mode: ALL_ROWS
Parsing user id: 54      (recursive depth: 1)

Rows     Row Source Operation
-------  ------------------------------------------------------
30000    TABLE ACCESS FULL 월요금납부실적 (cr=30004 pr=0 pw=0 time=151055 us)

**************************************************************************

INSERT INTO 납입방법별_월요금집계
(고객번호, 납입월, 납입방법코드, 납입금액)
VALUES (:B4, :B3, :B2, :B1 )

call      count    cpu    elapsed    disk    query    current    rows
------- ------- ------ ---------- ------- -------- ---------- -------
Parse         5   0.00       0.00       0        0          0       0
Execute  150000   2.62       2.47       2     2476     162171  150000
Fetch         0   0.23       0.17       0    30004          0       0
------- ------- ------ ---------- ------- -------- ---------- -------
total    150000   2.62       2.47       2     2476     162171  150000

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54      (recursive depth: 1)
```

반면, JAVA 프로그램을 수행할 때는 무려 126.82초가 소요되었다. 아래는 그때의 SQL 트레이스 결과이고, 총 303,000번 가량의 데이터베이스 Call이 발생한 것을 알 수 있다.

```
SELECT 고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷
FROM
   월요금납부실적 WHERE  납입월 = :1
  
call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute      1   0.00       0.00       2        2          0       0
Fetch     3001   0.14       0.20       9     3137          0   30000
------- ------ ------ ---------- ------- -------- ---------- -------
total     3003   0.14       0.20      11     3137          0   30000

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54

Rows     Row Source Operation
-------  ------------------------------------------------------
30000    TABLE ACCESS FULL 월요금납부실적 (cr=3135 pr=9 pw=0 time=165106 us)

**************************************************************************

INSERT INTO 납입방법별_월요금집계
(고객번호, 납입월, 납입방법코드, 납입금액)
VALUES (:1, :2, :3, :4)

call      count    cpu    elapsed    disk    query    current    rows
------- ------- ------ ---------- ------- -------- ---------- -------
Parse    150000   1.98       2.00       0        0          0       0
Execute  150000   8.75       9.20      27   150143     606212  150000
Fetch         0   0.00       0.17       0        0          0       0
------- ------- ------ ---------- ------- -------- ---------- -------
total    300000  10.73      11.20      27   150143     606212  150000

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54
```

select문에서 Fetch Call이 앞에서보다 1/10 수준으로 준 것은 JAVA에서 FetchSize 기본 설정이 10이기 때문이다. Array 단위 Fetch에 대해서는 뒤에서 다룬다.
그리고 JAVA에서 insert문은 애플리케이션 커서 캐싱 기법을 사용하지 않았으므로 Execute Call과 같은 횟수만큼 Parse Call이 발생했다.
PL/SQL에서는 자동으로 커서를 캐싱하므로 Parse Call이 5번에 그쳤다.

주목할 것은, JAVA에서 총 소요시간이 126.82초인데 반해 서버에서의 일량은 그에 훨씬 못 미친다는 사실이다.
순수하게 서버에서 처리한 시간은 10여 초에 불과하고 나머지는 네트워크 구간에서 소비한 시간, 그리고 데이터베이스 Call이 발생할 때마다 매번 OS로부터 CPU와 메모리 리소스를 할당받으려고 소비한 시간이다.
User Call이 Recursive Call에 비해 더 심각한 부하를 일으키는 이유가 바로 여기에 있다.
<br/><br/><br/>
아래와 같이 One-SQL로 통합하고 수행해 보면 1초가 채 걸리지 않는다.
```
INSERT INTO 납입방법별_월요금집계(납입월,고객번호,납입방법코드,납입금액)
SELECT x.납입월, x.고객번호
     , CHR(64 + Y.NO) 납입방법코드
     , DECODE(Y.NO, 1, 지로, 2, 자동이체, 3, 신용카드, 4, 핸드폰, 5, 인터넷)
FROM   월요금납부실적 x
    , (SELECT LEVEL NO FROM DUAL CONNECT BY LEVEL <= 5) y
WHERE x.납입월 = '200903'
AND   y.NO IN (
        DECODE(지로, 0, NULL, 1)
      , DECODE(자동이체, 0, NULL, 2)
      , DECODE(신용카드, 0, NULL, 3)
      , DECODE(핸드폰, 0, NULL, 4)
      , DECODE(인터넷, 0, NULL, 5)
      )
  
call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.01       0.02       0        1          0       0
Execute      1   0.45       0.66       2     1062       5079  150000
Fetch        0   0.00       0.00       0        0          0       0
------- ------ ------ ---------- ------- -------- ---------- -------
total        2   0.46       0.68       2     1063       5079  150000

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 54
```

데이터양이 많을 때는 위 쿼리도 소트 머지 조인 또는 해시 조인으로 유도하기 위한 약간의 튜닝이 필요하다.

```
INSERT INTO 납입방법별_월요금집계(납입월, 고객번호, 납입방법코드, 납입금액)
SELECT /*+ USE_MERGE(X Y) NO_EXPAND NO_MERGE(X) */ x.납입월, x.고객번호
     , CHR(64 + Y.NO) 납입방법코드
     , DECODE(Y.NO, 1, 지로, 2, 자동이체, 3, 신용카드, 4, 핸드폰, 5, 인터넷)
FROM  (SELECT 1 DUMMY, 납입월, 고객번호, 지로, 자동이체, 신용카드, 핸드폰, 인터넷
       FROM   월요금납부실적
       WHERE  납입월 = '200903') x
    , (SELECT 1 DUMMY, LEVEL NO FROM DUAL CONNECT BY LEVEL <= 5) y
WHERE x.DUMMY = y.DUMMY
AND   y.NO IN (
        DECODE(지로, 0, NULL, 1)
      , DECODE(자동이체, 0, NULL, 2)
      , DECODE(신용카드, 0, NULL, 3)
      , DECODE(핸드폰, 0, NULL, 4)
      , DECODE(인터넷, 0, NULL, 5)
) ;
```

다음 절에서 설명하는 Array Processing 기법을 활용하면, DBMS 외부에서 수행되는 JAVA 같은 프로그램에서도 네트워크 트래픽을 획기적으로 줄여 줘 굳이 One-SQL로 작성하지 않더라도 같은 수준의
성능개선 효과를 얻을 수 있다. 이 사실은, One-SQL로 로직을 통합했을 때 극적으로 성능 개선이 이루어지는 원리가 데이터베이스 Call 횟수를 줄이는 데에 있음을 반증한다.
<br/><br/><br/>
'납입방법별_월요금집계' 테이블을 읽어 '월요금납부실적'을 가공하고자 할 때는 어떻게 하면 될까? 혹시 아래와 같이 쿼리를 작성하는 개발자가 있을까 싶지만 뜻밖에 아주 많다.

```
INSERT INTO 월요금납부실적
(고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷)
SELECT K.고객번호, '200903' 납입월
     , A.납입금액 지로
     , B.납입금액 자동이체
     , C.납입금액 신용카드
     , D.납입금액 핸드폰
     , E.납입금액 인터넷
FROM   고객 K
    , (SELECT 고객번호, 납입금액 FROM 납입방법별_월요금집계
       WHERE  납입월 = '200903'
       AND    납입방법코드 = 'A') A
    , (SELECT 고객번호, 납입금액 FROM 납입방법별_월요금집계
       WHERE  납입월 = '200903'
       AND    납입방법코드 = 'B') B
    , (SELECT 고객번호, 납입금액 FROM 납입방법별_월요금집계
       WHERE  납입월 = '200903'
       AND    납입방법코드 = 'C') C
    , (SELECT 고객번호, 납입금액 FROM 납입방법별_월요금집계
       WHERE  납입월 = '200903'
       AND    납입방법코드 = 'D') D
    , (SELECT 고객번호, 납입금액 FROM 납입방법별_월요금집계
       WHERE  납입월 = '200903'
       AND    납입방법코드 = 'E') E
WHERE  A.고객번호(+) = K.고객번호
AND    B.고객번호(+) = K.고객번호
AND    C.고객번호(+) = K.고객번호
AND    D.고객번호(+) = K.고객번호
AND    E.고객번호(+) = K.고객번호 ;
```

SQL을 이처럼 작성해서는 고성능 DB 애플리케이션을 구축하기 어렵다. 데이터베이스 Call 횟수를 줄이려고 One-SQL로 구현하는 것도 중요하지만 어떻게 I/O 효율을 달성할지도 중요하다는 사실을 깨달아야 한다.
효율을 고려하지 않은 One-SQL은 누구나 작성할 수 있으며, I/O 효율의 핵심은 동일 레코드를 반복 액세스하지 않고 알마만큼 블록 액세스 양을 최소화할 수 있느냐에 달렸다.

I/O 효율을 달성하려면 쿼리를 아래와 같이 작성해야 한다. Cross-Table(또는 Pivot)을 만드는 아래 쿼리는 워낙 잘 알려진 패턴이므로 여기서 따로 설명하지는 않겠다.

```
INSERT INTO 월요금납부실적
(고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷)
SELECT 고객번호, 납입월
     , NVL(SUM(DECODE(납입방법코드, 'A', 납입금액)), 0) 지로
     , NVL(SUM(DECODE(납입방법코드, 'B', 납입금액)), 0) 자동이체
     , NVL(SUM(DECODE(납입방법코드, 'C', 납입금액)), 0) 신용카드
     , NVL(SUM(DECODE(납입방법코드, 'D', 납입금액)), 0) 핸드폰
     , NVL(SUM(DECODE(납입방법코드, 'E', 납입금액)), 0) 인터넷
FROM   납입방법별_월요금집계
WHERE  납입월 = '200903'
GROUP BY 고객번호, 납입월 ;
```

데이터베이스 Call이 많이 발생하도록 개발하는 또 다른 사례를 들어보자.
\[그림 5-4\](p.361)는 인터넷 쇼핑몰에서 카트에 담아둔 상품 중 일부를 선택한 후 곧바로 주문하거나 위시리스트(Wishlist)에 등록하는 화면이다.

5개 상품을 선택하고 '위시리스트' 버튼을 클릭했다고 가정하자.

위시리스트에 담는 메서드(method)를 아래처럼 구현했다면 5개 상품을 등록하려 할 때, 5번 Parse Call과 5번 Execute Call이 발생한다.

```
void insertWishList ( String p_custid, String p_goods_no ) {
  SQLStmt = "insert into wishlist "
          + "select custid, goods_no "
          + "from cart "
          + "where custid = ? "
          + "and  goods_no = ? " ;
  stmt = con.prepareStatement(SQLStmt);
  stmt.setString(1, p_custid);
  stmt.setString(2, p_goods_no);
  stmt.execute();
}
```

아래와 같이 구현했다면 Parse Call과 Execute Call이 각각 한 번씩만 발생한다.
단편적으로 말해, 24시간 내내 이 프로그램만 수행된다면 5배의 확장성을 갖는 것이며, AP 설계가 DBMS 성능을 좌우하는 중요한 요인임을 보여주는 사례라고 하겠다.

```
void insertWishList ( String p_custid, String[] p_goods_no ) {
  SQLStmt = "insert into wishlist "
          + "select custid, goods_no "
          + "from cart "
          + "where custid = ? "
          + "and goods_no in (?, ?, ?, ?, ? )" ;
  stmt = con.prepareStatement(SQLStmt);
  stmt.setString(1, p_custid);
  for (int i=0; i<5; i++){
    stmt.setString(i+2, p_goods_no[i]);
  }
  stmt.execute();
}
```