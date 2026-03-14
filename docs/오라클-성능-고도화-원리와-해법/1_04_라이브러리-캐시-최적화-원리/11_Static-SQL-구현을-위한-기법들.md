# 11. Static SQL 구현을 위한 기법들

Dynamic SQL를 자주 사용하게 되는 첫 번째 이유를 앞에서 살펴 보았다. 개발팀에 문의했을 때 두 번째로 많이 나온 사례는 조건절에 IN-List 항목이 가변적으로 변할 때였다.
이를 포함해 Dynamic SQL을 Static SQL로 바꾸는 기법을 몇 가지 소개하려고 한다.

미리 얘기하면, 여기서 소개하는 사례들은 Dynamic SQL로 작성하더라도 생성될 수 있는 최대 SQL 개수가 그다지 많지 않기 때문에 라이브러리 캐시에 그렇게 많은 부하를 주지는 않는다.
앞에서 보았던, 사용자가 선택적으로 입력한 검색조건에 따라 SQL이 다양하게 바뀌는 사례도 마찬가지였다.

문제는, 이런저런 이유로 Dynamic SQL을 사용하는 순간 조건절 비교 값까지 습관적으로 Literal 상수 값을 사용하도록 개발한다는 데에 있다.
그런 뜻에서, 꼭 필요한 경우가 아니라면 가급적 Static SQL로 작성하는 습관과 능력을 기를 수 있도록 몇 가지 사례를 공유하려는 것이다.
누구나 쉽게 생각해 낼 수 있는 단순한 방법들이지만 이런 사례들을 접함으로써 SQL을 Static하게 구현하는 것이 결코 어려운 일이 아님을 전달하고자 한다.
<br/>
<br/>
## (1) IN-List 항목이 가변적이지만 최대 경우 수가 적은 경우
LP 회원을 선택하는 팝업 창에서 사용자가 선택한 LP회원 목록을 선택하고자 할 때, Static 방식으로 SQL을 작성하려면 아래 7개 SQL을 미리 작성해 두어야 한다.

```
select * from LP회원 where 회원번호 in ('01')
select * from LP회원 where 회원번호 in ('01', '02')
select * from LP회원 where 회원번호 in ('01', '03')
select * from LP회원 where 회원번호 in ('01', '02', '03')
select * from LP회원 where 회원번호 in ('02')
select * from LP회원 where 회원번호 in ('02', '03')
select * from LP회원 where 회원번호 in ('03')
```

선택 가능한 회원수가 4개로 늘어나면 15개, 5개면 31개의 SQL이 필요하다.
SQL을 미리 작성해 두고 경우에 따라 그 중 하나를 선택하도록 프로그래밍하는 것은 귀찮은 일이므로 Dynamic SQL을 사용하는 손쉬운 방법을 선택하게 된다.

하지만 이를 Static SQL로 구현하기 위한 해법은 허무하게도 아래와 같이 간단하다.
```
select * from LP회원 where 회원번호 in ( :a, :b, :c )
```

사용자가 입력하지 않은 항목에 null 값을 입력하면 자동으로 결과집합에서 제외된다. 그런데도 IN-List를 동적으로 생성하도록 Dynamic SQL을 사용하는 경우를 종종 보게 되므로 여기서 언급하는 것이다.

상황에 따라 아래처럼 decode문을 사용해야 할 때도 있다. 4개의 체크박스에 각각 내부적으로 'all', '01', '02', '03' 값이 부여돼 있다고 가정하고 작성한 것이다.

```
select * from LP회원
where 회원번호 in ( decode(:a, 'all', '01', :b)
                 ,decode(:a, 'all', '02', :c)
                 ,decode(:a, 'all', '03', :d)
                )
```
<br/>

## (2) IN-List 항목이 가변적이고 최대 경우 수가 아주 많은 경우
IN-List 항목이 가변적이고 최대 개수도 고정적인 것은 앞 사례와 동일하다. 하지만 가능한 경우 수가 너무 많아 Static SQL로 일일이 작성해 두는 것은 불가능해 보인다.
그렇다고 바인드 변수를 사용하는 것도 쉬운 일은 아니다. 실행 시 바인드 변수에 값을 입력하는 코딩을 그만큼 많이 해야 하기 때문이다. 좀 더 쉽게 구현하는 방법이 없을까?

문제를 풀려면 발상의 전환이 필요하다. SQL 조건절에는 대개 좌변에 컬럼을 두고 우변에는 그것과 비교할 상수 또는 변수를 위치시킨다. 하지만 여기서는 생각을 바꿔 컬럼과 변수 위치를 서로 바꿔보자.

```
select * from 수시공시내역
where  공시일자 = :일자
and    :inlist like '%' || 분류코드 || '%'
```

조건절을 위와 같이 작성하고 사용자가 선택한 분류코드를 ',(comma)' 등 구분자로 연결해 아래처럼 String형 변수에 담아서 바인딩하고 실행시키면 된다.

```
:inlist := '01,03,08,14,17,24,33,46,48,53'
```

의외로 문제가 쉽게 풀린 느낌이다. 참고로, 문자열을 처리하는 오라클 내부 알고리즘 상 like 연산자보다 instr 함수를 사용하면 더 빠르므로 아래와 같이 작성할 것을 권고한다.

```
select * from 수시공시내역
where  공시일자 = :일자
and    INSTR(:inlist, 분류코드) > 0
```

인덱스 원리에 익숙하다면 like, instr 둘 다 컬럼을 가공한 형태이므로 분류코드를 인덱스 액세스 조건으로 사용 못해 성능상 문제가 될 수 있음을 직감했을 것이다. 이는 인덱스 구성에 따라 얘기가 달라진다.

```
[1] IN-List를 사용할 때
select * from 수시공시내역
where  공시일자 = :일자
and    분류코드 in ( ... )

[2] like 또는 instr 함수를 사용할 때
select * from 수시공시내역
where  공시일자 = :일자
and    INSTR(:inlist, 분류코드) >
```

인덱스 구성이 [분류코드 + 공시일자]일 때는 당연히 1번이 유리하다. 2번 SQL문은 이 인덱스를 사용하지 못하거나 Index Full Scan 해야만 한다.

하지만 인덱스 구성이 [공시일자 + 분류코드]일 때는 상황에 따라 다르다.
사용자가 선택한 항목 개수가 소수일 때는 1번이 유리하지만 다수일 때는 인덱스를 그만큼 여러 번 탐침해야 하므로 2번이 유리할 수도 있다.
2번 SQL의 유・불리는 인덱스 깊이(루트에서 리프블록에 도달하기까지 거치는 블록 수)와 데이터 분포에 따라 결정될 것이다.
하루치 데이터가 수십만건 이상 되는 경우가 아니라면 대개는 2번처럼 분류코드가 인덱스 필터 조건으로 사용되는 것만으로도 과도한 테이블 랜덤 액세스를 제거할 수 있어 만족할만한 성능을 얻을 수 있다.

어쩄든, 사용자가 대개 소수 항목만으로 조회하거나 인덱스 구성이 [분류코드 + 공시일자]일 때는 인덱스를 좀더 효율적으로 액세스할 수 있는 방법을 강구해야 한다. 어떤 방법이 있는지 살펴보자.

#### < 방법1 >
```
select /*+ ordered use_nl(B) */ B.*
from ( select 분류코드
       from   수시공시분류
       where  INSTR(:inlist, 분류코드) > 0 ) A
     , 수시공시내역 B
where B.분류코드 = A.분류코드
```

위 체크항목들을 화면에 뿌리기 위해 사용한 수시공시분류 테이블은 100개 이내의 작은 테이블일 것이므로 Full Scan으로 읽더라도 비효율이 없다.
작은 테이블을 Full Scan으로 읽으면서 NL 조인 방식으로 분류코드 값을 수시공시내역 테이블에 던져 준다.
그러면 수시공시내역 테이블에 있는 인덱스를 정상적으로 이용하면서 원하는 결과집합을 빠르게 얻을 수 있다.

#### < 방법2 >
```
select /*+ ordered use_nl(B) */ B.*
from  (select substr(:inlist, (rownum-1)*2+1, 2)
       from   수시공시분류
       where  rownum <= length(:inlist) / 2) A
     , 수시공시내역 B
where B.분류코드 = A.분류코드
```

여기서도 드라이빙 집합으로서 수시공시분류 테이블을 사용했지만, 화면에 뿌려진 항목 개수 이상의 레코드를 갖는다면 어떤 테이블을 사용해도 무방하다.

이 SQL을 실행할 때는 사용자가 선택한 분류코드를 아래처럼 구분자 없이 연결해 String형 변수에 담아서 바인딩한다.
```
:inlist := '01030814172433464853'
```

인라인뷰 A만 먼저 실행시켜 보면, 아래 10개 레코드를 갖는 집합이 된다. 코드값은 대개 고정길이를 사용한다는 데에 착안한 것이며, 여기서는 2자리라고 가정하고 있다.
코드값 길이가 3자리면, 위 SQL문에서 숫자 2를 모두 3으로 바꿔주기만 하면 된다.

```
01
03
08
14
17
24
33
46
48
53
```

이를 드라이빙 집합으로 삼아 수시공시내역 테이블을 NL 방식으로 조인한다면 <방법1>과 마찬가지로 수시공시내역 테이블에 있는 인덱스를 정상적으로 이용할 수 있다.

#### < 방법3 >
```
select /*+ ordered use_nl(B) */ B.*
from (select substr(:inlist, (level-1)*2+1, 2)
      from   dual
      connect by level <= length(:inlist) / 2) A
    , 수시공시내역 B
where B.분류코드 = A.분류코드
```

<방법2>와 같은 방식이라고 할 수 있다. 다만, dual 테이블을 이용해 집합을 동적으로 생성한다는 점만 다르다.
<br/>
<br/>
## (3) 체크 조건 적용이 가변적인 경우
```
select 회원번호, SUM(체결건수), SUM(체결수량), SUM(거래대금)
from   일별거래실적 e
where  거래일자 = :trd_dd
and    시장구분 = '유가'
and  exists (
   select 'x'
   from  종목
   where 종목코드 = e.종목코드
   and   코스피종목편입여부 = 'Y'
)
group by 회원번호
```

주식종목에 대한 회원사(=증권사)별 거래실적을 집계하는 쿼리다. Exists 서브쿼리는 코스피(KOSPI)에 편입된 종목만을 대상으로 거래실적을 집계하고자 할 때만 동적으로 추가된다.

사실 이 케이스는 라이브러리 캐시 최적화와는 그다지 상관이 없다. 나올 수 있는 경우의 수가 두 개뿐이기 때문이다. 그럼에도 Static SQL로 구현해 보려는 노력 속에 SQL을 아래와 같이 작성한 경우를 보았다.
문제는, 아이디어는 괜찮은데 성능이 오히려 이전보다 나빠진다는 데에 있다.

```
select 회원번호, SUM(체결건수), SUM(체결수량), SUM(거래대금)
from   일별거래실적 e
where  거래일자 = :trd_dd
and    시장구분 = '유가'
and  exists (
   select 'x'
   from   종목
   where  종목코드 = e.종목코드
   and    코스피종목편입여부 = decode(:check_yn, 'Y, 'Y', 코스피종목편입여부)
)
group by 회원번호
```

사용자가 코스피 종목 편입 여부와 상관없이 전 종목을 대상으로 집계하려고 할 때는 서브쿼리를 수행할 필요가 없다.
그럼에도 위와 같이 무리하게 SQL을 통합함으로써 항상 서브쿼리를 수행해야만 하는 비효율을 안게 된 것이다. SQL 트레이스 결과를 통해 확인해 보자.

### [전 종목을 대상으로 집계하고자 할 때]
아래에서 블록 I/O가 8,518개 발생한 것을 볼 수 있다. 지면관계상 Exists 절을 빼고 쿼리했을 때의 트레이스 결과를 보여주진 않았지만 600여 개에 불과했었다.
불필요한 서브쿼리를 수행함으로써 이전보다 성능이 더 떨어진 것이다.

```
:trd_dd := '20071228'
:check_yn := 'N'

Call     Count CPU Time Elapsed Time    Disk   Query  Current    Rows
------- ------ -------- ------------  ------ ------- -------- -------
Parse        1    0.000        0.000       0       0        0       0
Execute      1    0.000        0.000       0       0        0       0
Fetch      264    0.000        0.058       0    8518        0    2623
------- ------ -------- ------------  ------ ------- -------- -------
Total      266    0.000        0.058       0    8518        0    2623

Rows  Row Source Operation
----  -----------------------------------------------------
   0  STATEMENT
2623   FILTER  (cr=8518 pr=0 pw=0 time=36781 us)
2627    PARTITION RANGE SINGLE PARTITION: 20 20 (cr=641 pr=0 pw=0 time=107547 us)
2627     TABLE ACCESS BY LOCAL INDEX ROWID 일별거래실적 PARTITION: 20 20 (cr=641 ...)
2627      INDEX RANGE SCAN 일별거래실적_X01 PARTITION: 20 20 (cr=274 pr=0 ... )
2627    TABLE ACCESS BY INDEX ROWID 종목 (cr=7877 pr=0 pw=0 time=25166 us)
2623     INDEX RANGE SCAN 종목_PK (cr=5254 pr=0 pw=0 time=16450 us) (Object ID 79702)
```

### [사용자가 코스피 편입 종목만으로 집계하고자 할 때]
이 경우에는 쿼리를 통합했다고 손해를 보지는 않는다. 통합하기 전과 블록 I/O가 같다.

```
:trd_dd := '20071228'
:check_yn := 'Y'

Call     Count CPU Time Elapsed Time    Disk   Query  Current    Rows
------- ------ -------- ------------  ------ ------- -------- -------
Parse        1    0.000        0.000       0       0        0       0
Execute      1    0.000        0.000       0       0        0       0
Fetch       85    0.020        0.045       0    8208        0     839
------- ------ -------- ------------  ------ ------- -------- -------
Total       87    0.020        0.045       0    8208        0     839

Rows  Row Source Operation
----  -----------------------------------------------------
   0  STATEMENT
 839   FILTER  (cr=8208 pr=0 pw=0 time=13410 us)
2627    PARTITION RANGE SINGLE PARTITION: 20 20 (cr=293 pr=0 pw=0 time=5296 us)
2627     TABLE ACCESS BY LOCAL INDEX ROWID 일별거래실적 PARTITION: 20 20 (cr=293 ...)
2627      INDEX RANGE SCAN 일별거래실적_X01 PARTITION: 20 20 (cr=97 pr=0 pw=0 ... )
 839    TABLE ACCESS BY INDEX ROWID 종목 (cr=7915 pr=0 pw=0 time=27118 us)
2623     INDEX RANGE SCAN 종목_PK (cr=5292 pr=0 pw=0 time=18069 us) (Object ID 79702)
```

위 수행 결과를 볼 때, 2개 SQL로 분리하는 편이 낫다.

Static SQL 구현 능력을 향상시키는 차원에서 굳이 하나의 SQL로 통합하고 싶다면, 아래와 같이 함으로써 두 마리 토끼를 다 잡을 수 있다. 즉, I/O 효율과 라이브러리 캐시 효율을 모두 달성 가능하다.

```
select 회원번호, SUM(체결건수), SUM(체결수량), SUM(거래대금)
from   일별거래실적 e
where  거래일자 = :trd_dd
and    시장구분 = '유가'
and  exists (
    select 'x' from dual where :check_yn = 'N'
    union all
    select 'x'
    from 종목
    where 종목코드 = e.종목코드
    and   코스피종목편입여부 = 'Y'
    and   :check_yn = 'Y'
)
group by 회원번호
```

Exists 서브쿼리는 존재여부만 체크하는 것이므로 그 안에 union all을 사용하면 조인에 성공하는 첫 번째 레코드를 만나는 순간 더는 진행하지 않고 true를 리턴한다.
SQL 트레이스를 통해 앞에서 수행했던 결과와 비교해 보자.

### [전 종목을 대상으로 집계하고자 할 때]
위 쿼리가 실행될 때 :check_yn 변수에 'N' 값이 들어오면, 즉 코스피에 편입되지 않은 종목까지 모두 읽고자 한다면 union all 아래쪽 종목 테이블과는 조인을 시도하지 않고 곧바로 Exists 서브쿼리를
빠져 나온다. 10g부터 DUAL 테이블은 FAST DUAL 방식으로 수행되기 때문에 블록 I/O는 발생하지 않는다.

따라서 위처럼 SQL을 통합하면 아래 트레이스 결과에서 보듯이 블록 I/O가 통합하기 이전과 똑같고, 개발자가 작성한 것보다는 7.5% 수준으로 감소했다.
(차이를 느낄 수 없을 정도로 미미하긴 하지만 통합하기 전보다 필터링을 위해 CPU를 약간 더 소모한다는 사실은 알고 있자.)

```
:trd_dd := '20081228'
:check_yn := 'N'

Call     Count CPU Time Elapsed Time    Disk   Query  Current    Rows
------- ------ -------- ------------  ------ ------- -------- -------
Parse        1    0.000        0.000       0       0        0       0
Execute      1    0.000        0.000       0       0        0       0
Fetch      264    0.000        0.035       0     641        0    2627
------- ------ -------- ------------  ------ ------- -------- -------
Total      266    0.000        0.035       0     641        0    2627

Rows  Row Source Operation
----  -----------------------------------------------------
   0  STATEMENT
2627   FILTER  (cr=641 pr=0 pw=0 time=18443 us)
2627    PARTITION RANGE SINGLE PARTITION: 20 20 (cr=641 pr=0 pw=0 time=7924 us)
2627     TABLE ACCESS BY LOCAL INDEX ROWID 일별거래실적 PARTITION: 20 20 (cr=641 ...)
2627      INDEX RANGE SCAN 일별거래실적_X01 PARTITION: 20 20 (cr=274 pr=0 ... )
2627    UNION-ALL  (cr=0 pr=0 pw=0 time=9061 us)
2627     FILTER  (cr=0 pr=0 pw=0 time=4659 us)
2627      FAST DUAL  (cr=0 pr=0 pw=0 time=1874 us)
   0     TABLE ACCESS BY INDEX ROWID 종목  (cr=0 pr=0 pw=0 time=0 us)
   0      INDEX RANGE SCAN 종목_PK (cr=0 pr=0 pw=0 time=0 us) (Object ID 79702)
```

참고로, 9i 이전 버전이었다면 차라리 두 개 SQL로 분리하는 게 나을 수도 있다.
필터 처리 시 캐싱 효과가 있긴 하지만 최소한 해당 일자에 거래가 있었던 종목코드 개수만큼 DUAL 테이블을 반복적으로 읽어야 하기 때문이다.
DUAL 테이블을 Full Scan 방식으로 읽게 되므로 건건이 세그먼트 헤더를 포함해서 보통 2\~4개 정도의 블록 I/O를 추가로 일으키게 된다.

### [사용자가 코스피 편입 종목만으로 집계하고자 할 때]
이 경우는 통합 전에도 어차피 서브쿼리를 수행했어야 하므로 동일한 성능을 보인다.

```
:trd_dd := '20071228'
:check_yn := 'Y'

Call     Count CPU Time Elapsed Time    Disk   Query  Current    Rows
------- ------ -------- ------------  ------ ------- -------- -------
Parse        1    0.000        0.000       0       0        0       0
Execute      1    0.000        0.000       0       0        0       0
Fetch       85    0.040        0.053       0    8208        0     839
------- ------ -------- ------------  ------ ------- -------- -------
Total       87    0.040        0.053       0    8208        0     839

Rows  Row Source Operation
----  -----------------------------------------------------
   0  STATEMENT
 839   FILTER  (cr=8208 pr=0 pw=0 time=15177 us)
2627    PARTITION RANGE SINGLE PARTITION: 20 20 (cr=293 pr=0 pw=0 time=7919 us)
2627     TABLE ACCESS BY LOCAL INDEX ROWID 일별거래실적 PARTITION: 20 20 (cr=293 ...)
2627      INDEX RANGE SCAN 일별거래실적_X01 PARTITION: 20 20 (cr=97 pr=0 pw=0 ... )
 839    UNION-ALL  (cr=7915 pr=0 pw=0 time=36922 us)
   0     FILTER  (cr=0 pr=0 pw=0 time=1817 us)
   0      FAST DUAL  (cr=0 pr=0 pw=0 time=0 us)
 839     TABLE ACCESS BY INDEX ROWID 종목 (cr=7915 pr=0 pw=0 time=26760 us)
2623      INDEX RANGE SCAN 종목_PK (cr=5292 pr=0 pw=0 time=17769 us)
```
<br/>

## (4) select-list가 동적으로 바뀌는 경우
사용자 선택에 따라 화면에 출력해야 할 항목이 달라지는 경우가 종종 있다. 아래는 그런 요건 때문에 SQL을 동적으로 구성할 수 밖에 없었다면서 어떤 개발팀에서 가져온 사례다.

```
/* 1 : 평균 2: 합계 */
if ( pfmStrCmpTrim(INPUT->inData.gubun,"1",1) == 0 ){
  snprintf(..., " avg(계약수), avg(계약금액), avg(미결제약정금액) ");
} else {
  snprintf(..., " sum(계약수), sum(계약금액), avg(미결제약정금액) ");
}
```

이것을 Static SQL로 바꾸는 것은 너무 쉽다. decode 함수 또는 case 구문을 활용하면 된다.

```
/* 1 : 평균 2: 합계 */
decode(:gubun, '1', avg(계약수), sum(계약수)),
decode(:gubun, '1', avg(계약금액), sum(계약금액)),
decode(:gubun, '1', avg(미결제약정금액), sum(미결제약정금액)),
```

아래는 또 다른 개발팀에서 가져온 사례인데, 위의 것과 동일하다.
```
/*1:개별 2: 누적*/
if ( pfmStrCmpTrim(INPUT->inData.gubun,"1",1) == 0 ){
  snprintf(..., "sum(거래량) 거래량, sum(거래대금) 거래대금");
} else {
  snprintf(..., "sum(누적거래량) 거래량, sum(누적거래대금) 거래대금");
```

아래처럼 Static SQL로 바꾸면 된다.

```
/*1:개별 2: 누적*/
decode(:gubun, '1', sum(거래량), sum(누적거래량)) 거래량,
decode(:gubun, '1', sum(거래대금), sum(누적거래대금)) 거래대금
```

주의할 것은, 결과가 같더라도 아래와 같이 작성하면 decode 함수를 집계대상 건수만큼 반복 수행하게 되므로 성능이 더 나빠질 수 있다는 사실이다.
```
/*1:개별 2: 누락*/
sum(decode(:gubun, '1', 거래량, 누적거래량)) 거래량,
sum(decode(:gubun, '1', 거래대금, 누적거래대금)) 거래대금
```
<br/>

## (5) 연산자가 바뀌는 경우
조건절에 사용되는 입력항목이나 출력항목이 바뀌는 게아니라 비교 연산자가 그때그때 달라지는 경우는 어떻게 Static SQL로 구현할 수 있을까?

미만, 이하, 이상, 초과 중 사용자가 선택하는 항목이 무엇이냐에 따라 <, <=, >, >= 4가지 중 하나의 연산자로 바꿔야 하므로 Dynamic SQL이 불가피하다고 생각할 수 있다.
하지만 누구나 조금만 고민해 보면 쉽게 해법을 찾을 수 있다. 아래처럼 SQL을 작성하고 바인딩하는 값을 바꾸면 된다.

```
where 거래미형성률       between :min1 and :max1
and   일평균거래량       between :min2 and :max2
and   일평균거래대금      between :min3 and :max3
and   호가스프레드비율    between :min4 and :max4
and   가격연속성         between :min5 and :max5
and   시장심도          between :min6 and :max6
and   거래체결률         between :min7 and :max7
```

각 컬럼은 아래와 같은 도메인에 따라 표준화된 데이터 타입과 자릿수를 할당받는다.
|도메인|데이터 타입|
|:---:|:---:|
|거래량|NUMBER(9)|
|거래대금|NUMER(15)|
|가격연속성|NUMBER(5, 2)|
|......|......|

일평균거래량을 예로 들면, 거래량 도메인은 9자리 숫자형이고 정수 값만 허용하므로 입력 가능한 최소값은 0, 최대값은 999,999,999이다.
따라서 사용자가 1000주를 입력하면 사용자가 선택한 비교 연산자에 따라 아래와 같이 Between 시작값과 종료값을 바인딩하면 된다.

|구분|between 시작값(:min3)|between 종료값(:max3)|
|:---:|---:|---:|
|이하|0|1000|
|미만|0|999|
|이상|1000|999999999|
|초과|1001|999999999|

정수혀이 아닌 가격연속성을 하나 더 예로 들면, 가격연속성 도메인은 소수점 이하 2자리를 갖는 총 5자리 숫자형이므로 입력 가능한 최소값은 0.00, 최대값은 999.99이다.
따라서 사용자가 50%를 입력하면 사용자가 선택한 비교 연산자에 따라 아래와 같이 Between 시작값과 종료값을 바인딩하면 된다.

|구분|between 시작값(:min5)|between 종료값(:max5)|
|:---:|---:|---:|
|이하|0.00|50.00|
|미만|0.00|49.99|
|이상|50.00|999.99|
|초과|50.01|999.99|