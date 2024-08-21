# 06. Sort Area를 적게 사용하도록 SQL 작성

소트 연산이 불가피하다면 메모리 내에서 처리를 완료할 수 있도록 노력해야 한다. Sort Area 크기를 늘리는 방법도 있지만 그전에 Sort Area를 적게 사용할 방법부터 찾는 것이 순서다.

<br/>

## (1) 소트를 완료하고 나서 데이터 가공하기
특정 기간에 발생한 주문상품 목록을 파일로 내리고자 한다. 아래 두 SQL 중 어느 쪽이 Sort Area를 더 적게 사용할까?
#### [1번]
```
select lpad(상품번호, 30) || lpad(상품명, 30) || lpad(고객ID, 10)
    || lpad(고객명, 20) || to_char(주문일시, 'yyyymmdd hh24:mi:ss')
from   주문상품
where  주문일시 between :start and :end
order by 상품번호
```
#### [2번]
```
select lpad(상품번호, 30) || lpad(상품명, 30) || lpad(고객ID, 10)
    || lpad(고객명, 20) || to_char(주문일시, 'yyyymmdd hh24:mi:ss')
from (
  select 주문상품, 상품명, 고객ID, 고객명, 주문일시
  from   주문상품
  where  주문일시 between :start and :end
  order by 상품번호
)
```
1번 SQL은 레코드당 105(=30+30+10+20+15)바이트(헤더 정보는 제외하고 데이터 값만)로 가공된 결과치를 Sort Area에 담는다.
반면 2번 SQL은 가공되지 않은 상태로 정렬을 완료하고 나서 최종 출력할 때 가공하므로 1번 SQL에 비해 Sort Area를 훨씬 적게 사용한다.
실제 테스트해 보면 Sort Area 사용량에 큰 차이가 나는 것을 관찰할 수 있다.

<br/>

## (2) Top-N 쿼리
Top-N 쿼리 형태로 작성하며녀 소트 연산(=값 비교) 횟수를 최소화함은 물론 Sort Area 사용량을 줄일 수 있다. 우선 Top-N 쿼리 작성법부터 살펴보자.

SQL Server나 Sybase는 Top-N 쿼리를 아래와 같이 손쉽게 작성할 수 있다.
```
select TOP 10 거래일시, 체결건수, 체결수량, 거래대금
from   시간대별종목거래
where  종목코드 = 'KR123456'
and    거래일시 >= '20080304'
```
IBM DB2에서도 아래와 같이 쉽게 작성할 수 있다.
```
select 거래일시, 체결건수, 체결수량, 거래대금
from   시간대별종목거래
where  종목코드 = 'KR123456'
and    거래일시 >= '20080304'
FETCH FIRST 10 ROWS ONLY;
```
오라클에서는 아래처럼 인라인 뷰로 한번 감싸야 하는 불편함이 있다.
```
select * from (
  select 거래일시, 체결건수, 체결수량, 거래대금
  from   시간대별종목거래
  where  종목코드 = 'KR123456'
  and    거래일시 >= '20080304'
  order by 거래일시
)
where rownum <= 10
```
위 쿼리를 수행하는 시점에 [종목코드 + 거래일시] 순으로 구성된 인덱스가 존재한다면 옵티마이저는 그 인덱스를 이용함으로써 order by 연산을 대체할 수 있다.
아래 실행계획에서 sort order by 오퍼레이션이 나타나지 않은 것을 확인하기 바란다.
```
Execution Plan
--------------------------------------------------------------------------------------
 0      SELECT STATEMENT Optimizer=ALL_ROWS
 1   0    COUNT (STOPKEY)
 2   1      VIEW
 3   2        TABLE ACCESS (BY INDEX ROWID) OF '시간대별종목거래' (TABLE)
 4   3          INDEX (UNIQUE SCAN) OF '시간대별종목거래_PK' (INDEX (UNIQUE))
```
그뿐만 아니라 rownum 조건을 사용해 N건에서 멈추도록 했으므로 조건절에 부합하는 레코드가 아무리 많아도 매우 빠른 수행 속도를 낼 수 있다. 실행계획에 표시된 count stopkey가 그것을 의미한다.

### [Top-N 쿼리의 소트 부하 경감 원리]
[종목코드 + 거래일시] 순으로 구성된 인덱스가 없을 때는 어떤가? 종목코드만을 선두로 갖는 다른 인덱스를 사용하거나 Full Table Scan 방식으로 처리할 텐데, 이때는 정렬 작업이 불가피하다.
하지만 Top-N 쿼리 알고리즘이 효과를 발휘해 sort order by 부하를 경감시켜준다.

Top-N 쿼리 알고리즘에 대해 간단히 설명하면, rownum <= 10 이면 우선 10개 레코드를 담을 배열을 할당하고, 처음 읽은 10개 레코드를 정렬된 상태로 담는다.

이후 읽는 레코드에 대해서는 맨 우측에 있는 값(=가장 큰 값)과 비교해서 그보다 작은 값이 나타날 때만 배열 내에서 다시 정렬을 시도한다. 물론 맨 우측에 있던 값은 버린다.
이 방식으로 처리하면 전체 레코드를 정렬하지 않고도 오름차순(ASC)으로 최소값을 갖는 10개 레코드를 정확히 찾아낼 수 있다. 이것이 Top-N 쿼리가 소트 연산 횟수와 Sort Area 사용량을 줄여주는 원리다.

실제 소트 부하 경감 효과를 측정해 보자.

#### [효과 측정 : Top-N 쿼리가 작동할 때]
```
SQL> create table t as select * from all_objects;

SQL> alter session set workarea_size_policy = manual;

SQL> alter session set sort_area_size = 524288;

SQL> set autotrace traceonly statistics
SQL> select count(*) from t;

  COUNT(*)
----------
     49857

Statistics
---------------------------------------------------------
      ...  ...
      690  consistent gets
```
테이블을 스캔하면서 전체 레코드 개수를 구하는데 690개 블록을 읽었다.

이제 Top-N 쿼리를 수행해 보자. Top-N 쿼리가 작동하지 않을 때와 비교하려고 위에서 Sort Area 크기를 작게 설정한 것을 확인하기 바란다.
```
SQL> select *
  2  from (
  3    select * from t
  4    order by object_name
  5  )
  6  where rownum <= 10 ;

10 개의 행이 선택되었습니다.

Statistics
---------------------------------------------------------
        0  recursive calls
        0  db block gets
      690  consistent gets
        0  physical reads
      ...  ...
        1  sorts (memory)
        0  sorts (disk)
```
읽은 블록 수는 count(*) 쿼리일 때와 같다. 테이블 전체를 읽은 것이다. sorts 항목을 보면 메모리 소트 방식으로 정렬 작업을 한 번 수행하였다.
이제는 SQL 트레이스 결과인데, sort order by 옆에 stopkey가 표시되었고, physical write(=pw) 항목이 0인 것에 주목하자.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.078        0.083       0       690          0     10
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.078        0.084       0       690          0     10

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      0  STATEMENT
     10    COUNT STOPKEY (cr=690 pr=0 pw=0 time=83318 us)
     10      VIEW (cr=690 pr=0 pw=0 time=83290 us)
     10        SORT ORDER BY STOPKEY (cr=690 pr=0 pw=0 time=83264 us)
  49857          TABLE ACCESS FULL T (cr=690 pr=0 pw=0 time=299191 us)
```

#### [효과 측정 : Top-N 쿼리가 작동하지 않을 때]
아래는 Top-N 쿼리 알고리즘이 작동하지 않는 경우다. 쿼리 결과는 동일하도록 작성하였다.
```
SQL> select *
  2  from (
  3    select a.*, rownum no
  4    from (
  5      select * from t order by object_name
  6    ) a
  7  )
  8  where no <= 10 ;

Statistics
---------------------------------------------------------
        6  recursive calls
       14  db block gets
      690  consistent gets
      698  physical reads
      ...  ...
        0  sorts (memory)
        1  sorts (disk)
```
'sorts (disk)' 항목을 보고 정렬을 디스크 소트 방식으로 한 번 수행한 것을 알 수 있고, physical reads 항목이 698인 것도 눈에 띈다.
아래는 SQL 트레이스 결과인데, sort order by 옆에 stopkey가 없고 physical write(=pw) 항목이 698인 것을 확인하자.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.281        0.858     698       690         14     10
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.281        0.858     698       690         14     10

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      0  STATEMENT
     10    VIEW (cr=690 pr=698 pw=698 time=357962 us)
  49857      COUNT (cr=690 pr=698 pw=698 time=1604327 us)
  49857        VIEW (cr=690 pr=698 pw=698 time=1205452 us)
  49857          SORT ORDER BY (cr=690 pr=698 pw=698 time=756723 us)
  49857            TABLE ACCESS FULL T (cr=690 pr=0 pw=0 time=249345 us)
```
같은 양(690 블록)의 데이터를 읽고 정렬을 수행하였는데, 앞에서는 Top-N 쿼리 알고리즘이 작동해 메모리 내에서 정렬을 완료했지만 조금 전 쿼리는 디스크를 이용해야만 했다.

<br/>

## (3) 분석함수에서의 Top-N 쿼리
window sort 시에도 **rank()** 나 **row_number()** 를 쓰면 Top-N 쿼리 알고리즘이 작동해 max() 등 함수를 쓸 때보다 소트 부하를 경감시켜 준다. 테스트를 통해 같이 확인해 보자.

먼저 아래와 같이 테스트 데이터를 생성한다. 같은 ID가 10개씩 되도록 테이블을 만들고, Seq 컬럼을 두어 ID가 같을 때 레코드를 식별할 수 있도록 하였다.
```
create table t
as
select 1 id, rownum seq, owner, object_name, object_type, created, status
from   all_objects;

begin
  for i in 1..9
  loop
    insert into t
    select i+1 id, rownum seq
         , owner, object_name, object_type, created, status
    from   t
    where  id = 1;
    commit;
  end loop;
end;
/

alter session set workarea_size_policy = manual;
alter session set sort_area_size = 1048576;
```
아래는 마지막 이력 레코드를 찾는 쿼리다.
```
select id, seq, owner, object_name, object_type, created, status
from  (select id, seq
            , max(seq) over (partition by id) last_seq
            , owner, object_name, object_type, created, status
       from t)
where  seq = last_seq

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     2.750        9.175     13456      4536         9        10
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     2.750        9.175     13456      4536         9        10

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      0  STATEMENT
     10    VIEW (cr=4536 pr=13456 pw=8960 time=4437847 us)
 498570      WINDOW SORT (cr=4536 pr=13456 pw=8960 time=9120662 us)
 498570        TABLE ACCESS FULL T (cr=4536 pr=0 pw=0 time=1994341 us)
```
디스크 소트가 발생하도록 하려고 sort_area_size를 줄여 테스트하였고, 실제 window sort 단계에서 13,456개의 physical read(=pr)와 8,960개의 physical write(=pw)가 발생했다.

max() 함수 대신 rank() 함수를 사용해 보자.
```
select id, seq, owner, object_name, object_type, created, status
from  (select id, seq
            , rank() over (partition by id order by seq desc) rnum
            , owner, object_name, object_type, created, status
       from t)
where  rnum = 1

Call      Count  CPU Time Elapsed Time      Disk     Query   Current      Rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1     0.000        0.000         0         0         0         0
Execute       1     0.000        0.000         0         0         0         0
Fetch         2     0.969        1.062        40      4536        42        10
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         4     0.969        1.062        40      4536        42        10

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      0  STATEMENT
     10    VIEW (cr=4536 pr=40 pw=40 time=1061996 us)
    111      WINDOW SORT PUSHED RANK (cr=4536 pr=40 pw=40 time=1061971 us)
 498570        TABLE ACCESS FULL T (cr=4536 pr=0 pw=0 time=1495760 us)
```
여기서도 physical read와 physical write가 각각 40개씩 발생하긴 했지만 앞에서보다 훨씬 줄었다. 8초 가량 시간이 덜 소요된 것도 이 때문이다.