# 5.2 | 소트가 발생하지 않도록 SQL 작성

SQL 작성할 때 불필요한 소트가 발생하지 않도록 주의해야 한다.
Union, Minus, Distinct 연산자는 중복 레코드를 제거하기 위한 소트 연산을 발생시키므로 꼭 필요한 경우에만 사용하고, 성능이 느리다면 소트 연산을 피할 방법이 있는지 찾아봐야 한다.
조인 방식도 잘 선택해 줘야 한다.

<br/>

## (1) Union vs. Union All
SQL에 Union을 사용하면 옵티마이저는 상단과 하단 두 집합 간 중복을 제거하려고 소트 작업을 수행한다. 따라서 될 수 있으면 Union All을 사용해야 한다.

그런데 Union을 Union All로 변경하려다 자칫 결과 집합이 달라질 수 있으므로 주의해야 한다.
Union 대신 Union All을 사용해도 되는지를 정확히 판단하려면 데이터 모델에 대한 이해와 집합적 사고가 필요하다.
그런 능력이 부족하면 알 수 없는 데이터 중복, 혹시 모를 데이터 중복을 우려해 중복 제거용 연산자를 불필요하게 자주 사용하게 된다.

아래 SQL은 Union 상단과 하단 집합 사이에 인스턴스 중복 가능성이 없다. 결제수단코드 조건절에 다른 값을 입력했기 때문이다. 그런데도 Union을 사용함으로 인해 소트 연산을 발생시키고 있다.
```
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  결제수단코드 = 'M' and 결제일자 = '20180316'
UNION
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  결제수단코드 = 'C' and 결제일자 = '20180316'

Execution Plan
------------------------------------------------------------------------
  0        SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=2 Bytes=106)
  1    0     SORT (UNIQUE) (Cost=4 Card=2 Bytes=106)
  2    1       UNION-ALL
  3    2         FILTER
  4    3           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=1 ...)
  5    4             INDEX (RANGE SCAN) OF '결제_N1' (INDEX) (Cost=1 Card=1)
  6    2         FILTER
  7    6           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=1 ...)
  8    7             INDEX (RANGE SCAN) OF '결제_N1' (INDEX) (Cost=1 Card=1)
```
위아래 두 집합이 상호배타적이므로 Union 대신 Union All을 사용해도 된다.

아래 SQL은 Union 상단과 하단 집합 사이에 인스턴스 중복 가능성이 있다.
```
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  결제일자 = '20180316'
UNION
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  주문일자 = '20180316'

Execution Plan
------------------------------------------------------------------------
  0        SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=2 Bytes=106)
  1    0     SORT (UNIQUE) (Cost=2 Card=2 Bytes=106)
  2    1       UNION-ALL
  3    2         FILTER
  4    3           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=0 ...)
  5    4             INDEX (RANGE SCAN) OF '결제_N2' (INDEX) (Cost=0 Card=1)
  6    2         FILTER
  7    6           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=0 ...)
  8    7             INDEX (RANGE SCAN) OF '결제_N3' (INDEX) (Cost=0 Card=1)
```
결제일자와 주문일자 조건은 상호배타적 조건이 아니기 때문이다. 만약 Union을 Union All로 변경하면, 결제일자와 주문일자가 같은 결제 데이터가 중복해서 출력된다.

소트 연산이 일어나지 않도록 Union All을 사용하면서도 데이터 중복을 피하려면, 아래와 같이 하면 된다.
```
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  결제일자 = '20180316'
UNION ALL
select 결제번호, 주문번호, 결제금액, 주문일자 ...
from   결제
where  주문일자 = '20180316'
and    결제일자 <> '20180316'

Execution Plan
------------------------------------------------------------------------
  0        SELECT STATEMENT Optimizer=ALL_ROWS (Cost=0 Card=2 Bytes=106)
  1    0       UNION-ALL
  2    1         FILTER
  3    2           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=0 Card=1 ...)
  4    3             INDEX (RANGE SCAN) OF '결제_N2' (INDEX) (Cost=0 Card=1)
  5    1         FILTER
  6    5           TABLE ACCESS (BY INDEX ROWID) OF '결제' (TABLE) (Cost=0 ...)
  7    6             INDEX (RANGE SCAN) OF '결제_N3' (INDEX) (Cost=0 Card=1)
```
참고로, 결제일자가 Null 허용 컬럼이면 맨 아래 조건절을 아래와 같이 변경해야 한다.
```
and    (결제일자 <> '20180316' or 결제일자 is null)
```
아래와 같이 LNNVL 함수를 이용해도 된다.
```
and    LNNVL(결제일자 = '20180316')
```

## (2) Exists 활용
중복 레코드를 제거할 목적으로 Distinct 연산자를 종종 사용하는데, 이 연산자를 사용하면 조건에 해당하는 데이터를 모두 읽어서 중복을 제거해야 한다.
부분범위 처리는 당연히 불가능하고, 모든 데이터를 읽는 과정에 많은 I/O가 발생한다.

예를 들어, 상품과 계약 테이블이 있다.
계약_X2 인덱스 구성이 [상품번호 + 계약일자]일 때, 아래 쿼리는 상품유형코드 조건절에 해당하는 상품에 대해 계약일자 조건 기간에 발생한 계약 데이터를 모두 읽는 비효율이 있다.
상품 수는 적고 상품별 계약 건수가 많을수록 비효율이 큰 패턴이다.
```
select DISTINCT p.상품번호, p.상품명, p.상품가격, ...
from   상품 p, 계약 c
where  p.상품유형코드 = :pclscd
and    c.상품번호 = p.상품번호
and    c.계약일자 between :dt1 and :dt2
and    c.계약구분코드 = :ctpcd

Execution Plan
----------------------------------------------------------------------
  0       SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=1 Bytes=80)
  1    0    HASH (UNIQUE) (Cost=3 Card=1 Bytes=80)
  2    1      FILTER
  3    2        NESTED LOOPS
  4    3          NESTED LOOPS
  5    4            TABLE ACCESS (BY INDEX ROWID) OF '상품' (TABLE) (Cost=1 ... )
  6    5              INDEX (RANGE SCAN) OF '상품_X1' (INDEX) (Cost=1 Card=1)
  7    4            INDEX (RANGE SCAN) OF '계약_X2' (INDEX) (Cost=1 Card=1)
  8    3          TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE) (Cost=1 ...)
```
쿼리를 아래와 같이 바꿔보자.
```
select p.상품번호, p.상품명, p.상품가격, ...
from   상품 p
where  p.상품유형코드 = :pclscd
and    EXISTS (select 'x' from 계약 c
               where   c.상품번호 = p.상품번호
               and     c.계약일자 between :dt1 and :dt2
               and     c.계약구분코드 = :ctpcd)

Execution Plan
----------------------------------------------------------------------
  0       SELECT STATEMENT Optimizer=ALL_ROWS (Cost=2 Card=1 Bytes=80)
  1    0    FILTER
  2    1      NESTED LOOPS (SEMI) (Cost=2 Card=1 Bytes=80)
  3    2        TABLE ACCESS (BY INDEX ROWID) OF '상품' (TABLE) (Cost=1 Card=1 ... )
  4    3          INDEX (RANGE SCAN) OF '상품_X1' (INDEX) (Cost=1 Card=1)
  5    2        TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE) (Cost=1 Card=1 ... )
  6    5          INDEX (RANGE SCAN) OF '계약_X2' (INDEX) (Cost=1 Card=1)
```
Exists 서브쿼리는 데이터 존재 여부만 확인하면 되기 때문에 조건절을 만족하는 데이터를 모두 읽지 않는다.

위 쿼리로 말하면, 상품유형코드 조건절(p.상품유형코드 = :pclscd)에 해당하는 상품(c.상품번호 = p.상품번호)에 대해 계약일자 조건 기간(c.계약일자 between :dt1 and :dt2)에 발생한 계약 중
계약구분코드 조건절을 만족하는(c.계약구분코드 = :ctpcd) 데이터가 한건이라도 존재하는지만 확인한다. Distinct 연산자를 사용하지 않았으므로 상품 테이블에 대한 부분범위 처리도 가능하다.

**Distinct, Minus 연산자를 사용한 쿼리는 대부분 Exists 서브쿼리로 변환 가능하다** 아래는 Minus 연산자를 Not Exists 서브쿼리로 변환해서 튜닝한 사례다.
```
< 튜닝 전 >

SELECT ST.상황접수번호, ST.관제일련번호, ST.상황코드, ST.관제일시
  FROM 관제진행상황 ST
 WHERE 상황코드 = '0001'  -- 신고접수
   AND 관제일시 BETWEEN :V_TIMEFROM || '000000' AND :V_TIMETO || '235959'
MINUS
SELECT ST.상황접수번호, ST.관제일련번호, ST.상황코드, ST.관제일시
  FROM 관제진행상황 ST, 구조활동 RPT
 WHERE 상황코드 = '0001'
   AND 관제일시 BETWEEN :V_TIMEFROM || '000000' AND :V_TIMETO || '235959'
   AND RPT.출동센터ID = :V_CNTR_ID
   AND ST.상황접수번호 = RPT.상황접수번호
 ORDER BY 상황접수번호, 관제일시

< 튜닝 후 >

SELECT ST.상황접수번호, ST.관제일련번호, ST.상황코드, ST.관제일시
  FROM 관제진행상황 ST
 WHERE 상황코드 = '0001'  -- 신고접수
   AND 관제일시 BETWEEN :V_TIMEFROM || '000000' AND :V_TIMETO || '235959'
   AND NOT EXISTS (SELECT 'X' FROM 구조활동
                   WHERE 출동센터ID = :V_CNTR_ID
                   AND   상황접수번호 = ST.상황접수번호)
 ORDER BY ST.상황접수번호, ST.관제일시
```

<br/>

## (3) 조인 방식 변경
인덱스를 이용해 소트 연산을 생략하는 방법은 바로 이어서 설명하겠지만, 조인문일 때는 조인 방식도 잘 선택해 줘야 한다.

아래 SQL 문에서 계약_X01 인덱스가 [지점ID + 계약일시] 순이면 소트 연산을 생략할 수 있지만, 해시 조인이기 때문에 Sort Order By가 나타났다.
```
select c.계약번호, c.상품코드, p.상품명, p.상품구분코드, c.계약일시, c.계약금액
from   계약 c, 상품 p
where  c.지점ID = :brch_id
and    p.상품코드 = c.상품코드
order by c.계약일시 desc

Execution Plan
---------------------------------------------------------------------
  0       SELECT STATEMENT Optimizer=ALL_ROWS
  1    0    SORT (ORDER BY)
  2    1      HASH JOIN
  3    2        TABLE ACCESS (FULL) OF '상품' (TABLE)
  4    2        TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE)
  5    4          INDEX (RANGE SCAN) OF '계약_X01' (INDEX)
```
아래와 같이 계약 테이블 기준으로 상품 테이블과 NL 조인하도록 조인 방식을 변경하면 소트 연산을 생략할 수 있어 지점ID 조건을 만족하는 데이터가 많고 부분범위 처리 가능한 상황에서 큰 성능 개선 효과를
얻을 수 있다.
```
select /*+ leading(c) use_nl(p) */
       c.계약번호, c.상품코드, p.상품명, p.상품구분코드, c.계약일시, c.계약금액
from   계약 c, 상품 p
where  c.지점ID = :brch_id
and    p.상품코드 = c.상품코드
order by c.계약일시 desc

Execution Plan
---------------------------------------------------------------------
  0       SELECT STATEMENT Optimizer=ALL_ROWS
  1    0    NESTED LOOPS
  2    1      NESTED LOOPS
  3    2        TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE)
  4    3          INDEX (RANGE SCAN DESCENDING) OF '계약_X01' (INDEX)
  5    2        INDEX (RANGE SCAN) OF '상품_PK' (INDEX (UNIQUE))
  6    1      TABLE ACCESS (BY INDEX ROWID) OF '상품' (TABLE)
```
정렬 기준이 조인 키 컬럼이면 소트 머지 조인도 Sort Order By 연산을 생략할 수 있다.