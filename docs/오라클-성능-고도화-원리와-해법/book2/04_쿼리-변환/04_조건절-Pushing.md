# 04. 조건절 Pushing

뷰를 액세스하는 쿼리를 최적화할 때 옵티마이저는 1차적으로 뷰 Merging을 고려한다. 하지만 아래와 같은 이유로 뷰 Merging에 실패할 수 있다.

- 복합 뷰(Complex View) Merging 기능이 비활성화
- 사용자가 no_merge 힌트 사용
- Non-mergeable Views : 뷰 Merging 시행하면 부정확한 결과 가능성
- 비용기반 쿼리 변환이 작동해 No Merging 선택(10g 이후)

어떤 이유에서건 뷰 Merging을 실패했을 때, 옵티마이저는 포기하지 않고 2차적으로 조건절(Predicate) Pushing을 시도한다.
이는 뷰를 참조하는 쿼리 블록의 조건절을 뷰 쿼리 블록 안으로 Pushing하는 기능을 일컫는다.

조건절이 가능한 빨리 처리되도록 뷰 안으로 밀어 넣는다면, 뷰 안에서의 처리 일량을 최소화(group by 등 연산 작업 최소화, 인덱스 액세스 조건으로 사용)하게 됨은 물론 리턴되는 결과 건수를 줄임으로써
다음 단게에서 처리해야 할 일량을 줄일 수 있다.

### [조건절 Pushing 종류]
오라클은 조건절 Pushing과 관련해 다음과 같은 기술을 사용한다.
- **조건절(Predicate) Pushdown** : 쿼리 블록 밖에 있는 조건들을 쿼리 블록 안쪽으로 밀어 넣는 것을 말함
- **조건절(Predicate) Pullup** : 쿼리 블록 안에 있는 조건들을 쿼리 블록 밖으로 내오는 것을 말하며, 그것을 다시 다른 쿼리 블록에 Pushdown 하는 데 사용함(→ Predicate Move Around)
- **조인 조건(Join Predicate) Pushdown** : NL 조인 수행 중에 드라이빙 테이블에서 읽은 값을 건건이 Inner 쪽(=right side) 뷰 쿼리 블록 안으로 밀어 넣는 것을 말함

### [관련 힌트와 파라미터]
조건절 Pushdown과 Pullup은 항상 더 나은 성능을 보장하므로 별도의 힌트를 제공하지 않는다. 하지만, 조인 조건 Pushdown은 NL 조인을 전제로 하기 때문에 성능이 더 나빠질 수도 있다.
따라서 오라클은 조인 조건 Pushdown을 제어할 수 있도록 push_pred와 no_push_pred 힌트를 제공한다.

그런데 조인 조건 Pushdown 기능이 10g에서 비용기반 쿼리 변환으로 바뀌었고, 이 때문에 9i에서 빠르게 수행되던 쿼리가 10g로 이행하면서 오히려 느려지는 현상이 종종 나타나고 있다.
이때는 문제가 되는 쿼리 레벨에서 아래와 같이 힌트를 이용해 파라미터를 false로 변경하면 된다.
- Outer 조인 뷰에 대한 조인 조건 Pushdown은 9i에서도 비용기반으로 작동하였다.
```
select /*+ opt_param('_optimizer_push_pred_cost_based', 'false') * / * from ...
```

10g에서 시스템 환경에 따라 이 기능이 문제를 일으켜 쿼리 결과가 틀리는 문제도 발생하는 것으로 보고되고 있는데, 그때는 패치를 적용하거나 아래와 같이 시스템 레벨 변경이 불가피하다.
```
alter system set "_optimizer_push_pred_cost_based" = false;
```

### [Non-pushable View]
뷰 안에 rownum을 사용하면 Non-mergeable View가 된다고 앞 절에서 설명했는데, 동시에 Non-pushable View가 된다는 사실도 기억하기 바란다.
왜냐하면, rownum은 집합을 출력하는 단계에서 실시간 부여되는 값인데, 조건절 Pushing이 작동하면 기존에 없던 조건절이 생겨 같은 로우가 다른 값을 부여받을 수 있기 때문이다.
어떤 경우에도 옵티마이징 기법에 따라 쿼리 결과가 달라지는 일이 발생해선 안 된다.

분석함수(Analytic Function)를 사용해도 Non-mergeable, Non-pushable View가 되므로 주의하기 바란다.

지금부터 조건절 Pushdown, 조건절 Pullup, 조인 조건 Pushdown에 대해 자세히 살펴보자.

<br/>

## (1) 조건절 Pushdown
### [GROUP BY 절을 포함한 뷰에 대한 조건절 Pushdown]
group by 절을 포함한 복합 뷰(Complex View) Merging에 실패했을 때, 쿼리 블록 밖에 있는 조건절을 쿼리 블록 안쪽에 밀어 넣을 수 있다면 group by 해야 할 데이터량을 줄일 수 있다.
인덱스 상황에 따라서는 더 효과적인 인덱스 선택이 가능해지기도 한다.

아래 쿼리는 group by 절을 포함한 복합 뷰 사례인데, 9i 이후로는 복합 뷰 Merging이 기본적으로 활성화되므로 뷰 Merging이 발생할 것이다.
이를 막도록 _complex_view_merging 파라미터를 false로 바꾸거나 no_merge 힌트를 사용할 때의 실행계획을 확인해 보자.
```
alter sessioni set "_complex_view_merging"= false;

select deptno, avg_sal
from   (select deptno, avg(sal) avg_sal from emp group by deptno) a
where  deptno = 30

---------------------------------------------------------------------------
| Id  | Operation                        | Name           | Rows  | Bytes |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                |     1 |    26 |
|   1 |   VIEW                           |                |     1 |    26 |
|   2 |     SORT GROUP BY NOSORT         |                |     1 |     7 |
|   3 |       TABLE ACCESS BY INDEX ROWID| EMP            |     6 |    42 |
|*  4 |         INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     6 |       |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  4 - access("DEPTNO"=30)
```
뷰 Merging에 실패했지만 옵티마이저가 조건절을 뷰 안쪽으로 밀어 넣음(pushdown)으로써, emp_deptno_idx 인덱스를 사용한 것을 볼 수 있다.
조건절 Pushing이 작동 안 했다면 emp 테이블을 Full Scan 하고서 group by 이후에 deptno = 30 조건을 필터링했을 것이다.

이번에는 조인문으로 테스트해 보자.
```
select /*+ no_merge(a) * /
       b.deptno, b.dname, a.avg_sal
from  (select deptno, avg(sal) avg_sal from emp group by deptno) a
     , dept b
where  a.deptno = b.deptno
and    b.deptno = 30

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     1 |    39 |
|   1 |   NESTED LOOPS                     |                |     1 |    39 |
|   2 |     TABLE ACCESS BY INDEX ROWID    | DEPT           |     1 |    13 |
|*  3 |       INDEX UNIQUE SCAN            | DEPT_PK        |     1 |       |
|   4 |     VIEW                           |                |     1 |    26 |
|   5 |       SORT GROUP BY NOSORT         |                |     1 |     7 |
|   6 |         TABLE ACCESS BY INDEX ROWID| EMP            |     6 |    42 |
|*  7 |           INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     6 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("B"."DEPTNO"=30)
  7 - access("DEPTNO"=30)
```
이것은 뒤에서 설명하는 '조인 조건 Pushdown'과 구분되어야 한다. NL 조인을 수행하는 동안 dept 테이블에서 읽은 조인 컬럼(deptno) 값을 건건이 뷰 쿼리 블록에 pushdown 한 것이 아니기 때문이다.

여기서는 인라인 뷰 자체적으로 사전에 deptno = 30 조건절을 적용해 데이터량을 줄이고, group by 하고 나서 조인에 참여하였다.
deptno = 30 조건이 인라인 뷰에 pushdown 될 수 있었던 이유는, 다음 절에서 설명하는 '조건절 이행'이 먼저 일어났기 때문이다.
b.deptno = 30 조건이 조인 조건을 타고 a쪽에 전이됨으로써 아래와 같이 a.deptno = 30 조건절이 내부적으로 생성된 것이다.
```
select /*+ no_merge(a) * /
       b.deptno, b.dname, a.avg_sal
from  (select deptno, avg(sal) avg_sal from emp group by deptno) a
     , dept b
where  a.deptno = b.deptno
and    b.deptno = 30
and    a.deptno = 30
```
이 상태에서 a.deptno = 30 조건절이 인라인 뷰 안쪽으로 Pushing 된 것이므로 일반적인 조건절 Pushing으로 이해해야 한다.

### [UNION 집합 연산자를 포함한 뷰에 대한 조건절 Pushdown]
union 집합 연산자를 포함한 뷰는 Non-mergeable View에 속하므로 복합 뷰(Complex View) Merging 기능을 활성화하더라도 뷰 Merging에 실패한다.
따라서 조건절 Pushing을 통해서만 최적화가 가능하며, 아래는 그 사례를 보이고 있다.
```
create index emp_x1 on emp(deptno, job);

select *
from  (select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate
       from   emp
       where  job = 'CLERK'
       union all
       select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate
       from   emp
       where  job = 'SALESMAN') v
where  v.deptno = 30

---------------------------------------------------------------------------------
| Id  | Operation                        | Name   | Rows  | Bytes | Cost (%CPU) |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |        |     2 |   148 |      4   (0)|
|   1 |   VIEW                           |        |     2 |   148 |      4   (0)|
|   2 |     UNION-ALL                    |        |       |       |             |
|   3 |       TABLE ACCESS BY INDEX ROWID| EMP    |     1 |    33 |      2   (0)|
|*  4 |         INDEX RANGE SCAN         | EMP_X1 |     2 |       |      1   (0)|
|   5 |       TABLE ACCESS BY INDEX ROWID| EMP    |     1 |    33 |      2   (0)|
|*  6 |         INDEX RANGE SCAN         | EMP_X1 |     2 |       |      1   (0)|
---------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  4 - access("DEPTNO"=30 AND "JOB"='CLERK')
  6 - access("DEPTNO"=30 AND "JOB"='SALESMAN')
```
emp_x1 인덱스는 [deptno + job] 순으로 구성된 결합 인덱스고, 인덱스 선두컬럼인 deptno 조건이 뷰 쿼리 블록 안쪽에 기술되지 않았음에도 이 인덱스가 정상적인 Range Scan을 보이고 있다.
조건절 Pushing이 일어났기 때문이며, 실행계획 아래쪽 Predicate 정보를 통해서도 deptno 조건이 인덱스 액세스 조건으로 사용되었음을 알 수 있다.

아래는 조인 조건을 타고 전이된 상수 조건이 뷰 쿼리 블록에 Pushing된 경우다.
```
select /*+ ordered use_nl(e) * / d.dname, e.*
from   dept d
     , (select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate from emp
        where  job = 'CLERK'
        union all
        select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate from emp
        where  job = 'SALESMAN' ) e
where   e.deptno = d.deptno
and     d.deptno = 30

---------------------------------------------------------------------------------
| Id  | Operation                          | Name    | Rows  | Bytes | Cost (%CPU) |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |         |     2 |   174 |      5   (0)|
|   1 |   NESTED LOOPS                     |         |     2 |   174 |      5   (0)|
|   2 |     TABLE ACCESS BY INDEX ROWID    | DEPT    |     1 |    13 |      1   (0)|
|*  3 |       INDEX UNIQUE SCAN            | DEPT_PK |     1 |       |      0   (0)|
|   4 |     VIEW                           |         |     2 |   148 |      4   (0)|
|   5 |       UNION-ALL                    |         |       |       |             |
|   6 |         TABLE ACCESS BY INDEX ROWID| EMP     |     1 |    33 |      2   (0)|
|*  7 |           INDEX RANGE SCAN         | EMP_X1  |     2 |       |      1   (0)|
|   8 |         TABLE ACCESS BY INDEX ROWID| EMP     |     1 |    33 |      2   (0)|
|*  9 |           INDEX RANGE SCAN         | EMP_X1  |     2 |       |      1   (0)|
---------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("D"."DEPTNO"=30)
  7 - access("DEPTNO"=30 AND "JOB"='CLERK')
  9 - access("DEPTNO"=30 AND "JOB"='SALESMAN')
```

<br/>

## (2) 조건절 Pullup
조건절을 쿼리 블록 안으로 밀어 넣을 뿐만 아니라 안쪽에 있는 조건들을 바깥 쪽으로 끄집어 내기도 하는데, 이를 '조건절(Predicate) Pullup'이라고 한다.
그리고 그것을 다시 다른 쿼리 블록에 Pushdown 하는 데 사용한다(→ Predicate Move Around). 아래 실행계획을 보자.
```
select * from
  (select deptno, avg(sal) from emp where deptno = 10 group by deptno) e1
 ,(select deptno, min(sal), max(sal) from emp group by deptno) e2
where  e1.deptno = e.deptno

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     1 |    65 |
|*  1 |   HASH JOIN                        |                |     1 |    65 |
|   2 |     VIEW                           |                |     1 |    26 |
|   3 |       HASH GROUP BY                |                |     1 |     5 |
|   4 |         TABLE ACCESS BY INDEX ROWID| EMP            |     5 |    25 |
|*  5 |           INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
|   6 |     VIEW                           |                |     1 |    39 |
|   7 |       HASH GROUP BY                |                |     1 |     5 |
|   8 |         TABLE ACCESS BY INDEX ROWID| EMP            |     5 |    25 |
|*  9 |           INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - access("E1"."DEPTNO"="E2"."DEPTNO")
  5 - access("DEPTNO"=10)
  9 - access("DEPTNO"=10)
```
인라인 뷰 e2에는 deptno = 10 조건이 없지만 Predicate 정보를 보면 양쪽 모두 이 조건이 emp_deptno_idx 인덱스의 액세스 조건으로 사용된 것을 볼 수 있다.
아래와 같은 형태로 쿼리 변환이 일어난 것이다.
```
select * from
  (select deptno, avg(sal) from emp where deptno = 10 group by deptno) e1
 ,(select deptno, min(sal), max(sal) from emp where deptno = 10 group by deptno) e2
where  e1.deptno = e.deptno
```
아래는 Predicate Move Around 기능이 작동하지 않았을 때 어떤 비효율이 발생하는지를 보여준다.
opt_param 힌트를 이용해 이 기능을 비활성화시켰고, 이 때무네 9번 단게에서 Index FUll Scan이 나타난 것을 확인하기 바란다. (상황에 따라 Full Table Scan이 나타날 수도 있다.)
```
select /*+ opt_param('_pred_move_around', 'false') * / * from
  (select deptno, avg(sal) from emp where deptno = 10 group by deptno) e1
 ,(select deptno, min(sal), max(sal) avg_sal from emp group by deptno) e2
where  e1.deptno = e2.deptno

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     1 |    65 |
|*  1 |   HASH JOIN                        |                |     1 |    65 |
|   2 |     VIEW                           |                |     1 |    26 |
|   3 |       HASH GROUP BY                |                |     1 |     5 |
|   4 |         TABLE ACCESS BY INDEX ROWID| EMP            |     5 |    25 |
|*  5 |           INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
|   6 |     VIEW                           |                |     3 |   117 |
|   7 |       HASH GROUP BY                |                |     3 |    15 |
|   8 |         TABLE ACCESS BY INDEX ROWID| EMP            |    14 |    70 |
|   9 |           INDEX FULL SCAN          | EMP_DEPTNO_IDX |    14 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - access("E1"."DEPTNO"="E2"."DEPTNO")
  5 - access("DEPTNO"=10)
```

<br/>

## (3) 조인 조건 Pushdown
'조인 조건(Join Predicate) Pushdown'은 말 그대로 조인 조건절을 뷰 쿼리 블록 안으로 밀어 넣는 것으로서, NL 조인 수행 중에 드라이빙 테이블에서 읽은 조인 컬럼 값을 Inner쪽(=right side)
뷰 쿼리 블록 내에서 참조할 수 있도록 하는 기능이다. 조인 조건 Pushdown은 조건절 Pushdown의 일종이지만 이를 따로 분류한 이유를 앞에서 잠시 언급하였다.
지금까지 보았던 조인문에서의 조건절 Pushdown은 상수 조건이 조인 조건을 타고 전이된 것을 Pushing하는 기능이었던 반면, 지금 설명하는 조인 조건 Pushdown은 조인을 수행하는 중에 드라이빙 집합에서
얻은 값을 뷰 쿼리 블록 안에 실시간으로 Pushing하는 기능이다.

아래 예제 SQL을 보자.
```
select /*+ no_merge(e) push_pred(e) * / *
from   dept d, (select empno, ename, deptno from emp) e
where  e.deptno(+) = d.deptno
and    d.loc = 'CHICAGO'

---------------------------------------------------------------------------
| Id  | Operation                        | Name           | Rows  | Bytes |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                |     4 |   220 |
|   1 |   NESTED LOOPS OUTER             |                |     4 |   220 |
|*  2 |     TABLE ACCESS FULL            | DEPT           |     1 |    20 |
|   3 |     VIEW PUSHED PREDICATE        |                |     1 |    35 |
|   4 |       TABLE ACCESS BY INDEX ROWID| EMP            |     5 |    60 |
|*  5 |         INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  2 - filter("D"."LOC"='CHICAGO')
  5 - access("DEPTNO"="D"."DEPTNO")
```
위 쿼리의 실행계획과 Predicate 정보를 보면, 인라인 뷰 내에서 메인 쿼리에 있는 d.deptno 컬럼을 참조할 수 없음에도 옵티마이저가 이를 참조하는 조인 조건을 뷰 안쪽에 생성해준 것을 알 수 있다.
조인 조건 Pushdown이 일어난 것이며, 실행계획상 'VIEW PUSHED PREDICATE' 오퍼레이션이 나타나는 것을 통해서도 이를 알 수 있다.

조인 조건 Pushdown을 제어하는 힌트로는 아래 두 가지가 있다.
- push_pred : 조인 조건 Pushdown을 유도한다.
- no_push_pred : 조인 조건 Pushdown을 방지한다.

그리고 이를 제어하는 파라미터로는 아래 세 가지가 있다.
- _push_join_predicate : 뷰 Merging에 실패한 뷰 안쪽으로 조인 조건을 Pushdown하는 기능을 활성화한다.
  union 또는 union all을 포함하는 Non-mergeable 뷰에 대해서는 아래 두 파라미터가 따로 제공된다.

- _push_join_union_view : union all을 포함하는 Non-mergeable View 안쪽으로 조인 조건을 Pushdown하는 기능을 활성화한다.
- _push_join_union_view2 : union을 포함하는 Non-mergeable View 안쪽으로 조인 조건을 Pushdown하는 기능을 활성화한다.

위 항목은 10g 기준이며, 9i에서는 _push_join_union_view2 파라미터가 없다. 9i에서 union all을 포함한 뷰에 대한 조인 조건 Pushdown은 작동하지만 union에는 작동하지 않는다는 뜻이다.

### [GROUP BY 절을 포함한 뷰에 대한 조인 조건 Pushdown]
가장 질문을 많이 받는 사항 중 하나인데, 아래처럼 group by를 포함하는 뷰에 대한 조인 조건 Pushdown 기능은 11g에 와서야 제공되기 시작했다.

아래는 10g 실행계획이며, 조인 조건 Pushdown이 작동하지 않아 emp 쪽 인덱스를 full scan하는 것을 확인하기 바란다. dept 테이블에서 읽히는 deptn마다 emp 테이블 전체를 group by 하고 있다.
```
select /*+ leading(d) use_nl(e) no_merge(e) push_pred(e) */
       d.deptno, d.dname, e.avg_sal
from   dept d
     , (select deptno, avg(sal) avg_sal from emp group by deptno) e
where  e.deptno(+) = d.deptno

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     4 |   148 |
|   1 |   NESTED LOOPS OUTER               |                |     4 |   148 |
|   2 |     TABLE ACCESS FULL              | DEPT           |     4 |    44 |
|*  3 |     VIEW                           |                |     1 |    26 |
|   4 |       SORT GROUP BY                |                |     3 |    21 |
|   5 |         TABLE ACCESS BY INDEX ROWID| EMP            |    14 |    98 |
|   6 |           INDEX FULL SCAN          | EMP_DEPTNO_IDX |    14 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - filter("E"."DEPTNO"(+)="D"."DEPTNO")
```
아래는 11g에서 확인한 실행계획이다. 'VIEW PUSHED PREDICATE' 오퍼레이션(id=3)이 나타났고, emp_deptno_idx 인덱스를 통해 emp 테이블을 액세스한다.
즉, dept 테이블로부터 넘겨진 deptno에 대해서만 group by 하는 것이다.
```
select /*+ leading(d) use_nl(e) no_merge(e) push_pred(e) index(e (deptno)) * /
       d.deptno, d.dname, e.avg_sal
from   dept d
     , (select deptno, avg(sal) avg_sal from emp group by deptno) e
where  e.deptno(+) = d.deptno

-------------------------------------------------------------------------------
| Id  | Operation                            | Name           | Rows  | Bytes |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                |     4 |   116 |
|   1 |   NESTED LOOPS OUTER                 |                |     4 |   116 |
|   2 |     TABLE ACCESS FULL                | DEPT           |     4 |    64 |
|   3 |     VIEW PUSHED PREDICATE            |                |     1 |    13 |
|*  4 |       FILTER                         |                |       |       |
|   5 |         SORT AGGREGATE               |                |     1 |     7 |
|   6 |           TABLE ACCESS BY INDEX ROWID| EMP            |     5 |    35 |
|*  7 |             INDEX RANGE SCAN         | EMP_DEPTNO_IDX |     5 |       |
-------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  4 - filter(COUNT(*)>0)
  7 - access("DEPTNO"="D"."DEPTNO")
```
이 기능은 부분범위처리가 필요한 상황에서 특히 유용한데, 10g까지 이 기능이 제공되지 않아 아쉬움이 많았다. 10g 이하 버전이라면 어떻게 해야 할까?
다행히 여기서는 집계함수가 하나뿐이므로 아래처럼 스칼라 서브쿼리로 쉽게 변환할 수 있다.
```
select  d.deptno, d.dname
     , (select avg(sal) from emp where deptno = d.deptno)
from    dept d

-------------------------------------------------------------------------------
| Id  | Operation                            | Name           | Rows  | Bytes |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                |     4 |    44 |
|   1 |   SORT AGGREGATE                     |                |     1 |     7 |
|   2 |     TABLE ACCESS BY INDEX ROWID      | EMP            |     5 |    35 |
|   3 |       INDEX RANGE SCAN               | EMP_DEPTNO_IDX |     5 |       |
|   4 |   TABLE ACCESS FULL                  | DEPT           |     4 |    44 |
-------------------------------------------------------------------------------
```
집계함수가 두 개 이상일 때가 문제인데, 2장 6절에서 설명한 것처럼 필요한 컬럼 값들을 모두 결합하고서 바깥쪽 액세스 쿼리에서 substr 함수로 다시 분리하거나, 오브젝트 TYPE을 사용하는 방식을 고려해
볼 수 있다.

### [UNION 집합 연산을 포함한 뷰에 대한 조인 조건 Pushdown]
union 또는 union all을 포함한 뷰 쿼리 블록에 대한 조인 조건 Pushdown은 10g 이전부터 제공되던 기능이다. union all은 8.1.6부터, union은 10.1.0부터다.
아래 SQL과 실행계획을 보자.
```
create index dept_idx on dept(loc);
create index emp_idx on emp(deptno, job);

select /*+ push_pred(e) * / d.dname, e.*
from   dept d
     , (select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate from emp
        where job = 'CLERK'
        union all
        select deptno, empno, ename, job, sal, sal * 1.1 sal2, hiredate from emp
        where job = 'SALESMAN' ) e
where   e.deptno = d.deptno
and     d.loc = 'CHICAGO'

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |     2 |   200 |
|   1 |   NESTED LOOPS                     |                |     2 |   200 |
|   2 |     TABLE ACCESS BY INDEX ROWID    | DEPT           |     1 |    20 |
|*  3 |       INDEX RANGE SCAN             | DEPT_IDX       |     1 |       |
|   4 |     VIEW                           |                |     1 |    80 |
|   5 |       UNION-ALL PUSHED PREDICATE   |                |       |       |
|   6 |         TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    36 |
|*  7 |           INDEX RANGE SCAN         | EMP_IDX        |     2 |       |
|   8 |         TABLE ACCESS BY INDEX ROWID| EMP            |     1 |    36 |
|*  9 |           INDEX RANGE SCAN         | EMP_IDX        |     2 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("D"."LOC"='CHICAGO')
  7 - access("DEPTNO"="D"."DEPTNO" AND "JOB"='CLERK')
  9 - access("DEPTNO"="D"."DEPTNO" AND "JOB"='SALESMAN')
```
emp_idx 인덱스는 [deptno + job] 순으로 구성된 결합 인덱스고, 인덱스 선두컬럼인 deptno 조건이 뷰 쿼리 블록 안쪽에 기술되지 않았음에도 이 인덱스가 정상적인 Range Scan을 보이고 있다.
loc = 'CHICAGO' 조건에 해당하는 dept 테이블을 스캔하면서 얻은 deptno 값을 뷰 쿼리 블록 안에 제공했기 때문이며, 실행계획 아래쪽 Predicate 정보를 통해서도 deptno 조건이 인덱스 액세스
조건으로 사용되었음을 알 수 있다.

union all pushed predicate 오퍼레이션 단계(id=5)가 눈에 띈다.
참고로, 9i에서는 'union-all partition'이라고 표시되는데, 이는 파티션 뷰(Partition View)에 대한 Prunning 기능이 작동할 때 보던 것이어서 의미가 상통한다.
(union all을 이용해 뷰를 정의함으로써 수동으로 파티션 기능을 구현하는 것을 '파티션 뷰'라고 하며, 6장에서 설명함)

9i에서 use_nl 힌트를 push_pred와 함께 사용하면 조인 조건 Pushdown 기능이 작동하지 않는 현상이 나타나므로 주의하기 바란다.
이때는 push_pred 힌트만을 사용해야 하며, 조인 조건 Pushdown은 NL 조인을 전제로 하므로 굳이 use_nl 힌트를 사용할 필요는 없다.

### [Outer 조인 뷰에 대한 조인 조건 Pushdown]
Outer 조인에서 Inner 쪽 집합이 뷰 쿼리 블록일 때, 뷰 안에서 참조하는 테이블 개수에 따라 옵티마이저는 다음 2가지 방법 중 하나를 선택한다.

- [1] 뷰 안에서 참조하는 테이블이 단 하나일 때, 뷰 Merging을 시도한다.
- [2] 뷰 내에서 참조하는 테이블이 두 개 이상일 때, 조인 조건식을 뷰 안쪽으로 Pushing하려고 시도한다.

아래는 뷰에서 두 개 테이블을 참조하는 경우를 예시하고 있다.
```
select /*+ push_pred * /
       a.empno, a.ename, a.sal, a.hiredate, b.deptno, b.dname, b.loc, a.job
from   emp a
     , (select e.empno, d.deptno, d.dname, d.loc
        from   emp e, dept d
        where  d.deptno = e.deptno
        and    e.sal >= 1000
        and    d.loc in ( 'CHICAGO', 'NEW YORK') ) b
where   b.empno(+) = a.empno
and     a.hiredate >= to_date('19810901', 'yyyymmdd')

-------------------------------------------------------------------------------
| Id  | Operation                          | Name             | Rows  | Bytes |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                  |    14 |   952 |
|   1 |   NESTED LOOPS OUTER               |                  |    14 |   952 |
|   2 |     TABLE ACCESS BY INDEX ROWID    | EMP              |    14 |   476 |
|*  3 |       INDEX RANGE SCAN             | EMP_HIREDATE_IDX |    14 |       |
|   4 |     VIEW PUSHED PREDICATE          |                  |     1 |    34 |
|   5 |       NESTED LOOPS                 |                  |     1 |    35 |
|*  6 |         TABLE ACCESS BY INDEX ROWID| EMP              |     1 |    15 |
|*  7 |           INDEX UNIQUE SCAN        | EMP_PK           |     1 |       |
|*  8 |         TABLE ACCESS BY INDEX ROWID| DEPT             |     2 |    40 |
|*  9 |           INDEX RANGE SCAN         | DEPT_PK          |     1 |       |
-------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("A"."HIREDATE">=TO_DATE('1981-09-01 00:00:00', 'yyyy-mm-dd hh24:mi:ss'))
  6 - filter("E"."SAL">=1000)
  7 - access("E"."EMPNO"="A"."EMPNO")
  8 - filter("D"."LOC"='CHICAGO' OR "D"."LOC"='NEW YORK')
  9 - access("D"."DEPTNO"="E"."DEPTNO")
```
뷰 안에서 참조하는 테이블이 단 하나일 때도 no_merge 힌트를 사용해 뷰 Merging를 방지하면 위와 같으 조인 조건 Pushdown이 작동한다.

참고로, 앞에서 설명한 'UNION 집합 연산자를 포함한 뷰에 대한 조인 조건 Pushdown'은 10g부터 비용기반으로 작동하기 시작했지만 지금 설명한 Outer 조인 뷰에 대한 기능은 9i에서도 비용기반이었다.
'GROUP BY 절을 포함한 뷰에 대한 조인 조건 Pushdown'은 11g에 도입되면서부터 비용기반으로 작동한다.