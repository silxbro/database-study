# 05. Outer 조인

<br/>

## (1) Outer NL 조인
NL 조인은 그 특성상 Outer 조인할 때 방향이 한쪽으로 고정되며, Outer 기호(+)가 붙지 않은 테이블이 항상 드라이빙 테이블로 선택된다. leading 힌트를 이용해 순서를 바꿔 보려 해도 소용 없다.
```
select /*+ use_nl(d e) */
from   dept d, emp e
where  e.deptno(+) = d.deptno

Execution Plan
--------------------------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=14 Bytes=2K)
1   0    NESTED LOOPS (OUTER) (Cost=4 Card=14 Bytes=2K)
2   1      TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=4 Bytes=120)
3   1      TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=4 Bytes=348)
4   3        INDEX (RANGE SCAN) OF 'EMP_DEPTNO_INX' (INDEX) (Cost=0 Card=5)
```
NL 조인이 조인 순서에 가장 큰 영향을 받는다. 조인 순서 때문에 성능이 나빠지지 않게 하려면 불필요한 Outer 조인이 발생하지 않도록 주의해야 한다.

### [ERD 표기를 따르는 SQL 개발의 중요성]
데이터베이스 프로그래머에게는 ERD 표기법에 대한 정확한 이해가 필수적이다. ERD를 잘못 해석하면 설계 의도와 다른 데이터를 발생시킬 수 있기 때문이다.
SQL을 작성할 때는 각 속성의 Null 값 허용 여부를 반드시 확인해야 하고, 엔티티 간 관계(relationship)를 해석할 때도 카디널리티(crow`s foot, 1:1, 1:M, M:M 등)만 보지 말고
Optionality를 반드시 따져봐야 한다.

설계 의도를 잘못 해석하면 데이터 품질뿐만 아니라 쿼리 성능에도 영향을 미칠 수 있다.
예를 들어, 사원이 전혀 없는 유령 부서(또는 아직 부서원이 배정되지 않은 신규 부서)가 등록될 수 있다(부서 쪽 관계선이 점선).
따라서 사원 유무와 상관없이 모든 부서가 출력되도록 하려면 사원 쪽 모든 조건절에 Outer 기호(+)를 반드시 붙여 줘야 한다.

하지만 사원 없는 부서가 등록될 수 없는 경우(부서 쪽 관계선이 실선), 모든 부서가 출력되도록 하려고 굳이 Outer 조인할 필요가 없음에도 Outer 기호(+)를 붙인다면 성능이 나빠질 수 있는 것이다.

그리고 위 경우 모두 사원 쪽 부서번호가 필수(Mandatory) 컬럼이다. 즉, 소속 부서 없는 사원이 존재할 수 없다는 뜻이므로 테이블을 생성할 때 Not Null 제약을 두어야 한다.
DMBS에 실제 Not Null 제약을 생성했건 안 했건 그 컬럼에 Null 값이 들어올 수 없도록 하려는 것이 설계자의 의도이므로 프로그램도 그에 맞게 개발해야 한다.
- ERD 상에 표시된 설계자의 의도를 무시하고 테이블 생성 시 Not Null 제약을 두지 않는 경우를 종종 보는데, 결코 바람직한 현상이 아니다.
  데이터 품질이 중요하다는 것이 첫 번째 이유고, 옵티마이저에게 매우 중요한 정보로서 성능에도 영향을 준다는 것이 두 번째 이유다.

따라서 사원을 기준으로 부서 테이블과 Outer 조인하는 것은 불필요하며, 만약 Inner 조인을 했을 때 걸러지는 사원 레코드가 있다면 조회 프로그램이 아니라 입력/수정/삭제 프로그램을 고쳐야 할 일이다.
물론 데이터 정제(cleansing) 작업도 함께 실시해야 한다. 그런데도 혹시 있을지 모를 Null 값을 두려워해 습관적으로 Outer 기호(+)를 붙인다면 성능상 불이익이 생길 수 있음을 명심하기 바란다.

<br/>

## (2) Outer 소트 머지 조인
소트 머지 조인은 소트된 중간 집합을 이용한다는 점만 다를 뿐 처리 루틴이 NL 조인과 다르지 않다고 했다.
따라서 Outer 소트 머지 조인도 처리 방향이 한쪽으로 고정되며, Outer 기호(+)가 붙지 않은 테이블(Outer 테이블)이 항상 First 테이블로 선택된다.
leading 힌트를 이용해 순서를 바꿔 보려 해도 소용없다.
```
select /*+ use_merge(d e) */
from   dept d, emp e
where  e.deptno(+) = d.deptno

Execution Plan
--------------------------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=14 Bytes=2K)
1   0    MERGE JOIN (OUTER) (Cost=8 Card=14 Bytes=2K)
2   1      SORT (JOIN) (Cost=4 Card=4 Bytes=120)
3   2        TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=4 Bytes=120)
4   1      SORT (JOIN) (Cost=4 Card=14 Bytes=1K)
5   4        TABLE ACCESS (FULL) OF 'EMP' (TABLE) (Cost=3 Card=14 Bytes=120)
```

<br/>

## (3) Outer 해시 조인
Outer 해시 조인도 9i까지는 방향이 고정됐었다. 9i에서 Outer 해시 조인을 수행해 보면, Outer 기호(+)가 붙지 않은 테이블(Outer 테이블)이 항상 Build Input으로 선택된다.
```
select /*+ use_hash(d e) */
from   dept d, emp e
where  e.deptno(+) = d.deptno

Execution Plan
--------------------------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=14 Bytes=2K)
1   0    HASH JOIN (OUTER) (Cost=7 Card=14 Bytes=2K)
2   1      TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=4 Bytes=120)
3   1      TABLE ACCESS (FULL) OF 'EMP' (TABLE) (Cost=3 Card=14 Bytes=1K)
```
실행계획이 위와 같을 때 오라클은 아래와 같은 알고리즘을 사용해 Outer 해시 조인을 처리한다.
- [1] Outer 집합인 dept 테이블을 해시 테이블로 빌드(Build)한다.
- [2] Inner 집합인 emp 테이블을 읽으면서 해시 테이블을 탐색(Probe)한다.
- [3] 조인에 성공한 레코드는 곧바로 결과집합에 삽입하고, 조인에 성공했음을 해시 엔트리에 표시해 둔다.
- [4] Probe 단계가 끝나면 Inner 조인과 동일한 결과집합이 만들어진 상태다.
  이제 조인에 실패했던 레코드를 결과집합에 포함시켜야 하므로 해시 테이블을 스캔(PGA에서 스캔하므로 비용이 매우 낮음)하면서 체크가 없는 dept 엔트리를 결과집합에 삽입한다.
  그럼으로써 Outer 조인을 완성한다.

해시 조인은 특히 대용량 테이블을 조인할 때 자주 사용되는데, Outer 조인할 때 이처럼 조인 순서가 고정되다 보니 자주 성능 문제를 일으키곤 했다.
예를 들어, 주문 테이블을 기준으로 고객 테이블과 Outer 조인하는 경우에 대용량 주문 테이블을 해시 테이블로 빌드해야 하는 문제가 생긴다.

Hash Area가 부족해 디스크 쓰기와 읽기가 발생할 뿐만 아니라 주문 건수가 많은 고객일수록 해시 버킷 당 엔트리 개수가 많아져 해시 테이블을 탐색하는 효율이 크게 저하된다.
이 두 가지 요인에 의해 해시 조인 성능이 얼마나 나빠지는지 3절에서 충분히 설명하였다.

오라클은 이 문제를 해결하려고 10g에서 아래와 같은 Right Outer 해시 조인을 도입하게 되었다.
```
select /*+ use_hash(d e) swap_join_inputs(d) */ d.dname, e.ename
from   dept d, emp e
where  e.deptno = d.deptno(+)

Execution Plan
--------------------------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=9 Card=14 Bytes=280)
1   0    HASH JOIN (RIGHT OUTER) (Cost=9 Card=14 Bytes=280)
2   1      TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=4 Card=4 Bytes=44)
3   1      TABLE ACCESS (FULL) OF 'EMP' (TABLE) (Cost=4 Card=14 Bytes=126)
```
실행계획이 위와 같을 때 오라클은 아래와 같은 알고리즘을 사용해 Outer 해시 조인을 처리한다.

- [1] Inner 집합인 dept 테이블을 해시 테이블로 빌드(Build)한다.
- [2] Outer 집합인 emp 테이블을 읽으면서 해시 테이블을 탐색(Probe)한다.
- [3] Outer 조인이므로 조인 성공 여부에 상관없이 결과집합에 삽입한다.

이미 느꼈겠지만 Right Outer 해시 조인은 결국 Outer NL 조인과 같은 알고리즘을 사용한다.
Outer 해시 조인을 애초에 이런 방식으로 처리할 수 있었을 텐데 오라클은 왜 10g에 와서야 이 방식을 추가로 도입한 것일까?

### [Right Outer 해시 조인 탄생 배경]
일반적으로 Outer 조인은 1쪽 집합을 기준으로 하는 경우가 많다. 예를 들면, 고객을 기준으로 주문과 Outer 조인하거나 상품을 기준으로 주문과 Outer 조인하는 경우다.
모델상에서 보면 주문 기준으로 고객 또는 상품과 Outer 조인할 이유는 없다. '고객 없는 주문' 또는 '상품 없는 주문'을 허용하지 않도록 설계 돼 있기 때문이다.

오라클은 이런 사실을 감안해 Outer 테이블을 해시 테이블로 빌드(Build)하는 알고리즘을 애초에 선택하였다.
Inner 조인하고 나서 해시 테이블을 전체적으로 한 번 더 스캔하는 비효율을 감수하면서까지 말이다. 이유는, 작은 쪽 집합을 해시 테이블로 빌드하는 게 유리하기 때문이다.

그런데 일반적인 엔티티 관계(relationship) 속에서도 주문을 기준으로 고객 또는 상품과 Outer 조인해야 할 필요성이 생긴다.
모델 상으로는 부모 없는 자식 레코드가 있어선 안 되지만 FK를 설정하지 않은 채 DBMS를 운영하는 경우가 많다 보니 실제 그런 레코드들이 많이 생긴다.

프로그램 버그일 수도 있고, 활동성 없는 고객이나 오래된 상품 레코드를 지우면서 관련된 자식 레코드(주문 등)는 지우지 않아 생기는 현상이다.
데이터 정제(Cleansing) 작업할 때 M쪽 자식 테이블을 기준으로 Outer 조인하는 쿼리가 자주 사용되는 이유다.

그럴 때 주문 같은 초대용량 테이블을 해시 테이블로 생성해야 하기 때문에 성능이 여간 나쁘지 않은데, 정제 작업이 많은 데이터 이행(Migration) 업무를 해 본 독자라면 깊이 공감할 것이다.

'고객 없는 주문', '상품 없는 주문'이 발견됐을 때 이를 정제하지 않고 데이터를 그대로 이행하는 경우도 많다.
중요한 주문 데이터를 지우면 그와 관련된 각종 집계 값들이 안 맞을 수 있어 업무 담당자 입장에서도 쉽게 결정을 내리지 못하기 때문이다.
그러다 보니 주문을 기준으로 고객과 상품 테이블을 Outer 조인한 결과를 신시스템에 그대로(정합성이 안 맞더라도) 이행하는 경우가 생기고, 그런 작업을 담당한 개발자는 성능 때문에 곤혹스런 상황에 부닥치게 된다.

이런 성능 이슈를 해결하려고 오라클은 10g부터 Inner쪽 집합을 해시 테이블로 빌드할 수 있는 알고리즘을 추가하게 되었다.

### [9i 이전 버전에서 Outer 해시 조인 튜닝]
그럼 Right Outer 해시 조인이 도입되기 전 9i까지는 위와 같은 상황에서 어떻게 튜닝할 수 있을까?
```
select /*+ ordered index_ffs(o) full(c) full(o2) use_hash(o c) use_hash(o2)
           parallel_index(o) parallel(c) parallel(o2) */ c.*, o2.*
from   주문 o, 고객 c, 주문 o2
where  c.고객번호(+) = o.고객번호
and    o2.고객번호 = o.고객번호
and    o2.상품번호 = o.상품번호
and    o2.주문일시 = o.주문일시
```
위 쿼리를 보면, 처음 주문(o)과 고객(c) 테이블을 Outer 조인할 때는 주문 테이블에서 PK 인덱스만 빠르게 읽어 Outer 조인을 완성하고, 주문(o2) 테이블과 다시 한 번 조인하면서는 Inner 조인하도록
했다. 주문이 워낙 대용량이어서 인덱스 블록만 읽더라도 In-Memory 해시 조인은 불가능하겠지만 Build Input 크기를 줄임으로써 디스크 쓰기 및 읽기 작업을 최소화하려는 아이디어다.

하지만 이 방법은 디스크 쓰기와 읽기 작업을 줄여주는 효과는 있지만 해시 버킷 당 엔트리 개수가 많아서 생기는 문제는 피할 수가 없다.

만약 이 때문에 조인 성능이 느리다면 구간을 나눠 쿼리를 여러 번 수행하는 방법을 생각해 볼 수 있다.
주문 테이블은 주문일시로 Range 파티셔닝 돼 있을 것이고, 일정한 주문일시 구간 내에서의 고객별 주문 건수는 많지 않을 것이기 때문에 해시 버킷 당 엔트리 개수를 최소화할 수 있다.
그러면 고객 테이블을 반복적으로 읽는 비효율에도 불구하고 더 빠르게 수행될 것이다.

<br/>

## (4) Full Outer 조인
테스트를 위해 우선 '입금'과 '출금' 테이블을 아래와 같이 생성해 보자.
```
SQL> exec dbms_random.seed(150);

SQL> create table 입금
  2  as
  3  select rownum 일련번호
  4       , round(dbms_random.value(1, 20)) 고객ID
  5       , round(dbms_random.value(1000, 100000), -2) 입금액
  6  from  dual connect by level <= 10;

SQL> create table 출금
  2  as
  3  select rownum 일련번호
  4       , round(dbms_random.value(1, 20)) 고객ID
  5       , round(dbms_random.value(1000, 100000), -2) 출금액
  6  from  dual connect by level <= 10;

SQL> exec dbms_stats.gather_table_stats(user, '입금');
SQL> exec dbms_stats.gather_table_stats(user, '출금');
```

### ['Left Outer 조인 + Union All + Anti 조인(Not Exists 필터)' 이용]
입금과 출금, 두 테이블을 Full Outer 조인해 고객별 입금액과 출금액을 같이 집계하려고 한다. 이를 위해 가장 일반적으로 사용할 수 있는 방법은 다음과 같다.
```
SQL> set autotrace on emp
SQL> select a.고객ID, a.입금액, b.출금액
  2  from   (select 고객ID, sum(입금액) 입금액 from 입금 group by 고객ID) a
  3       , (select 고객ID, sum(출금액) 출금액 from 출금 group by 고객ID) b
  4  where   b.고객ID(+) = a.고객ID
  5  union all
  6  select 고객ID, null, 출금액
  7  from   (select 고객ID, sum(출금액) 출금액 from 출금 group by 고객ID) a
  8  where  not exists (select 'x' from 입금 where 고객ID = a.고객ID);

고객ID      입금액      출금액
---------- --------- ----------
        13      6800      14500
         2     23900       6200
         3     26600       2300
        19     40400       6900
         8     95700      
         1     23100      
         6     71000      
        18     34300      
         4    121900      
        17                 9200
         7                 3900
        16                 7500
         9                 1300

13 개의 행이 선택되었습니다.

------------------------------------------------------------------------------------
| Id  | Operation                  | Name | Rows  | Bytes | Cost  (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |      |    10 |   477 |    20   (60)| 00:00:01 |
|   1 |   UNION-ALL                |      |       |       |             |          |
|   2 |     HASH JOIN OUTER        |      |     9 |   468 |    11   (28)| 00:00:01 |
|   3 |       VIEW                 |      |     9 |   234 |     5   (20)| 00:00:01 |
|   4 |         HASH GROUP BY      |      |     9 |    63 |     5   (20)| 00:00:01 |
|   5 |           TABLE ACCESS FULL| 입금  |    10 |    70 |     4    (0)| 00:00:01 |
|   6 |       VIEW                 |      |     9 |   208 |     5   (20)| 00:00:01 |
|   7 |         HASH GROUP BY      |      |     9 |    48 |     5   (20)| 00:00:01 |
|   8 |           TABLE ACCESS FULL| 출금  |    10 |    60 |     4    (0)| 00:00:01 |
|   9 |     HASH GROUP BY          |      |     1 |     9 |    10   (20)| 00:00:01 |
|  10 |       HASH JOIN ANTI       |      |     1 |     9 |     9   (12)| 00:00:01 |
|  11 |         TABLE ACCESS FULL  | 출금  |    10 |    60 |     4    (0)| 00:00:01 |
|  12 |         TABLE ACCESS FULL  | 입금  |    10 |    30 |     4    (0)| 00:00:01 |
------------------------------------------------------------------------------------
```

### [ANSI Full Outer 조인]
위와 같이 쿼리를 복잡하게 작성하지 않고도 Full Outer 조인할 수 있도록 오라클 9i부터 아래와 같이 ANSI 구문을 지원하기 시작했다.
```
SQL> select nvl(a.고객ID, b.고객ID) 고객ID, a.입금액, b.출금액
  2  from   (select 고객ID, sum(입금액) 입금액 from 입금 group by 고객ID) a
  3          full outer join
  4         (select 고객ID, sum(출금액) 출금액 from 출금 group by 고객ID) b
  5      on a.고객ID = b.고객ID;

고객ID      입금액      출금액
---------- --------- ----------
        13      6800      14500
         2     23900       6200
         3     26600       2300
        19     40400       6900
         8     95700      
         1     23100      
         6     71000      
        18     34300      
         4    121900      
        17                 9200
         7                 3900
        16                 7500
         9                 1300

13 개의 행이 선택되었습니다.

------------------------------------------------------------------------------------
| Id  | Operation                      | Name | Rows  | Bytes | Cost  (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |      |    17 |   884 |    36   (12)| 00:00:01 |
|   1 |   VIEW                         |      |    17 |   884 |    36   (12)| 00:00:01 |
|   2 |     UNION-ALL                  |      |       |       |             |          |
|   3 |       HASH JOIN OUTER          |      |     9 |   612 |    11   (28)| 00:00:01 |
|   4 |         VIEW                   |      |     9 |   342 |     5   (20)| 00:00:01 |
|   5 |           HASH GROUP BY        |      |     9 |    63 |     5   (20)| 00:00:01 |
|   6 |             TABLE ACCESS FULL  | 입금  |    10 |    70 |     4    (0)| 00:00:01 |
|   7 |         VIEW                   |      |     8 |   240 |     5   (20)| 00:00:01 |
|   8 |           HASH GROUP BY        |      |     8 |    48 |     5   (20)| 00:00:01 |
|   9 |             TABLE ACCESS FULL  | 출금  |    10 |    60 |     4    (0)| 00:00:01 |
|  10 |       HASH GROUP BY            |      |     8 |    48 |    25    (4)| 00:00:01 |
|  11 |         FILTER                 |      |       |       |             |          |
|  12 |           TABLE ACCESS FULL    | 출금  |    10 |    60 |     4    (0)| 00:00:01 |
|  13 |           SORT GROUP BY NOSORT |      |     1 |     3 |     4    (0)| 00:00:01 |
|  14 |             TABLE ACCESS FULL  | 입금  |     1 |     3 |     4    (0)| 00:00:01 |
------------------------------------------------------------------------------------
```
그런데 위 실행계획을 보면 내부적으로는 'Left Outer 조인 + Union All + Anti 조인(Not Exists)' 방식을 그대로 사용하고 있다.
쿼리가 간단해졌을 뿐 입금과 출금 테이블을 각각 두 번씩 액세스하는 비효율은 그대로 있다.

### [Native Hash Full Outer 조인]
이에 오라클 11g에서 'Native Hash Full Outer 조인'을 선보였고, 필요하면 10.2.0.4 버전에서도 Hidden 파라미터 _optimizer_native_full_outer_join를 조정해 이 기능을 사용할 수 있다.
```
SQL> select /*+ opt_param('_optimizer_full_outer_join', 'force') */
  2         nvl(a.고객ID, b.고객ID) 고객ID, a.입금액, b.출금액
  2  from   (select 고객ID, sum(입금액) 입금액 from 입금 group by 고객ID) a
  3          full outer join
  4         (select 고객ID, sum(출금액) 출금액 from 출금 group by 고객ID) b
  5      on a.고객ID = b.고객ID;

고객ID      입금액      출금액
---------- --------- ----------
         1     23100
         6     71000      
        13      6800      14500
         2     23900       6200
         4    121900
         8     95700        
         3     26600       2300
        18     34300      
        19     40400       6900
        17                 9200
         7                 3900
        16                 7500
         9                 1300

13 개의 행이 선택되었습니다.

------------------------------------------------------------------------------------------
| Id  | Operation                    | Name     | Rows  | Bytes | Cost  (%CPU)| Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |          |     9 |   468 |     9   (34)| 00:00:01 |
|   1 |   VIEW                       | V_FOJ_0  |     9 |   468 |     9   (34)| 00:00:01 |
|   2 |     HASH JOIN FULL OUTER     |          |     9 |   468 |     9   (34)| 00:00:01 |
|   3 |       VIEW                   |          |     8 |   208 |     4   (25)| 00:00:01 |
|   4 |         HASH GROUP BY        |          |     8 |    48 |     4   (25)| 00:00:01 |
|   5 |           TABLE ACCESS FULL  | 출금      |    10 |    60 |     3    (0)| 00:00:01 |
|   6 |       VIEW                   |          |     9 |   234 |     4   (25)| 00:00:01 |
|   7 |         HASH GROUP BY        |          |     9 |    63 |     4   (25)| 00:00:01 |
|   8 |           TABLE ACCESS FULL  | 입금      |    10 |    70 |     3    (0)| 00:00:01 |
------------------------------------------------------------------------------------------
```
실행계획을 보면, 입금과 출금 테이블을 각각 한 번씩만 액세스한다는 것이 가장 큰 변화다. 입금액이 null인 레코드가 마지막에 출력된 것을 통해, 내부적으로 어떤 식으로 처리하는지 추정할 수 있다.

- [1] 출금 테이블을 해시 테이블로 빌드(Build)한다.
- [2] 입금 테이블로 해시 테이블을 탐색(Probe)한다.
- [3] 조인 성공 여부에 상관없이 결과집합에 삽입하고, 조인에 성공한 출금 레코드에는 체크 표시를 해 둔다.
- [4] Probe 단계가 끝나면 Right Outer 해시 조인한 것과 동일한 결과집합이 만들어진다.
  이제 해시 테이블을 스캔하면서 체크 표시가 없는 출금 레코드를 결과집합에 삽입함으로써 Full Outer 조인을 완성한다. 입금액이 null인 레코드가 마지막에 출력된 이유가 바로 이것이다.

### [Union All을 이용한 Full Outer 조인]
아래와 같이 union all을 이용하면 버전에 상관없이 Full Outer 조인된 결과집합을 얻을 수 있다. 두 테이블을 각각 한 번씩만 액세스하였으며, 조인 대신 sort(또는 hash) group by 연산을 수행한다.
```
SQL> select 고객ID, sum(입금액) 입금액, sum(출금액) 출금액
  2  from (
  3    select 고객ID, 입금액, to_number(null) 출금액
  4    from   입금
  5    union all
  6    select 고객ID, to_number(null) 입금액, 출금액
  7    from   출금
  8  )
  9  group by 고객ID ;

고객ID      입금액      출금액
---------- --------- ----------
         1     23100
         6     71000      
        13      6800      14500
         2     23900       6200
         4    121900
         8     95700
        17                 9200    
         3     26600       2300
        18     34300
         7                 3900  
        19     40400       6900
         9                 1300         
        16                 7500
         
13 개의 행이 선택되었습니다.

----------------------------------------------------------------------------------
| Id  | Operation                | Name | Rows  | Bytes | Cost  (%CPU)| Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT         |      |    20 |   780 |     9   (12)| 00:00:01 |
|   1 |   HASH GROUP BY          |      |    20 |   780 |     9   (12)| 00:00:01 |
|   2 |     VIEW                 |      |    20 |   780 |     8    (0)| 00:00:01 |
|   3 |       UNION-ALL          |      |       |       |             |          |
|   4 |         TABLE ACCESS FULL| 입금  |    10 |    70 |     4    (0)| 00:00:01 |
|   5 |         TABLE ACCESS FULL| 출금  |    10 |    60 |     4    (0)| 00:00:01 |
----------------------------------------------------------------------------------
```