# 03. 뷰 Merging

## (1) 뷰 Merging 이란?
```
< 쿼리1 >
select *
from   (select * from emp   where job = 'SALESMAN') a
     , (select * from dept  where loc = 'CHICAGO') b
where  a.deptno = b.deptno
```
위 <쿼리1>과 같이 습관적으로 인라인 뷰를 많이 사용하는 분들을 종종 볼 수 있는데, 그렇게 작성하는 것이 더 읽기 편하기 때문이다. 서브쿼리도 마찬가지다.
서브쿼리로 표현하면 아무래도 조인문보다 더 직관적으로 읽힌다.

그런데 사람의 눈으로 볼 때는 쿼리를 블록화하는 것이 더 읽기 편할지 모르지만 최적화를 수행하는 옵티마이저의 시각에서는 더 불편하다.
그런 탓에 옵티마이저는 가급적 <쿼리2>처럼 쿼리 블록을 풀어내려는 습성을 갖는다.(옵티마이저 개발팀이 그렇게 만들었다.)
```
< 쿼리 2 >
select *
from   emp a, dept b
where  a.deptno = b.deptno
and    a.job = 'SALESMAN'
and    b.loc = 'CHICAGO'
```
따라서 위에서 본 <쿼리1>의 뷰 쿼리 블록은 액세스 쿼리 블록(뷰를 참조하는 쿼리 블록)과의 머지(merge) 과정을 거쳐 <쿼리2>와 같은 형태로 변환되는데, 이를 '뷰 Merging'이라고 한다.
이처럼 뷰 Merging을 거친 쿼리라야 옵티마이저가 더 다양한 액세스 경로(Access Path)를 조사 대상으로 삼을 수 있게 된다. 이 기능을 제어하는 힌트로는 merge와 no_merge가 있다.

<쿼리1>처럼 뷰를 사용하면 쿼리 성능이 더 느려진다는 속설이 있는데, 과연 그럴까? <쿼리1>과 <쿼리2>는 뷰 Merging을 통해 100% 같은 실행계획으로써 수행되며, 쿼리 성능에 하등 차이가 없다.

뷰 Merging과 다음 절에서 설명하는 조건절(Predicate) Pushing이 불가능한 형태의 뷰를 사용했을 때 성능이 느려지기도 하므로 뷰 때문에 쿼리 성능이 느려진다는 말도 전혀 틀린 말은 아니다.
하지만 막연한 경험치를 가지고 뷰 사용을 꺼리는 것보다는 옵티마이저의 쿼리 변환 원리를 정확히 이해함으로써 적절한 때에 뷰를 사용할 수 있어야 한다.

<br/>

## (2) 단순 뷰(Simple View) Merging
조건절과 조인문만을 포함하는 단순 뷰(Simple View)는 no_merge 힌트를 사용하지 않는 한 언제든 Merging이 일어난다.
반면, group by 절이나 distinct 연산을 포함하는 복합 뷰(Complex View)는 파라미터 설정(_complex_view_merging = true / 9i부터는 기본 값이 true) 또는 힌트 사용에 의해서만
뷰 Merging이 가능하다. 또한 집합(set) 연산자, connect by, rownum 등을 포함하는 복합 뷰(Non-mergable Views)는 아예 뷰 Merging이 불가능하다.

아래는 조건절 하나만을 가진 단순 뷰(Simple View)의 예시다.
```
create or replace view emp_salesman
as
select empno, ename, job, mgr, hiredate, sal, comm, deptno
from   emp
where  job = 'SALESMAN' ;
```
위 단순 뷰와 조인하는 간단한 조인문을 작성해 보자.
```
select e.empno, e.ename, e.job, e.mgr, e.sal, d.dname
from   emp_salesman e, dept d
where  e.deptno = d.deptno
and    e.sal >= 1500 ;
```
위 쿼리를 뷰 Merging 하지 않고 그대로 최적화한다면 아래와 같은 실행계획이 만들어진다. (단순 뷰는 항상 Merging 되므로 no-merge 힌트로 유도하였다.)
```
Execution Plan
--------------------------------------------------------------------------------------
 0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=2 Bytes=156)
 1   0    NESTED LOOPS (Cost=3 Card=2 Bytes=156)
 2   1      VIEW OF 'EMP_SALESMAN' (VIEW) (Cost=2 Card=2 Bytes=130)
 3   2        TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=2 Card=2 ...)
 4   3          INDEX (RANGE SCAN) OF 'EMP_SAL_IDX' (INDEX) (Cost=1 Card=7)
 5   1      TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (TABLE) (Cost=1 Card=1 Bytes=13)
 6   5        INDEX (UNIQUE SCAN) OF 'DEPT_PK' (INDEX (UNIQUE)) (Cost=0 Card=1)
```
뷰 Merging이 작동한다면 변환된 쿼리는 아래와 같은 모습일 것이다.
```
select e.empno, e.ename, e.job, e.mgr, e.sal, d.dname
from   emp e, dept d
where  e.deptno = d.deptno
and    e.job = 'SALESMAN'
and    e.sal >= 1500 ;
```
그리고 그때의 실행계획은 다음과 같이 일반 조인문을 처리하는 것과 똑같은 형태가 된다.
```
Execution Plan
--------------------------------------------------------------------------------------
 0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=2 Bytes=84)
 1   0    NESTED LOOPS (Cost=3 Card=2 Bytes=84)
 2   1      TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=2 Card=2 Bytes=58)
 3   2        INDEX (RANGE SCAN) OF 'EMP_SAL_IDX' (INDEX) (Cost=1 Card=7)
 4   1      TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (TABLE) (Cost=1 Card=1 Bytes=13)
 5   4        INDEX (UNIQUE SCAN) OF 'DEPT_PK' (INDEX (UNIQUE)) (Cost=0 Card=1)
```

<br/>

## (3) 복합 뷰(Complex View) Merging
아래 항목을 포함하는 복합 뷰(Complex View)는 _complex_view_merging 파라미터를 true로 설정할 때만 Merging이 일어난다.
- group by 절
- select-list에 distince 연산자 포함

8i에서는 _complex_view_merging 파라미터의 기본 값이 false 이므로 기본적으로 복합 뷰 Merging이 작동하지 않는다. 복합 뷰 Merging을 원할 때는 merge 힌트를 사용해야만 한다.

9i부터는 이 파라미터가 기본적으로 true로 설정돼 있으므로 동일한 결과가 보장되는 한 복잡 뷰 Merging이 항상 일어난다. 뷰를 Merging하면 더 나은 실행계획을 생산할 가능성이 높다고 믿기 때문에 그렇게 하는 것이며, 휴리스틱(Heuristic 또는 Rule-based) 쿼리 변환의 전형이라고 할 수 있다.
이를 막고자 할 때 no_merge 힌트를 사용한다. (뷰 안에 rownum을 넣어도 된다.)

10g에서도 복합 뷰 Merging을 일단 시도하지만, 원본 쿼리에 대해서도 비용을 같이 계산해 Merging 헀을 때의 비용이 더 낮을 때만 그것을 채택한다(→ 비용기반 쿼리 변환).
10g에서도 뷰 Merging을 강제하고 싶을 때는 merge 힌트를 사용하면 된다.

참고로, 앞 절에서 설명한 서브쿼리 Unnesting도 뷰 Merging과 똑같은 진화과정을 거쳐 비용기반 쿼리 변환 방식으로 작동하게 되었다.

_complex_view_merging 파라미터를 true로 설정하더라도 아래 항목들을 포함하는 복합 뷰는 Merging 될 수 없다(→ Non-mergeable Views).
- 집합(set) 연산자(union, union all, intersect, minus)
- connect by 절
- ROWNUM pseudo 컬럼
- select-list에 집계 함수(avg, count, max, min, sum) 사용: group by 없이 전체를 집계하는 경우를 말함
- 분석 함수(Analytic Function)

아래는 복합 뷰를 포함한 쿼리 예시다.
```
select d.dname, avg_sal_dept
from   dept d
    , (select deptno, avg(sal) avg_sal_dept
       from emp
       group by deptno) e
where  d.deptno = e.depnto
and    d.loc = 'CHICAGO'
```
뷰 쿼리 블록을 액세스 쿼리 블록과 Merging하고 나면 아래와 같은 형태가 된다.
```
select d.dname, avg(sal)
from   dept d, emp e
where  d.deptno = e.deptno
and    d.loc = 'CHICAGO'
group by d.rowid, d.dname
```
뷰 Merging이 일어난다면 두 쿼리는 똑같이 아래 실행계획을 사용한다.
```
Execution Plan
--------------------------------------------------------------------------------------
 0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=5 Card=1 Bytes=28)
 1   0    HASH (GROUP BY) (Cost=5 Card=1 Bytes=28)
 2   1      TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=5 Bytes=25)
 3   2        NESTED LOOPS (Cost=4 Card=5 Bytes=140)
 4   3          TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=1 Bytes=23)
 5   3          INDEX (RANGE SCAN) OF 'EMP_IDX' (INDEX) (Cost=0 Card=5)
```
위 쿼리가 뷰 Merging을 통해 얻을 수 있는 이점은, dept.loc = 'CHICAGO'인 데이터만 선택해서 조인하고, 조인에 성공한 집합만 group by 한다는 데에 있다.
만약 뷰를 Merging하지 않는다면 emp 테이블에 있는 모든 데이터를 group by 해서 조인하고 나서야 loc = 'CHICAGO' 조건을 필터링하게 되므로 emp 테이블을 스캔하는 과정에서 불필요한 레코드
액세스가 많이 발생하게 된다.

<br/>

## (4) 비용기반 쿼리 변환의 필요성
9i에서 복합 뷰(Complex View)를 무조건 Merging 하도록 구현한 것은, 옵티마이저 개발팀의 경험과 테스트 결과에 비추어 볼 때 보편적으로 더 나은 성능을 보였기 때문일 것이다.
그런데 다른 쿼리 변환은 대개 더 나은 성능을 제공하지만, 복합 뷰 Merging은 그렇지 못할 떄가 많다.

예를 들어, 앞서 보았던 쿼리를 뷰 Merging 했을 때의 성패는 loc = 'CHICAGO' 조건에 달렸다.
이 조건에 의해 선택된 deptno가 emp 테이블에서 많은 비중을 차지한다면 오히려 Table Full Scan을 감수하더라도 group by로 먼저 집합을 줄이고 나서 조인하는 편이 더 나을 것이다.
9i 환경에서 튜닝할 때, no_merge 힌트를 사용하거나 뷰 안에 rownum을 넣어주는 튜닝 기법을 자주 사용하게 되는 이유가 여기에 있다.

그래서 10g부터는 비용기반 쿼리 변환 방식으로 전환하게 되었고, 이 기능을 제어하기 위한 파라미터가 _optimizer_cost_based_transformation이다. 설정 가능한 값으로는 아래 5가지가 있다.
- on
- off
- exhaustive
- linear
- iterative

비용기반 서브쿼리 Unnesting도 이 파라미터에 의해 영향을 받는다.
참고로, 바로 다음 절에서 설명하는 조건절(Predicate) Pushing도 비용기반 쿼리 변환 방식으로 전환되었지만 이 기능은 별도의 파라미터(_optimizer_push_pred_cost_based)로 제어된다.
그리고 오라클 10.2.0.2에서 _optimizer_connect_by_cost_based 파라미터가 추가된 것을 통해 connect by도 비용기반 하에 쿼리 변환이 일어나게 되었음을 알 수 있다.

비용기반 쿼리 변환이 휴리스틱 쿼리 변환보다 고급 기능이긴 하지만 파싱 과정에서 더 많은 일을 수행해야만 한다.
약간의 하드 파싱 부하를 감소하더라도 더 나은 실행계획을 얻으려는 것이므로 이들 파라미터를 off 시키는 것은 바람직하지 않다.

```
select /*+ opt_param('_optimizer_push_pred_cost_based', 'false') */ from ...
```
실제로 10g에서 조인 조건(Join Predicate) Pushdown 기능이 비용기반 쿼리 변환으로 바뀌면서 쿼리 성능이 느려지는 경우가 자주 발생한다.
이때는 문제가 되는 쿼리 레벨에서 위와 같이 힌트를 이용해 파라미터를 false로 변경하면 된다.

<br/>

## (5) Merging 되지 않은 뷰의 처리방식
뷰 Merging을 시행했을 때 오히려 비용이 더 증가한다고 판단되거나(10g부터) 부정확한 결과집합이 만들어질 가능성이 있을 때 옵티마이저는 뷰 Merging을 포기한다.
어떤 이유에서건 뷰 Merging이 이루어지지 않았을 땐 2차적으로 조건절 Pushing(다음 절에서 설명함)을 시도한다.
하지만 이마저도 실패한다면 뷰 쿼리 블록을 개별적으로 최적화하고, 거기서 생성된 서브플랜(Subplan)을 전체 실행계획을 생성하는 데 사용한다.
실제 쿼리를 수행할 때도 뷰 쿼리의 수행 결과를 액세스 쿼리에 전달하는 방식을 사용한다.

아래는 단순 뷰(Simple View)를 참조하므로 버전에 상관없이 항상 뷰 Merging이 일어난다.
```
select /*+ leading(e) use_nl(d) */ *
from   dept d
     , (select * from emp) e
where  e.deptno = d.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.016        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.001       0        20          0     14
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.016        0.001       0        20          0     14

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     14  NESTED LOOPS (cr=20 pr=0 pw=0 time=146 us)
     14    TABLE ACCESS FULL EMP (cr=4 pr=0 pw=0 time=240 us)
     14    TABLE ACCESS BY INDEX ROWID DEPT (cr=16 pr=0 pw=0 time=622 us)
     14      INDEX UNIQUE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=257 us)
```
아래는 no_merge 힌트를 사용해 뷰 Merging을 방지했을 때의 SQL 트레이스 결과다.
```
select /*+ leading(e) use_nl(d) */ *
from   dept d
     , (select /*+ NO_MERGE */ * from emp) e
where  e.deptno = d.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.001       0        20          0     14
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.000        0.001       0        20          0     14

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     14  NESTED LOOPS (cr=20 pr=0 pw=0 time=129 us)
     14    VIEW (cr=4 pr=0 pw=0 time=270 us)
     14      TABLE ACCESS FULL EMP (cr=4 pr=0 pw=0 time=222 us)
     14    TABLE ACCESS BY INDEX ROWID DEPT (cr=16 pr=0 pw=0 time=314 us)
     14      INDEX UNIQUE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=132 us)
```
여기서 오해하지 말 것은, 실행계획에 'VIEW' 라고 표시된 오퍼레이션 단계가 추가되었다고 해서 다음 단계로 넘어가기 전에 중간집합을 생성하는 것은 아니라는 점이다.
따라서 위 실행계획이 위와 같더라도 완전한 부분범위 처리가 가능하다.

아래처럼 뷰가 NL 조인에서 Inner 테이블로서 액세스될 때는 어떻게 처리될 것으로 예상하는가?
```
select /*+ leading(d) use_nl(d) */ *
from   dept d
     , (select /*+ NO_MERGE */ * from emp) e
where  e.deptno = d.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.001       0        20          0     14
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.000        0.001       0        20          0     14

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     14  NESTED LOOPS (cr=17 pr=0 pw=0 time=122 us)
      4    TABLE ACCESS FULL DEPT (cr=4 pr=0 pw=0 time=80 us)
     14    VIEW (cr=13 pr=0 pw=0 time=198 us)
     56      TABLE ACCESS FULL EMP (cr=13 pr=0 pw=0 time=241 us)
```
이때도 VIEW 처리 단계에서 중간집합을 생성하지는 않는다. 따라서 드라이빙 테이블 dept에서 읽은 건수만큼 emp 테이블에 대한 Full Scan을 반복한다.
emp 테이블을 Full Scan하는 단계에서 읽은 블록 개수(=13)와 출력된 결과 건수(=56)가 이를 잘 말해주고 있다.

아래처럼 인라인 뷰에 order by 절을 추가하고 다시 수행해보자.
```
select /*+ leading(d) use_nl(e) */ *
from   dept d
     , (select /*+ NO_MERGE */ * from emp ORDER BY ENAME) e
where  e.deptno = d.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.002       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.000       0         7          0     14
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.000        0.003       0         7          0     14

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     14  NESTED LOOPS (cr=7 pr=0 pw=0 time=169 us)
      4    TABLE ACCESS FULL DEPT (cr=4 pr=0 pw=0 time=87 us)
     14    VIEW (cr=3 pr=0 pw=0 time=208 us)
     56      SORT ORDER BY (cr=3 pr=0 pw=0 time=255 us)
     14        TABLE ACCESS FULL EMP (cr=3 pr=0 pw=0 time=61 us)
```
emp 테이블 액세스 단계에서 14건만 리턴하고 블록 I/O도 3개로 줄어들었다. 즉, emp 테이블은 한 번만 Full Scan 했고, 소트 수행 후 PGA에 저장된 중간집합을 반복 액세스한 것을 알 수 있다.
따라서 추가적인 블록 I/O가 발생하지 않았다.