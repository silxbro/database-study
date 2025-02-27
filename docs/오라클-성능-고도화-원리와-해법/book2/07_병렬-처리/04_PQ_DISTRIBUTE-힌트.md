# 04. PQ_DISTRIBUTE 힌트

4가지 병렬 조인 방식의 동작 원리와 특징을 살펴보았고, 이제 이들을 제어하기 위해 사용되는 pq_distribute 힌트를 소개하려고 한다.
<br/>
<br/>
## (1) pq_distribute 힌트의 용도
조인되는 양쪽 테이블의 파티션 구성, 데이터 크기 등에 따라 병렬 조인을 수행하는 옵티마이저의 선택이 달라질 수 있다. 대개 옵티마이저의 선택이 최적이라고 할 수 있지만 가끔 그렇지 못한 경우가 있다.
그럴 때 pq_distribute 힌트를 사용함으로써 옵티마이저의 선택을 무시하고 사용자가 직접 조인을 위한 데이터 분배 방식을 결정할 수 있다.

- 옵티마이저가 파티션된 테이블을 적절히 활용하지 못하고 동적 재분할을 시도할 때
- 기존 파티션 키를 무시하고 다른 키 값으로 동적 재분할하고 싶을 때
- 통계정보가 부정확하거나 통계정보를 제공하기 어려운 상황(→ 옵티마이저가 잘못된 판단을 하기 쉬운 상황)에서 실행계획을 고정시키고자 할 때
- 기타 여러 가지 이유로 데이터 분배 방식을 변경하고자 할 때

병렬 방식으로 조인을 수행하기 위해서는 프로세스들이 서로 "독립적으로" 작업할 수 있도록 사전 준비작업이 필요하다. 먼저 데이터를 적절히 배분하는 작업이 선행되어야 한다.

병렬 쿼리는 '분할 & 정복(Divide & Conquer) 원리'에 기초한다. 그 중에서도 병렬 조인을 위해서는 '분배 & 조인(Distribute & Join 원리'가 작동함을 이해하는 것이 매우 중요하다.
이때, pq_distribute 힌트는 조인에 앞서 데이터를 분배(distribute)하는 과정에만 관여하는 힌트임을 반드시 기억할 필요가 있다.

예를 들어, 아래 실행계획을 보면 테이블은 양쪽 모두 Hash 방식으로 분배했지만 조인은 소트 머지 조인 방식으로 수행하였다.
즉, 데이터를 재분배하기 위해 해시 함수를 사용하는 것일 뿐 조인 방식(method)과는 무관하다는 것이다.

```
select /*+ ordered use_merge(e) parallel(d 4) parallel(e 4)
           pq_distribute(e hash hash) */ *
from   dept d, emp e
where  e.deptno = d.deptno

---------------------------------------------------------------------------------
| Id  | Operation                        | Name     |   TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |          |       |      |            |
|   1 |   PX COORDINATOR                 |          |       |      |            |
|   2 |     PX SEND QC (RANDOM)          | :TQ10002 | Q1,02 | P->S | QC (RAND)  |
|   3 |       MERGE JOIN                 |          | Q1,02 | PCWP |            |
|   4 |         SORT JOIN                |          | Q1,02 | PCWP |            |
|   5 |           PX RECEIVE             |          | Q1,02 | PCWP |            |
|   6 |             PX SEND HASH         | :TQ10000 | Q1,00 | P->P | HASH       |
|   7 |               PX BLOCK ITERATOR  |          | Q1,00 | PCWC |            |
|   8 |                 TABLE ACCESS FULL| DEPT     | Q1,00 | PCWP |            |
|   9 |         SORT JOIN                |          | Q1,02 | PCWP |            |
|  10 |           PX RECEIVE             |          | Q1,02 | PCWP |            |
|  11 |             PX SEND HASH         | :TQ10001 | Q1,01 | P->P | HASH       |
|  12 |               PX BLOCK ITERATOR  |          | Q1,01 | PCWC |            |
|  13 |                 TABLE ACCESS FULL| EMP      | Q1,01 | PCWP |            |
---------------------------------------------------------------------------------
```

다시 말하지만, pq_distribute 힌트는 병렬 조인에 앞선 사전 정지 작업으로서 데이터를 어떻게 분배할지를 결정하는 힌트지, 조인 방식을 결정하는 힌트가 아니다.
<br/>
<br/>
## (2) 구문 이해하기
pq_distribute 힌트의 사용법은 다음과 같다.

> /*+ **PQ_DISTRIBUTE**( table, outer_distribution, inner_distribution) */
>
> - table: **inner** 테이블명 또는 alias
> - outer_distribution: outer 테이블의 distribution 방식
> - inner_distribution: inner 테이블의 distribution 방식

많은 개발자 또는 튜너들이 첫 번째 인자의 의미를 이해하는 데에 어려움을 느끼는 것 같은데, use_nl 힌트가 pq_distribute 힌트를 이해하는 데에 도움이 되므로 잠시 살펴보자.
use_nl 힌트 사용법은 아래와 같다.
```
select /*+ ordered use_nl(B) use_nl(C) use_hash(D) * / *
from   A, B, C, D
where  ...

"A → B → C → D 순으로 조인하되, B, C와 조인할 때는 Nested Loop 방식으로 조인하고, D와 조인할 때는 해시 방식으로 조인하라는 뜻이다."
```

pq_distribute 힌트도 마찬가지다. ordered 또는 leading 힌트에 의해 먼저 처리되는 outer 테이블을 기준으로 그 집합과 조인되는 inner 테이블을 첫 번째 인자로 지정하면 된다.
그리고 그 조인 과정에서의 outer, inner 테이블에 대한 분배방식을 각각 두 번째, 세 번째 인자로 지정하는 것이다.
조인 순서를 먼저 고정시키는 것이 중요하므로 ordered 또는 leading 힌트를 같이 사용하는 것이 올바른 사용법이라고 할 수 있다.
```
SQL> SELECT /*+ ordered
  2             use_hash(b) use_nl(c) use_merge(d)
  3             full(a) full(b) full(c) full(d)
  4             parallel(a, 16) parallel(b, 16)  parallel(c, 16) parallel(d, 16)
  5             pq_distribute(b, none, partition)
  6             pq_distribute(c, none, broadcast)
  7             pq_distribute(d, hash, hash) */ .....
  8  FROM   상품기본이력임시 a, 상품 b, 코드상세 c, 상품상세 d
  9  WHERE  a.상품번호 = b.상품번호
 10  AND    .....
```

위 병렬 조인문에 기술된 힌트를 풀어서 설명하면 아래와 같다. 숫자는 라인번호를 뜻한다.

- [1] FROM절에 나열된 순서대로 조인하라.
- [2] b(상품) 테이블과는 해시 조인, c(코드상셰) 테이블과는 NL 조인, d(상품상세) 테이블과는 소트 머지 조인을 하라.
- [3] a, b, c, d 네 테이블을 Full Scan 하라.
- [4] a, b, c, d 네 테이블을 병렬로 처리하라.
- [5] 상품(b) 테이블과 조인할 때, inner 테이블(=상품)을 outer 테이블(=상품기본이력임시)에 맞춰 파티셔닝하라.
- [6] 코드상세(c) 테이블과 조인할 때, inner 테이블(=코드상세)을 Broadcase하라.
- [7] 상품상세(d) 테이블과 조인할 때, 양쪽 모두를 Hash 방식으로 동적 파티셔닝하라.
  <br/>

## (3) 분배방식 지정
inner 테이블 지정하는 방법을 알았고, 이제 두 번째와 세 번째 인자를 통해 분배방식을 어떻게 지정하는지 간단히 살펴보자.

### [pq_distribute(inner, none, none)]
Full-Partition Wise 조인으로 유도할 때 사용한다. 당연히, 양쪽 테이블 모두 조인 컬럼에 대해 같은 기준으로 파티셔닝(equi-partitioning) 돼 있을 때만 작동한다.

### [pq_distribute(inner, partition, none)]
Partial-Partition Wise 조인으로 유도할 때 사용하며, outer 테이블을 inner 테이블 파티션 기준에 따라 파티셔닝하라는 뜻이다.
당연히, inner 테이블이 조인 키 컬럼에 대해 파티셔닝 돼 있을 때만 동작한다.

### [pq_distribute(inner, none, partition)]
Partial-Partition Wise 조인으로 유도할 때 사용하며, inner 테이블을 outer 테이블 파티션 기준에 따라 파티셔닝하라는 뜻이다.
당연히, outer 테이블이 조인 키 컬럼에 대해 파티셔닝 돼 있을 때만 작동한다.

### [pq_distribute(inner, hash, hash)]
조인 키 컬럼을 해시 함수에 적용하고 거기서 반환된 값을 기준으로 양쪽 테이블을 동적으로 파티셔닝하라는 뜻이다.

### [pq_distribute(inner, broadcast, none)]
outer 테이블을 Broadcast 하라는 뜻이다.

### [pq_distribute(inner, none, broadcast)]
inner 테이블을 Broadcast 하라는 뜻이다.
<br/>
<br/>
## (4) pq_distribute 힌트를 이용한 튜닝 사례
통계 정보가 없는 상태에서 병렬 조인하면 옵티마이저가 아주 큰 테이블을 Broadcast하는 경우를 종종 보게 된다.
임시 테이블을 많이 사용하는 야간 배치나 데이터 이행(Migration) 프로그램에서 그런 문제가 자주 발생하는 이유가 여기에 있다.

아래는 데이터 이행 도중 실제 문제가 발생했던 사례다. 실행계획은 9i의 것이다.
```
SQL> INSERT /*+ APPEND */ INTO 상품기본이력 ( ... )
  2> SELECT /*+ PARALLEL(A, 32) PARALLEL(B, 32) PARALLEL(C, 32), PARALLEL(D, 32) */ ......
  3> FROM   상품기본이력임시 a, 상품 b, 코드상세 c, 상품상세 d
  4> WHERE  a.상품번호 = b.상품번호
  5> AND    ...
  6> /

INSERT /*+ append */ INTO 상품기본이력 (
*
1행에 오류:
ORA-12801: 병렬 질의 서버 P013에 오류신호가 발생했습니다
ORA-01652: 256(으)로 테이블 공간 TEMP에서 임시 세그먼트를 확장할 수 없습니다

경  과: 01:39:56.08

--------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name         | Rows  | Pstart| Pstop |IN-OUT| PQ Distrib |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |              |  5248 |       |       |      |            |
|   1 |   LOAD AS SELECT              |              |       |       |       |      |            |
|   2 |     HASH JOIN                 |              |  5248 |       |       | P->S | QC (RAND)  |
|   3 |       HASH JOIN OUTER         |              |  5248 |       |       | P->P | BROADCAST  |
|   4 |         HASH JOIN             |              |  5248 |       |       | PCWP |            |
|   5 |           PARTITION HASH ALL  |              |       |     1 |   128 | PCWP |            |
|   6 |             TABLE ACCESS FULL | 상품기본이력임시 |  5248 |     1 |   128 | P->P | BROADCAST  |
|   7 |           TABLE ACCESS FULL   | 상품          |  7595K|       |       | PCWP |            |
|   8 |         TABLE ACCESS FULL     | 코드상세       |    26 |       |       | P->P | BROADCAST  |
|   9 |       TABLE ACCESS FULL       | 상품상세       |  7595K|       |       | PCWP |            |
--------------------------------------------------------------------------------------------------
```

1시간 40분간 수행되던 SQL이 임시 세그먼트를 확장할 수 없다는 오류 메시지를 던지면서 멈춰 버렸고, 분석해 보니 상품기본이력임시 테이블에 통계 정보가 없던 것이 원인이었다.
실제 천만 건에 이르는 큰 테이블이었는데, 통계 정보가 없어 옵티미ㅏ이저가 5,248건의 작은 테이블로 판단한 것을 볼 수 있다.

이 큰 테이블을 32개 병렬 서버에게 Broadcast하는 동안 과도한 프로세스 간 통신이 발생했고, 결국 Temp 테이블스페이스를 모두 소진하고서 멈췄다.

pq_distribute 힌트를 이용해 데이터 분배 방식을 조정하고 나서 다시 수행해 본 결과, 아래와 같이 2분 29초 만에 작업을 완료하였다.
```
SQL> INSERT /*+ APPEND */ INTO 상품기본이력 ( ... )
  2> SELECT /*+ ORDERED PARALLEL(A, 16) PARALLEL(B, 16) PARALLEL(C, 16), PARALLEL(D, 16)
  3>            PQ_DISTRIBUTE(B, NONE, PARTITION)
  4>            PQ_DISTRIBUTE(C, NONE, BROADCAST)
  5>            PQ_DISTRIBUTE(D, HASH, HASH) */ .....
  6> FROM   상품기본이력임시 a, 상품 b, 코드상세 c, 상품상세 d
  7> WHERE  a.상품번호 = b.상품번호
  8> AND    ...
  9> /

876902 개의 행이 만들어졌습니다.

경  과: 00:02:29.00

--------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name         | Rows  | Pstart| Pstop |IN-OUT| PQ Distrib |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |              |  5248 |       |       |      |            |
|   1 |   LOAD AS SELECT              |              |       |       |       |      |            |
|   2 |     HASH JOIN                 |              |  5248 |       |       | P->S | QC (RAND)  |
|   3 |       HASH JOIN OUTER         |              |  5248 |       |       | P->P | HASH       |
|   4 |         HASH JOIN             |              |  5248 |       |       | PCWP |            |
|   5 |           PARTITION HASH ALL  |              |       |     1 |   128 | PCWP |            |
|   6 |             TABLE ACCESS FULL | 상품기본이력임시 |  5248 |     1 |   128 | PCWP |            |
|   7 |           TABLE ACCESS FULL   | 상품          |  7595K|       |       | P->P | PARK (KEY) |
|   8 |         TABLE ACCESS FULL     | 코드상세       |    26 |       |       | P->P | BROADCAST  |
|   9 |       TABLE ACCESS FULL       | 상품상세       |  7595K|       |       | P->P | HASH       |
--------------------------------------------------------------------------------------------------
```

10g부터는 통계정보가 없을 때 동적 샘플링이 일어나므로 그럴 가능성이 매우 낮아졌다. 하지만 테이블 간 조인을 여러 번 거치면 옵티마이저가 예상한 조인 카디널리티가 점점 부정확해지게 마련이다.
예를 들어, a → b → c → d → e 순으로 조인을 진행하는데, a, b, c, d를 조인하고 난 시점에도 결과 건수가 여전히 많을 수 있다.
그렇다면 해시/해시 분배 방식이 효과적인데 옵티마이저가 조인 카디널리티를 계산할 때는 그 시점의 결과 건수가 매우 적은 것으로 판단해 Broadcast 방식을 선택할 수도 있는 것이다.

데이터 분포가 고르지 않은 컬럼이 조건절에 많이 사용되거나, 다른 테이블과 조인되기 전 인라인 뷰 내에서 많은 가공이 이루어져 정확한 카디널리티 계산이 어려울 때 이런 오류 발생 가능성은 더욱 커진다.