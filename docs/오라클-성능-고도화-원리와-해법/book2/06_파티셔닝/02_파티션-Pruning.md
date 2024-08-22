# 02. 파티션 Pruning

'prune'은 '쓸데없는 가지를 치다', '불필요한 부분을 제거한다'는 뜻이다.
용어에서 알 수 있듯이, 파티션 Pruning(=Elimination)은 하드파싱이나 실행 시점에 SQL 조건절을 분석하여 읽지 않아도 되는 파티션 세그먼트를 액세스 대상에서 제외시키는 기능이다.
파티션 테이블에 대한 쿼리나 DML을 수행할 때 극적인 성능 개선을 가져다주는 핵심 원리가 파티션 Pruning에 있다고 할 수 있다.

기본 파티션 Pruning 기법을 먼저 설명하고, 고급 파티션 Pruning 기법으로서 서브쿼리 Pruning과 조인 필터 Pruning을 이어서 설명한다.

<br/>

## (1) 기본 파티션 Pruning
기본 파티션 Pruning 기법으로는 아래 두 가지가 있다.

- **정적(Static) 파티션 Pruning** : 파티션 키 컬럼을 상수 조건으로 조회하는 경우에 작동하며, 액세스할 파티션이 쿼리 최적화 시점에 미리 결정되는 것이 특징이다.
  실행계획의 Pstart(partition start)와 Pstop(partition stop) 컬럼에는 액세스할 파티션 번호가 출력된다.

- **동적(Dynamic) 파티션 Pruning** : 파티션 키 컬럼을 바인드 변수로 조회하면 쿼리 최적화 시점에는 액세스할 파티션을 미리 결정할 수 없다.
  실행 시점이 돼서야 사용자가 입력한 값에 따라 결정되며, 실행계획의 Pstart와 Pstop 컬럼에는 'KEY'라고 표시된다.
  NL 조인할 때도 Inner 테이블이 조인 컬럼 기준으로 파티셔닝 돼 있다면 동적 Pruning이 작동한다.

### [파티션 Pruning 기능에 따른 실행계획 비교]
SQL과 실행계획을 보면서 파티션 Pruning이 실제 어떻게 작동하는지 확인해 보자.
```
create table t ( key, no, data )
partition by range(no) (
  partition p01 values less than(11)
, partition p02 values less than(21)
, partition p03 values less than(31)
, partition p04 values less than(41)
, partition p05 values less than(51)
, partition p06 values less than(61)
, partition p07 values less than(71)
, partition p08 values less than(81)
, partition p09 values less than(91)
, partition p10 values less than(maxvalue)
)
as
select lpad(rownum, 6, '0'), mod(rownum, 100)+1, lpad(rownum, 10, '0')
from   dual
connect by level <= 999999
```
먼저 no 컬럼 기준으로 Range 파티션 테이블을 생성했다. 아래는 상수 조건을 사용해 정적 Pruning을 테스트한 것으로서, 10개 중 3개(3~5번) 파티션만 읽는 것을 알 수 있다.
```
select count(*) from t where no between 30 and 50

----------------------------------------------------------------------------
| Id  | Operation                   | Name | Rows  | Bytes | Pstart| Pstop |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |      |     1 |    13 |       |       |
|   1 |   SORT AGGREGATE            |      |     1 |    13 |       |       |
|   2 |     PARTITION RANGE ITERATOR|      |   143K|  1822K|     3 |     5 |
|   3 |       TABLE ACCESS FULL     | T    |   143K|  1822K|     3 |     5 |
----------------------------------------------------------------------------
```
이번에는 바인드 변수로 동적 Pruning을 테스트해 보자.
```
select count(*) from t where no between :a and :b

------------------------------------------------------------------------------
| Id  | Operation                     | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |      |     1 |    13 |       |       |
|   1 |   SORT AGGREGATE              |      |     1 |    13 |       |       |
|   2 |     FILTER                    |      |       |       |       |       |
|   3 |       PARTITION RANGE ITERATOR|      |  2310 | 30030 |   KEY |   KEY |
|   4 |         TABLE ACCESS FULL     | T    |  2310 | 30030 |   KEY |   KEY |
------------------------------------------------------------------------------
```
위 실행계획에서 시작과 종료 파티션 번호 대신 'KEY'라고 표시된 것은 하드파싱 시점에 액세스할 파티션을 결정할 수 없기 때문이다.

파티션 컬럼에 IN-List 조건을 사용하면 상수 값이더라도 아래처럼 'KEY(I)'라고 표시된다.
```
select count(*) from t where no in (30, 50)

------------------------------------------------------------------------------
| Id  | Operation                 | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT          |      |     1 |    13 |       |       |
|   1 |   SORT AGGREGATE          |      |     1 |    13 |       |       |
|   2 |     PARTITION RANGE INLIST|      | 23671 |   300K|KEY(I) |KEY(I) |
|   3 |       TABLE ACCESS FULL   | T    | 23671 |   300K|KEY(I) |KEY(I) |
------------------------------------------------------------------------------
```
이번에는 NL 조인을 테스트하기 위해 아래와 같이 n 테이블을 만들어 보자.
```
create table n
as
select level no from dual connect by level <= 100;
```
아래 실행계획에서 알 수 있듯이 NL 조인에서도 n 테이블로부터 읽히는 값에 따라 t 테이블에 동적 Pruning이 일어난다.
```
select /*+ leading(n) use_nl(t) */ *
from   n, t
where  t.no = n.no

------------------------------------------------------------------------------
| Id  | Operation                   | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |      |  1097K|    48M|       |       |
|   1 |   NESTED LOOPS              |      |  1097K|    48M|       |       |
|   2 |     TABLE ACCESS FULL       | N    |   100 |  1300 |       |       |
|   3 |     PARTITION RANGE ITERATOR|      | 10975 |   353K|   KEY |   KEY |
|   4 |       TABLE ACCESS FULL     | T    | 10975 |   353K|   KEY |   KEY |
------------------------------------------------------------------------------
```
결합 파티션일 때는 어떤 식으로 Pruning이 일어나는지 살펴보자.
```
create table t ( key, no, data )
partition by range(no) subpartition by hash(key) subpartitions 16 (
  partition p01 values less than(11)
, partition p02 values less than(21)
, partition p03 values less than(31)
, partition p04 values less than(41)
, partition p05 values less than(51)
, partition p06 values less than(61)
, partition p07 values less than(71)
, partition p08 values less than(81)
, partition p09 values less than(91)
, partition p10 values less than(maxvalue)
)
as
select lpad(rownum, 6, '0'), mod(rownum, 50)+1, lpad(rownum, 10, '0')
from   dual
connect by level <= 999999
```
테이블을 no 컬럼 기준 Range 파티셔닝, key 컬럼 기준 해시 파티셔닝하였다. 아래는 상수 조건을 사용해 정적 Pruning을 테스트한 것이다.
```
select count(*) from t where no between 30 and 50

----------------------------------------------------------------------------
| Id  | Operation                   | Name | Rows  | Bytes | Pstart| Pstop |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |      |     1 |    13 |       |       |
|   1 |   SORT AGGREGATE            |      |     1 |    13 |       |       |
|   2 |     PARTITION RANGE ITERATOR|      |   576K|  7323K|     3 |     5 |
|   3 |       PARTITION HASH ALL    |      |   576K|  7323K|     1 |    16 |
|   4 |         TABLE ACCESS FULL   | T    |   576K|  7323K|    33 |    80 |
----------------------------------------------------------------------------
```
id 2번 라인은 Range 파티션에 대한 Prunning 정보를, id 3번 라인은 해시 파티션에 대한 Prunning 정보를 표시하고 있다.
Range 파티션에선 10개 중 3개(3~5번)를 읽었지만 각각 서브파티션을 16개씩 읽어 총 48(=3✕16)개 파티션을 읽는 것을 알 수 있다.

바인드 변수를 사용할 때는 실행계획에 어떻게 표시되는지 살펴보자.
```
select count(*) from t where no between :a and :b

------------------------------------------------------------------------------
| Id  | Operation                     | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |      |     1 |    13 |       |       |
|   1 |   SORT AGGREGATE              |      |     1 |    13 |       |       |
|   2 |     FILTER                    |      |       |       |       |       |
|   3 |       PARTITION RANGE ITERATOR|      |  2226 | 28938 |   KEY |   KEY |
|   4 |         PARTITION HASH ALL    |      |  2310 | 28938 |     1 |    16 |
|   5 |           TABLE ACCESS FULL   | T    |  2226 | 28938 |   KEY |   KEY |
------------------------------------------------------------------------------
```
Range 파티션에는 파티션 목록을 확정할 수 없어 'KEY'라고 표시하였지만 서브 파티션에는 액세스할 주 파티션별로 16개씩 읽게 됨을 표시하고 있다.
해시 서브파티션 키 컬럼에 조건절을 사용하지 않았기 때문이다.

### [파티션 Pruning 기능에 따른 I/O 수행량 비교]
파티션 Pruning 작동 여부에 따라 수행 일량에 어떤 차이가 생기는지 SQL 트레이스를 통해 확인해 보자.
```
select * from t where no = 1 and key = '000100'

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     0.016        0.007         0        49         0         1
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     0.016        0.007         0        49         0         1

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  PARTITION RANGE SINGLE PARTITION: 1 1 (cr=49 pr=0 pw=0 time=5915 us)
      1    PARTITION HASH SINGLE PARTITION: 6 6 (cr=49 pr=0 pw=0 time=5859 us)
      1      TABLE ACCESS FULL T PARTITION: 6 6 (cr=49 pr=0 pw=0 time=5724 us)
```
위에서는 주 파티션과 서브 파티션에 Pruning이 작동해 하나의 서브 파티션에서 49개 블록만 읽었다.

이번에는 서브 파티션에 대한 Pruning이 작동하지 못하도록 서브 파티션 키 컬럼을 함수로 가공하고 다시 수행해 보자.
```
select * from t where no = 1 and to_number(key) = 100

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     0.063        1.056       528       776         0         1
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     0.063        1.056       528       776         0         1

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  PARTITION RANGE SINGLE PARTITION: 1 1 (cr=776 pr=528 pw=0 time=1056056 us)
      1    PARTITION HASH ALL PARTITION: 1 16 (cr=776 pr=528 pw=0 time=1056027 us)
      1      TABLE ACCESS FULL T PARTITION: 1 16 (cr=776 pr=528 pw=0 time=1055868 us)
```
16개 서브 파티션에서 776개 블록을 읽었다. 인덱스와 마찬가지로, 파티션 키 컬럼도 함부로 가공해선 안 된다는 사실을 잘 말해주고 있다.

묵시적 형변환이 일어나는 경우도 테스트해 보자.
```
select * from t where no = 1 and key = 100

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     0.078        0.955       528       776         0         1
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     0.078        0.955       528       776         0         1

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  PARTITION RANGE SINGLE PARTITION: 1 1 (cr=776 pr=528 pw=0 time=954975 us)
      1    PARTITION HASH ALL PARTITION: 1 16 (cr=776 pr=528 pw=0 time=954945 us)
      1      TABLE ACCESS FULL T PARTITION: 1 16 (cr=776 pr=528 pw=0 time=954780 us)
```
파티션 키 컬럼을 명시적으로 가공했을 때와 같은 블록 I/O가 발생한 것을 볼 수 있고, 내부적인 형변환 떄문임을 아래 Predicate 정보를 통해 알 수 있다.
```
Predicate Information (identified by operation id):
---------------------------------------------------
  3 - filter("NO"=1 AND TO_NUMBER("KEY")=100)
```
이번에는 주 파티션 키 컬럼을 가공함으로써 서브 파티션 키 컬럼에 묵시적 형변환이 일어났을 때의 I/O 발생량을 확인해 보자.
```
select * from t where to_char(no) = '1' and key = 100

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     1.297        7.119      3588      4114         0         1
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     1.297        7.119      3588      4114         0         1

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  PARTITION RANGE ALL PARTITION: 1 10 (cr=4114 pr=3588 pw=0 time=7118551 us)
      1    PARTITION HASH ALL PARTITION: 1 16 (cr=4114 pr=3588 pw=0 time=7118465 us)
      1      TABLE ACCESS FULL T PARTITION: 1 160 (cr=4114 pr=3588 pw=0 time=7116955 us)
```
10개 주 파티션마다 16개씩, 총 160개 파티션에서 4,114개 블록을 읽었다.

### [동적 파티션 Pruning 시 테이블 레벨 통계 사용]
바인드 변수를 사용하면 최적화 시점에 파티션을 확정할 수 없어 동적 파티션 Pruning이 일어난다고 했는데, 같은 이유로 쿼리 최적화에 테이블 레벨 통계가 사용된다.
반면, 정적 파티션 Pruning일 때는 파티션 레벨 통계가 사용된다.

테이블 레벨 통게는 파티션 레벨 통계보다 다소 부정확하기 때문에 옵티마이저가 가끔 잘못된 실행계획을 수립하는 경우가 생기며, 이는 바인드 변수 때문에 생기는 대표적인 부작용 중 하나다.

<br/>

## (2) 서브쿼리 Pruning
조인에 사용되는 고급 파티션 Pruning 기법으로는 아래 두 가지가 있다.
- 서브쿼리 Pruning(8i~)
- 조인 필터 Pruning(11g~)

우선 서브쿼리 Pruning부터 살펴볼 텐데, 아래와 같은 쿼리가 있다고 하자.
```
select d.분기, o.주문일자, o.고객ID, o.상품ID, o.주문수량, o.주문금액
from   일자 d, 주문 o
where  o.주문일자 = d.일자
and    d.분기 >= 'Q20071'
```
NL 조인할 때 Inner 테이블이 조인 컬럼 기준으로 파티셔닝 돼 있다면 동적 Pruning이 작동한다고 했다.
주문은 대용량 거래 테이블이므로 주문일자 기준으로 월별 Range 파티셔닝 돼 있을테고, 일자 테이블을 드라이빙해 NL 조인한다면 분기 >= 'Q30071' 기간에 포함되는 주문 레코드만 읽을 수 있다.

하지만 대용량 주문 테이블을 Random 액세스 위주의 NL 방식으로 조인한다면 결코 좋은 성능을 기대하기 어렵다. 그렇다고 해시 조인이나 소트 머지 조인으로 처리하기도 부담스럽다.
2007년 1분기 이후 주문 데이터만 필요한데도 주문 테이블로부터 모든 파티션을 읽어 조인하고서 나중에 분기 조건을 필터링해야 하기 때문이다.

이럴 때 오라클은 Recursive 서브쿼리를 이용한 동적 파티션 Pruning을 고려한다. 이른바 '서브쿼리 Pruning'이라고 불리는 메커니즘으로서, 위 쿼리에 대해 내부적으로 아래와 같은 서브쿼리가 수행된다.
```
select distinct TBL$OR$IDX$PART$NUM(주문, 0, 1, 0, a.일자)
from  (select 일자 from 일자 where 분기 >= 'Q20071') a
order by 1
```
이 쿼리를 수행하면 액세스해야 할 파티션 번호 목록이 구해지며, 이를 이용해 필요한 주문 파티션만 스캔할 수 있다. 아래는 서브쿼리 Pruning이 작동할 때의 실행계획이다.
```
-----------------------------------------------------------------------------------
| Id  | Operation                     | Name | Rows   | Bytes  | Pstart  | Pstop  |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |      |        |        |         |        |
|*  1 |   HASH JOIN                   |      | 480951 | 22415K |         |        |
|*  2 |     TABLE ACCESS FULL         | 일자  |     12 |     89 |         |        |
|   3 |     PARTITION RANGE SUBQUERY  |      | 480951 | 15262K | KEY(SQ) |KEY(SQ) |
|   4 |       TABLE ACCESS FULL       | 주문  | 480951 | 15262K | KEY(SQ) |KEY(SQ) |
-----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
  1 - access("O"."주문일자"="D"."일자"
  2 - filter("D"."분기">='Q20074')
```
KEY(SQ)에서 SQ는 SubQuery를 뜻한다. 이 방식으로 파티션을 Pruning 하려면 드라이빙 테이블을 한 번 더 읽게 되므로 경우에 따라 총 비용이 오히려 증가할 수 있다.
따라서 서브쿼리 Pruning 적용 여부는 옵티마이저가 비용을 고려해 내부저으로 결정한다. 옵티마이저 결정에 영향을 미치는 Hidden 파라미터는 아래 표와 같다.
- 8i와 9i 버전에서는 'Partition Range Subquery'라고 표시되지 않는다. 따라서 Partition Range All이 Partition Range Iterator로 바뀌는 것을 통해 작동 여부를 확인해야 한다.
  explain plan 명령 후 plan_table을 조회할 때 other 컬럼에 표시되는 Recursive 서브쿼리를 통해서도 확인 가능하다.

|파라미터명|기본값|설명|
|:---:|:---:|:---|
|_subquery_pruning_cost_factor|20|파티션 테이블이 recursive 서브쿼리 대상 테이블보다 20배 이상 클 때만<br/>이 방식을 사용하겠다는 뜻이다. 바꿔 말하면, recursive 서브쿼리 수행 비용이<br/>파티션 테이블에 있는 모든 데이터를 액세스하는 비용의 5%를 초과하지 않아야<br/>효과적이라는 가정을 세운 것이다. (1/20 = 0/05)|
|_subquery_pruning_reduction|50|드라이빙 테이블 조건절에 의해 액세스되는 데이터가 드라이빙 테이블 전체 건수의<br/>50%를 넘지 않아야 함을 뜻한다. 드라이빙 테이블 조건절에 의한 선택도를 가지고<br/>반대편 파티션 테이블 Pruning 비율을 추정하는 것이다.|
|_subquery_pruning_enabled|true|서브쿼리 Pruning을 활성화|

제거될 것으로 예상되는 파티션 개수가 상당히(기본 값에 의하면 50%) 많고, where 조건절을 가진 드라이빙 테이블이 파티션 테이블에 비해 상당히 작을 때만 서브쿼리 Pruning이 작동한다는 사실을 알 수 있다.

참고로, 아래와 같이 설정하면 옵티마이저에 의해 계산된 비용과 상관없이 항상 서브쿼리 Pruning을 실시한다.
- _subquery_pruning_cost_factor = 1
- _subquery_pruning_reduction = 100

## (3) 조인 필터 Pruning
서브쿼리 Pruning은 드라이빙 테이블을 한 번 더 액세스하는 추가 비용이 발생한다.
그래서 11g부터 오라클은 블룸 필터(Bloom Filter) 알고리즘을 기반으로 한 조인 필터(Join Filter, Bloom Filter) Pruning 방식을 도입하였다.

### [블룸 필터(Bloom Filter) 알고리즘]
먼저 블룸 필터 알고리즘에 대해 살펴보자.
참고로 아래 내용은, 스위스 취리히 trivadis(www.trivadis.com)에서 컨설턴트로 활약 중인 Chriistian Antognini의 설명을 좀 더 이해하기 쉽게 재구성한 것이다.
그리고 블룸 필터에 대한 더 자세한 설명은 WIKIPEDIA(http://en.wikipedia.org/wiki/Bloom_filter)에서 얻을 수 있다.

아래처럼 A, B 두 개의 집합이 있고, 두 집합 간의 교집합을 찾으려고 한다.

> 집합 A = { 175, 442, 618 }
>
> 집합 B = { 175, 327, 432, 548 }

여기서는 원소가 각각 3개, 4개 밖에 없어 육안으로도 쉽게 찾을 수 있지만 각각 10,000개씩이라면 어떻게 할까? 집합 B에 있는 각 원소마다 집합 A를 뒤지면서 같은 값을 가진 원소가 있는지 확인해 봐야 한다.
교집합이 크다면, 많은 일량에도 불구하고 비효율은 없다고 할 수 있다. 하지만 교집합이 매우 작다면 이 방식은 비효율적이다.
지금 설명하려는 블룸 필터는 교집합이 작을 때 큰 효과를 발휘하며, 기본 알고리즘은 다음과 같다.

- [1] n 비트 Array를 할당하고(1~n), 각 비트를 0으로 설정한다.
- [2] n개의 값(1~n 중 하나)을 리턴하는 m개의 해시 함수를 정의하며, 서로 다른 해시 알고리즘을 사용한다. m개의 해시 함수는 다른 입력 값에 대해 우연히 같은 출력 값을 낼 수도 있다.
  (심지어 같은 알고리즘을 사용하더라도 다른 입력 값에 대해 같은 출력 값을 낼 수 있는 것이 해시 함수의 특징이다.)
- [3] 집합 A의 각 원소마다 차례로 m개의 해시 함수를 "모두" 적용한다. 그리고 각 해시 함수에서 리턴된 값(1~n 중 하나)에 해당하는 비트(1~n 중 하나)를 "모두" 1로 설정한다.
- [4] 교집합을 찾을 준비가 끝났다. 이제 집합 B의 각 원소마다 차례로 m개의 해시 함수를 "모두" 적용한다. 그리고 원소별로 해시 함수에서 리턴된 값(1~n)에 해당하는 비트를 "모두" 확인한다.
  - 하나라도 0으로 설정돼 있으면 그 원소는 집합 A에 없는 값이다. 잘못된 음수(false negative)는 불가능하기 때문이다. 즉, 집합 A에 있는 값인데 0으로 남았을 리가 없다. (0을 음수라고 가정하자.)
  - 모두 1로 설정돼 있으면 그 원소는 집합 A에 포함될 가능성이 있는 값이므로 이때만 집합 A를 찾아가 실제 같은 값을 가진 원소가 있는지 찾아본다.
    모두 1로 설정돼 있는데도 집합 A에 실제 그 값이 있는지 확인해야 하는 이유는, 잘못된 양수(false positive)가 가능하기 때문이다. 즉, 집합 A에 없는 값인데 모두 1로 설정될 수 있다.

설명만으로는 쉽게 이해하기 어려울 것이다. 예시를 보면서 단계별로 다시 설명해 보자.
- [1] 0부터 9번까지 10개 비트를 할당하고 모두 0으로 설정한다.
- [2] 0부터 9까지 총 10개의 값을 리턴할 수 있는 두 개의 함수(h1, h2)를 정의한다. 두 함수는 서로 다르게 구현되었다.

  ```
  SQL> create or replace function h1(e number) return number
    2  as
    3  begin
    4    return mod(e, 10);
    5  end;
    6  /

  SQL> create or replace function h2(e number) return number
    2  as
    3  begin
    4    return mod(ceil(e/10), 10);
    5  end;
    6  /

  SQL> select rownum no, h1(rownum) r1, h2(rownum) r2
    2  from   dual
    3  connect by level <= 100;

             NO            R1            R2
  ------------- ------------- -------------
              1             1             1
              2             2             1
              3             3             1
              4             4             1
              5             5             1
              6             6             1
              7             7             1
              8             8             1
              9             9             1
             10             0             1
             11             1             2
             12             2             2
             13             3             2
             14             4             2
             15             5             2
           ....           ...           ...
           ....           ...           ...
  ```

- [3] 집합 A의 각 원소(175, 442, 618)마다 차례로 h1, h2 함수를 적용하고, 각 해시 함수에서 리턴된 값(0~9 중 하나)에 해당하는 비트르 모두 1로 설정한다.

  아래는 집합 A에 있는 세 값을 h1, h2 함수에 입력했을 때의 출력 값이다.

  |입력 값|h1 출력 값|h2 출력 값|
    |:---:|:---:|:---:|
  |175|5|8|
  |442|2|5|
  |618|8|2|

  따라서 3번 과정을 모두 완료하고 나서 비트 값을 확인해 보면, 2, 5, 8 비트가 1로 설정돼 있다.

- [4] 교집합을 찾을 준비가 끝났다. 집합 B의 각 원소마다 차례로 h1, h2 함수를 적용하고, 리턴된 값(0~9 중 하나)에 해당하는 비트를 모두 확인한다.
  하나라도 0으로 설정돼 있으면 그 원소는 집합 A에 없는 값이다. 모두 1로 설정돼 있으면 그 원소는 집합 A에 포함될 가능성이 있는 값이므로 이때만 집합 A를 찾아가 실제 같은 값을 가진 원소가 있는지 찾아본다.

블룸 필터 역할은, 교집합에 해당하는 원소(175)를 찾는 데 있지 않고 교집합이 아닌 것이 확실한 원소(327, 432)를 찾는 데에 있음을 이해할 것이다.
327과 432를 제외한 나머지 원소(175, 548)에 대해서만 집합 A를 확인해서 실제 교집합인지를 가린다.

### [블룸 필터 알고리즘에서 false positive를 줄이는 방법]
집합 B의 원소 548에서 false positive가 발생해 불필요하게 집합 A를 확인해야만 했다. 10비트보다 큰 100비트 Array를 사용하면 어떤 결과가 나올까?

더 많은 비트를 할당(더 많은 공간이 필요)하거나 더 많은 해시 함수를 사용(더 많은 시간을 소비)하면 false positive 발생 가능성은 줄어든다.
공간/시간의 효율성과 false positive 발생 가능성은 서로 트레이드 오프(Trade-off) 관계이므로 적정한 개수의 비트와 해시 함수를 사용하는 것이 과제다.

### [조인 필터(=블룸 필터) Pruning]
지금까지 블룸 필터 알고리즘에 대해 비교적 자세히 설명하였다. 오라클은 성능 향상을 위해 여러 곳에 이 알고리즘을 사용하고 있는데, 그 중 대표적인 것이 파티션 Pruning이다.
11g부터 적용되기 시작한 이 기능은 파티션 테이블과 조인할 때, 읽지 않아도 되는 파티션을 제거해 주는 것으로서 '조인 필터 Pruning' 또는 '블룸 필터 Pruning'이라고 부른다.

앞에서 서브쿼리 파티션 Pruning에서 보았던 쿼리에 이 기능을 적용하면 실행계획에 어떤 변화가 생기는지 살펴보자.
```
select d.분기, o.주문일자, o.고객ID, o.상품ID, o.주문수량, o.주문금액
from   일자 d, 주문 o
where  o.주문일자 = d.일자
and    d.분기 >= 'Q20071'

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
 480591  HASH JOIN (cr=3827 pr=0 pw=0 time=4946 us cost=655 size=21002700 ... )
     12    PART JOIN FILTER CREATE :BF0000 (cr=4 pr=0 pw=0 time=18 us cost=4 ... )
     12      TABLE ACCESS FULL 일자 (cr=4 pr=0 pw=0 time=6 us cost=4 size=10388 ... )
 480591    PARTITION RANGE JOIN-FILTER PARTITION: :BF0000 :BF0000 (cr=3823 pr=0 ... )
 480591      TABLE ACCESS FULL 주문 PARTITION: :BF0000 :BF0000 (cr=3823 pr=0 ... )
```
part join filter create와 partition range join-filter를 포함하는 두 개 오퍼레이션 단계가 눈에 띈다.
전자는 블룸 필터를 생성하는 단계, 후자는 블룸 필터를 이용해 파티션 Pruning하는 단계를 나타낸다. 좀 더 풀어서 설명하면 다음과 같다.

- [1] 일자 테이블로부터 분기 >= 'Q20071' 조건에 해당하는 레코드를 읽어 해시 테이블을 만들면서 블룸 필터를 생성한다.
  즉, 조인 컬럼인 일자 값에 매핑되는 주문 테이블 파티션 번호를 찾아 n개의 해시함수에 입력하고 거기서 출력된 값을 이용해 비트 값을 설정한다.
- [2] 일자 테이블을 다 읽고 나면 주문 테이블의 파티션 번호별로 비트 값을 확인함으로써 읽지 않아도 되는 파티션 목록을 취합한다.
- [3] 취합된 파티션을 제외한 나ㅜ 머지 파티션만 읽어 조인한다.

다시 말하지만 블룸 필터 역할은, 교집합 아님이 확실한 원소를 찾는 데에 있다.
이 알고리즘을 사용하는 조인 필터 Pruning도 조인 대상 집합을 확실히 포함하는 파티션을 찾는 게 아니라, 확실히 포함하지 않는 파티션을 찾는다. 그런 후 그 파티션 목록을 제외한 나머지 파티션만 스캔한다.

위 쿼리는 기본적으로 2007년 1월부터 현재까지의 파티션만 읽지만, false positive가 발생해 그 이전 파티션 중 일부를 불필요하게 읽을 가능성은 있다.
하지만 이런 고급 파티션 Pruning 기법을 활용하지 않았을 때 모든 파티션을 읽는 것에 비하면 훨씬 효율적이다. 서브쿼리 Pruning과 비교해 보면, 드라이빙 테이블이 클수록 조인 필터 Pruning이 유리하다.

만약, 조인 필터 Pruning 기능을 비활성화하고 싶다면, _bloom_pruning_enabled 파라미터를 false로 설정하면 된다.

<br/>

## (4) SQL 조건절 작성 시 유의사항
고객 테이블을 아래처럼 가입일 기준으로 Range 월 파티셔닝 하였다.
```
create table 고객
partition by range(가입일)
( partition m01 values less than('20090201')
, partition m02 values less than('20090301')
, partition m03 values less than('20090401')
, partition m04 values less than('20090501')
, partition m05 values less than('20090601')
, partition m06 values less than('20090701')
, partition m07 values less than('20090801')
, partition m08 values less than('20090901')
, partition m09 values less than('20091001')
, partition m10 values less than('20091101')
, partition m11 values less than('20091201')
, partition m12 values less than('20100101'))
as
select rownum 고객ID
     , dbms_random.string('a', 20) 고객명
     , to_char(to_date('20090101', 'yyyymmdd') + (rownum - 1), 'yyyymmdd') 가입일
from   dual
connect by level <= 365;
```
아래처럼 2009년 10월에 가입한 고객을 조회할 때 like 연산자를 사용했더니 두 개(9~10번) 파티션을 스캔하는 것을 볼 수 있다.
왜 m10 파티션만 읽지 않고 m09 파티션까지 읽은 것일까?
```
select * from 고객
where  가입일 like '200910%'

--------------------------------------------------------------------------
| Id  | Operation                 | Name | Rows  | Bytes | Pstart| Pstop |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT          |      |    31 | 62651 |       |       |
|   1 |   PARTITION RANGE ITERATOR|      |    31 | 62651 |     9 |    10 |
|   2 |     TABLE ACCESS FULL     | 고객  |    31 | 62651 |     9 |    10 |
--------------------------------------------------------------------------
```
아래처럼 '20091001' 보다 작은 수많은 문자가 존재하는데, 이 값들이 입력된다면 파티션 정의상 2009년 9월 파티션(m09)에 저장된다.
- 200910, 20091000, ...
- 2009100+, 2009100-, 2009100*, 2009100/, ...
- 2009100%, 20099100$, 2009100#, ...

이 상태에서 위와 같이 like '200910%' 조건으로 검색한다면 이들 값까지 모두 결과집합에 포함시켜야 하므로 옵티마이저는 m09 파티션을 스캔하지 않을 수 없다.
사용자가 이런 값들을 입력하지 않더라도 옵티마이저 입장에서는 알 수 없는 일이다.

따라서 위와 같이 (월이 아닌) 일자로써 파티션 키 값을 정의했다면 아래와 같이 between 연산자를 이용해 정확한 값 범위를 주고 쿼리해야 한다.
```
select * from 고객
where  가입일 between '20091001' and '20091031'

------------------------------------------------------------------------
| Id  | Operation               | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |      |    31 | 62651 |       |       |
|   1 |   PARTITION RANGE SINGLE|      |    31 | 62651 |    10 |    10 |
|   2 |     TABLE ACCESS FULL   | 고객  |    31 | 62651 |    10 |    10 |
------------------------------------------------------------------------
```
1장 7절 인덱스 스캔 효율화 원리에서 like보다 between이 유리한 이유를 설명했는데, 지금 본 것처럼 파티션 Pruning을 위해서라도 like보다는 가급적 between 연산자를 이용해 정확한 검색구간을
지정하는 것이 바람직하다.

위와 같은 상황에서, 만약 일일이 SQL을 수정하기 곤란하다면 아래처럼 연월(yyyymm)로써 파티션 키 값을 다시 정의해 주면 된다.
```
create table 고객
partition by range(가입일)
( partition m01 values less than('200902')
, partition m02 values less than('200903')
, ...
, ...
, partition m11 values less than('200912')
, partition m12 values less than('201001') )
as
select rownum 고객ID
     , dbms_random.string('a', 20) 고객명
     , to_char(to_date('20090101', 'yyyymmdd') + (rownum - 1), 'yyyymmdd') 가입일
from   dual
connect by level <= 365;
```
이제 like로 검색하더라도 아래처럼 정확히 m10 파티션만 스캔한다. between 연산자는 이때도 정확한 파티션 Pruning이 일어난다.
다시 강조하지만, 파티션 설계와 상관없이 옵티마이저가 효율적인 선택을 할 수 있도록 하려면 between 연산자를 사용하기 바란다.
```
select * from 고객
where  가입일 like '200910%'

------------------------------------------------------------------------
| Id  | Operation               | Name | Rows  | Bytes | Pstart| Pstop |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |      |    31 | 62651 |       |       |
|   1 |   PARTITION RANGE SINGLE|      |    31 | 62651 |    10 |    10 |
|   2 |     TABLE ACCESS FULL   | 고객  |    31 | 62651 |    10 |    10 |
------------------------------------------------------------------------
```