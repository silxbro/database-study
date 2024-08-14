# 06. IOT, 클러스터 테이블 활용

<br/>

## (1) IOT란?
이전에 테이블 액세스 없이 인덱스만 읽고 처리하도록 튜닝하는 기법에 대해 살펴보았다. Random 액세스가 발생하지 않도록 테이블을 아예 인덱스 구조로 생성하면 어떨까?
실제 오라클은 그런 식으로 테이블을 생성하는 방법을 제공하는데, 이를 'IOT(Index-Organized Table)'라고 부른다.

테이블을 찾아가기 위한 rowid를 갖는 일반 인덱스와 달리 IOT는 모든 행 데이터를 리프 블록에 저장하고 있다. IOT에서는 "인덱스 리프 블록이 곧 데이터 블록"인 셈이다.

테이블을 인덱스 구조로 만드는 구문은 아래와 같다.
```
create table index_org_t ( a number primary key, b varchar(10) )
organization index;
```
우리가 일반적으로 사용하는 테이블은 '힙 구조 테이블'이라고 부르며, 테이블 생성 시 대개 생략하지만 아래와 같이 organization 옵션을 명시할 수도 있다.
```
create table index_org_t ( a number primary key, b varchar(10) )
organization heap;
```
일반적인 힙 구조 테이블로의 데이터 삽입은 Random 방식으로 이루어진다. 즉, Freelist로부터 할당 받은 블록에 정해진 순서 없이 값을 입력한다.
반면, IOT는 인덱스 구조 테이블이므로 정렬 상태를 유지하며 데이터를 삽입한다.

IOT는 SQL 서버나 Sybase에서 말하는 '클러스터형 인덱스(Clustered Index)'와 비슷한 개념이라고 할 수 있다. 다만 오라클 IOT는 PK 컬럼 순으로만 정렬할 수 있다는 점이 다르다.
- SQL 서버는 Unique 하지 않은 일반 컬럼들을 정렬 기준으로 삼을 수 있는데, 그때는 내부적으로 생성한 일련번호를 함께 저장함으로써 레코드를 유일하게 식별할 수 있게 한다.
  그래야 클러스터형 인덱스를 가리키는 제2의 인덱스(비클러스터형 인덱스)로부터 해당 레코드를 찾아갈 수 있기 때문이다.

### [IOT의 장점과 단점]
IOT는 인위적으로 클러스터링 팩터를 좋게 만드는 방법 중 하나다.
같은 값을 가진 레코드들이 100% 정렬된 상태로 모여 있기 때문에 Random 액세스가 아닌 Sequential 방식으로 데이터를 액세스할 수 있고, 이 떄문에 넓은 범위를 액세스할 때 유리하다.

PK 컬럼 기준으로 데이터가 모여 있더라도 선행 컬럼이 '=' 조건이 아니면 조회 대상 레코드들이 서로 흩어져 많은 스캔을 유발하지만, 적어도 테이블 Random 액세스는 발생하지 않아 빠른 성능을 낼 수 있다.

테이블을 IOT로 생성하면 PK 인덱스를 위한 별도의 세그먼트를 생성하지 않아도 돼 저장공간을 절약하는 부수적인 이점도 생긴다.

IOT의 가장 큰 단점으로는 데이터 입력 시 성능이 느리다는 점을 주로 꼽는다. 이 때문에 IOT를 기피하는 설계자 또는 DBA를 자주 만나는데, 실제 테스트해 보면 그럴 정도로 늦지는 않다.
일반 힙 구조 테이블에 PK 인덱스를 생성하지 않았을 때와 비교하면 당연히 많은 차이가 나겠지만 똑같이 PK 인덱스를 두고 비교해 보면 차이가 거의 없음을 발견할 것이다.

그럼에도 둘 간의 성능 차이가 클 때는 인덱스 분할(Split) 발생량 차이 때문이다. IOT는 인덱스 구조이므로 중간에 꽉 찬 블록에 새로운 값을 입력할 일이 종종 생기고 그럴 때 인데스 분할(Split)이 발생한다.
그런데 IOT가 PK 이외에 많은 컬럼을 갖는다면 리프 블록에 저장해야 할 데이터량이 늘어나 그만큼 인덱스 분할 발생빈도도 높아진다.
컬럼 수가 그렇게 많은 테이블이라면 인덱스 스캔 효율 때문에라도 IOT 대상으로는 부적합하다.

IOT에 Direct Path Insert가 작동하지 않는 것도 큰 제약 중 하나이며, 이 때문에 성능이 느린 것은 어쩔 수가 없다.

<br/>

## (2) IOT, 언제 사용할 것인가?
IOT는 아래와 같은 상황에서 유용하다.
### [1] 크기가 작고 NL 조인으로 반복 룩업(Lookup)하는 테이블
코드성 테이블이 주로 여기에 속한다. NL 조인에서 Inner쪽 룩업 테이블로서 액세스되는 동안 건건이 인덱스와 테이블 블록을 다 읽는다면 비효율적이다.
따라서 그런 테이블을 IOT로 구성해 주면 적어도 테이블은 반복 액세스하지 않아도 된다.

다만, IOT 구성 시 PK 이외 속성의 크기 때문에 인덱스 높이(height)가 증가한다면 역효과가 날 수 있으므로 이를 반드시 확인하기 바란다.

### [2] 폭이 좁고 긴(=로우 수가 많은) 테이블
두 테이블 간 M:M 관계를 해소하기 위한 Association(=Intersection) 테이블이 주로 여기에 속한다.

예를 들어, 고객과 컨텐츠 테이블이 있고, Association 테이블로 컨텐츠방문 테이블이 있다고 하자. 고객이나 컨텐츠 테이블은 PK 이외에도 많은 컬럼으로 구성된다.
하지만 컨텐츠방문 테이블은 PK(고객ID, 컨텐츠ID, 방문일시) 이외 컬럼이 전혀 없거나 있더라도 아주 소수에 불과하다(=폭이 좁다).

PK 인덱스는 어차피 생성해야 할 테고, 그러면 테이블과 거의 중복된 데이터를 갖게 된다. 그럴 때 IOT로 구성해 주면 중복을 피할 수 있다.

### [3] 넓은 범위를 주로 검색하는 테이블
주로 Between, Like 같은 조건으로 넓은 범위를 검색하는 테이블이라면, IOT 구성을 고려해볼만하다. 특히, PK 이외 컬럼이 별로 없는 통계성 테이블에는 최적의 솔루션이라고 할 수 있다.

통계성 테이블은 주로 넓은 범위 조건으로 검색하는데다, 일반적으로 PK구성 컬럼은 많으면서 일반(Non-Key) 속성은 몇 개 되질 않는다.
PK 구성 컬럼이 많은 만큼 분석 관점(다차원 모델링에서 말하는 dimension)과 액세스 경로가 아주 다양한데, 이를 위해 B*Tree 결합 인덱스를 계속 추가해 나가는 것은 저장공간이나 DML 부하
측면에서 문제가 많다.

그럴 때 테이블을 IOT로 구성하면 효과적이다. 우선 PK 인덱스를 위한 별도 공간이 필요하지 않다는 점이 맘에 든다.
정렬 순서를 잘 정의하면 인덱스를 여러 개 두지 않고도 다양한 검색 조건에 대응할 수 있고, 무엇보다 테이블 Random 액세스가 전혀 발생하지 않는다는 점에서 강력하다.

정렬 순서는 어떻게 정하는 것이 좋을까? 통계성 테이블 검색 시 항상 사용되는 일자 컬럼은 대부분 between 조건이므로 선두 컬럼으로 부적합하다.
'=' 조건으로 항상 사용되는 컬럼 한두 개를 찾아 선두에 두고 바로 이어서 일자 컬럼이 오도록 IOT를 구성하는 것이 효과적이다.
범위검색 조건인 일자 뒤쪽에 놓인 컬럼이 '=' 조건으로 입력되더라도 스캔 범위를 줄이는 데에 큰 역할을 못해 생기는 비효율이 있긴 하지만 Random 액세스가 발생하지 않는 것만으로도 대부분 만족스러운
성능을 낼 수 있다.

데이터량이 너무 많아지면 인덱스 높이(height)가 증가하고 관리 부담이 커진다는 우려가 생길 수 있는데, 뒤에서 설명할 Partitioned IOT를 활용할 수 있으므로 걱정하지 않아도 된다.

참고로, 팩트(fact)성 테이블에 대한 인덱스 구성전략을 얘기할 때면 흔히 비트맵 인덱스가 거론되곤 한다.
하지만 여러 비트맵 인덱스로 Bit-Wise 오퍼레이션을 수행한 결과 테이블 액세스량이 획기적으로 줄어드는 경우가 아니라면 즉, Random 액세스 발생량이 많다면 비트맵 인덱스는 성능 개선에 그다지 도움이
되지 못한다.

- #### [비트맵 인덱스에 대한 오해]

  성별처럼 Distinct Value 개수가 적은 컬럼으로 조회할 때 비트맵 인덱스를 사용하면 빠르다과 생각하는 분들이 많다.
  즉, 넓은 범위를 조회할 때 B*Tree 인덱스보다 성능을 크게 향상시켜 준다는 얘긴데, 과연 그럴까?

  그런 컬럼이라야 비트맵 인덱스의 저장 효율이 좋은 것은 사실이지만 조회 성능이 그다지 좋지는 않다.
  테이블 Random 액세스 발생 측면에서는 B*Tree 인덱스와 똑같기 때문이며, 스캔할 인덱스 블록이 줄어드는 정도의 이점만 생긴다.

  요컨대, 하나의 비트맵 인덱스 단독으로는 쓰임새가 별로 없다.
  비트맵 인덱스가 가진 여러 가지 특징(특히, 용량이 작고 여러 인덱스를 동시에 사용할 수 있다는 특징) 때문에 읽기 위주의 대용량 DW 환경에 적합한 것일 뿐 대용량 데이터 조회에 유리한 것은 아님을
  기억하기 바란다.

### [4] 데이터 입력과 조회 패턴이 서로 다른 테이블
회사에 100명의 영업사원이 있다고 하자. 영업사원들의 일별 실적을 집계하는 테이블이 있는데, 한 블록에 100개 레코드가 담긴다. 그러면 매일 한 블록씩 1년이면 365개 블록이 생긴다.

이처럼 실적등록은 일자별로 진행되지만 실적조회는 주로 사원별로 이루어진다. 예를 들어, 일상적으로 아래 쿼리가 가장 많이 수행된다고 하자.
```
select substr(일자, 1, 6) 월도
     , sum(판매금액) 총판매금액, avg(판매금액) 평균판매금액
from   영업실적
where  사번 = 'S1234'
and    일자 between '20090101' and '20091231'
group by substr(일자, 1, 6)
```
그러면 인덱스를 경유해 사원마다 365개 테이블 블록을 읽어야 한다. 클러스터링 팩터가 매우 안 좋기 때문이며, 입력과 조회 패턴이 서로 달라서 생기는 현상이다.
그럴 때 아래와 같이 사번이 첫 번째 정렬 기준이 되도록 IOT를 구성해 주면, 한 블록만 읽고 처리할 수 있다.
```
create table 영업실적 ( 사번 varchar2(5), 일자 varchar2(8), ...
     , constraint 영업실적_PK primary key (사번, 일자) organization index;
```

<br/>

## (3) Partitioned IOT
수억 건에 이르는 일별상품별계좌별거래 테이블이 있다고 하자. 그리고 아래 쿼리 1처럼 넓은 범위의 거래일자를 기준으로 특정 상품을 조회하는 쿼리가 자주 수행된다.

### [쿼리 1]
```
select 거래일자, 지점번호, 계좌번호, sum(거래량), sum(거래금액)
from   일별상품별계좌별거래
where  상품번호 = 'P7006050009'
and    거래일자 between '20080101' and '20080630'
group by 거래일자, 지점번호, 계좌번호
```
상품별 거래건수가 워낙 많아 쿼리 1에 인덱스를 사용하면 Random 액세스 부하가 심하게 발생한다.
거래일자 기준으로 월별 Range 파티셔닝돼 있다면 인덱스를 이용하기보다 필요한 파티션만 Full Scan하는 편이 오히려 빠르겠지만 다른 종목의 거래 데이터까지 모두 읽는 비효율이 생긴다.

[상품번호 + 거래일자] 순으로 정렬되도록 IOT를 구성하면 읽기 성능이야 획기적으로 개선되겠지만, 수억 건에 이르는 테이블을 단일 IOT로 구성하는 것은 관리상 여간 부담스럽지 않다.

더구나, 아래 쿼리 2 때문에라도 그럴 수가 없다. 단일 IOT를 구성하면 쿼리 2는 수억 건에 이르는 데이터를 Full Scan 해야만 한다.

### [쿼리 2]
```
select 상품번호, 거래일자, sum(거래량), sum(거래금액)
from   일별상품별계좌별거래
where  거래일자 between '20080101' and '20080630'
group by 상품번호, 거래일자
```
이럴 때 'Partitioned IOT'를 구성하면 두 마리 토끼를 다 잡을 수 있다.
- 거래일자 기준 Range 파티셔닝
- [상품번호 + 거래일자] 순으로 PK를 정의하고, IOT 구성

쿼리 1을 위해서라면 입력한 거래일자 구간에 속하는 파티션 IOT를 골라 필요한 종목 거래 데이터만 스캔하면 되고, 쿼리 2을 위해서는 검색구간에 속하는 파티션 세그먼트 전체를 Full Scan하면 된다.

<br/>

## (4) Overflow 영역
앞에서 여러 차례 언급했듯이 PK 이외 컬럼이 많은 테이블일수록 IOT로 구성하기에 부적합하다. 인덱스 분할에 의한 DML 부하는 물론, 검색을 위한 스캔량도 늘어나기 때문이다.

그럼에도 성능 향상을 위해 IOT가 꼭 필요하다면 어떻게 해야 할까? 그럴 때 Overflow 기능이 도움이 될 수 있는데, 아래 사례를 보면서 설명해 보자.

```
CREATE TABLE 불공정거래적출 (
-- PK 속성
    종목코드           VARCHAR2(12) NOT NULL
  , 적출일자           VARCHAR2(8)  NOT NULL
  , 회원번호           VARCHAR2(5)  NOT NULL
  , 지점번호           VARCHAR2(5)  NOT NULL
  , 계좌번호           VARCHAR2(12) NOT NULL
  , 적출유형코드        VARCHAR2(2)  NOT NULL

  , 상품구분코드        VARCHAR2(1)  NOT NULL
  , 적출건수           NUMBER(15)   NOT NULL

- 시스템 관리 속성
  , 생성자ID          VARCHAR(10)  NOT NULL
  , 수정자ID          VARCHAR(10)  NOT NULL
  , 생성일시           VARCHAR(10)  NOT NULL
  , 수정일시           VARCHAR(10)  NOT NULL

  , CONSTRAINT 불공정거래적출_PK PRIMARY KEY
       ( 종목코드, 적출일자, 회원번호, 지점번호, 계좌번호, 적출유형코드 )
)
ORGANIZATION INDEX
OVERFLOW TABLESPACE TBS_OVERFL01
PCTTHRESHOLD 30
INCLUDING 적출건수 ;
```
위 불공정거래적출 테이블에서 생성자ID, 수정자ID, 생성일시, 수정일시는 시스템 내부적인 필요에 의해 생겨난 관리 속성이지 업무요건에 의한 것은 아니다.
따라서 값은 저장해 두지만 출력이나 조회조건으로는 거의 사용되지 않는다.

만약 이런 컬럼들을 다른 주요 컬럼과 분리 저장할 수 있다면 IOT 활용성을 높일 수 있는데, 다행히 오라클이 그런 기능을 제공한다.
위 스크립트 하단에 지정한 옵션들이 그것이며, 각각 아래와 같은 의미를 갖는다.

- **OVERFLOW TABLESPACE** : Overflow 세그먼트가 저장될 테이블스페이스를 지정한다. (SYS_IOT_OVER_38645 등의 이름을 가진 세그먼트가 자동으로 생성됨)
- **PCTTHRESHOLD** : DEFAULT 값은 50이다. 예를 들어 이 값이 30이면, 블록 크기의 30%를 초과하기 직전 컬럼까지만 인덱스 블록에 저장하고 그 뒤쪽 컬럼은 모두 Overflow 세그먼트에 저장한다.
  물론 로우 전체 크기가 지정된 비율 크기보다 작다면 모두 인덱스 블록에 저장한다.
  테이블을 생성하는 시점에 모든 컬럼의 데이터 타입 Max 길이를 합산한 크기가 이 비율 크기보다 작다면 Overflow 세그먼트는 불필요하지만 만약 초과된다면 오라클은 Overflow Tablespace 옵션을
  반드시 지정하도록 강제하는 에러를 던진다.
- **INCLUDING** : Including에 지정한 컬럼까지만 인덱스 블록에 저장하고 나머지는 무조건 Overflow 세그먼트에 저장한다.

오라클은 Pctthreshold 또는 Including 둘 중 하나를 만족하는 컬럼을 Overflow 영역에 저장한다.
즉, Including 이전에 위치한 컬럼이라도 Pcttreshold에 지정된 비율 크기를 초과한다면 Overflow 영역에 저장된다. 반대의 경우도 마찬가지다.

반드시 기억할 것은, Overflow 영역을 읽을 때도 건건이 Random 액세스가 발생한다는 사실이다.
따라서 Overflow 세그먼트에 저장된 컬럼 중 일부를 자주 액세스해야 하는 상황이 발생한다면 IOT 액세스 효율은 급격히 저하된다.
Pctthreshold를 신중히 선택해야 함은 물론 Including 절에 어떤 컬럼을 지정하느냐가 관건이라고 하겠다.

다행스러운 것은, Overflow 영역에도 버퍼 Pinning 효과가 나타나기 때문에 연속적으로 같은 Overflow 블록을 읽을 때는 Random 블록 I/O를 최소화할 수 있다.

<br/>

## (5) Secondary 인덱스
IOT에 추가적인 secondary 인덱스를 생성할 때 고려해야 할 몇 가지 주요 이슈가 있는데, 결론부터 말하면 IOT는 secondary 인덱스 추가 가능성이 크지 않을 때만 선택하는 것이 바람직하다.

오라클 secondary 인덱스를 설명하기에 앞서 MS-SQL 서버의 비클러스터형 인덱스 진화 과정부터 살펴보자.

### [MS-SQL 서버의 비클러스터형 인덱스 진화 과정]
SQL 서버에서 IOT처럼 인덱스 구조로 생성한 테이블을 '클러스터형 인덱스(Clustered Index)'라 부르고, 여기에 추가로 생성한 2차 인덱스들은 '비클러스터형 인덱스(Non-Clustered Index)'라고 부른다.

SQL 서버 6.5 이전에는 비클러스터형 인덱스가 클러스터형 인덱스 레코드를 직접 가리키는 rowid를 갖도록 하였다.
문제는, 인덱스 분할에 의해 클러스터형 인덱스 레코드 위치가 변경될 때마다 비클러스터형 인덱스(한 개 이상일 수 있음)가 갖는 rowid 정보를 모두 갱신해 주어야 한다는 데 있다.

실제로, DML 부하가 심하다고 느낀 마이크로소프트는 7.0 버전부터 비클러스터형 인덱스가 rowid 대신 클러스터형 인덱스의 키 값을 갖도록 구조를 변경하였다.
이제 키 값을 갱신하지 않는 한, 인덱스 분할 때문에 비클러스터형 인덱스를 갱신할 필요가 없어진 것이다.

그러나 세상에 공짜는 없는 법이다. DML 부하가 줄어든 대신, 비클러스터형 인덱스를 이용할 때 이전보다 더 많은 I/O가 발생하는 부작용을 떠안게 되었다.
비클러스터형 인덱스에서 읽히는 레코드마다 건건이 클러스터형 인덱스 수직 탐색을 반복하기 때문이다. 당연히 클러스터형 인덱스 높이(height)가 증가할수록 블록 I/O도 증가한다.

### [오라클 Logical Rowid]
오라클은 IOT를 개발하면서 SQL 서버 6.5와 7.0 이후 버전이 갖는 두 가지 액세스 방식을 모두 사용할 수 있도록 설계하였다.

IOT 레코드의 위치는 영구적이지 않기 때문에 오라클은 secondary 인덱스로부터 IOT 레코드를 가리킬 때 물리적 주소 대신 logical rowid를 사용한다.
logical rowid는 PK와 physical guess로 구성된다.
```
Logical rowid = PK + physical guess
```
physical guess는 secondary 인덱스를 "최초 생성하거나 재생성(Rebuild)한 시점"에 IOT 레코드가 위치했던 데이터 블록 주소(DBA)다.
인덱스 분할에 의해 IOT 레코드가 다른 블록으로 이동하더라도 secondray 인덱스에 저장된 physical guess 값은 갱신되지 않는다.
SQL 서버 6.5에서 발생한 것과 같은 DML 부하를 없애기 위함이고, 레코드 이동이 발생하면 정확한 값이 아닐 수 있기 때문에 'guess'란 표현을 사용한 것이다.

이처럼 두 가지 정보를 다 가짐으로써 오라클은 상황에 따라 다른 방식으로 IOT를 액세스할 수 있게 하였고, 경우에 따라서는 두 가지 방식을 다 사용할 때도 있다.

### [PCT_DIRECT_ACCESS]
dba/all/user_indexes 테이블을 조회하면 pct_direct_access 값을 확인할 수 있다.
이는 secondray 인덱스가 유효한 physical guess를 가진 비율(Direct 액세스 성공 비율)을 나타내는 지표로서, secondary 인덱스 탐색 효율을 결정짓는 매우 중요한 값이다.

통계정보 수집을 통해 얻어지는 이 값이 100% 미만이면 오라클은 바로 PK를 이용해 IOT를 탐색한다.
100%일 때만 physical guess를 이용하는데, 레코드를 찾아갔을 때 해당 레코드가 다른 곳으로 이동하고 없으면(PK 값 비교를 통해 이동 여부를 알 수 있음) PK로 다시 IOT를 탐색한다.
그런 비율이 높아지면 성능은 당연히 나빠진다.

인덱스를 최초 생성하거나 재생성(Rebuild)하고 나면 (통계정보를 따로 수집해 주지 않더라도) pct_direct_access 값은 100이다.
이때는 physical guess로 바로 액세스하고, 성공률도 100%이므로 비효율이 없다.

문제는, 휘발성이 강한(→ 레코드 위치가 자주 변하는) IOT의 경우 시간이 지나면서 physical guess에 의한 액세스 실패 확률이 높아져 성능이 점점 저하된다는 데에 있다.
그럴 때는 통계정보를 다시 수집해 pct_direct_access가 실제 physical guess 성공률(100 미만)을 반영하도록 해 주어야 한다.
그때부터 오라클은 physical guess를 거치지 않고 곧바로 PK로 IOT를 탐색할 것이다.

물론 아래처럼 인덱스를 Rebuild하거나 update block references 옵션을 이용해 physical guess를 주기적으로 갱신해 준다면 가장 효과적이다.
```
alter index iot_second_idx REBUILD;
alter index iot_second_idx UPDATE BLOCK REFRENCES;
```
secondary 인덱스 physical guess를 갱신하더라도 통계정보를 재수집한 이후부터 Direct 액세스로 전환된다는 사실을 기억하기 바란다.
인덱스 분할이 발생하더라도 통계정보를 재수집한 이후부터 PK를 이용하는 것과 마찬가지다.

Direct 액세스 성공 확률이 비교적 높은 상태에서 통계정보만 재수집하는 바람에 PK 액세스로 전환하는 일이 생겨도 문제다. 통계정보를 수집하는 순간 pct_direct_access가 100 미만으로 떨어지기 때문이다.

secondary 인덱스 튜닝 방안에 대해서는 잠시 후 다시 살펴보기로 하고, 지금은 IOT에 secondary 인덱스를 만들어 실제 트랜잭션과 통계정보 수집에 따라 pct_direct_access 값이 어떻게 바뀌는지 확인해보자.
```
SQL> create table t1 (
  2      cl number not null
  3    , c2 number
  4    , c3 number
  5    , c4 number
  6    , constraint  t1_pk primary key (c1)  )
  7  organization index;                → IOT 생성

SQL> create index t1_x1 on t1 (c2) ;    → secondary 인덱스 생성
```
IOT를 만들고 secondary 인덱스까지 만들었다. secondary 인덱스를 처음 생성하고 나면 통계정보 여부와 상관없이 pct_direct_access 통계치는 100이라고 했다.
이제 이 테이블에 1,000개 레코드를 입력해 보자.
```
SQL> insert into t1
  2  select rownum, rownum, rownum, rownum
  3  from all_objects
  4  where  rownum <= 1000;

SQL> commit;

SQL> select index_name, PCT_DIRECT_ACCESS
  2  from user_indexes
  3  where index_name = 'T1_X1';

INDEX_NAME                PCT_DIRECT_ACCESS
------------------------- -----------------
T1_X1                                   100
```
레코드를 입력했지만 pct_direct_access 값은 여전히 100이다. 이 상태에서 t1_x1 인덱스를 이용해 테이블을 액세스하면 physical guess가 활용될 것이다.

통계정보를 수집하고 pct_direct_access 값을 다시 확인해 보자.
```
SQl> exec dbms_stats.gather_index_stats(user, 't1_x1');

SQL> select index_name, PCT_DIRECT_ACCESS
  2  from user_indexes
  3  where index_name = 'T1_X1';

INDEX_NAME                PCT_DIRECT_ACCESS
------------------------- -----------------
T1_X1                                    64
```
64로 바뀌었고, 인덱스 분할에 의해 36% 정도 레코드가 다른 블록으로 이동했음을 의미한다.
만약 통계정보 수집 직전(pct_direct_access=100)에 t1_x1 인덱스를 이용해 테이블을 액세스해 보면 36% 정도는 Direct 액세스에 실패해 PK로 다시 IOT를 탐색했을 것이다.
이제는 pct_direct_access가 100 미만으로 떨어졌기 때문에 physical guess 활용 없이 PK로 IOT를 탐색한다.

이제 physical guess를 정확한 값으로 갱신해 보자.
```
SQL> alter index t1_x1 UPDATE BLOCK REFERENCES;

SQL> select index_name, PCT_DIRECT_ACCESS
  2  from user_indexes
  3  where index_name = 'T1_X1';

INDEX_NAME                PCT_DIRECT_ACCESS
------------------------- -----------------
T1_X1                                    64
```
physical guess를 갱신했지만 통계정보 상으로 pct_direct_access는 여전히 64%다. 이 상태에서 실제 Direct 액세스 성공률은 100%이겠지만 여전히 physical guess는 활용되지 않는다.
```
SQL> exec dbms_stats.gather_index_stats(user, 't1_x1');

SQL> select index_name, PCT_DIRECT_ACCESS
  2  from user_indexes
  3  where index_name = 'T1_X1';

INDEX_NAME                PCT_DIRECT_ACCESS
------------------------- -----------------
T1_X1                                   100
``` 
통계정보를 재수집하고 나니 pct_direct_access가 100으로 바뀐 것을 볼 수 있다. 이제 다시 t1_x1 인덱스로부터 physical guess를 이용한 Direct 액세스가 활성화된다.

- #### [PCT_DIRECT_ACCESS에 대한 오해]

  인덱스 통계상 pct_direct_access 값이 일정 비율 이하로 떨어지면, physical guess로 탐색했다가 PK로 다시 IOT를 스캔하는 비율이 높아져 성능이 떨어진다고 설명한 글을 본 적이 있다.

  하지만 지금까지 설명한 것처럼 이 값이 100% 미만이면 오라클은 PK를 이용해 바로 IOT를 탐색한다. 오히려 휘발성 IOT에서 이 값이 100%를 가리킬 때가 더 문제일 수 있다.

### [비휘발성 IOT에 대한 Secondary 인덱스 튜닝 방안]
비휘발성(읽기전용이거나 맨 우측 블록에만 값이 입력되어 IOT 레코드 위치가 거의 변하지 않는) 테이블이라면 Direct 액세스 성공률이 높을 것이다.
따라서 pct_direct_access 값이 100을 가리키도록 유지하는 것이 효과적인 튜닝 방안이다. 데이터가 쌓이는 양에 따라 한 달에 한 번 또는 일년에 한 번 정도만 physical guess를 갱신해주면 된다.

읽기전용 테이블이면 pct_direct_access 값이 100을 가리키도록 한 상태에서 더 이상 통계정보를 수집하지 않으면 되겠지만, 맨 우측에 지속적으로 값이 입력되는 경우라면 통계정보 수집이 필수적이다.
(뒤에서 보겠지만 Right-Growing IOT이더라도 pct_direct_access 값이 100이 아닐 수 있다.) 그럴 때는 통계정보 수집 직후에 아래 프로시저를 이용해 값을 직접 설정해 주면 된다.
```
SQL> exec dbms_stats.set_index_stats (user, 't1_x1', guessq => 100);
```
physical guess에 의한 Direct 액세스 성공률이 100%에 가깝다면 일반 테이블을 인덱스 rowid로 액세스할 때와 거의 같은 수준의 성능을 보이므로 secondary 인덱스를 추가하는 데 대한 부담을 덜 수 있다.

### [휘발성 IOT에 대한 Secondary 인덱스 튜닝 방안]
휘발성이 강한(IOT 레코드 위치가 자주 변하는) IOT에 secondary 인덱스를 추가할 때는 각별한 주의가 필요하고, 처음 IOT를 설계할 때부터 이에 대한 고려가 있어야 한다.

휘발성이어서 physical guess에 의한 Direct 액세스 성공률이 낮다면 두 가지 선택을 할 수 있다.

첫 번째는, 주기적으로 physical guess를 정확한 값으로 갱신해 주는 것으로서, 주로 secondary 인덱스 크기가 작을 때 쓸 수 있는 방법이다.

두 번째는, 아예 physical guess가 사용되지 못하도록 pct_direct_access 값을 100 미만으로 떨어뜨리는 것으로서, 인덱스 크기가 커서 주기적으로 physical guess를 갱신해 줄 수 없을 때
쓸 수 있는 방법이다. 인덱스 분할이 어느 정도 발생한 상태에서 통계정보를 수집해 주면 된다.

두 번째 방법을 쓰면 일반 테이블을 인덱스 rowid로 액세스할 때보다 느려지겠지만 선택도가 매우 낮은 secondary 인덱스 위주로 구성해 주면 큰 비효율은 없다.
예를 들어, [상품번호 + 거래일자 + 고객번호]가 PK인 주문 테이블을 IOT로 구성한다고 하자.
[상품번호 =, 거래일자 between] 조건으로 조회할 때는 넓은 범위의 주문 데이터를 액세스할 가능성이 높으므로 IOT를 통해 큰 성능 개선을 이룰 수 있다.

[고객번호 =, 거래일자 between] 조건에 대해서는 [고객번호 + 거래일자] 순으로 secondray 인덱스를 구성하면 되고 고객별 주문량이 대개 소수이므로 액세스 과정에서 발생하는 비효율은 미미할 것이다.

### [Right-Growing IOT에서 pct_direct_access가 100 미만으로 떨어지는 이유]
앞에서 트랜잭션과 통계정보 수집에 따라 pct_direct_access 값이 어떻게 바뀌는지 확인했는데, 강의 중에 이 테스트를 진행하고 나서 아래와 같은 질문을 받았다.
```
"t1 테이블에 값을 1부터 1,000까지 차례로 입력했기 때문에 기존 레코드 위치가 바뀔 이유가 없다. 그런데 왜 secondary 인덱스의 pct_direct_access 값이 64로 떨어졌는가?"
```
맨 우측 블록에만 값이 입력되는 Right-Growing IOT라면 인덱스 분할이 발생하더라도 기존 레코드의 주소 값이 바뀔 이유가 없는 것이 사실이다.
새로운 인덱스 블록을 맨 우측에 추가해 거기에 값을 입력하기 때문이다. 그런데도 이를 가리키는 secondary 인덱스의 physical guess 정확도가 떨어진 이유가 무엇일까?

이것은 인덱스 높이(height)가 2단계로 증가하면서 생기는 현상이다.

최초 IOT 인덱스 블록이 단 하나인 상태에서 블록이 꽉 차면 기존 100번 블록을 그대로 둔 채 101번 리프 블록과 102번 루트 블록이 새로 생길 것 같다.

100번 블록 레코드를 새로 할당한 101번 블록에 모두 복제하고 100번 블록은 루트 레벨로 올라간다. 그러고 나서 새로 추가되는 값들은 102번 리프 블록에 입력된다.
이 때문에 100번 블록을 가리키던 secondary 인덱스 physical guess가 모두 부정확해진 것이다.

이후 102번 블록 우측에 리프 블록이 계속 추가될 때는 IOT 레코드 이동이 발생하지 않는다. 우리가 기대했던 그대로다.
물론 값이 중간으로 들어와 101번 또는 102번 블록이 50:50으로 분할될 때는 레코드 이동이 발생한다.

시간이 흘러 인덱스 레벨이 한 단계 더 올라가는 순간, 아까처럼 100번 블록이 통째로 다른 블록으로 복제된다. 100번 블록 정보를 새로 할당한 103번 블록에 모두 복제하고, 100번 블록은 다시 루트 레벨로 올라간다.
하지만 이 경우에는 105번 블록이 추가될 뿐 다른 리프 블록에는 변화가 없어 secondary 인덱스 physical guess에도 영향을 주지 않는다.

오라클은 왜 이런 식으로 인덱스 레벨을 조정하는 걸까? 인덱스 루트 블록은 매우 특별한 블럭이다. 인덱스를 탐색할 때 항상(index fast full scan만 제외하고) 시작점으로 사용되기 때문이다.

Table Full Scan 시에는 매번 테이블 세그먼트 헤더로부터 익스텐트 정보를 얻고 스캔을 시작하지만 인덱스 스캔 시에는 실행계획에 담고 있는 루트 블록 주소로 곧바로 찾아간다.
따라서 루트 블록 주소가 바뀌면 해당 인덱스를 참조하는 많은 실행계획이 영향을 받게 돼 시스템에 엄청난 파급 효과를 일으킬 것이다.
(물론 실행계획이 만들어지는 하드파싱 시점에는 세그먼트 헤더를 읽어 루트 블록 주소값을 알아내지만 실행될 때마다 세그먼트 헤더를 조회하지는 않는다.)

### [IOT_REDUNDANT_PKEY_ELIM]
secondary 인덱스에는 physical guess와 함께 PK 컬럼 값을 저장하므로 만약 PK 컬럼 개수가 많은 IOT라면 데이터 중복 때문에 저장공간을 낭비하고 스캔 과정에서도 많은 비효율을 안게 된다.
secondray 인덱스가 여러 개면 문제는 더 심각해진다.

PK 컬럼을 저장하기 때문에 생기는 이점도 있는데, 아래 테스트 결과를 보자.
```
SQL> create table emp_iot
  2  ( empno, ename, job, mgr, hiredate, sal, com, deptno
  3  , constraint pk_emp_iot primary key ( empno ) )
  4  organization index
  5  as
  6  select * from scott.emp;

SQL> create index iot_secondary_index on emp_iot( ename );

SQL> set autotrace traceonly explain
SQL> select * from emp_iot where ename = 'SMITH';

Execution Plan
----------------------------------------------------------------
0      SELECT STATEMENT Optimizer=CHOOSE (Cost=3 Card=1 Bytes=87)
1   0    INDEX (UNIQUE SCAN) OF 'PK_EMP_IOT' (UNIQUE) (Cost=1 Card=1 Bytes=87)
2   1      INDEX (RANGE SCAN) OF 'IOT_SECONDARY_INDEX' (NON_UNIQUE) (Cost=1 Card=1)
```
secondary 인덱스 컬럼인 ename 조건으로 모든 컬럼(*)을 조회하니까 secondary 인덱스와 IOT를 모두 액세스했다.
이번에는 secondary 인덱스 키 컬럼(ename)과 PK 컬럼(empno)만 읽도록 해 보자.
```
SQL> selec empno from emp_iot where ename = 'SMITH';

Execution Plan
----------------------------------------------------------------
0      SELECT STATEMENT Optimizer=CHOOSE (Cost=3 Card=1 Bytes=87)
1   0    INDEX (RANGE SCAN) OF 'IOT_SECONDARY_INDEX' (NON_UNIQUE) (Cost=1 Card=1 ... )
```
secondary 인덱스만 읽고 처리를 완료하였다. (이를 통해 secondary 인덱스에 실제 PK 컬럼이 저장돼 있음도 확인되었다.)

여기서 한 가지 의문이 생길 수 있다. 컨텐츠방문 테이블을 예로 들어보자.
만약 이 테이블을 IOT로 구성하고 [컨텐츠ID + 방문일시]를 키(key)로 갖는 secondary 인덱스 '컨텐츠방문_X01'을 생성하면, 아래와 같이 컨텐츠ID, 방문일시 두 컬럼이 중복 저장될까?

|인덱스명|인덱스 키|Logical Rowid|
:---:|:---:|:---:|
|컨텐츠방문_X01|(컨텐츠ID, 방문일시)|(고객ID, 컨텐츠ID, 방문일시) + (physical guess)|

오라클은 secondary 인덱스의 Logical Rowid가 인덱스 키와 중복되면, 이를 제거하고 아래와 같이 고객ID만을 저장한다.

|인덱스명|인덱스 키|Logical Rowid|
:---:|:---:|:---:|
|컨텐츠방문_X01|(컨텐츠ID, 방문일시)|(고객ID) + (physical guess)|

dba/all/user_indexes를 조회하면 iot_redundant_pkey_elim 통계치를 볼 수 있는데, 이 값이 'YES'이면 secondary 인덱스 키와 PK 컬럼 간에 하나 이상 중복 컬럼이 있어 오라클이 이를
제거했음을 의미한다.

<br/>

## (6) 인덱스 클러스터 테이블
지금까지 테이블 Random 액세스를 줄이는 방안으로서 IOT를 소개하였고, 지금부터는 클러스터 테이블(clustered table)에 대해 설명하려고 한다.
클러스터 테이블에는 인덱스 클러스터와 해시 클러스터 두 가지가 있는데, 인덱스 클러스터부터 살펴보자.

인덱스 클러스터 테이블은 클러스터 키(여기서는 deptno) 값이 같은 레코드가 한 블록에 모이도록 저장하는 구조를 사용한다.
한 블록에 모두 담을 수 없을 때는 새로운 블록을 할당해 클러스터 체인으로 연결한다.

심지어 여러 테이블 레코드가 물리적으로 같이 저장될 수도 있다. 여러 테이블을 서로 조인된 상태로 저장해 두는 것인데, 일반적으로는 하나의 데이터 블록이 여러 테이블에 의해 공유될 수 없음을 상기하기 바란다.

이름 때문에 SQL 서버나 Sybase에서 말하는 '클러스터형 인덱스(Clustered Index)'와 같다고 생각할지 모르지만 클러스터형 인덱스는 오히려 IOT에 가깝다.
인덱스 클러스터는 키 값이 같은 데이터를 물리적으로 한 곳에 저장해 둘 뿐, IOT처럼 정렬하지는 않는다.

인덱스 클러스터 테이블을 구성하려면 먼저 아래와 같이 클러스터를 생성해야 한다.
```
SQL> create cluster c_deptno# ( deptno number(2) ) index;
```
그리고 클러스터에 테이블을 담기 전에 아래와 같이 클러스터 인덱스를 반드시 정의해야 한다.
왜냐하면, 클러스터 인덱스는 데이터 검색 용도로 사용될 뿐만 아니라 데이터가 저장될 위치를 찾을 때도 사용되기 때문이다.
```
SQL> create index i_deptno# on cluster c_deptno#;
```
클러스터 인덱스도 일반적인 B*Tree 인덱스 구조를 사용하지만, 해당 키 값을 저장하는 첫 번째 데이터 블록만을 가리킨다는 점에서 다르다.

클러스터 인덱스의 키 값은 항상 Unique(중복 값이 없음)하며, 테이블 레코드와 1:M 관계를 갖는다.
일반 테이블에 생성한 인덱스 레코드는 테이블 레코드와 1:1 대응 관계를 갖는다고 설명한 것을 기억할 것이다.

이런 구조적 특성 때문에 클러스터 인덱스를 스캔하면서 값을 찾을 때는 Random 액세스가 (클러스터 체인을 스캔하면서 발생하는 Random 액세스는 제외) 값 하나당 한 번씩 밖에 발생하지 않는다.
클러스터에 도달해서는 Sequential 방식으로 스캔하기 때문에 넓은 범위를 읽더라도 비효율이 없다는 게 핵심 원리다.

인덱스 클러스터 테이블에는 아래 두 가지 유형이 있다.
- 단일 테이블 인덱스 클러스터
- 다중 테이블 인덱스 클러스터

뒤에서 계속 단일 테이블 클러스터를 기준으로 설명하므로 여기서는 다중 테이블 클러스터를 생성하는 예제를 간단히 살펴보고 넘어가자.
인덱스 클러스터와 클러스터 인덱스는 이미 앞서 정의하였으므로 여기에 테이블을 담기만 하면 된다.

```
SQL> create table emp
  2  cluster c_deptno# (deptno)
  3  as
  4  select * from scott.emp;

SQL> create table dept
  2  cluster c_deptno# (deptno)
  3  as
  4  select * from scott.dept;

SQL> selectt owner, table_name from dba_tables where cluster_name = 'C_DEPTNO#';

OWNER            TABLE_NAME
---------------- -----------------------
ORAKING          DEPT
ORAKING          EMP
```
앞서 생성한 c_deptno# 클러스터에 dept와 emp 두 테이블이 같이 담기도록 정의하였다. 여기서, 클러스터에 생성한 i_deptno# 인덱스를 dept와 emp가 공유한다는 사실도 기억하기 바란다.
물론 클러스터 인덱스 외에 각 테이블별로 제2, 제3의 인덱스를 생성할 수도 있다.
```
SQL> break on deptno skip 1
SQL> select d.deptno, e.empno, e.ename
  2       , dbms_rowid.rowid_block_number(d.rowid) dept_block_no
  3       , dbms_rowid.rowid_block_number(e.rowid) emp_block_no
  4  from   dept d, emp e
  5  where  e.deptno = d.deptno
  6  order by d.deptno;

 DEPTNO     EMPNO ENAME      DEPT_BLOCK_NO EMP_BLOCK_NO
------- --------- ---------- ------------- ------------
     10      7934 MILLER               524          524
             7839 KING                 524          524
             7782 CLARK                524          524

     20      7788 SCOTT                527          527
             7566 JONES                527          527
             7369 SMITH                527          527
             7876 ADAMS                527          527
             7902 FORD                 527          527

     30      7499 ALLEN                528          528
             7521 WARD                 528          528
             7654 MARTIN               528          528
             7698 BLAKE                528          528
             7844 TURNER               528          528
             7900 JAMES                528          528

14 개의 행이 선택되었습니다.
```
위 결과를 통해 'deptno가 같은 dept, emp 레코드'가 '같은 블록'에 담긴 것을 알 수 있다.

### [인덱스 클러스터는 넓은 범위를 검색할 때 유리]
넓은 범위 데이터를 검색할 때 클러스터 테이블이 실제 유용한지 SQL 트레이스를 통해 확인해보자.
```
SQL> create cluster objs_cluster# ( object_type VARCHAR2(19) ) index ;

SQL> create index objs_cluster_idx on cluster objs_cluster#;

SQL> create table objs_cluster    → 클러스터 테이블
  2  cluster objs_cluster# ( object_type )
  3  as
  4  select * from all_objects
  5  order by dbms_random.value;

SQL> create table objs_regular    → 일반 테이블
  2  as
  3  select * from objs_cluster
  4  order by dbms_random.value;

SQL> create index objs_reguler_idx on objs_regular(object_type);

SQl> alter table objs_regular modify object_name null;

SQL> alter table objs_cluster modify object_name null;
```
인덱스 클러스터 테이블(objs_cluster)과 일반 힙 구조 테이블(objs_regular)을 만들었다. 이제 SQL 트레이스를 건 상태에서 object_type = 'TABLE' 조건으로 각 테이블을 조회해 보자.
```
select /*+ index(t objs_regular_idx) */ count(object_name)
from   objs_regular t
where  object_type = 'TABLE';

Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    1  SORT AGGREGATE (cr=642 pr=0 pw=0 time=5774 us)
 1763    TABLE ACCESS BY INDEX ROWID OBJS_REGULAR (cr=642 pr=0 pw=0 time=38863 us)
 1763      INDEX RANGE SCAN OBJS_REGULAR_IDX (cr=6 pr=0 pw=0 time=8922 us)
 ```
일반 B*Tree 인덱스와 힙 구조 테이블을 이용했더니 테이블을 1,763번 Random 액세스하는 동안 636(=642-6)개의 블록 I/O가 발생하였다.
```
select count(object_name)
from   objs_cluster t
where  object_type = 'TABLE';

Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    1  SORT AGGREGATE (cr=23 pr=0 pw=0 time=5774 us)
 1763    TABLE ACCESS CLUSTER OBJS_CLUSTER (cr=23 pr=0 pw=0 time=7075 us)
    1      INDEX UNIQUE SCAN OBJS_CLUSTER_IDX (cr=1 pr=0 pw=0 time=18 us)
 ```
이번에는 B*Tree 클러스터 인덱스를 통해 클러스터 테이블을 액세스했더니 Random 액세스는 단 1회만 발생하였고, 클러스터를 스캔하는 동안 22(=23-1)개의 블록 I/O가 발생하였다.

클러스터 인덱스를 '=' 조건으로 액세스할 때는 항상 Unique Scan이 나타난다는 사실에도 주목하기 바란다.

### [클러스터 테이블과 관련한 성능 이슈]
방금 본 것처럼 클러스터 테이블은 넓은 범위를 검색할 때 유리하다. 이런 장점에도 클러스터 테이블이 실무적으로 자주 활용되지 않는 이유는 DML 부하 때문이다.
(사실 이보다 더 큰 이유는 클러스터 테이블 기능을 잘 모른다는 데에 있다.)

IOT에서도 설명했듯이 일반적인 힙 구조 테이블에 데이터를 입력할 때는 Freelist로부터 할당받은 공간에 정해진 순서 없이 값을 입력한다. 반면, IOT는 정렬 상태를 유지하면서 값을 입력한다고 했다.
클러스터 테이블은 IO처럼 정렬 상태를 유지하지는 않지만 정해진 블록을 찾아서 값을 입력해야 하기 때문에 DML 성능이 다소 떨어진다.
특히, 전에 없던 값을 입력할 때는 블록을 새로 할당 받아야 하기 때문에 더 느리다.

하지만 클러스터를 구성하지 않는 대신 인덱스를 생성할거면 DML 부하는 어차피 비슷하다고 볼 수 있다.
특히 이미 블록이 할당된 클러스터 키 값을 입력할 때는 별 차이가 없고, 만약 계속 새로운 값이 입력돼 많이 느려진다면 클러스터 키를 잘못 선정한 경우라고 할 수 있다.
클러스터 테이블을 구성하면서 기존에 사용하던 인덱스 두세 개를 없앨 수 있다면 DML 부하가 오히려 감소할 수도 있다.

수정이 자주 발생하는 컬럼은 클러스터 키로 선정하지 않는 것이 좋지만, 삭제 작업 때문에 클러스터 테이블이 불리할 것은 없다.
다만, 전체 데이터를 지우거나 테이블을 통째로 Drop할 때 성능 문제가 생길 수 있다.

전체 데이터를 지울 때는 Truncate Table 문장을 쓰는 것이 빠른데, 클러스터 테이블에는 이 문장을 쓸 수가 없다. 단일 테이블 클러스터일 때도 마찬가지다.
또한 테이블을 Drop하려 할 때도 내부적으로 건건이 delete가 수행된다는 사실을 기억할 필요가 있다. 다중 테이블 클러스터를 생각해보면 너무 당연한 사실이다.

따라서 전체 데이터를 빠르게 지우고 싶을 때는 아래와 같이 클러스터를 Truncate 하거나 Drop 하는 것이 가장 빠르다.
물론 다중 테이블 클러스터일 때는 클러스터링된 테이블이 모두 삭제되므로 주의해야 한다.
```
truncate cluster objs_cluster#;
drop cluster objs_cluster# including tables;
```
DML 부하 외에 클러스터 테이블과 관련해 고려해야 할 성능 이슈로는 다음고 같은 것들이 있다.
- Direct Path Loading을 수행할 수 없다.
- 파티셔닝 기능을 함께 적용할 수 없다. IOT의 경우는 Partitioned IOT가 가능하고, 이를 통해 효과적으로 성능 문제를 해결한 사례를 앞에서 보았다.
- 다중 테이블 클러스터를 Full Scan할 때는 다른 테이블 데이터까지 스캔하기 때문에 불리하다.

### [SIZE 옵션]
클러스터 키 하나당 레코드 개수가 많지 않을 때 클러스터마다 한 블록씩 통째로 할당하는 것은 낭비다. 그래서 오라클은 하나의 블록에 여러 키 값이 같이 상주할 수 있도록 SIZE 옵션을 두었다.

SIZE 옵션은 한 블록에 여러 클러스터 키가 같이 담기더라도 하나당 가질 수 있는 최소 공간(바이트 단위)을 미리 예약하는 기능이다.
반대로 말하면, 하나의 블록에 담을 최대 클러스터 키 개수를 결정짓는다고 할 수 있다.
예를 들어, 블록 크기가 8KB일 때 SIZE 옵션으로 2000 바이트를 지정하면 한 블록당 최대 4개 클러스터 키만을 담을 수 있으며, 아래 수행 결과가 그런 사실을 잘 말해주고 있다.
```
SQL> show parameter block_size

NAME                                TYPE          VALUE
----------------------------------- ------------- ------
db_block_size                       integer       8192

SQL> drop cluster emp_cluster# including tables;

SQL> create cluster emp_cluster# ( empno number(4) ) pctfree 0 size 2000 index ;

SQL> create index emp_cluster_idx on cluster emp_cluster#;

SQL> create table emp
  2  cluster emp_cluster# ( empno )
  3  as
  4  select * from scott.emp;

SQL> select emp.empno, emp.ename, dbms_rowid.rowid_block_number(rowid) block_no
  2  from   emp;

    EMPNO ENAME        BLOCK_NO
--------- ----------- ---------
     7902 FORD            68404
     7934 MILLER          68404    (2개)
     7369 SMITH           68406
     7499 ALLEN           68406
     7521 WARD            68406
     7566 JONES           68406    (4개)
     7654 MARTIN          68407
     7698 BLAKE           68407
     7782 CLARK           68407
     7788 SCOTT           68407    (4개)
     7839 KING            68408
     7844 TURNER          68408
     7876 ADAMS           68408
     7900 JAMES           68408    (4개)
```
emp 레코드 하나가 2,000 바이트에 한참 못 미치지만 각 블록당 4개의 empno를 입력하고는 새로운 블록에 담기 시작한 것을 볼 수 있다.

하나의 블록에 담을 "최대" 클러스터 키 개수를 결정짓는다는 것은 4개 미만의 키 값이 담길 수도 있다는 뜻이기도 하다.
예를 들어, 같은 empno를 가진 1,000바이트 크기의 emp 레코드를 차례로 입력하면 아래와 같이 8개 레코드가 한 블록에 모두 담길 수 있다.
```
SQL> drop table emp purge;

SQL> create table emp
  2  cluster emp_cluster# ( empno )
  3  as
  4  select empno, ename, lpad('*', 970) data  → 한 로우가 1000 바이트쯤 되도록
  5  from   scott.emp, (select rownum no from dual connect by level <= 10)
  6  where  empno = 7900;

SQL> select empno, ename, dbms_rowid.rowid_block_number(rowid) block_no
  2  from   emp ;

    EMPNO ENAME        BLOCK_NO
--------- ----------- ---------
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68406
     7900 JAMES           68407
     7900 JAMES           68407
```
위 테스트 결과를 통해, 같은 블록 내에 공간이 있다면(최대 클러스터 키 개수를 초과하지 않는 범위 내에서) 계속 그곳에 저장하고, 그 블록마저 차면 새로운 블록을 할당해서 계속 저장한다는 사실을 알 수 있다.
SIZE 옵션은 공간을 미리 예약해 두는 것일 뿐 그 크기를 초과했다고 값을 저장하지 못하도록 하지는 않는다는 것이다.

SIZE 옵션 때문에 데이터 입력이 방해 받지는 않지만 대부분 클러스터 키 값이 한 블록씩을 초과한다면 굳이 이 옵션을 두어 클러스터 체인이 발생하도록 할 이유는 없다.
조금 전처럼 같은 값이 한꺼번에 입력된다면 클러스터 체인을 최소화할 수 있지만 그것은 운이 좋았을 뿐이다.
같은 키 값을 가진 데이터가 물리적으로 서로 모여서 저장되도록 하려고 클러스터 테이블을 사용하는 것인데, 이 옵션을 너무 작게 설정하면 그 효과가 반감된다는 사실을 기억하자.

반대로 너무 크게 설정하면 공간을 낭비할 수 있으며, 판단 기준은 클러스터 키마다의 평균 데이터 크기다. 참고로, SIZE 옵션을 지정하지 않으면 한 블록에 하나의 클러스터 키만 담긴다.

<br/>

## (7) 해시 클러스터 테이블
해시 클러스터 테이블은 해시 함수에서 반환된 값이 같은 데이터를 물리적으로 함께 저장하는 구조다. 클러스터 키로 데이터를 검색하고 저장할 위치를 찾을 때는 해시 함수를 사용한다.
해시 함수가 인덱스 역할을 대신하는 것이며, 해싱 알고리즘을 이용해 클러스터 키 값을 데이터 블록 주소로 변환해 준다.

해시 클러스터 테이블도 인덱스 클러스터 테이블처럼 아래 두 가지 유형이 있다.
- 단일 테이블 해시 클러스터
- 다중 테이블 해시 클러스터

해시 클러스터의 가장 큰 제약사항은 '=' 검색만 가능하다는 점이다. 따라서 거의 대부분 '=' 조건으로만 검색되는 컬럼을 해시 키로 선정해야 한다.

물리적인 인덱스를 따로 갖고 있지 않기 때문에 해시 클러스터 키로 검색할 때는 그만큼 블록 I/O가 덜 발생한다는 이점이 생긴다. 아래 테스트가 그런 사실을 잘 말해주고 있다.
```
SQL> create cluster username_cluster# ( username varchar2(30) )
  2  hashkeys 100 size 50;

SQL> create table user_cluster
  2  cluster username_cluster# ( username )
  3  as
  4  select * from all_users;

SQL> create table user_regular as select * from all_users;

SQL> create unique index user_regular_idx on user_regular(username);

SQL> alter table user_regular modify user_id null;
SQL> alter table user_cluster modify user_id null;

SQL> alter session set sql_trace = true;

SQL> declare
  2    1_user_id user_regular.user_id%type;
  3  begin
  4    for c in (selectt owner from objs_regular where owner <> 'PUBLIC')
  5    loop
  6      select user_id into 1_user_id from user_regular where username = c.owner;
  7      select user_id into 1_user_id from user_cluster where username = c.owner;
  8    end loop;
  9  end;
 10  /
```
아래는 위 스크립트를 수행할 때의 SQL 트레이스 결과다.
```
SELECT USER_ID FROM USER_REGULAR WHERE USER_NAME = :B1

call      count       cpu      elapsed    disk     query    current   rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1      0.00         0.00       0         0          0      0
Execute   30101      0.75         0.64       0         0          0      0
Fetch     30101      0.81         0.81       0     60202          0  30101
-------- ------ --------- ------------ ------- --------- ---------- ------
Total     60203      1.56         1.45       0     60202          0  30101
  
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
30101  TABLE ACCESS BY INDEX ROWID USER_REGULAR (cr=60202 pr=0 pw=0 time=924994 ...)
30101    INDEX UNIQUE SCAN USER_REGULAR_IDX (cr=30101 pr=0 pw=0 time=383440 us)


SELECT USER_ID FROM USER_CLUSTER WHERE USER_NAME = :B1

call      count       cpu      elapsed    disk     query    current   rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1      0.00         0.00       0         0          0      0
Execute   30101      0.34         0.40       0         3          0      0
Fetch     30101      0.62         0.48       0     30101          0  30101
-------- ------ --------- ------------ ------- --------- ---------- ------
Total     60203      0.96         0.89       0     30104          0  30101
  
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
30101  TABLE ACCESS HASH USER_CLUSTER (cr=30101 pr=0 pw=0 time=410828 us)
 ```

<br/>

## (8) IOT와 클러스터 테이블을 동시에 적용한 튜닝 사례
마지막으로, IOT와 클러스터 테이블을 동시에 적용해 튜닝했던 사례를 소개하고자 한다.
```
select a.카드번호, a.고객id, a.고객명, a.발급일자
     , a.성별, a.생년월일, a.결혼기념일, a.핸드폰번호, a.전화번호
     , a.우편번호, a.우편번호주소, a.상세주소
     , a.최종구매일자, a.구매횟수, a.구매금액 구매금액1, x.구매금액 구매금액2
     , a.누적포인트, a.실사용가능포인트
from  (select x.고객id, x.관리지점, sum(구매금액) 구매금액
       from   고객별품목별구매내역 x
       where  x.관리지점 = 'A6123E'
       and    x.구매일자 between '20010101' and '20090430'
       and    x.구매지점 not in ('AAAAAA', 'BBBBBB')
       and    (x.품목 = '0001' or x.품목 = '0002')
       and    not (x.품목 = '0003' or x.품목 = '0004')
       group by x.고객id, x.관리지점) x
     , 고객마스터 a
where  x.관리지점 = 'A6123E'
and    a.고객id = x.고객id
and    a.관리지점 = x.관리지점
and    a.전화확인 = '01'
and    a.sms거부 = 'N'
and    a.핸드폰번호 is not null

Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.005       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch      1108    12.970      292.657   81233    121571          0  11062
-------- ------ --------- ------------ ------- --------- ---------- ------
Total      1110    12.970      292.662   81233    121571          0  11062
  
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
11062  NESTED LOOPS (cr=121571 r=81233 w=0 time=292379730 us)
14661    VIEW (cr=76364 r=68123 w=0 time=252514157 us)
14661      SORT GROUP BY (cr=76364 r=68123 w=0 time=252503412 us)
19099        TABLE ACCESS BY INDEX ROWID 고객별품목별구매내역 (cr=76364 r=68123 ...)
75773          INDEX RANGE SCAN 고객별품목별구매내역_PK (cr=853 r=853 w=0 time= ...)
11062    TABLE ACCESS BY INDEX ROWID 고객마스터 (cr=45207 r=13110 w=0 time= ...)
14661      INDEX UNIQUE SCAN 고객마스터_PK (cr=30431 r=10849 w=0 time= ...)
```
위 쿼리에 사용된 두 테이블 인덱스 구성은 다음과 같다.
```
- 고객마스터_PK : 고객ID
- 고객별품목별구매내역_PK : 관리지점 + 구매일자 + 품목 + 고객ID + 구매지점
- 고객별품목별구매내역_X01 : 고객ID + 구매일자
```
쿼리를 보면 고객마스터와 고객별품목별구매내역 모두 지점별로 데이터가 관리된다는 사실을 알 수 있다. 따라서 관리지점별로 대량의 데이터를 집계하는 경우가 많다.

그런데 관리지점별 고객 수가 수만 명에 이르고 구매일자 검색범위에 해당하는 구매내역도 소만 건에 이르기 때문에 NL 조인으로는 좋은 성능을 기대하기 어려운 상황이다.

해시 조인이나 소트 머지 조인으로 유도하더라도 마찬가지다. 관리지점 조건으로 각각 인덱스를 이용해 테이블을 액세스하는 과정에서 이미 많은 Random I/O가 발생하기 때문이다.

이런 상황에서 어떻게 튜닝하는 것이 효과적일까? 우선 고객마스터는 관리지점 기준으로 클러스터 테이블을 구성하는 것이 가장 최적의 솔루션이다.

문제는 고객별품목별구매내역 테이블에 있다. 이런 형태의 거래 테이블은 일자 컬럼 기준으로 Range 파티션을 구성하는 것이 일반적이지만, 불필요한 관리지점까지 Full Scan하는 비효율이 있다.

관리지점마다 따로 저장되도록 'Range + 리스트' 결합 파티셔닝을 생각해 볼 수 있지만 관리지점이 추가될 때마다 파티션 구성을 변경해 주어야 하는 관리 부담이 생긴다.
게다가 일 단위 파티션이 아닌 한, 검색범위에 포함되지 않는 일자까지 모두 Full Scan하는 비용도 적지 않을 것이다.

그래서 관리지점, 구매일자가 정렬 기준 선두 컬럼이 되도록 IOT를 구성하기로 결정하였다.
```
- 고객마스터_PK : 고객ID
- 고객마스터_X01 : 관리지점 (→ 클러스터 키 인덱스)
- 고객별품목별구매내역_PK : 관리지점 + 구매일자 + 품목 + 고객ID + 구매지점 (→ IOT)
- 고객별품목별구매내역_X01 : 고객ID + 구매일자
```
아래는 IOT와 클러스터 테이블을 구성하고 나서 SQL 트레이스를 수집한 것이다.
```
select /*+ leading(a) use_hash(x) */ → 해시 조인으로 유도
       a.카드번호, a.고객id, a.고객명, a.발급일자, ...
from   ( ... ) x, 고객마스터 a
where  ...

Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.005       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch      1108     1.120        2.803    2995      2995          0  11062
-------- ------ --------- ------------ ------- --------- ---------- ------
Total      1110     1.120        2.803    2995      2995          0  11062
  
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
11062  HASH JOIN (cr=2995 r=2995 w=0 time=2619887 us)
61185    TABLE ACCESS CLUSTER 고객마스터 (cr=2142 r=2142 w=0 time=1776145 us)
    1      INDEX UNIQUE SCAN 고객마스터_X01 (cr=1 r=1 w=0 time=647 us)
14661    VIEW (cr=853 r=853 w=0 time=537164 us)
14661      SORT GROUP BY (cr=853 r=853 w=0 time=527150 us)
19099        INDEX RANGE SCAN 고객별품목별구매내역_PK (cr=853 r=853 w=0 time= ... )
```
참고로, 레코드 건수로 봐서는 고객별품목별구매내역을 Build Input으로 삼고 고객마스터를 Probe Input으로 삼는 게 유리해 보인다.
하지만 고객마스터를 Probe Input으로 사용하면 1,108번의 Fetch Call마다 클러스터 스캔이 발생해 오히려 블록 I/O가 늘어난다.
반면, 고객별품목별구매내역은 sort group by를 거쳐 PGA에 저장된 중간 집합이므로 Fetch 시 추가적인 블록 I/O가 발생하지 않는다.