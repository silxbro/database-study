# 02. AutoTrace

#### ✅ AutoTrace 결과에는 SQL을 튜닝하는데 유용한 정보들을 많이 포함하고 있어 가장 즐겨 사용되는 도구 중 하나다

```
SQL> set autotrace on
SQL> select * from scott.emp where empno = 7900;

 EMPNO ENAME   JOB         MGR HIREDATE             SAL   COMM  DEPTNO
------ ------- ------- ------- ----------------- ------ ------ -------
  7900 JAMES   CLERK      7698 81/12/03 00:00:00    950             30

1개의 행이 선택되었습니다.


Execution Plan
------------------------------------------------------------
Plan hash value: 4024650034

-------------------------------------------------------------------------------
| Id  | Operation                      | Name   | Rows  | Bytes  | Cost (%CPU)|
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |        |     1 |     32 |     1   (0)|
|   1 |   TABLE ACCESS BY INDEX ROWID  | EMP    |     1 |     32 |     1   (0)|
|*  2 |     INDEX UNIQUE SCAN          | EMP_PK |     1 |        |     0   (0)|
-------------------------------------------------------------------------------

Predicate Information (identified by operation id):
-------------------------------------------------------------------------------
  2 - access("EMPNO"=7900)

Statistics
-------------------------------------------------------------------------------
          1  recursive calls
          0  db block gets
          2  consistent gets
          0  physical reads
          0  redo size
        743  bytes sent via SQL*Net to client
        374  bytes received via SQL*Net from client
          1  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```

아래와 같은 옵션 조합에 따라 필요한 부분만 출력해 볼 수 있다.

#### (1) set autotrace on
SQL을 실제 수행하고 그 결과와 함께 실행계획 및 실행통계를 출력한다.
#### (2) set autotrace on explain
SQL을 실제 수행하고 그 결과와 함께 실행계획을 출력한다.
#### (3) set autotrace on statistics
SQL을 실제 수행하고 그 결과와 함께 실행통계를 출력한다.
#### (4) set autotrace traceonly
SQL을 실제 수행하지만 그 결과는 출력하지 않고 실행계획과 통계만 출력한다.
#### (5) set autotrace traceonly explain
SQL을 실제 수행하지 않고 실행계획만을 출력한다.
#### (6) set autotrace traceonly statistics
SQL을 실제 수행하지만 그 결과는 출력하지 않고 실행통계만을 출력한다.

- (1) ~ (3)은 수행결과를 출력해야 하므로 쿼리를 실제 수행한다.
- (4), (6)은 실행통계를 보여줘야 하므로 쿼리를 실제 수행한다.
- (5)는 실행계획만 출력하면 되므로 쿼리를 실제 수행하지 않는다. SQL*Plus에서 실행계획을 가장 쉽고 빠르게 확인해 볼 수 있는 방법이다.

#### ✅ 실행통계 확인을 위한 권한부여
AutoTrace 기능을 실행계획 확인 용도로만 사용한다면 plan_table만 생성돼 있으면 된다.
하지만 실행통계까지 함께 확인하려면 v_$sesstat, v_$statname, v_$mystat 뷰에 대한 읽기 권한이 필요하다.

따라서 dba, select_catalog_role 등의 롤(Role)을 부여받지 않은 일반사용자들에게는 별도의 권한 설정이 필요하다.
이들 뷰에 대한 읽기 권한을 일일이 부여해도 되지만 plustrace 롤(Role)을 생성하고 필요한 사용자들에게 이 롤을 부여하는 것이 관리상 편리하다. 아래처럼 하면 된다.

```
SQL> @?/sqlplus/admin/plustrace.sql
SQL> grant plutrace to scott;
```
#### ✅ 실행통계 활성화 시 세션 추가
참고로, statistics 모드로 AutoTrace를 활성화시키면 새로운 세션이 하나 열리면서 현재 세션의 통계정보를 대신 쿼리해서 보여준다.
쿼리를 실행하기 전 현재 세션의 수행 통계 정보를 어딘가에 저장했다가 쿼리 실행 후 수행 통계와의 델타(Delta) 값을 계산해 보여주는 방식이다.
만약 같은 세션에서 수행한다면 세션통계를 쿼리할 때의 수행통계까지 뒤섞이기 때문에 별도의 세션을 사용하는 것이다.

```
SQL> @session

USERNAME OSUser@Terminal  PROGRAM      STATUS   LoginTIM SID SERIAL# OSPID
-------- ---------------- ------------ -------- -------- --- ------- -----
ORAKING  oraking@SHCHO    sqlplus.exe  ACTIVE   08:49:37  13    3309 11546
```

현재 위처럼 한 개 세션이 존재하는 상황에서 statistics 옵션을 활성화해 보자.

```
SQL> set autotrace on statistics

SQL> @session

USERNAME OSUser@Terminal  PROGRAM      STATUS   LoginTIM SID SERIAL# OSPID
-------- ---------------- ------------ -------- -------- --- ------- -----
ORAKING  oraking@SHCHO    sqlplus.exe  ACTIVE   08:49:37  13    3309 11546
ORAKING  oraking@SHCHO    sqlplus.exe  INACTIVE 09:32:30  22    1460 11592
```

statistics 모드로 AutoTrace를 활성화하니까 새로운 세션이 하나 추가되었음을 확인할 수 있다. 이번에는 explain 모드로 바꿔 보자.

```
SQL> set autotrace on explain

SQL> @session

USERNAME OSUser@Terminal  PROGRAM      STATUS   LoginTIM SID SERIAL# OSPID
-------- ---------------- ------------ -------- -------- --- ------- -----
ORAKING  oraking@SHCHO    sqlplus.exe  ACTIVE   08:49:37  13    3309 11546
```

아까 새롭게 열렸던 세션이 사라진 것을 확인하기 바란다. statistics 옵션을 활성화했을 때 내부적으로 수행되는 SQL문은 아래와 같다.

```
SELECT PT.VALUE FROM SYS.V_$SESSTAT PT
WHERE PT.SID = :1
-- AND PT.STATISTIC# IN (7,40,41,42,115,236,237,238,242,243)  -- 9i
AND PT.STATISTIC# IN (7,47,50,54,134,335,336,337,341,342)     -- 10g
ORDER BY PT.STATISTIC#
```