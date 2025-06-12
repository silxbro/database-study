# 10. Dynamic SQL 사용 기준
<br/>

## (1) Dynamic SQL 사용에 관한 기본 원칙
지금까지 설명한 라이브러리 캐시 최적화 원리와 Static, Dynamic SQL 정의에 비추어 Dynamic SQL 사용에 관한 기본 원칙을 다음과 같이 정리할 수 있다.

- [1] Static SQL을 지원하는 개발환경이라면 Static SQL로 작성하는 것을 원칙으로 한다. Static SQL은 PreCompile 과정을 거치므로 런타임 시 안정적인 프로그램 Build가 가능하다는 장점이 있다.
  그리고 Dynamic SQL을 사용하면 애플리케이션 커서 캐싱 기능이 작동하지 않는 경우가 있는데, 이 기능이 필요한 상황(예를 들어, 루프 내에서 반복 수행되는 쿼리)에서 Dynamic SQL을 사용하면 성능이
  나빠지기 때문이다.

- [2] 아래 경우에는 Dynamic SQL를 사용해도 무방하다.
    - PreCompile 과정에서 컴파일 에러가 나는 구문을 사용할 때. 예를 들어, Pro*C에서 스칼라 서브쿼리, 분석함수, ANSI 조인 등
    - 상황과 조건에 따라 생성될 수 있는 SQL 최대 개수가 많아 Static SQL로 일일이 나눠서 작성하려면 개발 생산성이 저하되고 유지보수 비용이 매우 커질 때

- [3] [2]번 경우에 해당해서 Dynamic SQL를 사용하더라도 조건절에는 바인드 변수를 사용하는 것을 원칙으로 한다.
  특히, 사용빈도가 높고 조건절 컬럼의 값 종류가 매우 많을 때(예를 들어, 계좌번호, 상품번호, 회원번호 등)는 반드시 준수한다.

- [4] [3]번 바인드 변수 사용원칙을 준수하되 아래 경우는 예외적으로 인정한다.
    - 배치 프로그램이나 DW, OLAP 등 정보계 시스템에서 사용되는 Long Running 쿼리.
      이들 쿼리는 파싱 소요시간이 쿼리 총 소요시간에서 차지하는 비중이 매우 낮고, 수행빈도가 낮아 하드파싱에 의한 라이브러리 캐시 부하를 유발할 가능성이 적음
    - OLTP성 애플리케이션이더라도 사용빈도가 매우 낮아 하드파싱에 의한 라이브러리 캐시 부하를 유발할 가능성이 없을 때.
      예외적으로 인정하는 것이므로 단순히 바인드 변수 정의하는 게 귀찮다고 그렇게 해서는 안됨
    - 조건절 컬럼의 값 종류(Distinct Value)가 소수일 때. 특히 값 분포가 균일하지 않아 옵티마이저가 컬럼 히스토그램 정보를 활용하도록 유도하고자 할 때.
        - [예] 증권시장구분코드 = { '유가', '코스닥', '주식파생', '상품파생' }

Static(=Embedded) SQL을 지원하지 않는 개발 환경이라면 모든 SQL이 Dynamic SQL이지만 런타임 시 SQL이 동적으로 바뀌도록 개발하는 것만큼은 삼가야 한다.
그런 환경에서는 Static과 Dynamic SQL을 편의상 아래와 같이 재정의하고, 위에서 제시한 기본 원칙을 동일하게 적용할 것을 권고한다.

- Static SQL : SQL Repository(주로 XML 파일 형태로 관리)에 완성된 형태로 저장한 SQL
- Dynamic SQL : SQL Respository에 불완전한 형태로 저장한 후 런타임 시 상황과 조건에 따라 동적으로 생성되도록 작성한 SQL
  <br/><br/>

## (2) 기본 원칙이 잘 지켜지지 않는 첫 번째 이유, 선택적 검색 조건
이렇게 Dynamic SQL 사용 원칙을 정하고 개발을 시작해도, 중간에 점검해 보면 여전히 잘 지켜지지 않는다.
Static SQL을 지원하는 개발 환경에서조차 자주 Dynamic SQL를 사용해 조건절을 동적으로 구성한다.
그 원인을 개발팀에 물어 보면, 가장 많은 비중을 차지하는 것이 \[그림 4-13\](p.317)처럼 검색 조건이 다양해 사용자 선택에 따라 조건절이 동적으로 바뀌는 경우다.

크기가 작은 테이블이라면 상관없겠지만 대상 테이블이 대용량인데도 화면을 이처럼 설계하는 것은 참 무책임하다는 생각을 하게 된다.
이렇게 다양한 검색 조건을 한 화면에서 처리해 준다고 하면 업무 담당자 입장에서는 싫다고 마다할 리 없다. 심지어는 필수입력 항목도 전혀 없고, 검색기간도 무제한이다.
그런데 성능은 누가 담보할 것인가? 조회 버튼을 누를 때마다 10분씩 소요된다면 기능이 좋다고 해서 만족할 사용자는 아마 없을 것이다.
항상 개발 막바지에 가서 이런 화면들 때문에 성능 이슈가 불거지는 것을 많이 목격했다.

대용량 테이블에 대한 조회 요건이 이렇게 복잡하다면 적어도 현업과의 협의를 통해 필수입력 항목을 수렴해서 프로그램에 반영해야 한다.
또한 기간 조건(Between, <, >, <=, >= 등)에 대해서는 입력 값 범위를 가능한 짧게 제한하려는 노력을 반드시 해야 한다. 업무 담당자를 만나서 인터뷰해 보면 의외로 문제가 쉽게 풀리는 경우가 많다.
설계 단계에서부터 성능을 고려한 업무 요건 도출이 필요한 이유가 여기에 있다.
<br/><br/>

현업과의 협의를 거쳐 필수항목 위주로 화면을 구성하더라도 업무요건 상 조회 조건이 다양한 경우는 있기 마련이다. 그럴 때 SQL을 효과적으로 작성하려면 어떻게 접근해야 하는지 같이 고민해 보자.
\[그림 4-14\](p.318)처럼 조금은 단순화시킨 사례를 가지고 얘기를 진행하려고 한다.

거래일자만 필수입력 조건이고, 나머지는 모두 선택적 입력 조건이라고 가정하자.

SQL 작성에 대한 가이드가 없는 상황에서 위와 같은 조회 프로그램을 구현할 때면 개발자는 십중팔구 아래와 같이 SQL을 작성한다.
```
select 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
from   일별종목거래
where  거래일자 between :시작일자 and :종료일자
%option
```

그러고 나서 사용자의 선택이나 입력 값에 따라 %option 부분에 조건절을 아래와 같이 동적으로 붙여나간다.

```
%option = " and 종목코드 = 'KR123456' and 투자자유형코드 = '1000' "
```

SQL 작성 표준이 존재하는 대형 개발 프로젝트라면 당연히 Dynamic SQL 사용을 제한하므로 표준을 준수하려고 아래와 같은 방법을 사용하게 된다.
참고로, NVL을 사용하는 것 외에 몇 가지 방법이 더 있는데, 각각의 장단점에 대해서는 뒤에서 다시 다루기로 하겠다.

```
select 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
from   일별종목거래
where  거래일자 between :시작일자 and :종료일자
and    종목코드 = nvl(:종목코드, 종목코드)
and    투자자유형코드 = nvl(:투자자유형코드, 투자자유형코드)
and    주문매체구분코드 = nvl(:주문매체구분코드, 주문매체구분코드)
```

이렇게 코딩하면 사용자가 어떻게 값을 입력하더라도 단 한 개의 실행계획을 공유하면서 반복 재사용하게 되므로 라이브러리 캐시 효율 측면에서는 최상의 선택이다. 하지만 쿼리 수행 속도 측면에서는 어떤가?
인덱스를 이용한 액세스가 효과적인 상황에서 낭패를 보게 된다. 인덱스를 전혀 사용 못 하거나(거래일자가 인덱스 선두 컬럼이 아닐 때) 사용하더라도 비효율적으로 사용하기 때문이다.

라이브러리 캐시 효율과 I/O 효율을 모두 고려하면서 SQL을 개발하려면, 남아 있는 방법은 한 가지뿐이다. 종목코드, 투자자유형코드, 주문매체구분코드 입력여부에 따라 SQL을 모두 분리해서 개발하는 것이다.
그럴 경우 이 3가지 선택적 입력 조건을 처리하는 데에만 모두 8개의 SQL을 따로 작성해야 한다.

- [1] 거래일자 between
- [2] 거래일자 between, 종목코드 =
- [3] 거래일자 between, 종목코드 =, 투자자유형코드 =
- [4] 거래일자 between, 종목코드 =, 투자자유형코드 =, 주문매체구분코드 =
- [5] 거래일자 between, 종목코드 =, 주문매체구분코드 =
- [6] 거래일자 between, 투자자유형코드 =
- [7] 거래일자 between, 투자자유형코드 =, 주문매체구분코드 =
- [8] 거래일자 between, 주문매체구분코드 =

여기서는 SQL이 간단하므로 8개쯤 작성하는 게 대수롭지 않을 수 있지만 실무에서 작성되는 SQL은 대개 짧으면 수십 라인, 길면 수백 라인에 이른다. 몇 천 줄짜리 SQL도 가끔 본다.
빠듯한 개발 일정에 쫓기는 프로젝트에서 이렇게 개발하라고 하는 것은 현실성이 떨어지며 불가능에 가깝다.

인덱스 원리를 잘 아는 SQL 튜닝 전문가 시각에서 보면, 위 요건을 만족하려고 SQL을 8개나 만들 필요는 없다.
변별력이 좋지 않은 컬럼은 인덱스 액세스 효율에 도움이 되지 않으므로, 인덱스 구성을 고려해 변별력이 좋은 컬럼 중심으로 2\~3개 SQL로 분기하면 된다.
애플리케이션에서 IF문을 이용해 분기하거나 아래처럼 union all을 사용하는 방법이 있다.

> < 인덱스 구성 >
>
> - INDEX01  :  종목코드 + 거래일자
> - INDEX02  :  투자자유형코드 + 거래일자 + 주문매체구분코드
> - INDEX03  :  거래일자 + 주문매체구분코드
```
select 거래일자, 투자자유형코드, 회원번호
     , 체결건수, 체결수량, 거래대금
from   일별종목거래
where  :종목코드 is not null
and    거래일자 between :시작일자 and :종료일자
and    종목코드 = :종목코드
and    투자자유형코드 = nvl(:투자자유형, 투자자유형코드)
and    주문매체구분코드 = nvl(:주문매체, 주문매체구분코드)
union all
select 거래일자, 투자자유형코드, 회원번호
     , 체결건수, 체결수량, 거래대금
from   일별종목거래
where  :종목코드 is null and :투자자유형 is not null
and    거래일자 between :시작일자 and :종료일자
and    투자자유형코드 = :투자자유형
and    주문매체구분코드 = nvl(:주문매체, 주문매체구분코드)
union all
select 거래일자, 투자자유형코드, 회원번호
     , 체결건수, 체결수량, 거래대금
from   일별종목거래
where  :종목코드 is null and :투자자유형 is null
and    거래일자 between :시작일자 and :종료일자
and    주문매체구분코드 = nvl(:주문매체, 주문매체구분코드)
```

하지만 개발 기간 내내 SQL마다 이런 식으로 최적의 인덱스 구성전략을 고민하면서 개발한다는 게 결코 쉬운 일은 아니다.
그리고 데이터 분포와 인덱스 구성 등을 고려해 이와 같은 형태로 SQL을 최적화할 수 있는 고급 개발자가 그리 많지 않다는 현실도 인정해야 한다.

위와 같이 union all로 분기하는 기법은, 일반적인 SQL 작성 표준보다는 튜닝차원에서 접근하고 필요에 따라 적절히 활용하도록 하는 것이 타당하다.
<br/>
<br/>
## (3) 선택적 검색 조건에 대한 현실적인 대안
그렇다면 현실적인 대안은? 앞 절에서 이미 정리했듯이 Static SQL 사용을 원칙으로 하되 사용자 입력 조건에 따라 생성될 수 있는 SQL 최대 개수가 너무 많을 때는 Dynamic SQL 사용을 허용하는 것이다.
조건절에 따른 SQL 개수가 많더라도 그 중 일부만 주로 사용되므로 실질적인 하드 파싱 부하는 거의 없다. 다만, 라이브러리 캐시 효율화의 핵심인 바인드 변수 사용 원칙만큼은 준수하도록 해야 한다.

개발 언어나 툴에서 Dynamic Method를 제공하는 목적에 맞게, 부하를 최소화하는 수준에서 잘 활용하는 것이 최선이라고 생각한다.
아래는 PL/SQL에서 조건절을 동적으로 구성하되 바인드 변수를 사용하도록 코딩한 예제다. JAVA, Pro*C 등에서도 활용 가능한 패턴이다.
(JAVA에서 아래와 같은 패턴을 사용하지 않으면 if...else 구문을 두 번씩 작성하게 된다. SQL문을 위해 한 번, 바인드 변수를 위해 한 번)

```
SQLStmt := 'SELECT 거래일자, 종목코드, 투자자유형코드, '
        || '주문매체코드, 체결건수, 체결수량, 거래대금 '
        || 'FROM 일별종목거래 '
        || 'WHERE 거래일자 BETWEEN :1 AND :2 ' ;

If :종목코드 IS NULL Then
  SQLStmt := SQLStmt || 'AND :3 IS NULL ' ;
Else
  SQLStmt := SQLStmt || 'AND 종목코드 = :3 ' ;
End If;

If :투자자유형 IS NULL Then
  SQLStmt := SQLStmt || 'AND :4 IS NULL ' ;
Else
  SQLStmt := SQLStmt || 'AND 투자자유형코드 = :4 ' ;
End If;

EXECUTE IMMEDIATE SQLStmt
  INTO :A, :B, :C, :D, :E, :F, :G
  USING :시작일자, :종료일자, :종목코드, :투자자유형코드;
```

아래는 Visual Basic 스타일로 변환한 것으로서, Delphi 등에서도 활용 가능한 패턴이다.
```
SQLStmt = "SELECT 거래일자, 종목코드, 투자자유형코드 " & _
          "     , 주문매체코드, 체결건수, 체결수량, 거래대금 " & _
          "FROM 일별종목거래 WHERE 거래일자 BETWEEN ? AND ? "

comm.Parameters.Append _
    comm.CreateParameter("시작일자",adVarchar,adParamInput,8,시작일자.Text)

comm.Parameters.Append _
    comm.CreateParameter("종료일자",adVarchar,adParamInput,8,종료일자.Text)

If 종목.Text <> "" Then
  SQLStmt = SQLStmt & "AND 종목코드 = ? "
  comm.Parameters.Append _
  comm.CreateParameter("종목코드", adVarchar, adParamInput, 8, 종목.Text)
End If

If 투자자유형.Text <> "" Then
  SQLStmt = SQLStmt & "AND 투자자유형코드 = ? "
  comm.Parameters.Append _
  comm.CreateParameter("종목코드", adVarchar, adParamInput, 2, 투자자유형.Text)
End If

comm.CommandText = SQLStmt
Set rs = comm.Execute
```

이처럼 SQL을 Dynamic하게 구성하면, 인덱스를 설계할 때 다소 불편하다는 단점이 있다.
SQL Repository에서 SQL을 수집해 테이블별 액세스 유형을 분석하면서 인덱스 설계를 해야 하는데, 조건절이 프로그램 수행 도중에 동적으로 바뀌기 때문이다.
그리고 옵티마이저 힌트를 사용해 튜닝하기도 곤란하다.

이런 단점이 있긴 하지만 개발 생산성도 무시할 수 없으므로 Dynamic SQL을 적재적소에 잘 활용하라고 권고 아닌 권고를 하는 것이다. 튜닝은 말 그대로 튜닝이다.
개발이 어느 정도 진행된 시점에 성능 요건을 만족하기 어려운 대상을 추출해서 튜닝하고, 그때 튜닝 관점에서 필요한 조치들을 취하면 된다고 생각한다.
(안타까운 것은, 최근 국내에서 가장 빠르게 시장 점유율을 높여가고 있는 개발 프레임워크에서 조건절을 동적으로 붙여 나가는 방식으로 SQL을 작성하면 바인드 변수를 사용할 수 없다는 제약을 갖는다.
빨리 개선되기를 기대해 본다.)

인덱스 설계 문제에 대해 얘기하자면, 완성된 형태의 SQL들은 SQL Repository에 저장된 것을 참조하고 그렇지 않은 것들은 수행된 최종 SQL들을 수집(SQL 트레이스 또는 v$sql 등 활용)해서 자주
나타나는 액세스 유형을 기준으로 인덱스 설계를 진행하면 된다.(솔직히 인덱스 설계를 위해 SQL Repository나 프로그램 소스를 직접 열어가며 작업하는 경우는 거의 없으며, 대부분 후자의 방식을 따른다.)

Pro*C, PL/SQL, Visual Basic, Delphi, 어떤 프로그램에서 수행했든 위와 같이 Dynamic 패턴으로 SQL을 던지면 오라클 서버를 통해 수집되는 최종 SQL은 아래와 같은 형태다.

```
SELECT 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
FROM   일별종목거래
WHERE  거래일자 BETWEEN ? AND ?

SELECT 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
FROM   일별종목거래
WHERE  거래일자 BETWEEN ? AND ?
AND    종목코드 = ?

SELECT 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
FROM   일별종목거래
WHERE  거래일자 BETWEEN ? AND ?
AND    투자자유형코드 = ?

SELECT 거래일자, 종목코드, 투자자유형코드, 주문매체코드
     , 체결건수, 체결수량, 거래대금
FROM   일별종목거래
WHERE  거래일자 BETWEEN ? AND ?
AND    종목코드 = ?
AND    투자자유형코드 = ?
```

Static SQL로 작성했을 때와 차이가 없다. 따라서 이들 액세스 경로(Access Path) 기준으로 인덱스 설계를 하면 되고, 이것을 Static SQL로 작성했다고 해서 인덱스 구성전략이 달라지지는 않는다.

인덱스 구성전략만으로 튜닝이 되지 않을 때는 옵티마이저 힌트를 사용해야 하는데, 조건절이 위와 같이 동적으로 바뀐다면 힌트를 함부로 사용할 수 없다.
그때는 할 수 없이 Static SQL을 사용해야 하며, 인덱스 구성과 컬럼 분포, 자주 사용되는 액세스 유형들을 고려해 SQL을 통합하고 힌트를 기술할 수 있는 형태로 재작성해야만 한다.
<br/><br/>

이렇게 설명해 놓고 한가지 걱정이 생기는데, 여기 설명한 내용을 근거로 Dynamic SQL를 무분별하게 사용하려는 개발자가 생겨나지 않을까 싶다.
다시 한번 강조하지만, 원칙은 Static SQL로 작성하는 것이며, 방법이 없거나 SQL이 너무 복잡할 때만 Dynamic SQL을 꺼내 들려고 노력해야 한다.
그런 뜻에서 Dynamic SQL을 Static SQL로 바꿔서 구현한 사례들을 다음 절에서 소개하려고 한다.

그전에, 선택적 입력 조건을 처리할 때 NVL 대신 사용할 수 있는 몇 가지 방법들이 있다고 했는데, 여기서 각각의 특징을 살펴보고 넘어가자.
<br/>
<br/>
## (4) 선택적 검색 조건에 사용할 수 있는 기법 성능 비교

### A. [OR 조건을 사용하는 경우]
```
select * from 일별종목거래
where (:isu_cd is null or isu_cd = :isu_cd)

Execution Plan
-------------------------------------------------------------
0    SELECT STATEMENT Optimizer=ALL_ROWS
1  0  TABLE ACCESS (FULL) OF '일별종목거래' (TABLE)
```

항상 Table Full Scan으로 처리되므로 인덱스 활용이 필요할 때는 이 방식을 사용해선 안 된다.

### B. [LIKE 연산자를 사용하는 경우]
```
select * from 일별종목거래
where  isu_cd like :isu_cd || '%'

Execution Plan
-------------------------------------------------------------
0    SELECT STATEMENT Optimizer=ALL_ROWS
1  0  TABLE ACCESS (BY LOCAL INDEX ROWID) OF '일별종목거래' (TABLE)
2  1   INDEX (RANGE SCAN) OF '일별종목거래_PK' (INDEX (UNIQUE))
```

인덱스 사용이 가능하지만 사용자가 :isu_cd 값을 입력하지 않았을 때 Table Full Scan이 유리한데도 인덱스를 사용하게 되므로 성능이 나빠질 수 있다.
또 주의할 점은, null 허용 컬럼일 때 결과집합이 달라질 수 있으므로 반드시 not null 컬럼일 때만 사용해야 한다는 것이다.

### C. [NVL 함수를 사용하는 경우]
```
select * from 일별종목거래
where  isu_cd = nvl(:isu_cd, isu_cd)

Execution Plan
-------------------------------------------------------------
0    SELECT STATEMENT Optimizer=ALL_ROWS
1  0  CONCATENATION
2  1   FILTER
3  2    TABLE ACCESS (FULL) OF '일별종목거래' (TABLE)
4  1   FILTER
5  4    TABLE ACCESS (BY LOCAL INDEX ROWID) OF '일별종목거래' (TABLE)
6  5     INDEX (RANGE SCAN) OF '일별종목거래_PK' (INDEX (UNIQUE))
```

### D. [DECODE 함수를 사용하는 경우]
```
select * from 일별종목거래
where  isu_cd = decode(:isu_cd, null, isu_cd, :isu_cd)

Execution Plan
-------------------------------------------------------------
0    SELECT STATEMENT Optimizer=ALL_ROWS
1  0  CONCATENATION
2  1   FILTER
3  2    TABLE ACCESS (FULL) OF '일별종목거래' (TABLE)
4  1   FILTER
5  4    TABLE ACCESS (BY LOCAL INDEX ROWID) OF '일별종목거래' (TABLE)
6  5     INDEX (RANGE SCAN) OF '일별종목거래_PK' (INDEX (UNIQUE))
```

C와 D 방식은, 사용자의 :isu_cd 입력 여부에 따라 Full Table Scan과 Index Scan으로 실행계획이 자동 분기된다.
단, 앞서 설명한 like 연산자처럼 nvl 또는 decode 함수를 사용할 때도 해당 컬럼이 not null 컬럼이어야 하며, null 허용 컬럼일 때는 결과집합이 달라지므로 주의해야 한다.
사용자가 :isu_cd 값을 입력하지 않았을 때는 조건절이 isu_cd = isu_cd가 되는데, isu_cd 컬럼 값이 null일 때 오라클은 false를 리턴하기 때문이다.
참고로, null = null 비교가 가능한 DBMS도 있기는 하다. 잘 이해가 되지 않는다면 아래 결과를 참고하시라.

```
SQL> select * from dual
  2  where NULL = NULL;

선택된 레코드가 없습니다.

SQL> select * from dual
  2  where NULL IS NULL;

DU
--
X

1 개의 행이 선택되었습니다.
```

nvl 또는 decode를 여러 컬럼에 대해 사용했을 때는 그 중 변별력이 가장 좋은 컬럼 기준으로 한번만 분기가 일어난다는 사실도 기억할 필요가 있고, 복잡한 옵션 조건을 처리할 때 이 방식에만 의존하기
어려운 이유가 여기에 있다.

### E. [UNION ALL을 사용하는 경우]
```
select * from 일별종목거래
where  :isu_cd is null
union all
select * from 일별종목거래
where  :isu_cd is not null
and    isu_cd = :isu_cd

Execution Plan
-------------------------------------------------------------
0    SELECT STATEMENT Optimizer=ALL_ROWS
1  0  UNION-ALL
2  1   FILTER
3  2    TABLE ACCESS (FULL) OF '일별종목거래' (TABLE)
4  1   FILTER
5  4    TABLE ACCESS (BY LOCAL INDEX ROWID) OF '일별종목거래' (TABLE)
6  5     INDEX (RANGE SCAN) OF '일별종목거래_PK' (INDEX (UNIQUE))
```
<br/>

5가지 방식에 대한 선택 기준을 정리해 보면 아래와 같다.

- [1] not null 컬럼일 때는 nvl, decode를 사용(C와 D)하는 것이 편하다.
- [2] null 값을 허용하고 인덱스 액세스 조건으로 의미있는 컬럼이라면 union all을 사용(E)해 명시적으로 분기해야 한다.
- [3] 인덱스 액세스 조건으로 참여하지 않는 경우, 즉 인덱스 필터 또는 테이블 필터 조건으로만 사용되는 컬럼이라면( :c is null or col = :c) 또는 (c like :c || '%') 어떤 방식을 사용
  (A와 B)해도 무방하다.