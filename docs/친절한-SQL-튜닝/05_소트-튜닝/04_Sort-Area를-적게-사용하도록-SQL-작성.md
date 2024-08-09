# 5.4 | Sort Area를 적게 사용하도록 SQL 작성

소트 연산이 불가피하다면 메모리 내에서 처리를 완료할 수 있도록 노력해야 한다. Sort Area 크기를 늘리는 방법도 있지만, 그전에 Sort Area를 적게 사용할 방법부터 찾는 것이 순서다.

<br/>

## (1) 소트 데이터 줄이기
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
from  (
  select 상품번호, 상품명, 고객ID, 고객명, 주문일시
  from   주문상품
  where  주문일시 between :start and :end
  order by 상품번호
)
```

1번 SQL은 레코드당 107(=30+30+10+20+17) 바이트로 가공한 결과집합을 Sort Area에 담는다. 반면, 2번 SQL은 가공하지 않은 상태로 정렬을 완료하고 나서 최종 출력할 때 가공한다.
따라서 2번 SQL이 Sort Area를 훨씬 적게 사용한다.

아래 두 SQL 중에서는 어느 쪽이 Sort Area를 더 적게 사용할까?

#### [1번]
```
SELECT *
FROM   예수금원장
ORDER BY 총예수금 desc

Execution Plan
----------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=184K Card=2M Bytes=716M)
1   0    SORT (ORDER BY) (Cost=184K Card=2M Bytes=716M)
2   1      TABLE ACCESS (FULL) OF '예수금원장' (TABLE) (Cost=25K Card=2M Bytes=716M)
```

#### [2번]
```
SELECT 계좌번호, 총예수금
FROM   예수금원장
ORDER BY 총예수금 desc

Execution Plan
----------------------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS (Cost=31K Card=2M Bytes=17M)
1   0    SORT (ORDER BY) (Cost=31K Card=2M Bytes=17M)
2   1      TABLE ACCESS (FULL) OF '예수금원장' (TABLE) (Cost=24K Card=2M Bytes=17M)
```
당연히 2번 SQL이 적게 사용한다. 1번 SQL은 모든 컬럼을 Sort Area에 저장하는 반면, 2번 SQL은 계좌번호와 총예수금만 저장하기 때문이다.
실행계획에서 맨 우측열을 보면, 1번 SQL은 716MB, 2번 SQL은 17MB를 처리했다. 두 SQL 모두 테이블을 Full Scan 했으므로 읽은 데이터량은 똑같지만, 소트한 데이터량이 다르므로 성능도 다르다.
229개 컬럼을 가진 테이블로 직접 테스트해 본 결과, 1번 SQL은 14.41초, 2번 SQL은 1.2초가 소요되었다.

<br/>

## (2) Top N 쿼리의 소트 부하 경감 원리
Top N 쿼리에 대해서는 이전에 살펴봤다. Top N 쿼리에 소트 연산을 생략할 수 있도록 인덱스를 구성했을 때, 'Top N Stopkey' 알고리즘이 어떤 성능 효과를 가져다주는지를 살펴봤다.
이번에는 인덱스로 소트 연산을 생략할 수 없을 때, Top N 쿼리가 어떻게 작동하는지 설명할 차례다.

전교생 1,000명 중 가장 큰 학생 열 명을 선발하려고 한다. 다른 학교와 농구시합을 앞두고 있어서다. 어떤 방법이 있을까?

만약 전교생을 키 순서대로 정렬한 학생명부가 있다면 가장 위쪽에 있는 열 명을 선발하면 된다. 'Top N Stopkey' 알고리즘이다.
그런 학생명부를 미리 준비해 두지 않았다면, 아래와 같은 방법이 가장 효과적이지 않을까 싶다.

- [1] 전교생을 운동장에 집합시킨다.
- [2] 맨 앞줄 맨 왼쪽에 있는 학생 열 명을 단상 앞으로 불러 키 순서대로 세운다.
- [3] 나머지 990명을 한 명씩 교실로 들여보내면서 현재 Top 10 위치에 있는 학생과 키를 비교한다. 더 큰 학생이 나타나면, 현재 Top 10 위치에 있는 학생을 교실로 들여보낸다.
- [4] Top 10에 새로 진입한 학생 키에 맞춰 자리를 재배치한다.

전교생이 다 교실로 들어갈 때까지 3번과 4번 작업을 반복하면, 최종적으로 그 학교에서 가장 키 큰 학생 열 명만 운동장에 남게 된다. 지금 설명한 알고리즘을 지금부터 'Top N 소트' 알고리즘이라고 부르자.

이전에 설명했던 페이징 쿼리로 돌아가 보자.
```
select *
from  (
  select rownum no, a.*
  from
    (
      select 거래일시, 체결건수, 체결수량, 거래대금
      from   종목거래
      where  종목코드 = 'KR123456'
      and    거래일시 >= '20180304'
      order by 거래일시
    ) a
  where rownum <= (:page * 10)
)
where no >= (:page - 1)*10 + 1
```
아래는 인덱스로 소트 연산을 생략할 수 없어 Table Full Scan 방식으로 처리할 때의 SQL 트레이스다.
```
Call     Count  CPU Time Elapsed Time    Disk   Query   Current   Rows
------- ------ --------- ------------ ------- ------- --------- ------
Parse        1     0.000        0.000       0       0         0      0
Execute      1     0.000        0.000       0       0         0      0
Fetch        2     0.078        0.083       0     690         0     10
------- ------ --------- ------------ ------- ------- --------- ------
Total        4     0.078        0.084       0     690         0     10

Rows  Row Source Operation
----- --------------------------------------------------------------
    0 STATEMENT
   10   COUNT STOPKEY (cr=690 pr=0 pw=0 time=83318 us)
   10    VIEW (cr=690 pr=0 pw=0 time=83290 us)
   10      SORT ORDER BY STOPKEY (cr=690 pr=0 pw=0 time=83264 us)
49857        TABLE ACCESS FULL 종목거래 (cr=690 pr=0 pw=0 time=299191 us)
```
실행계획에 Sort Order By 오퍼레이션이 나타났다.
Table Full Scan 대신 종목코드가 선두인 인덱스를 사용할 수도 있지만, 바로 뒤 컬럼이 거래일시가 아니면 소트 연산을 생략할 수 없으므로 지금처럼 Sort Order By 오퍼레이션이 나타난다.

여기서 Sort Order By 옆에 'Stopkey'라고 표시된 부분을 주목하기 바란다.
소트 연산을 피할 수 없어 Sort Order By 오퍼레이션을 수행하지만 'Top N 소트' 알고리즘이 작동한다는 사실을 실행계획에 표시하고 있다.
이 알고리즘이 작동하면, 소트 연산(=값 비교) 횟수와 Sort Area 사용량을 최소화해 준다. 예를 들어, page 변수에 1을 입력하면 열 개 원소를 담을 배열(Array) 공간만 있으면 된다.

열 개짜리 배열로 최상위 열 개 레코드를 찾는 방법은 앞서 농구팀 선발 과정과 같다. 처음 읽은 열 개 레코드를 거래일시 오름차순(ASC)으로 정렬해서 배열에 담는다.
이후 읽는 레코드에 대해서는 배열 맨 끝(큰 쪽 끝)에 있는 값과 비교해서 그보다 작은 값이 나타날 때만 배열 내에서 다시 정렬한다. 기존에 맨 끝에 있던 값은 버린다.

코드를 다 정렬하지 않고도 오름차순(ASC)으로 최소값을 갖는 열 개 레코드를 정확히 찾아낼 수 있다. 이것이 'Top N 소트' 알고리즘이 소트 연산 횟수와 Sort Area 사용량을 줄여주는 원리다.

아래는 AutoTrace 결과다. 위에서 본 SQL 트레이스에서 Physical Read(=pr)와 Physical Write(=pw)가 전혀 발생하지 않았는데, 그 사실을 AutoTrace에서도 확인할 수 있다.
'physical reads'와 'sorts (disk)' 항목이 0이다.
```
Statistics
-----------------------
  0  recursive calls
  0  db block gets
690  consistent gets
  0  physical reads  **
...  ...
  1  sorts (memory)
  0  sorts (dist)    **
```
참고로 많은 공간이 필요 없다는 사실을 증명하기 위해 아래와 같이 Sort Area를 작게 설정하고 테스트하였다.
```
SQL> alter session set workarea_size_policy = manual;

SQL> alter session set sort_area_size = 524288;
```

<br/>

## (3) Top N 쿼리가 아닐 때 발생하는 소트 부하
SQL을 좀 더 간결하게 표현하기 위해 다음과 같이 Order By 아래 쪽 ROWNUM 조건절을 제거하고 수행해 보자.
```
select *
from  (
  select rownum no, a.*
  from
    (
      select 거래일시, 체결건수, 체결수량, 거래대금
      from   종목거래
      where  종목코드 = 'KR123456'
      and    거래일시 >= '20180304'
      order by 거래일시
    ) a
  where rownum <= (:page * 10)
)
where no between (:page-1)*10 + 1 and (:page * 10)
```
아래는 SQL 트레이스 결과다. Sort Area는 앞에서와 똑같이 설정하고 테스트하였다.
```
Call     Count  CPU Time Elapsed Time    Disk   Query   Current   Rows
------- ------ --------- ------------ ------- ------- --------- ------
Parse        1     0.000        0.000       0       0         0      0
Execute      1     0.000        0.000       0       0         0      0
Fetch        2     0.281        0.858     698     690        14     10
------- ------ --------- ------------ ------- ------- --------- ------
Total        4     0.281        0.858     698     690        14     10

Rows  Row Source Operation
----- --------------------------------------------------------------
    0 STATEMENT
   10   VIEW (cr=690 pr=698 pw=698 time=357962 us)
49857     COUNT (cr=690 pr=698 pw=698 time=1604327 us)
49857       VIEW (cr=690 pr=698 pw=698 time=1205452 us)
49857         SORT ORDER BY (cr=690 pr=698 pw=698 time=756723 us)
49857           TABLE ACCESS FULL 종목거래 (cr=690 pr=0 pw=0 time=249345 us)
```
실행계획에서 Stopkey가 사라졌다. 'Top N 소트' 알고리즘이 작동하지 않았다는 뜻이다. 그 결과로 Physical Read(pr=698)와 Physical Write(pw=698)가 발생했다.
같은 양(690 블록)의 데이터를 읽고 정렬을 수행했는데, 앞에서는 'Top N 소트' 알고리즘이 작동해 메모리 내에서 정렬을 완료했지만 지금은 디스크를 이용해야만 했다.

아래는 AutoTrace 결과다. 'sorts (disk)' 항목이 1이므로 정렬 과정에 Temp 테이블스페이스를 이용했다는 사실을 알 수 있다.
```
Statistics
-----------------------
  6  recursive calls
 14  db block gets
690  consistent gets
698  physical reads  **
...  ...
  0  sorts (memory)  **
  1  sorts (dist)    **
```

## (4) 분석함수에서의 Top N 소트
윈도우 함수 중 rank나 row_number 함수는 max 함수보다 소트 부하가 적다. Top N 소트 알고리즘이 작동하기 때문이다. 테스트를 통해 확인해 보자.

아래는 max 함수를 이용해 모든 장비에 대한 마지막 이력 레코드를 찾는 쿼리다.
```
select 장비번호, 변경일자, 변경순번, 상태코드, 메모
from  (select 장비번호, 변경일자, 변경순번, 상태코드, 메모
            , max(변경순번) over (partition by 장비번호) 최종변경순번
       from   상태변경이력
       where  변경일자 = :upd_dt)
where  변경순번 = 최종변경순번

Call     Count  CPU Time Elapsed Time    Disk   Query   Current   Rows
------- ------ --------- ------------ ------- ------- --------- ------
Parse        1     0.000        0.000       0       0         0      0
Execute      1     0.000        0.000       0       0         0      0
Fetch        2     2.750        9.175   13456    4536         9     10
------- ------ --------- ------------ ------- ------- --------- ------
Total        4     2.750        9.175   13456    4536         9     10

Rows  Row Source Operation
------ --------------------------------------------------------------
     0 STATEMENT
    10   VIEW (cr=4536 pr=13456 pw=8960 time=4437847 us)
498570     WINDOW SORT (cr=4536 pr=13456 pw=8960 time=9120662 us)
498570       TABLE ACCESS FULL 상태변경이력 (cr=4536 pr=0 pw=0 time=1994341 us)
```

Sort Area를 작게 설정한 상태에서 테스트했기 때문에 Window Sort 단계에서 13,456개 physical read(=pr)와 8,960개 physical write(=pw)가 발생했다.

같은 상황에서 max 대신 rank 함수를 사용해 보자.
```
select 장비번호, 변경일자, 변경순번, 상태코드, 메모
from  (select 장비번호, 변경일자, 변경순번, 상태코드, 메모
            , rank() over (partition by 장비번호 order by 변경순번 desc) rnum
       from   상태변경이력
       where  변경일자 = :upd_dt)
where  rnum = 1

Call     Count  CPU Time Elapsed Time    Disk   Query   Current   Rows
------- ------ --------- ------------ ------- ------- --------- ------
Parse        1     0.000        0.000       0       0         0      0
Execute      1     0.000        0.000       0       0         0      0
Fetch        2     0.969        1.062      40    4536        42     10
------- ------ --------- ------------ ------- ------- --------- ------
Total        4     0.969        1.062      40    4536        42     10

Rows  Row Source Operation
------ --------------------------------------------------------------
     0 STATEMENT
    10   VIEW (cr=4536 pr=40 pw=40 time=1061996 us)
   111     WINDOW SORT PUSHED RANK (cr=4536 pr=40 pw=40 time=1061971 us)
498570       TABLE ACCESS FULL 상태변경이력 (cr=4536 pr=0 pw=0 time=1495760 us)
```
여기서도 physical read와 physical write가 각각 40개씩 발생하긴 했지만 max 함수 쓸 때보다 훨씬 줄었다. 시간이 8초 가량 덜 소요된 것도 확인하기 바란다.