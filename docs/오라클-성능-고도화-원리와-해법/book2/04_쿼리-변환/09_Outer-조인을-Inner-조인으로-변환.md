# 09. Outer 조인을 Inner 조인으로 변환

Outer 조인문을 작성하면서 일부 조건절에 Outer 기호(+)를 빠뜨리면 Inner 조인할 때와 같은 결과가 나온다. 이럴 때 옵티마이저는 Outer 조인을 Inner 조인문으로 바꾸는 쿼리 변환을 시행한다.
아래 Predicate 정보를 확인하기 바란다.
```
select *
from   emp e, dept d
where  d.deptno(+) = e.deptno
and    d.loc = 'DALLAS'
and    e.sal >= 1000

---------------------------------------------------------------------------
| Id  | Operation                        | Name           | Rows  | Bytes |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                |     5 |   250 |
|*  1 |   TABLE ACCESS BY INDEX ROWID    | EMP            |     5 |   160 |
|   2 |     NESTED LOOPS                 |                |     5 |   250 |
|   3 |       TABLE ACCESS BY INDEX ROWID| DEPT           |     1 |    18 |
|*  4 |         INDEX RANGE SCAN         | DEPT_LOC_IDX   |     1 |       |
|*  5 |     INDEX RANGE SCAN             | EMP_DEPTNO_IDX |     5 |       |
---------------------------------------------------------------------------

Predicate Information (identified by operation id) :
----------------------------------------------------
  1 - filter("E"."SAL">=1000)
  4 - access("D"."LOC"='DALLAS')
  5 - access("D"."DEPTNO"="E"."DEPTNO")
```
옵티마이저가 굳이 이런 쿼리 변환을 시행하는 이유는 조인 순서를 자유롭게 결정하기 위해서다. Outer NL 조인, Outer 소트 머지 조인 시 드라이빙 테이블은 항상 Outer 기호가 붙지 않은 쪽으로 고정된다.
Outer 해시 조인의 경우, 10g부터 자유롭게 조인 순서가 바뀌도록 개선되었지만 9i까지는 해시 조인도 순서가 고정적이었따. 이처럼 조인 순서를 자유롭게 결정하지 못하는 것이 쿼리 최적화에 큰 걸림돌일 수 있다.

만약 위 쿼리에서 sal >= 1000 조건에 부합하는 사원 레코드가 매우 많고, loc = 'DALLAS' 조건에 부합하는 부서에 속한 사원이 매우 적다면 dept 테이블을 먼저 드라이빙하는 것이 유리하다.
그럼에도 Outer 조인 때문에 항상 emp 테이블을 드라이빙해야 한다면 불리한 조건에서 최적화하는 것이 된다. SQL을 작성할 때 불필요한 Outer 조인을 삼가야 하는 이유가 여기에 있다.

Outer 조인을 써야 하는 상황이라면 Outer 기호를 정확히 구사해야 올바른 결과집합을 얻을 수 있음에 유념하자. ANSI Outer 조인문일 때는 Outer 기호 대신 조건절 위치에 신경을 써야 한다.
Outer 조인에서 Inner쪽 테이블에 대한 필터 조건을 아래처럼 where절에 기술한다면 Inner 조인할 때와 같은 결과집합을 얻게 된다. 따라서 옵티마이저가 Outer 조인을 아예 Inner 조인으로 변환해 버린다.
```
select *
from   dept d left outer join emp e on d.deptno = e.deptno
where  e.sal > 1000
```
제대로 된 Outer 조인 결과집합을 얻으려면 sal > 1000 조건을 아래와 같이 on절에 기술해 주어야 한다.
```
select *
from   dept d left outer join emp e on d.deptno = e.deptno and e.sal > 1000
```
ANSI Outer 조인문에서 where절에 기술한 Inner쪽 필터 조건이 의미 있게 사용되는 경우는 아래처럼 is null 조건을 체크하는 경우뿐이며, 조인에 실패하는 레코드를 찾고자 할 때 흔히 사용되는 SQL이다.
```
select (
from   dept d left outer join emp e on d.deptno = e.deptno
where  e.empno is null
```
Outer 쪽 필터조건은 on절에 기술하든 where절에 기술하든 결과집합이나 성능에 하등 차이가 없다.