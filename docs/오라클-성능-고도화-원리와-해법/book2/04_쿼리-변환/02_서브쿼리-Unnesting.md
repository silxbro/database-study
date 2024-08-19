# 02. 서브쿼리 Unnesting

<br/>

## (1) 서브쿼리의 분류
서브쿼리(Subquery)는 하나의 SQL 문장 내에서 괄호로 묶인 별도의 쿼리 블록(Query Block)을 말한다. 즉 쿼리에 내장된 또 다른 쿼리다.
서브쿼리를 DBMS마다 조금씩 다르게 분류하는데, 오라클 매뉴얼에는 아래 3가지로 분류돼 있다.

- **[1] 인라인 뷰(Inline View)** : from 절에 나타나는 서브쿼리를 말한다.
- **[2] 중첩된 서브쿼리(Nested Subquery)** : 결과집합을 한정하기 위해 where 절에 사용된 서브쿼리를 말한다. 특히, 서브쿼리가 메인쿼리에 있는 컬럼을 참조하는 형태를
  '상관관계 있는(Correlated) 서브쿼리'라고 부른다.
- **[3] 스칼라 서브쿼리(Scalar Subquery)** : 한 레코드당 정확히 하나의 컬럼 값만을 리턴하는 것이 특징이다.
  주로 select-list에서 사용되지만 몇 가지 예외사항을 뺸다면 컬럼이 올 수 있는 대부분 위치에서 사용 가능하다.

이들 서브쿼리를 참조하는 메인 쿼리도 하나의 쿼리 블록(Query Block)이며, 옵티마이저는 쿼리 블록 단위로 최적화를 수행한다.
즉, 쿼리 블록 단위로 최적의 액세스 경로(Access Path)와 조인 순서(Join Order), 조인 방식(Join Method)을 선택하는 것을 목표로 한다.

그런데 각 서브쿼리를 최적화했다고 해서 쿼리 전체가 최적화됐다고 말할 수는 없다. 옵티마이저가 숲을 바라보는 시각으로 쿼리를 이해하려면 먼저 서브쿼리를 풀어내야만 한다.

서브쿼리를 풀어내는 두 가지 쿼리 변환 중 '서브쿼리 Unnesting'은 중첩된 서브쿼리(Nested Subquery)와 관련 있고, 다음 절에서 설명하는 '뷰 Merging'은 인라인 뷰와 관련 있다.
(뷰 Merging은 인라인 뷰 뿐만 아니라 '저장된 뷰'에도 작동한다.)

<br/>

## (2) 서브쿼리 Unnesting의 의미
'nest'의 사전적 의미를 찾아보면, "상자 등을 차곡차곡 포개넣다"라는 설명이 있다. 즉, '중첩'의 의미를 갖는다.
여기에 '부정' 또는 '반대'의 의미가 있는 접두사 'un-'을 붙인 'unnest'는 "중첩된 상태를 풀어낸다"는 뜻이 된다.
따라서 '서브쿼리 Unnesting'은 중첩된 서브쿼리(Nested Subquery)를 풀어내는 것을 말하며, 풀어내지 않고 그대로 두는 것은 '서브쿼리 No-Unnesting'이라고 말할 수 있다.

'중첩된 서브쿼리(nested subquery)'는 메인쿼리와 부모와 자식이라는 종속적이로 계층적인 관계가 존재한다. 따라서 논리적인 관점에서 그 처리과정은 IN, Exists를 불문하고 필터 방식이어야 한다.
즉, 메인 쿼리에서 읽히는 레코드마다 서브쿼리를 반복 수행하면서 조건에 맞지 않는 데이터를 골라내는 것이다.

하지만 서브쿼리를 처리하는 데 있어 필터 방식이 항상 최적의 수행속도를 보장하지 못하므로 옵티마이저는 아래 둘 중 하나를 선택한다.

- [1] 동일한 결과를 보장하는 조인문으로 변환하고 나서 최적화한다. 이를 일컬어 '**서브쿼리 Unnesting**'이라고 한다.
- [2] 서브쿼리를 Unnesting하지 않고 원래대로 둔 상태에서 최적화한다.
  메인쿼리와 서브쿼리를 별도의 서브플랜(Subplan)으로 구분해 각각 최적화를 수행하며, 이때 서브쿼리에 필터(Filter) 오퍼레이션이 나타난다.

[1]번 서브쿼리 Unnesting은 메인과 서브쿼리 간의 계층구조를 풀어 서로 같은 레벨(flat한 구조)로 만들어 준다는 의미에서 '**서브쿼리 Flatting**'이라고도 부른다.
이렇게 쿼리 변환이 이루어지고 나면 일반 조인문처럼 다양한 최적화 기법을 사용할 수 있게 된다.

[2]번처럼, Unnesting하지 않고 쿼리 블록별로 최적화할 때는 각각의 최적이 쿼리문 전체의 최적을 달성하지 못할 때가 많다.
그리고 Plan Generator가 고려대상으로 삼을만한 다양한 실행계획을 생성해 내는 작업이 매우 제한적인 범위 내에서만 이루어진다.

- ### [서브쿼리의 또 다른 최적화 기법]

  서브쿼리를 위해 오라클 옵티마이저가 사용하는 최적화 기법이 한 가지 더 있다.
  where 조건절에 사용된 서브쿼리가 [1] 메인쿼리와 상관관계에 있지 않으면(Non-Correlated, 서브쿼리에서 메인 쿼리를 참조하지 않음) [2] 단일 로우를 리턴(single-row subquery)하는 아래와
  같은 형태의 서브쿼리를 처리할 때 나타나는 방식이다.(첫 번째는 스칼라 서브쿼리지만 두 번째는 두 개의 컬럼을 리턴하므로 스칼라 서브쿼리가 아니다.)

  ```
  select * from tab1 where key1 = (select avg(col1) from tab2);

  select * from tab1 where (key1, key2) =
      (select col1, col2 from tab2 where col3 >= 5000 and rownum = 1);
  ```

  위와 같은 형태의 서브쿼리는 Fetch가 아닌 Execute 시점에 먼저 수행한다. 그리고 그 결과 값을 메인 쿼리에 상수로 제공하는, 아래와 같은 방식으로 수행한다.

  ```
  select * from tab1 where key1 = :value1;

  select * from tab1 where (key1, key2) = (:value1, :value2);
  ```

  조건절에서 서브쿼리를 'in'이 아닌 '=' 조건으로 비교한다는 것은 서브쿼리가 단일 로우를 리턴하게 됨을 의미하므로 이런 방식을 사용할 수 있는 것이다.
  만약 이들 서브쿼리가 2개 이상의 로우를 리턴한다면 ORA-01427(single-row subquery returns more than one row) 에러가 발생하므로 대개 rownum <= 1 같은 stopkey 조건이나
  min, max, avg 등 집계함수가 사용된다.

<br/>

## (3) 서브쿼리 Unnesting의 이점
서브쿼리를 메인쿼리와 같은 레벨로 풀어낸다면 다양한 액세스 경로와 조인 메소드를 평가할 수 있다.
특히 옵티마이저는 많은 조인테크닉을 가지기 때문에 조인 형태로 변환했을 때 더 나은 실행계획을 찾을 가능성이 높아진다.

이런 이점 때문에 옵티마이저는 서브쿼리 Unnesting을 선호한다. 그래서 오라클 9i에서는 정확히 같은 결과집합임이 보장된다면 무조건 서브쿼리를 Unnesting 하려고 시도한다.
즉, 휴리스틱 쿼리 변환 방식으로 작동한다.

10g부터는 서브쿼리 Unnesting이 비용기반 쿼리 변환 방식으로 전환되었다.
따라서 변환된 쿼리의 예상 비용이 더 낮을 때만 Unnesting된 버전을 사용하고, 그렇지 않을 때는 원본 쿼리 그대로 필터 방식으로 최적화한다.

서브쿼리 Unnesting과 관련된 힌트로는 아래 두 가지가 있다.

- unnest : 서브쿼리를 Unnesting 함으로써 조인방식으로 최적화하도록 유도한다.
- no_unnest : 서브쿼리를 그대로 둔 상태에서 필터 방식으로 최적화하도록 유도한다.

<br/>

## (4) 서브쿼리 Unnesting 기본 예시
실제 서브쿼리 Unnesting이 어떤 식으로 작동하는지 살펴보자. 아래처럼 IN 서브쿼리를 포함하는 SQL문이 있다.
```
select * from emp
where  deptno in (select deptno from dept)
```
이 SQL 문을 Unnesting 하지 않고 그대로 최적화한다면 옵티마이저는 아래와 같이 필터 방식의 실행계획을 수립한다.
```
select * from emp
where deptno in (select /*+ no_unnest */ deptno from dept)

---------------------------------------------------------------------
| Id  | Operation            | Name    | Rows  | Bytes | Cost (%CPU)|
---------------------------------------------------------------------
|   0 | SELECT STATEMENT     |         |     3 |    99 |     3   (0)|
|*  1 |   FILTER             |         |       |       |            |
|   2 |     TABLE ACCESS FULL| EMP     |    10 |   330 |     3   (0)|
|*  3 |     INDEX UNIQUE SCAN| DEPT_PK |     1 |     2 |     0   (0)|
---------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - filter( EXISTS (SELECT 0 FROM "DEPT" "DEPT" WHERE "DEPTNO"=:B1))
  3 - access("DEPTNO"=:B1)
```
옵티마이저가 서브쿼리 Unnesting을 선호하므로 이를 방지하려고 no_unnest 힌트를 사용해 실행계획을 유도하였다.
Predicate 정보를 보면 필터 방식으로 수행된 서브쿼리의 조건절이 바인드 변수로 처리된 부분(DEPTNO = :B1)이 눈에 띄는데, 이것을 통해 옵티마이저가 서브쿼리를 별도의 서브플랜(Subplan)으로
최적화한다는 사실을 알 수 있다. 메인 쿼리도 하나의 쿼리 블록이므로 서브쿼리를 제외한 상태에서 별도로 최적화가 이루어졌다.(아무 조건절이 없으므로 Full Table Scan이 최적이다.)

이처럼, Unnesting하지 않은 서브쿼리를 수행할 때는 메인 쿼리에서 읽히는 레코드마다 값을 넘기면서 서브쿼리를 반복 수행한다.
(Unnesting 하지 않았지만 내부적으로 IN 서브쿼리를 Exists 서브쿼리로 변환한다는 사실도 Predicaate 정보를 통해 알 수 있다.)

unnest 힌트를 사용하거나 옵티마이저가 스스로 Unnesting을 선택한다면, 변환된 쿼리는 아래와 같은 조인문 형태가 된다.
(사용자가 발행한 SQL 텍스트를 변환하는 것은 아니며, 파서(Parser)에 의해 생성된 파싱트리 내에서 변환한다.)
```
select *
from   (select deptno from dept) a, emp b
where  b.deptno = a.deptno
```
그리고 이것은 다시 다음 절에서 설명하는 뷰 Merging 과정을 거쳐 최종적으로 아래와 같은 형태가 된다.
```
select emp.* from dept, emp
where  emp.deptno = dept.deptno
```
아래는 서브쿼리에 unnest 힌트를 주고 실행계획을 확인한 결과다. 서브쿼리인데도 일반적인 Nested Loop 조인 방식으로 수행된 것을 볼 수 있다. 위 조인문을 수행할 때와 정확히 같은 실행 계획이다.
```
select * from emp
where  deptno in (select /*+ unnest */ deptno from dept)

------------------------------------------------------------------------------------
| Id  | Operation                    | Name           | Rows  | Bytes | Cost (%CPU)|
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                |    10 |   350 |     2   (0)|
|   1 |   TABLE ACCESS BY INDEX ROWID| EMP            |     3 |    99 |     1   (0)|
|   2 |     NESTED LOOPS             |                |    10 |   350 |     2   (0)|
|   3 |       INDEX FULL SCAN        | DEPT_PK        |     4 |     8 |     1   (0)|
|   4 |       INDEX RANGE SCAN       | EMP_DEPTNO_IDX |     3 |       |     0   (0)|
------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  4 - access("DEPTNO"="DEPTNO")
```

<br/>

## (5) Unnesting된 쿼리의 조인 순서 조정
여기서 주목할 점은, Unnesting에 의해 일반 조인문으로 변환된 후에는 emp, dept 어느 쪽이든 드라이빙 집합으로 선택될 수 있다는 사실이다.
선택은 옵티마이저의 몫이며, 판단 근거는 데이터 분포를 포함한 통계정보에 있다.

서브쿼리 쪽 dept 테이블이 먼저 드라이빙되었을 때의 실행계획은 위에서 보았고, 아래는 메인쿼리 쪽 emp 테이블이 드라이빙된 경우다.
```
--------------------------------------------------------------------------------
| Id  | Operation            | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |         |    14 |   560 |     3   (0)| 00:00:01 |
|   1 |   NESTED LOOPS       |         |    14 |   560 |     3   (0)| 00:00:01 |
|   2 |     TABLE ACCESS FULL| EMP     |    14 |   518 |     3   (0)| 00:00:01 |
|*  3 |     INDEX UNIQUE SCAN| DEPT_PK |     1 |     3 |     0   (0)| 00:00:01 |
--------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("DEPTNO"="DEPTNO")
```
이 실행계획은 순서가 다르게 결정될 수 있음을 보이기 위해 힌트를 사용해 유도한 것이며, 힌트에 의해 조정될 수 있다는 것은 데이터 분포에 따라 옵티마이저도 같은 선택을 할 수 있다는 뜻이다.

Unnesting된 쿼리의 조인 순서를 조정하는 방법에 대해 살펴보자. 메인 쿼리 집합을 먼저 드라이빙하는 것은 쉽다. 아래처럼 leading(emp) 힌트를 사용하면 된다.
```
select /*+ leading(emp) */ * from emp
where  deptno in (select /*+ unnest */ deptno from dept)
```
서브쿼리 쪽 집합을 드라이빙하는 경우를 보자. 서브쿼리에서 메인 쿼리에 있는 테이블을 참조할 수는 있지만 메인 쿼리에서 서브쿼리 쪽 테이블을 참조하지는 못하므로 아래와 같은 방식을 사용할 수 없다.
(이상한 말이지만 10g부터는 아래와 같이 하더라도 조인 순서가 조정된다.)
```
select /*+ leading(dept) */ from emp
where  deptno in (select /*+ unnest */ deptno from dept)
```
그럴 때는 leading 힌트 대신 아래와 같이 ordered 힌트를 사용하면 서브쿼리 쪽 테이블을 직접 참조하지 않아도 되므로 원하는 대로 조인 순서를 유도할 수 있다.
이것을 통해, Unnesting 된 서브쿼리가 from 절에서 앞쪽에 위치함을 알 수 있다.
```
select /*+ ordered */ * from emp
where  deptno in (select /*+ unnest */ deptno from dept)
```
10g부터는 쿼리 블록(Query Block)마다 이름을 지정할 수 있는 qb_name 힌트가 제공되므로 아래처럼 쉽고 정확하게 제어할 수 있다.
```
select /*+ leading(dept@qb1) */ * from emp
where  deptno in (select /*+ unnest qb_name(qb1) */ deptno from dept)
```

<br/>

## (6) 서브쿼리가 M쪽 집합이거나 Nonunique 인덱스일 때
지금까지 본 예제는 메인 쿼리의 emp 테이블과 서브쿼리의 dept 테이블이 M:1 관계이기 때문에 일반 조인문으로 바꾸더라도 쿼리 결과가 보장된다.
옵티마이저는 dept 테이블 deptno 컬럼에 PK 제약이 설정된 것을 통해 dept 테이블이 1쪽 집합이라는 사실을 알 수 있다. 따라서 안심하고 쿼리 변환을 실시한다.

만약 서브쿼리 쪽 테이블이 조인 컬럼에 PK/Unique 제약 또는 Unique 인덱스가 없다면, 일반 조인문처럼 처리했을 때 어떻게 될까?

#### <사례1>
```
select * from dept
where  deptno in (select deptno from emp)
```
위 쿼리는 1쪽 집합을 기준으로 M쪽 집합을 필터링하는 형태이므로 당연히 서브쿼리 쪽 emp 테이블 deptno 컬럼에는 Unique 인덱스가 없다.
dept 테이블이 기준 집합이므로 결과집합은 이 테이블의 총 건수를 넘지 않아야 한다.
그런데 옵티마이저가 임의로 아래와 같은 일반 조인문으로 변환한다면 M쪽 집합인 emp 테이블 단위의 결과집합이 만들어지므로 결과 오류가 생긴다.
```
select *
from  (select deptno from emp) a, dept b
where  b.deptno = a.deptno
```

#### <사례2>
```
select * from emp
where  deptno in (select deptno from dept)
```
위 쿼리는 M쪽 집합을 드라이빙해 1쪽 집합을 서브쿼리로 필터링하도록 작성되었으므로 조인문으로 바꾸더라도 결과에 오류가 생기지는 않는다.
하지만 dept 테이블 deptno 컬럼에 PK/Unique 제약 또는 Unique 인덱스가 없다면 옵티마이저는 emp와 dept 간의 관계를 알 수 없고, 결과를 확신할 수 없으니 일반 조인문으로의 쿼리 변환을
시도하지 않는다. (만약 SQL 튜닝 차원에서 위 쿼리를 사용자가 직접 조인문으로 바꿨는데, 어느 순간 dept 테이블 deptno 컬럼에 중복 값이 입력되면서 결과에 오류가 생기더라도 옵티마이저에게는 책임이 없다.)

이럴 때 옵티마이저는 두 가지 방식 중 하나를 선택하는데, Unnesting 후 어느 쪽 집합이 먼저 드라이빙 되느냐에 따라 달라진다.
- 1쪽 집합임을 확신할 수 없는 서브쿼리 쪽 테이블이 드라이빙된다면, 먼저 sort unique 오퍼레이션을 수행함으로써 1쪽 집합으로 만든 다음에 조인한다.
- 메인 쿼리 쪽 테이블이 드라이빙된다면 세미 조인(Semi Join) 방식으로 조인한다. 이것이 세미 조인(Semi Join)이 탄생하게 된 배경이다.

### [Sort Unique 오퍼레이션 수행]
dept 테이블에서 PK 제약을 제거하고, deptno 컬럼에 Nonunique 인덱스를 생성하고 나서 실행계획을 다시 확인해 보자.
```
alter table dept drop primary key;

create index dept_deptno_idx on dept(deptno);


select * from emp
where  deptno in (select deptno from dept);

------------------------------------------------------------------------
| Id  | Operation                    | Name            | Rows  | Bytes |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                 |    11 |   440 |
|   1 |   TABLE ACCESS BY INDEX ROWID| EMP             |     4 |   148 |
|   2 |     NESTED LOOPS             |                 |    11 |   440 |
|   3 |       SORT UNIQUE            |                 |     4 |    12 |
|   4 |         INDEX FULL SCAN      | DEPT_DEPTNO_IDX |     4 |    12 |
|*  5 |       INDEX RANGE SCAN       | EMP_DEPTNO_IDX  |     5 |       |
------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  5 - access("DEPTNO"="DEPTNO")
```
실제로 dept 테이블은 Unique한 집합이지만 옵티마이저는 이를 확신할 수 없어 sort unique 오퍼레이션을 수행하였다. 아래와 같은 형태로 쿼리 변환이 일어난 것이다.
- 10gR2에서 order by를 생략하면 hash unique 방식으로 수행된다.

```
select b.*
from  (select /*+ no_merge */ distinct deptno from dept order by deptno) a, emp b
where  b.deptno = a.deptno
```
직접 테스트해 보는 과정에서 위와 같은 실행계획이 안 나타난다면(즉, 옵티마이저가 세미 조인 방식을 선택한다면) 아래와 같이 힌트를 사용해 유도할 수 있다.
Unnesting한 다음에 서브쿼리 쪽 테이블이 드라이빙 집합으로 선택되도록 하는 것이다.
```
-- 오라클 9i
select /*+ ordered use_nl(emp) */ * from emp
where  deptno in (select /*+ unnest */ deptno from dept)

-- 오라클 10g 이후
select /*+ leading(dept@qb1) use_nl(emp) */ * from emp
where  deptno in (select /*+ unnest qb_name(qb1) */ deptno from dept)
```

### [세미 조인 방식으로 수행]
아래는 세미 조인 방식으로 수행될 때의 실행계획이다.
```
select * from emp
where  dept in (select deptno from dept)

-----------------------------------------------------------------------
| Id  | Operation            | Name     | Rows  | Bytes | Cost  (%CPU)|
-----------------------------------------------------------------------
|   0 | SELECT STATEMENT     |          |    10 |   350 |      3   (0)|
|   1 |   NESTED LOOPS SEMI  |          |    10 |   350 |      3   (0)|
|   2 |     TABLE ACCESS FULL| EMP      |    10 |   330 |      3   (0)|
|*  3 |     INDEX RANGE SCAN | DEPT_IDX |     4 |     8 |      0   (0)|
-----------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  3 - access("DEPTNO"="DEPTNO")
```
NL 세미 조인으로 수행할 때는 sort unique 오퍼레이션을 수행하지 않고도 결과집합이 M쪽 집합으로 확장되는 것을 방지하는 알고리즘을 사용한다.
기본적으로 NL 조인과 동일한 프로세스로 진행하지만, Outer(=Driving) 테이블의 한 로우가 Inner 테이블의 한 로우와 조인에 성공하는 순간 진행을 멈추고 Outer 테이블의 다음 로우를 계속 처리하는 방식이다.
아래 pseudo 코드를 참고한다면 어렵지 않게 이해할 수 있다.
```
for(i=0; ; i++) {            // outer loop
  for(j=0; ; j++) {          // inner loop
    if (i==j) break;
  }
}
```
만약 옵티마이저가 세미 조인 방식을 선택하지 않는다면 아래와 같이 힌트를 사용해 유도할 수 있다. Unnesting한 다음에 메인 쿼리 쪽 테이블이 드라이빙 집합으로 선택되도록 하는 것이다.
```
select /*+ leading(emp) */ * from emp
where  deptno in (select /*+ unnest nl_sj */ deptno from dept)
```
세미 조인 방식으로 변환할 때의 장점은, NL 세미 조인뿐만 아니라 해시 세미 조인, 소트머지 세미 조인도 가능하다는 데에 있다.
사용자가 직접 유도할 때는 unnest 힌트와 함께 각각 hash_sj, merge_sj 힌트를 사용하면 된다.

<br/>

## (7) 필터 오퍼레이션과 세미조인의 캐싱 효과
옵티마이저가 쿼리 변환을 수행하는 이유는, 전체적인 시각에서 더 나은 실행계획을 수립할 가능성을 높이는 데에 있다.
서브쿼리를 Unnesting해 조인문으로 바꾸고 나면 NL 조인은 물론 해시 조인, 소트 머지 조인 방식을 선택할 수 있고, 조인 순서도 자유롭게 선택할 수 있음을 앞에서 살펴보았다.

서브쿼리를 Unnesting하지 않으면 쿼리를 최적화하는 데 있어 선택의 폭이 넓지 않아 불리하다. 메인 쿼리를 수행하면서 건건이 서브쿼리를 반복 수행하는 단순한 필터 오퍼레이션을 사용할 수 밖에 없기 때문이다.
대량의 집합을 기준으로 이처럼 Random 액세스 방식으로 서브쿼리 집합을 필터링한다면 결코 빠른 수행 속도를 얻을 수 없다.

다행히 오라클은 필터 최적화 기법을 한 가지 갖고 있는데, 서브쿼리 수행 결과를 버리지 않고 내부 캐시에 저장하고 있다가 같은 값이 입력되면 저장된 값을 출력하는 방식이다.
이전에 스칼라 서브쿼리의 캐싱 효과를 설명했는데, 그것과 같다.

조나단 루이스 설명에 의하면, 오라클 8i와 9i에서 256개, 10gdptj 1,024개 해시 엔트리를 캐싱한다고 한다. 테스트 결과를 기반으로 추측된 값이라고도 밝혔다.
실제 캐싱할 수 있는 엔트리 개수가 몇 개이건 간에 서브쿼리와 조인되는 컬럼의 Distinct Value 개수가 캐시 상한선을 초과하지만 않는다면 필터 오퍼레이션은 매우 효과적인 수행방식일 것이다.

테스트를 위해 아래와 같이 emp 테이블을 100번 복제한 t_emp 테이블을 생성해 보자.
```
create table t_emp
as
select *
from   emp
    , (select rownum no from dual connect by level <= 100);
```
이제 t_emp 테이블은 1,400개 로우를 갖는다. 아래는 방금 생성한 t_emp 테이블을 기준으로 dept 테이블에 대한 서브쿼리 필터링을 수행하는 쿼리다.
no_unnest 힌트를 이용해 필터 방식으로 수행되도록 하였다.
```
select count(*) from t_emp t
where exists (select /*+ no_unnest */ 'x' from dept
              where deptno = t.deptno and loc is not null)

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.003       0        18          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.000        0.003       0        18          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=18 pr=0 pw=0 time=2854 us)
   1400    FILTER (cr=18 pr=0 pw=0 time=25325 us)
   1400      TABLE ACCESS FULL T_EMP (cr=12 pr=0 pw=0 time=7049 us)
      3      TABLE ACCESS BY INDEX ROWID DEPT (cr=6 pr=0 pw=0 time=122 us)
      3        INDEX UNIQUE SCAN DEPT_PK (cr=3 pr=0 pw=0 time=55 us) (Object ID 57571)
```
수행 결과, dept 테이블에 대한 필터링을 1,400번 수행했음에도 읽은 블록 수는 인덱스에서 3개, 테이블에서 3개, 총 6개 뿐이다. 그리고 거기서 리턴된 결과 건수도 3개에 그치는 것을 볼 수 있다.
t_emp 테이블 deptno에는 10, 20, 30 세 개의 값만 있기 때문에 서브쿼리를 3번만 수행했고, 그 결과를 캐시에 저장한 상태에서 반복적으로 재사용했음을 잘 보여주고 있다.

NL 세미 조인에도 이런 캐싱 효과가 가능하지 않을까? 물론 가능하다. 하지만 9i까지는 캐싱 효과가 나타나지 않았다. 아래는 9i에서 NL 세미 조인으로 수행했을 때의 결과다.
```
select count(*) from t_emp t
where exists (select /*+ unnest nl_sj */ 'x' from dept
              where deptno = t.deptno and loc is not null)

Call      count       cpu      elapsed    dist     query    current   rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1      0.00         0.00       0         0          0      0
Execute       1      0.00         0.00       0         0          0      0
Fetch         2      0.05         0.05       0      1414          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4      0.05         0.05       0      1414          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=1414 pr=0 pw=0 time=52907 us)
   1400    NESTED LOOPS SEMI (cr=1414 pr=0 pw=0 time=46453 us)
   1400      TABLE ACCESS FULL T_EMP (cr=12 pr=0 pw=0 time=6613 us)
   1400      TABLE ACCESS BY INDEX ROWID DEPT (cr=1402 pr=0 pw=0 time=20490 us)
   1400        INDEX UNIQUE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=6948 us) (Object ID 39130)
```
필터 캐싱 효과가 없어 1,400번 서브쿼리가 수행된 것을 알 수 있다. 서브쿼리를 수행하는 단계에서는 블록 I/O가 1,402개 발생하였고, 리턴된 결과 건수도 1,400건이다.
그리고 NL 조인에서 Inner 쪽 인덱스 루트 블록에 대한 버퍼 Pinning 효과는 여기서도 나타난다.
위 Row Source Operation에서 dept_pk 인덱스에 대한 탐색이 1,400번 일어났지만 cr은 단 2개에 그친 것을 확인하기 바란다. (앞서 본 필터 방식에서 이 효과가 안 나타난 것도 확인하기 바란다.)

10g부터는 아래에서 보듯 NL 세미 조인도 캐싱 효과를 갖는다. 그동안 캐싱 효과를 앞세워 명맥을 유지하던 필터 오퍼레이션이 설 자리를 잃게 되었다.
```
select count(*) from t_emp t
where exists (select /*+ unnest nl_sj */ 'x' from dept
              where deptno = t.deptno and loc is not null)

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.000        0.001       0        17          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.000        0.001       0        17          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=17 pr=0 pw=0 time=849 us)
   1400    NESTED LOOPS SEMI (cr=17 pr=0 pw=0 time=15464 us)
   1400      TABLE ACCESS FULL T_EMP (cr=12 pr=0 pw=0 time=4220 us)
      3      TABLE ACCESS BY INDEX ROWID DEPT (cr=5 pr=0 pw=0 time=73 us)
      3        INDEX UNIQUE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=31 us) (Object ID 57571)
```

<br/>

## (8) Anti 조인
not exists, not in 서브쿼리도 Unnesting 하지 않으면 아래와 같이 필터 방식으로 처리된다.
```
select * from dept d
where  not exists
  (select /*+ no_unnest */ 'x' from emp where deptno = d.deptno);

---------------------------------------------------------------------------
| Id  | Operation            | Name         | Rows  | Bytes | Cost  (%CPU)|
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |              |     3 |    60 |      5   (0)|
|   1 |   FILTER             |              |       |       |             |
|   2 |     TABLE ACCESS FULL| DEPT         |     4 |    80 |      3   (0)|
|   3 |     INDEX RANGE SCAN | EMP_DEPT_IDX |     2 |     6 |      1   (0)|
---------------------------------------------------------------------------
```
기본 처리루틴은 exists 필터와 동일하며, 조인에 성공하는 레코드가 하나도 없을 때만 결과집합에 포함시킨다는 점이 다르다.

- exists 필터 : 조인에 성공하는 (서브) 레코드를 만나는 순간 결과집합에 담고 다른 (메인) 레코드로 이동한다.
- not exists 필터 : 조인에 성공하는 (서브) 레코드를 만나는 순간 버리고 다음 (메인) 레코드로 이동한다. 조인에 성공하는 (서브) 레코드가 하나도 없을 때만 결과집합에 담는다.

똑같은 쿼리를 Unnesting하면 아래와 같이 Anti 조인 방식으로 처리된다.
```
select * from dept d
where  not exists
  (select /*+ unnest nl_aj */ 'x' from emp where deptno = d.deptno);

-----------------------------------------------------------------------------
| Id  | Operation            | Name           | Rows  | Bytes | Cost  (%CPU)|
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |                |     1 |    23 |      3   (0)|
|   1 |   NESTED LOOPS ANTI  |                |     1 |    23 |      3   (0)|
|   2 |     TABLE ACCESS FULL| DEPT           |     4 |    80 |      3   (0)|
|   3 |     INDEX RANGE SCAN | EMP_DEPTNO_IDX |     9 |    27 |      0   (0)|
-----------------------------------------------------------------------------

select * from dept d
where  not exists
  (select /*+ unnest merge_aj */ 'x' from emp where deptno = d.deptno);

---------------------------------------------------------------------------------------
| Id  | Operation                      | Name           | Rows  | Bytes | Cost  (%CPU)|
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                |     1 |    23 |      4  (25)|
|   1 |   MERGE JOIN ANTI              |                |     1 |    23 |      4  (25)|
|   2 |     TABLE ACCESS BY INDEX ROWID| DEPT           |     4 |    80 |      2   (0)|
|   3 |       INDEX FULL SCAN          | DEPT_PK        |     4 |       |      1   (0)|
|   4 |     SORT UNIQUE                |                |     9 |    27 |      2  (50)|
|   5 |       INDEX FULL SCAN          | EMP_DEPTNO_IDX |     9 |    27 |      1   (0)|
---------------------------------------------------------------------------------------

select * from dept d
where  not exists
  (select /*+ unnest hash_aj */ 'x' from emp where deptno = d.deptno);

-----------------------------------------------------------------------------
| Id  | Operation            | Name           | Rows  | Bytes | Cost  (%CPU)|
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |                |     1 |    23 |      5  (20)|
|   1 |   HASH JOIN ANTI     |                |     1 |    23 |      5  (20)|
|   2 |     TABLE ACCESS FULL| DEPT           |     4 |    80 |      3   (0)|
|   3 |     INDEX FULL SCAN  | EMP_DEPTNO_IDX |    14 |    42 |      1   (0)|
-----------------------------------------------------------------------------
```
NL Anti 조인과 머지 Anti 조인은 기본 처리루틴이 not exists 필터와 같지만, 해시 Anti 조인은 조금 다르다. 해시 Anti 조인으로 수행할 때는, 먼저 dept를 해시 테이블로 만든다.
emp를 스캔하면서 해시 테이블을 탐색하고, 조인에 성공한 엔트리에만 표시를 한다. 마지막으로, 해시 테이블을 스캔하면서 표시가 없는 엔트리만 결과집합에 담는 방식이다.

<br/>

## (9) 집계 서브쿼리 제거
집계 함수(Aggregate Function)를 포함하는 서브쿼리를 Unnesting 하고, 이를 다시 분석 함수(Analytic Function)로 대체하는 쿼리 변환이 10g에서 도입되었다.
```
select d.deptno, d.dname, e.empno, e.ename, e.sal
from   emp e, dept d
where  d.deptno = e.deptno
and    e.sal >= (select avg(sal) from emp where deptno = d.deptno)
```
위 쿼리를 Unnesting하면 1차적으로 아래와 같은 쿼리가 만들어진다.
```
select d.deptno, d.dname, e.empno, e.ename, e.sal
from   (select deptno, avg(sal) avg_sal from emp group by deptno) x, emp e, dept d
where  d.deptno = e.deptno
and    e.deptno = x.deptno
and    e.sal >= x.avg_sal
```
옵티마이저는 한 번 더 쿼리 변환을 시도해 인라인 뷰를 Merging하거나 그대로 둔 채 최적화할 수 있다.
(서브쿼리에 /*+ unnest merge */와 /*+ unnest no_merge */ 힌트를 추가함으로써 이 두 가지 경우를 테스트해 볼 수 있다.)

10g부터 옵티마이저가 선택할 수 있는 옵션이 한 가지 더 추가되었는데, 서브쿼리로부터 전환된 인라인 뷰를 제거하고 아래와 같이 메인 쿼리에 분석 함수를 사용하는 형태로 변환하는 것이다.
이 기능은 _remove_aggr_subquery 파라미터에 의해 제어되며, 비용기반으로 작동한다.
```
select deptno, dname, empno, ename, sal
from (
  select d.deptno, d.dname, e.empno, e.ename, e.sal
      , (case when e.sal >= avg(sal) over (partition by d.deptno)
         then e.rowid end) max_sal_rowid
  from   emp e, dept d
  where  d.deptno = e.deptno
)
where max_sal_rowid is not null
```
아래는 10g에서 집계 서브쿼리 제거(Aggregate Subquery Elimination) 기능이 작동했을 때의 실행계획이다.
쿼리에선 emp 테이블을 두 번 참조했지만 실행계획상으로는 한 번만 액세스했고, 대신 window buffer 오퍼레이션 단계가 추가되었다.
```
select d.deptno, d.dname, e.empno, e.ename, e.sal
from   dept d, emp e
where  d.deptno = e.deptno
and    e.sal = (select max(sal) from emp where deptno = d.deptno);

-----------------------------------------------------------------------------
| Id  | Operation                          | Name           | Rows  | Bytes |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                |    14 |   938 |
|*  1 |   VIEW                             | VW_WIF_1       |    14 |   938 |
|   2 |     WINDOW BUFFER                  |                |    14 |   574 |
|   3 |       NESTED LOOPS                 |                |    14 |   574 |
|   4 |         TABLE ACCESS BY INDEX ROWID| EMP            |    14 |   420 |
|   5 |           INDEX FULL SCAN          | EMP_DEPTNO_IDX |    14 |       |
|   6 |         TABLE ACCESS BY INDEX ROWID| DEPT           |     1 |    11 |
|*  7 |           INDEX UNIQUE SCAN        | DEPT_PK        |     1 |       |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - filter("VW_COL_6" IS NOT NULL)
  7 - access("D"."DEPTNO"="E"."DEPTNO")
```
아래는 집계 서브쿼리 제거 기능이 작동하지 못하도록 파라미터를 변경할 때의 실행계획이다.
```
alter session set "_remove_aggr_subquery" = false;
select d.deptno, d.dname, e.empno, e.ename, e.sal
from   dept d, emp e
where  d.deptno = e.deptno
and    e.sal = (select max(sal) from emp where deptno = d.deptno);

---------------------------------------------------------------------------------
| Id  | Operation                              | Name           | Rows  | Bytes |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |                |     1 |    56 |
|*  1 |   TABLE ACCESS BY INDEX ROWID          | EMP            |     1 |    17 |
|   2 |     NESTED LOOPS                       |                |     1 |    56 |
|   3 |       NESTED LOOPS                     |                |     3 |   117 |
|   4 |         VIEW                           |                |     3 |    78 |
|   5 |           HASH GROUP BY                |                |     3 |    21 |
|   6 |             TABLE ACCESS BY INDEX ROWID| EMP            |    14 |    98 |
|   7 |               INDEX FULL SCAN          | EMP_DEPTNO_IDX |    14 |       |
|   8 |         TABLE ACCESS BY INDEX ROWID    | DEPT           |     1 |    13 |
|*  9 |           INDEX UNIQUE SCAN            | DEPT_PK        |     1 |       |
|* 10 |       INDEX RANGE SCAN                 | EMP_DEPTNO_IDX |     5 |       |
---------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - filter("E"."SAL"="VW_COL_1")
  9 - access("DEPTNO"="D"."DEPTNO")
 10 - access("D"."DEPTNO"="E"."DEPTNO")
```

<br/>

## (10) Pushing 서브쿼리
앞에서 설명한 것처럼 Unnesting 되지 않은 서브쿼리는 항상 필터 방식으로 처리되며, 대개 실행계획 상에서 맨 마지막 단계에 처리된다.
만약 서브쿼리 필터링을 먼저 처리했을 때 다음 수행 단계로 넘어가는 로우 수를 크게 줄일 수 있다면 성능은 그만큼 향상된다.
Pushing 서브쿼리는 이처럼 실행계획 상 가능한 앞 단계에서 서브쿼리 필터링이 처리되도록 강제하는 것을 말하며, 이를 제어하기 위해 사용하는 옵티마이저 힌트가 push_subq이다.
(인라인 뷰 안에 조건절을 밀어 넣는 '조건절(Predicate) Pushing'과 헷갈리지 말기 바란다.)

Pushing 서브쿼리는 Unnesting 되지 않은 서브쿼리에서만 작동한다는 사실을 기억할 필요가 있다. 따라서 push_subq 힌트는 항상 no_unnest 힌트와 같이 기술하는 것이 올바른 사용법이다.
만약 옵티마이저가 서브쿼리를 Unnesting 하기로 결정한다면 push_subq 힌트는 무용지물이 되기 때문이다.

push_subq 힌트가 잘 작동하지 않는다는 질문을 종종 받는데, 대개 no_unnest 힌트를 같이 사용하지 않은 경우였다. 그리고 9i와 10g 사이에 push_subq 힌트를 기술하는 위치가 바뀐 것에도 기인한다.
아래 예시를 참조하기 바란다.
```
-- 오라클 9i에서 push_subq 힌트 사용하기
select /*+ PUSH_SUBQ leading(e1) use_nl(e2) */ sum(e1.sal), sum(e2.sal)
from   emp1 e1, emp2 e2
where  emp1.no = e2.no
and    emp1.empno = e2.empno
and    exists (select /*+ NO_UNNEST */ 'x' from dept
          where  deptno = e1.deptno
          and    loc = 'NEW YORK')

-- 오라클 10g에서 push_subq 힌트 사용하기
select /*+ leading(e1) use_nl(e2) */ sum(e1.sal), sum(e2.sal)
from   emp1 e1, emp2 e2
where  emp1.no = e2.no
and    emp1.empno = e2.empno
and    exists (select /*+ NO_UNNEST PUSH_SUBQ */ 'x' from dept
          where  deptno = e1.deptno
          and    loc = 'NEW YORK')
```
서브쿼리가 여러 개일 때 push_subq 힌트를 서브쿼리에 직접 기술해야 세밀한 제어가 가능하므로 10g에서 바뀐 것이다.

Pushing 서브쿼리를 통해 얼만큼 성능 개선 효과가 나타나는지 테스트해 보자. 테스트를 위해 아래와 같이 emp 테이블을 1,000번 복제한 emp1과 emp2 두 개 테이블을 생성해 보자.
이제 이 두 테이블은 각각 14,000개 로우를 갖는다.
```
create table emp1 as
select * from emp, (select rownum no from dual connect by level <= 1000);

create table emp2 as select * from emp1;

alter table emp1 add constraint emp1_pk primary key(no, empno);

alter table emp2 add constraint emp1_pk primary key(no, empno);
```
아래는 emp1과 emp2 테이블을 조인하고 나서 서브쿼리 필터링을 수행할 때의 트레이스 결과다.
```
select /*+ leading(e1) use_nl(e2) */ sum(e1.sal), sum(e2.sal)
from   emp1 e1, emp2 e2
where  emp1.no = e2.no
and    emp1.empno = e2.empno
and    exists (select /*+ NO_UNNEST NO_PUSH_SUBQ */ 'x'
          from  dept where  deptno = e1.deptno
          and   loc = 'NEW YORK')

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.484        0.493       0     28103          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.484        0.493       0     28103          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=28103 pr=0 pw=0 time=493306 us)
   3000    FILTER (cr=28103 pr=0 pw=0 time=486253 us)
  14000      NESTED LOOPS (cr=28097 pr=0 pw=0 time=602032 us)
  14000        TABLE ACCESS FULL EMP1 (cr=28097 pr=0 pw=0 time=602032 us)
  14000        TABLE ACCESS BY INDEX ROWID EMP2 (cr=95 pr=0 pw=0 time=42023 us)
  14000          INDEX UNIQUE SCAN EMP2_PK (cr=14002 pr=0 pw=0 time=164606 us)
      1      TABLE ACCESS BY INDEX ROWID DEPT (cr=6 pr=0 pw=0 time=78 us)
      3        INDEX UNIQUE SCAN DEPT_PK (cr=3 pr=0 pw=0 time=36 us)
```
아래는 emp2 테이블과 조인하기 전에 서브쿼리 필터링을 먼저 수행할 때의 트레이스 결과다.
앞에서는 emp2 테이블과의 조인 시도 횟수가 14,000번이었지만 여기서는 서브쿼리를 필터링한 결과 건수가 3,000건이므로 emp2 테이블과의 조인 횟수도 3,000번으로 줄었다.
읽은 블록 수도 28,103개에서 6,103개로 줄었다.
```
select /*+ leading(e1) use_nl(e2) */ sum(e1.sal), sum(e2.sal)
from   emp1 e1, emp2 e2
where  emp1.no = e2.no
and    emp1.empno = e2.empno
and    exists (select /*+ NO_UNNEST PUSH_SUBQ */ 'x'
          from  dept where  deptno = e1.deptno
          and   loc = 'NEW YORK')

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.125        0.129       0      6103          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.125        0.129       0      6103          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=6103 pr=0 pw=0 time=493306 us)
   3000    NESTED LOOPS (cr=6103 pr=0 pw=0 time=486253 us)
   3000      TABLE ACCESS FULL EMP1 (cr=101 pr=0 pw=0 time=18230 us)
      1        TABLE ACCESS BY INDEX ROWID DEPT (cr=6 pr=0 pw=0 time=135 us)
      3          INDEX UNIQUE SCAN DEPT_PK (cr=3 pr=0 pw=0 time=63 us)
   3000      TABLE ACCESS BY INDEX ROWID EMP2 (cr=6002 pr=0 pw=0 time=100092 us)
   3000        INDEX UNIQUE SCAN EMP2_PK (cr=3002 pr=0 pw=0 time=41733 us)
```
서브쿼리가 조인으로 풀릴 때 서브쿼리에서 참조하는 테이블이 먼저 드라이빙되도록 제어할 목적으로 push_subq 힌트를 사용한다고 잘못 알고 있는 분들을 자주 만난다.
서브쿼리가 조인으로 풀린다는 것은 Unnesting 되었다는 뜻인데, 다시 말하지만 Pushing 서브쿼리는 Unnesting 되지 않은 서브쿼리의 처리 순서를 제어하는 기능이다.
Unnesting된 서브쿼리의 조인 순서를 조정하는 방법에 대해서는 앞에서 이미 살펴보았다.