# 02. User Call vs. Recursive Call

앞에서는 데이터베이스 Call을 커서의 활동상태에 따라 Parse, Execute, Fetch로 나누었는데, Call이 어디서 발생하느냐에 따라 User Call과 Recursive Call로 나눌 수도 있다.

SQL 트레이스 파일을 TKProf 유틸리티로 포맷팅하면 맨 아래쪽에 아래와 같은 Overall Total 통계가 나온다.
이 중 NON-RECURSIVE 통계가 User Call에 해당하며, 그 아래쪽 RECURSIVE 통계가 Recursive Call에 해당한다.

```
OVERALL TOTALS FOR ALL NON-RECURSIVE STATEMENTS

call      count      cpu    elapsed     disk      query  current     rows
------- ------- -------- ---------- -------- ---------- -------- --------
Parse     90872   168.94     352.19        1        150       71        0
Execute  176628  2793.18    3084.44    20929  182492646   721270   241853
Fetch    267612  1088.07    7229.70  6020601   73918515        3  1423357
------- ------- -------- ---------- -------- ---------- -------- --------
total    535112  4050.19   10666.34  6041531   256411311  721344  1665210

Misses in library cache during parse: 62590
Misses in library cache during execute: 1980

OVERALL TOTALS FOR ALL RECURSIVE STATEMENTS

call      count      cpu    elapsed     disk      query  current     rows
------- ------- -------- ---------- -------- ---------- -------- --------
Parse     36153     0.88       1.78        2          2      380        0
Execute 3118796   152.34     684.24   479003    6884676   450267   144495
Fetch   3012681   143.35     316.24    41707   18561627       37  3221497
------- ------- -------- ---------- -------- ---------- -------- --------
total   6167630   296.57    1002.27   520712   25446305   450684  3365992

Misses in library cache during parse: 1076
Misses in library cache during execute: 1022
```

**User Call**은 OCI(Oracle Call Interface)를 통해 오라클 외부로부터 들어오는 Call을 말한다.
User Call이 클라이언트가 아닌 WAS에서 발생하는 이유는, 실제 시스템 사용자가 어디에 위치하느냐와는 별개로 DBMS 입장에서의 사용자는 WAS이기 때문이다.
그리고 이는 개념적인 것으로서 실제 WAS가 DBMS와 같은 하드웨어 머신에 있을 수도 있다.

동시 접속자 수가 적을 때는 잘 드러나지 않지만 Peak 시간대에 시스템 장애를 발생시키는 가장 큰 주범은 뭐니뭐니해도 User Call이다.
User Call이 많이 발생하도록 개발된 애플리케이션은 결코 좋은 성능을 낼 수 없으며, 이는 개발자의 기술력에 의해서도 좌우되지만 많은 경우 애플리케이션 설계와 프레임워크 기술구조에 기인한다.
이를테면, Array Processing을 제대로 지원하지 않는 프레임워크, 화면 페이지 처리에 대한 잘못된 표준가이드, 사용자 정의 함수/프로시저에 대한 무조건적인 제약 등이 그것이다.
그리고 프로시저 단위 모듈을 지나치게 잘게 쪼개서 SQL을 건건이 호출하도록 설계하는 것도 대표적이다.

DBMS 성능과 확장성(Scalability)을 높이려면 User Call을 최소화하려는 노력이 무엇보다 중요하며, 이를 위해 아래와 같은 기능과 기술을 적극적으로 활용해야만 한다.

- [1] Loop 쿼리를 해소하고 집합적 사고를 통해 One-SQL로 구현
- [2] Array Processing : Array 단위 Fetch, Bulk Insert/Update/Delete
- [3] 부분범위처리 원리 활용
- [4] 효과적인 화면 페이지 처리
- [5] 사용자 정의 함수/프로시저/트리거의 적절한 활용

Recursive Call은 오라클 내부에서 발생하는 Call을 말한다.
SQL 파싱과 최적화 과정에서 발생하는 Data Dictionary 조회, PL/SQL로 작성된 사용자 정의 함수/프로시저/트리거 내에서의 SQL 수행이 여기에 해당한다.
Recursive Call을 최소화하려면, 바인드 변수를 적극적으로 사용해 하드파싱 발생횟수를 줄여야 한다.
그리고 PL/SQL로 작성한 프로그램이 어떤 특징을 가지며 내부적으로 어떻게 수행되는지를 잘 이해하고 시의 적절하게 사용해야만 한다.
무조건 사용하지 못하도록 제약하거나 무분별하게 사용하지 말아야 한다는 뜻이다.

아래는 SQL 트레이스 결과에서 발췌한 것인데, recursive depth가 2로 표시돼 있다.
특정 프로시저를 호출했는데 거기서 또 다른 프로시저를 호출한 경우이며, 그 마지막 프로시저에서 사용된 SQL에 대한 수행통계인 것이다.
이처럼 Recursive Depth가 깊어지도록 프로그래밍하는 것은 바람직하지 않다.
PL/SQL은 가상머신(Virtual Machine) 상에서 수행되는 인터프리터(Interpreter) 언어이므로 빈번한 호출 시 컨텍스트 스위칭(Context Switching) 때문에 성능이 매우 나빠진다.
성능을 위해서라면 PL/SQL 프로그램에 대한 지나친 모듈화는 지양해야 한다.

```
SELECT BIRTHDAY , ......
FROM ......

call     count    cpu    elapsed   disk    query    current    rows
------- ------ ------ ---------- ------ -------- ---------- -------
Parse        1   0.00       0.00      0        0          2       0
Execute    486   0.05       0.01      0        0          0       0
Fetch      486   0.00       0.00      0     1944          0     486
------- ------ ------ ---------- ------ -------- ---------- -------
total      973   0.05       0.02      0     1944          2     486

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: ALL_ROWS
Parsing user id: 296      (recursive depth: 2)

Rows     Row Source Operation
-------  ----------------------------------------------------
    486  ......
```

또한, 대용량 데이터 조회 쿼리에서 함수를 잘못 사용하면 건건이 함수 호출이 발생해 성능이 극도로 저하되는 경험을 많이 했을 것이다.
대용량 데이터를 조회할 때는 함수를 부분범위처리가 가능한 상황에서 제한적으로 사용해야 하며, 될 수 있으면 조인 또는 스칼라 서브쿼리 형태로 변환하려는 노력이 필요하다.

지금까지 설명한 데이터베이스 Call 종류와 앞으로 설명할 튜닝 원리를 간단히 요약하면 아래 표와 같다.

|Call구분|User|Recursive|
|:---:|:---|:---|
|**Parse**|- Parse Call 최소화 → 애플리케이션 커서 캐싱 기법<br/>&nbsp;&nbsp;(사용자 정의 함수/프로시저에서는 자동으로 기능함)|- Parse Call 최소화 → 애플리케이션 커서 캐싱 기법<br/>&nbsp;&nbsp;(사용자 정의 함수/프로시저에서는 자동으로 기능함)|
|**Execute**|- Array Processing<br/>- One-SQL로 구현<br/>- AP 설계 및 표준 가이드|- Array Processing<br/>- 함수 반복 호출 최소화<br/>&nbsp;&nbsp;→ 조인・스칼라 서브쿼리 활용|
|**Fetch**|- Array 단위 Fetch<br/>- 부분범위처리 활용<br/>- 페이지 처리|- Cursor FOR Loop문 사용<br/>&nbsp;&nbsp;(10g 이후부터 적용)<br/><br/>|