# 07. OR-Expansion

## (1) OR-Expansion 기본
아래 쿼리가 그대로 수행된다면 OR 조건이므로 Full Table Scan으로 처리될 것이다. 아니면 Index Combine이 작동할 수도 있다.
```
select * from emp
where  job = 'CLERK' or deptno = 20
```
만약 job과 deptno에 각각 생성된 인덱스를 사용하고 싶다면 아래와 같이 union all 형태로 바꿔주면 된다.
```
select * from emp
where  job = 'CLERK'
union all
select * from emp
where  deptno = 20
and    LNNVL(job='CLERK')
```
사용자가 쿼리를 직접 바꿔주지 않아도 옵티마이저가 그런 작업을 대신해 주는 경우가 있는데, 이를 'OR-Expansion'이라고 한다.
아래는 OR-Expansion 쿼리 변환이 일어났을 때의 실행계획과 Predicate 정보다.
```
-------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes |
-------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     7 |   224 |
|   1 |   CONCATENATION                |                |       |       |
|   2 |     TABLE ACCESS BY INDEX ROWID| EMP            |     3 |    96 |
|*  3 |       INDEX RANGE SCAN         | EMP_JOB_IDX    |     3 |       |
|*  4 |     TABLE ACCESS BY INDEX ROWID| EMP            |     4 |   128 |
|*  5 |       INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
-------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  3 - access("JOB"='CLERK')
  4 - filter(LNNVL("JOB"='CLERK')
  5 - access*("DEPTNO"-20)
```
job과 deptno 컬럼을 선두로 갖는 두 인덱스가 각각 사용되었고, union all 위쪽 브랜치는 job = 'CLERK'인 집합을 읽고 아래쪽 브랜치는 deptno = 20인 집합만을 읽는다.

분기된 두 쿼리가 각각 다른 인덱스를 사용하긴 하지만, emp 테이블 액세스가 두 번 일어난다.
따라서 중복 액세스되는 영역 (deptno=20이면서 job='CLERK')의 데이터 비중이 작을수록 효과적이고, 그 반대의 경우라면 오히려 쿼리 수행 비용이 증가한다.
OR-Expansion 쿼리 변환이 처음부터 비용기반으로 작동한 것도 이 때문이다.

중복 액세스되더라도 결과집합에는 중복이 없게 하려고 union all 아래쪽에 오라클이 내부적으로 LNNVL 함수를 사용한 것을 확인하기 바란다.
job <> 'CLERK'이거나 job is null인 집합만을 읽으려는 것이며, 이 함수는 조건식이 false이거나 알 수 없는(Unknown) 값일 때 true를 리턴한다.

OR-Expansion을 제어하기 위해 사용하는 힌트로는 use_concat과 no_expand 두 가지가 있다.
use_concat은 OR-Expansion을 유도하고자 할 때 사용하고, no_expand는 이 기능을 방지하고자 할 때 사용한다.

```
select /*+ USE_CONCAT */ * from emp
where  job = 'CLERK' or deptno = 20;

select /*+ NO_EXPAND */ * from emp
where  job = 'CLERK' or deptno = 20;
```
OR-Expansion 기능을 아예 작동하지 못하도록 막으려면 아래와 같이 _no_or_expansion 파라미터를 true로 설정하면 되고, 그럴 경우 use_concat 힌트를 사용하더라도 OR-Expansion이 일어나지 않는다.
기본 값은 false다.
```
alter session set "_no_or_expansion" = true;
```

<br/>

## (2) OR-Expansion 브랜치별 조인 순서 최적화
OR-Expansion에 의해 분기된 브랜치마다 각각 다른 조인 순서를 가질 수 있음은 매우 중요한 사실이다. 아래 쿼리를 보자.
```
select /*+ NO_EXPAND */ * from emp e, dept d
where  d.deptno = e.deptno
and    e.sal >= 2000
and    (e.job = 'SALESMAN' or d.loc = 'CHICAGO');

-------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes |
-------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     4 |   200 |
|   1 |   NESTED LOOPS                 |                |     4 |   200 |
|   2 |     TABLE ACCESS BY INDEX ROWID| EMP            |    11 |   352 |
|*  3 |       INDEX RANGE SCAN         | EMP_SAL_IDX    |    11 |       |
|*  4 |     TABLE ACCESS BY INDEX ROWID| DEPT           |     1 |    18 |
|*  5 |       INDEX UNIQUE SCAN        | DEPT_PK        |     1 |       |
-------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  3 - access("SAL">=2000)
  4 - filter("E"."JOB"='SALESMAN' OR "D"."LOC"='CHICAGO')
  5 - access("D"."DEPTNO"="E"."DEPTNO")
```
no_expand 힌트를 사용해 OR-Expansion하지 못하도록 막았으므로 sal >= 2000 조건으로 emp 테이블을 먼저 읽어 조인한 후에 dept 테이블을 액세스하는 단계에서
[e.job = 'SALESMAN' or d.loc = 'CHICAGO'] 조건 필터링이 이루어지고 있다.
만약 드라이비이 조건(sal >= 2000)의 변별력이 나빠 조인 액세스 건수가 많고 오히려 최종 필터되는 OR로 묶인 두 조건의 변별력이 좋다면, 위 실행계획은 매우 비효율적이다.

emp 테이블 job과 deptno 컬럼, dept 테이블 loc 컬럼에 대한 각각 인덱스를 만들고 아래와 같이 OR-Expansion이 일어나도록 유도해 보자.
```
select /*+ USE_CONCAT */ * from emp e, dept d
where  d.deptno = e.deptno
and    e.sal >= 2000
and    (e.job = 'SALESMAN' or d.loc = 'CHICAGO');

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     6 |   300 |
|   1 |   CONCATENATION                    |                |       |       |
|*  2 |     TABLE ACCESS BY INDEX ROWID    | EMP            |     4 |   128 |
|   3 |       NESTED LOOPS                 |                |     4 |   200 |
|   4 |         TABLE ACCESS BY INDEX ROWID| DEPT           |     1 |    18 |
|*  5 |           INDEX RANGE SCAN         | DEPT_LOC_IDX   |     1 |       |
|*  6 |         INDEX RANGE SCAN           | EMP_DEPTNO_IDX |     5 |       |
|   7 |     NESTED LOOPS                   |                |     2 |   100 |
|*  8 |       TABLE ACCESS BY INDEX ROWID  | EMP            |     2 |    64 |
|*  9 |         INDEX RANGE SCAN           | EMP_JOB_IDX    |     3 |       |
|* 10 |       TABLE ACCESS BY INDEX ROWID  | DEPT           |     1 |    18 |
|* 11 |         INDEX UNIQUE SCAN          | DEPT_PK        |     1 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  2 - filter("E"."SAL">=2000)
  5 - access("D"."LOC"='CHICAGO')
  6 - access("D"."DEPTNO"="E"."DEPTNO")
  8 - filter("E"."SAL">=2000)
  9 - access("E"."JOB"='SALESMAN')
 10 - filter(LNNVL("D"."LOC"='CHICAGO'))
 11 - access("D"."DEPTNO"="E"."DEPTNO")
```
loc = 'CHICAGO'인 집합을 생성하는 쿼리(위쪽 브랜치)와 job = 'SALESMAN'인 집합을 생성하는 쿼리(아래쪽 브랜치)가 각기 다른 인덱스와 조인 순서를 가지고 실행되었다.
위쪽 브랜치는 dept가 먼저 드라이빙되고 아래쪽 브랜치는 emp가 먼저 드라이빙된 것을 확인하기 바란다. 여기서도 두 쿼리의 교집합이 두 번 출력되는 것을 방지하려고 LNNVL 함수가 사용되었다.
(실제 위와 같은 실행계획이 나오지 않으면 힌트를 써서 유도할 수도 있다.)

<br/>

## (3) 같은 컬럼에 대한 OR-Expansion
아래 두 쿼리는 내용적으로 100% 같은 쿼리다.
```
< 쿼리1 >
select * from emp
where  (deptno = 10 or deptno = 30)
and    ename = :ename;

< 쿼리2 >
select * from emp
where  deptno in (10, 30)
and    ename = :ename;
```
따라서 이들 쿼리도 아래와 같이 OR-Expansion 처리가 가능하다.
```
select * from emp
where  deptno = 30
and    ename = :ename
union all
select * from emp
where  deptno = 10
and    ename = :ename

-------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes |
-------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     2 |    74 |
|   1 |   CONCATENATION                |                |       |       |
|*  2 |     TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    37 |
|*  3 |       INDEX RAGNE SCAN         | EMP_DEPTNO_IDX |     5 |       |
|*  4 |     TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    37 |
|*  5 |       INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
-------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  2 - filter("ENAME"=:ENAME)
  3 - access("DEPTNO"=30)
  4 - filter("ENAME"=:ENAME)
  5 - access("DEPTNO"=10)
```
실제로 9i까지는 위와 같은 형태 즉, 같은 컬럼에 대한 OR 조건이나 IN-List도 OR-Expansion이 작동할 수 있었지만 10g부터는 기본적으로 아래와 같이 IN-List Iterator 방식으로만 처리된다.
```
-------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes |
-------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     1 |    32 |
|   1 |   INLIST ITERATOR              |                |       |       |
|*  2 |     TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    32 |
|*  3 |       INDEX RAGNE SCAN         | EMP_DEPTNO_IDX |     9 |       |
-------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  2 - filter("ENAME"=:ENAME)
  3 - access("DEPTNO"=10 OR "DEPTNO"=30)
```
만약 억지로 OR-Expansion으로 유도하려면 use_concat 힌트에 인자를 제공하면 되지만, IN-List Iterator에 비해 나은 점이 없으므로 굳이 그렇게 할 이유는 없다.

주의할 사항이 한 가지 있다. 9i까지는 OR 조건이나 IN-List를 힌트를 이용해 OR-Expansion으로 유도하면 뒤쪽에 놓인 값(위 쿼리에서는 30)이 항상 먼저 출력됐었다.
하지만 10g CPU 비용 모델에서는 위와 같이 OR-Expansion으로 유도했을 때 통계적으로 카디널리티가 작은 값이 먼저 출력된다.
9i 이전처럼 뒤쪽에 놓인 값이 먼저 출력되도록 하려면 12절 (3)항 '조건절 비교 순서'에서 설명하는 ordered_predicates 힌트를 사용하거나 IO 비용 모델로 바꿔주어야 한다.

10g 이후 버전이더라도 비교 연산자가 '=' 조건이 아닐 때는 일반적인 use_concat 힌트만으로도 같은 컬럼에 대한 OR-Expansion이 잘 작동한다.
```
select /*+ use_concat */ * from emp
where  deptno = 10 or dept >= 30

-------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes |
-------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     7 |   259 |
|   1 |   CONCATENATION                |                |       |       |
|   2 |     TABLE ACCESS BY INDEX ROWID| EMP            |     3 |   111 |
|*  3 |       INDEX RAGNE SCAN         | EMP_DEPTNO_IDX |     3 |       |
|   4 |     TABLE ACCESS BY INDEX ROWID| EMP            |     4 |   148 |
|*  5 |       INDEX RAGNE SCAN         | EMP_DEPTNO_IDX |     6 |       |
-------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  3 - access("DEPTNO"=10)
  5 - access("DEPTNO">=30)
      filter(LNNVL("DEPTNO"=10))
```

<br/>

## (4) nvl/decode 조건식에 대한 OR-Expansion
사용자가 선택적으로 입력하는 조건절에 대해 nvl 또는 decode 함수를 이용할 수 있다. 아래 쿼리는 deptno 검색 조건을 사용자가 선택적으로 입력할 수 있는 경우에 대비하기 위한 것이다.
```
select * from emp
where  deptno = nvl(:deptno, deptno)
and    ename like :ename || '%'
```
오라클 9i부터 쿼리를 위와 같이 작성하면, 아래와 같은 형태로 OR-Expansion 쿼리 변환이 일어난다.
```
select * from emp
where  :deptno is null
and    deptno is not null
and    ename like :ename || '%'
union all
select * from emp
where  :deptno is not null
and    deptno = :deptno
and    ename like :ename || '%'
```
:deptno 변수 값의 null 여부에 따라 위 또는 아래 브랜치만 수행하는 것이다. 아래와 같이 decode 함수를 사용하더라도 같은 처리가 일어난다.
```
select * from emp
where  deptno = decode(:deptno, null, deptno, :deptno)
and    ename like :ename || '%'
```
옵티마이저에 의해 자동으로 OR-Expansion이 일어났을 때 실행계획은 아래와 같다.
```
---------------------------------------------------------------------------
| Id  | Operation                        | Name           | Rows  | Bytes |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                |     3 |   111 |
|   1 |   CONCATENATION                  |                |       |       |
|   2 |     FILTER                       |                |       |       |
|   3 |       TABLE ACCESS BY INDEX ROWID| EMP            |     2 |    74 |
|   4 |         INDEX RAGNE SCAN         | EMP_ENAME_IDX  |     2 |       |
|   5 |     FILTER                       |                |       |       |
|   6 |       TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    37 |
|   7 |         INDEX RAGNE SCAN         | EMP_DEPTNO_IDX |     5 |       |
---------------------------------------------------------------------------
```
무엇보다 중요한 것은  :deptno 변수 값 입력 여부에 따라 다른 인덱스를 사용한다는 사실이다.
실행계획을 보면 :deptno 변수에 null 값을 입력했을 때 사용되는 위쪽 브랜치는 emp_ename_idx 인덱스를 사용했고, null 값이 아닌 값을 입력했을 때 사용되는 아래쪽 브랜치는 emp_deptno_idx
인덱스를 사용했다.

이 유용한 기능을 제어하는 파라미터는 _or_expand_nvl_predicate이다.
오래전부터 SQL 튜너들이 union all을 이용해 위와 같은 튜닝 기법을 많이 사용해 왔는데, 이제는 옵티마이저가 스스로 그런 처리를 함으로써 많이 편리해진 것은 사실이다.
하지만 nvl 또는 decode를 여러 컬럼에 대해 사용했을 때는 그 중 변별력이 가장 좋은 컬럼 기준으로 한 번만 분기가 일어난다.
옵션 조건이 복잡할 때는 이 방식에만 의존하기 어려운 이유가 여기에 있고, 그럴 때는 여전히 수동으로 union all 분기를 해 줘야만 한다.