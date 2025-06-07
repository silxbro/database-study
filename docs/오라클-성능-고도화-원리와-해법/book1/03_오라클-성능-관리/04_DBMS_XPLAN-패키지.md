# 04. DBMS_XPLAN 패키지

오라클 9.2 버전에 소개된 dbms_xplan 패키지를 통해 plan_table에 저장된 실행계획을 좀 더 쉽게 출력해 볼 수 있게 되었다.
오라클은 9i부터 plan_table에 더 많은 정보들을 담기 시작했고, 이 패키지를 이용하지 않더라도 직접 쿼리해 보면 과거보다 더 많은 유용한 정보를 얻어낼 수 있다.

오라클 10g부터는 라이브러리 캐시에 캐싱돼 있는 SQL 커서에 대한 실행계획은 물론 Row Source별 수행통계까지 손쉽게 출력해 볼 수 있도록 기능이 확장되었다.
AWR에 수집된 과거 수행됐던 SQL에 대한 실행계획을 확인하는 것도 가능하다.
<br/>
<br/>
## (1) 예상 실행계획 출력
앞에서 @?/rdbms/admin/utlxpls 스크립트를 사용해 실행계획을 출력하는 방법을 이미 보았는데, 그 스크립트를 열어 보면 내부적으로 dbms_xplan 패키지를 호출하고 있는 것을 볼 수 있다.

```
select plan_table_output
from table(dbms_xplan.display('plan_table', null, 'serial'));
```

첫 번째 인자에는 실행계획이 저장된 plan table명을 입력하고, 두 번째 인자에는 statement_id를 입력하면 된다.
두 번째 옵션이 NULL일 때는 가장 마지막 explain plan 명령에 사용했던 쿼리의 실행계획을 보여준다.
병렬 쿼리에 대한 실행계획을 수집했다면 @?/rdbms/admin/utlxpls 스크립트를 수행함으로써 병렬 항목에 대한 정보까지 볼 수 있다.

그 외에도 dbms_xplan.display 함수를 직접 쿼리하면 세 번째 인자를 통해 다양한 포맷 옵션을 선택할 수 있다. 직접 해 보면 어떻게 다른지 쉽게 알 수 있으므로 일일이 설명하지는 않겠다.

```
explain plan set statment_id = 'SQL1' for
select *
from   emp e, dept d
where  d.deptno = e.deptno
and    e.sal >= 1000 ;

select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'BASIC'));

select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'TYPICAL'));

select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'SERIAL'));
```

basic 옵션을 사용하면 ID, Operation, Name 컬럼만 보이는데, format 인자를 아래처럼 구사하면 Rows, Bytes, Cost 컬럼까지 출력해 준다.

```
select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1'
                                      ,'BASIC ROWS BYTES COST'));

--------------------------------------------------------------------------------------
| Id  | Operation                        | Name        | Rows  | Bytes | Cost  (%CPU)|
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |             |    12 |   660 |     3    (0)|
|   1 |   TABLE ACCESS BY INDEX ROWID    | DEPT        |     1 |    18 |     1    (0)|
|   2 |     NESTED LOOPS                 |             |    12 |   660 |     3    (0)|
|   3 |       TABLE ACCESS BY INDEX ROWID| EMP         |    12 |   444 |     2    (0)|
|   4 |         INDEX RANGE SCAN         | EMP_SAL_IDX |    12 |       |     1    (0)|
|   5 |       INDEX RANGE SCAN           | DEPT_PK     |     1 |       |     0    (0)|
--------------------------------------------------------------------------------------
```

ROWS, BYTES, COST 이외에 추가로 사용할 수 있는 옵션으로는 다음과 같은 것들이 있다.

- PARTITION
- PARALLEL
- PREDICATE
- PROJECTION
- ALIAS
- REMOTE
- NOTE

위 모든 항목들을 다 출력해 보이려면 일일이 나열할 필요없이 all 옵션을 사용하면 된다.

```
select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'ALL'));
```

그리고 아래처럼 outline 옵션을 사용하면 같은 실행계획을 수립하는 데 필요한 힌트 목록을 보여준다.

```
select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'OUTLINE'));

Outline Data
------------

  /* +
       BEGIN_OUTLINE_DATA
       USE_NL(@"SEL$1" "D"@"SEL$1")
       LEADING(@"SEL$1" "E"@"SEL$1" "D"@"SEL$1")
       INDEX(@"SEL$1" "D"@"SEL$1" ("DEPT"."DEPTNO"))
       INDEX(@"SEL$1" "E"@"SEL$1" ("EMP"."SAL"))
       OUTLINE_LEAF(@"SEL$1")
       ALL_ROWS
       OPTIMIZER_FEATURES_ENABLE('10.2.0.1')
       IGNORE_OPTIM_EMBEDDED_HINTS
       END_OUTLINE_DATA
  */
```

아래는 all과 outline을 함께 사용한 것과 같다.

```
select * from table(dbms_xplan.display('PLAN_TABLE', 'SQL1', 'ADVANCED'));
```
<br/>

## (2) 캐싱된 커서의 실제 실행계획 출력
커서가 무엇인지는 이후 자세히 다루지만 여기서 간단히 정의해 보면, 하드 파싱 과정을 거쳐 메모리에 적재된 SQL과 Parse Tree, 실행계획, 그리고 그것을 실행하는데 필요한 정보를 담은 SQL Area를
말한다. 오라클은 라이브러리 캐시에 캐싱돼 있는 각 커서에 대한 수행통계를 볼 수 있도록 v$sql 뷰를 제공한다. 이 뷰의 활용방안에 대해서는 뒤쪽에서 다시 설명한다.
이것과 함께 sql_id 값으로 조인해서 사용할 수 있도록 제공되는 뷰가 몇 가지 더 있는데, 그 중 활용도가 가장 높은 것이 v$sql_plan과 v$sql_plan_statistics다.
그리고 이 두 뷰를 합쳐서 보여주는 것이 v$sql_plan_statistics_all이다.

v$sql_plan 뷰의 활용방법부터 살펴보자.

```
SQL> set serveroutput off
SQL> select *
  2  from   emp e, dept d
  3  where  d.deptno = e.deptno
  4  and    e.sal >= 1000 ;
...

12 개의 행이 선택되었습니다.
```

위와 같이 쿼리를 수행하고 나면 캐싱된 커서 정보를 v$sql에서 조회할 수 있고, v$sql_plan을 통해 실제 수행하면서 사용했던 실행계획까지 확인해 볼 수 있다.
v$sql_plan을 조회하려면 마지막 수행한 SQL의 sql_id와 child_number 값을 알아야 하는데, 아래 쿼리를 이용해 쉽게 찾을 수 있다.
위에서 쿼리 수행 전에 serveroutput을 off 시키는 것을 잊지 말자. 안 그러면 dbms_output.disable 프로시저를 호출하는 커서의 sql_id를 찾게 된다.

```
SQL> column prev_sql_id       new_value  sql_id
SQL> column prev_child_number new_value  child_no
SQL> select prev_sql_id, prev_child_number
  2  from   v$session
  3  where  sid=userenv('sid')
  4  and    username is not null
  5  and    prev_hash_value <> 0 ;

PREV_SQL_ID   PREV_CHILD_NUMBER
------------- -----------------
7f5y19ywtkwgt                 0

1 개의 행이 선택되었습니다.
```

v$sql_plan 뷰를 일반 plan table처럼 쿼리에서 원하는 방식으로 포맷팅할 수 있지만 dbms_xplan.display_cursor 함수를 이용하면 편하다.

```
select * from table(dbms_xplan.display_cursor('[ sql_id] ',[ child_no],'[ format] '));
```

이 함수는 라이브러리 캐시에 현재 캐싱돼 있는 SQL 커서의 실제 실행계획과, 실행계획을 만들면서 예상했던 Rows, Bytes, Cost, Time 정보를 보여준다.
sql_id와 child_number를 매번 찾는 게 귀찮다면(당연히 귀찮다.) 첫 번째, 두 번째 인자에 NULL 값을 넣어주면 된다.
맨 우측 format 인자에는 앞에서 dbms_xplan.display 함수에 사용했던 옵션들을 그대로 사용할 수 있고 출력 포맷도 같다.

```
SQL> select * from table(dbms_xplan.display_cursor
  2    ('&sql_id', &child_no, 'BASIC ROWS BYTES COST PREDICATE'));
---------------------------------------------------------------------------------------
| Id  | Operation                        | Name        | Rows   | Bytes | Cost  (%CPU)|
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |             |        |       |     3  (100)|
|   1 |   TABLE ACCESS BY INDEX ROWID    | DEPT        |      1 |    18 |     1    (0)|
|   2 |     NESTED LOOPS                 |             |     12 |   660 |     3    (0)|
|   3 |       TABLE ACCESS BY INDEX ROWID| EMP         |     12 |   444 |     2    (0)|
|*  4 |         INDEX RANGE SCAN         | EMP_SAL_IDX |     12 |       |     1    (0)|
|*  5 |       INDEX RANGE SCAN           | DEPT_PK     |      1 |       |     0    (0)|
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   4 - access("E"."SAL">=1000)
   5 - access("D"."DEPTNO"="E"."DEPTNO")
```

참고로, dbms_xplan.display_awr 함수를 이용하면 AWR에 수집된 과거 수행했던 SQL에 대해서도 같은 분석작업을 진행할 수 있다.
<br/>
<br/>
## (3) 캐싱된 커서의 Row Source별 수행 통계 출력
SQL문에 gather_plan_statistics 힌트를 사용하거나, 시스템 또는 세션 레벨에서 statistics_level 파라미터를 all로 설정하면(개발 DB에서는 유용하지만 운영 DB의 시스템 레벨에서 all로
설정하는 것은 삼가야 한다.), 오라클은 실제 SQL을 수행하는 동안의 실행계획 각 오퍼레이션 단계(Row Source)별로 수행 통계를 수집한다.
참고로, '_rowsource_execution_statistics' 파라미터를 true로 설정하거나, SQL 트레이스를 걸어도 Row Source별 수행 통계가 수집된다.

조회할 때는 v$sql_plan_statistics 또는 v$sql_plan_statistics_all 뷰를 이용하면 된다. 예제를 보면서 어떻게 활용할 수 있는지 확인해 보자.

```
SQL> set serveroutput off
SQL> select /*+ gather_plan_statistics */ *
  2  from   emp e, dept d
  3  where  d.deptno = e.deptno
  4  and    e.sal >= 1000 ;
...

12 개의 행이 선택되었습니다.
```

이제 Row Source 별 수행 통계가 수집되었다. v$sql_plan_statistics를 조회하려면 마지막 수행한 SQL의 sql_id와 child_number 값을 알아야 하며, 방법은 앞에서 이미 살펴보았다.
아래처럼 쿼리하면 SQL 트레이스를 통해 볼 수 있는 Row Source Operation과 같은 정보가 출력된다.
select-list가 길고 복잡해 다 보여주지 못하는 것에 대해 미안하게 생각하지만 case문을 적절히 사용하면 아래처럼 출력되도록 쉽게 구현할 수 있다.

```
SQL> select last_output_rows "Rows", ......
  2  from   v$sql_plan_statistics_all
  3  where  sql_id = '&sql_id'
  4  and    child_number = &child_no
  5  order by id ;

Rows Row Source Operation
---- ----------------------------------------------------------------
  12 TABLE ACCESS BY INDEX ROWID DEPT (cr=9 pr=0 time=174 us)
  25   NESTED LOOPS (cr=7 pr=0 time=1835 us)
  12     TABLE ACCESS BY INDEX ROWID EMP (cr=4 pr=0 time=306 us)
  12       INDEX RANGE SCAN EMP_SAL_IDX (cr=2 pr=0 time=136 us)
  12     INDEX RANGE SCAN DEPT_PK (cr=3 pr=0 time=224 us)

5 개의 행이 선택되었습니다.
```

SQL 트레이스를 걸어도 같은 결과를 얻을 수 있으므로 TKProf를 이용하기 전에 먼저 Row Source Operation을 확인하고, Call 통계 정보는 필요할 떄만 확인하는 식으로 이용할 수 있다.

v$sql_plan_statistics 뷰에는 모든 통계항목에 대해 마지막 수행 통계치와 누적 통계치를 조회할 수 있도록 컬럼이 두 개씩 제공된다. 아래와 같은 식이다.

- output_rows, last_output_rows
- cr_buffer_gets, last_cr_buffer_gets
- disk_reads, last_disk_reads

따라서 마지막 수행통계나 누적 수행통계를 자유롭게 뽑아볼 수 있고, 위에서 예시한 과정을 스크립트로 만들어 두면 편리하게 사용할 수 있다. 아래는 스크립트 활용 예시로서, 누적 값을 출력해 보이고 있다.
왼쪽 Execs는 수행횟수다.

```
SQL> @rsop

(1) last  (2) Sum --> 2

Execs  Rows Row Source Operation
----- ----- -----------------------------------------------------------------------------------
   23   276 TABLE ACCESS BY INDEX ROWID DEPT (cr=207 pr=4 time=49885 us)
   23   575   NESTED LOOPS (cr=161 pr=3 time=0912860 us)
   23   276     TABLE ACCESS BY INDEX ROWID EMP (cr=92 pr=2 time=30595 us)
   23   276       INDEX RANGE SCAN EMP_SAL_IDX (cr=46 pr=1 time=14853 us)
   23   276     INDEX RANGE SCAN DEPT_PK (cr=69 pr=1 time=17436 us)

5 개의 행이 선택되었습니다.
```

dbms_xplan.display_cursor 함수는 우리를 대신해 위 뷰를 읽어 깔끔하게 포맷팅해주는 기능을 제공한다.
아래처럼 dbms_xplan.display 함수에는 없던 iostats, memstats, allstats 옵션을 사용하면 실제 수행 시 Row Source별 수행통계를 보여준다.
여기서도 가장 최근에 수행한 SQL을 찾을 때는 첫 번째, 두 번째 인자에 NULL 값을 입력하면 된다.

```
select * from table(dbms_xplan.display_cursor('7f5y19ywtkwgt', 0, 'IOSTATS'));
select * from table(dbms_xplan.display_cursor('7f5y19ywtkwgt', 0, 'MEMSTATS'));
select * from table(dbms_xplan.display_cursor('7f5y19ywtkwgt', 0, 'IOSTATS MEMSTATS'));
select * from table(dbms_xplan.display_cursor('7f5y19ywtkwgt', 0, 'ALLSTATS'));
```

아래는 출력된 결과 샘플인데, 여기서 E-Rows는 SQL을 수행하기 전 옵티마이저가 각 Row Source별로 예상했던 로우 수로서 v$sql_plan에서 읽어온 값이다.
A-Rows는 실제 수행 시 읽었던 로우 수로서, v$sql_plan_statistics에서 읽어 온 항목이다.
이처럼 옵티마이저의 예상 로우 수와 수행 시 실제 로우 수를 비교해 보여주므로 옵티마이저의 행동을 관찰할 때 유용하게 사용할 수 있다.

```
-----------------------------------------------------------------------------------
| Id  | Operation  | Name       | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-----------------------------------------------------------------------------------
|   1 |  TABLE ACC | DEPT       |      1 |      1 |     12 |00:00:00.01 |       9 |
|   2 |   NESTED L |            |      1 |     12 |     25 |00:00:00.01 |       7 |
|   3 |    TABLE A | EMP        |      1 |     12 |     12 |00:00:00.01 |       4 |
|*  4 |     INDEX  | EMP_SAL_ID |      1 |     12 |     12 |00:00:00.01 |       2 |
|*  5 |    INDEX R | DEPT_PK    |     12 |      1 |     12 |00:00:00.01 |       3 |
-----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   4 - access("E"."SAL">=1000)
   5 - access("D"."DEPTNO"="E"."DEPTNO")
```

기본적으로 누적 값(last_ 접두사가 붙지 않은 컬럼)을 보여주며, 아래처럼 format 옵션에 last를 추가해주면 마지막 수행했을 때의 일량(last_ 접두사가 붙은 컬럼)을 보여준다.

```
select * from table(dbms_xplan.display_cursor
                        ('7f5y19ywtkwgt', 0, 'ALLSTATS LAST'));
```