# 01. SQL과 옵티마이저

SQL(Structured Query Language)을 4세대 언어(4GL)라고 말한다.
4세대 언어라고 하면, Visual Basic, Delphi, PowerBuilder 같은 GUI 환경의 비쥬얼 개발 툴을 일컫는 말인데, SQL이 왜 4세대 언어인지 의아할 것이다.
4세대 언어에서는 버튼을 만들 때 마우스 드래그 한번이면 충분하고 그 버튼을 만들기 위한 코딩은 내부에서 자동으로 이루어진다.
우리가 **DBMS에 명령을 날릴 때도 SQL이라고 하는 구조화된 질의언어를 통해 원하는 결과 집합을 요구할 뿐 그 결과집합을 얻기 위한 처리절차를 개발자가 직접 기술하지는 않기 때문에 SQL도 4세대 언어**라고
부르는 것이다.

개발 업무를 시작하면서 처음부터 범용 RDBMS만을 사용해 왔다면 당연한 것 아닌가라고 여기겠지만 파일 시스템이나 dBase III +, FoxPro, Clipper 같은 xBase 계열에서 데이터베이스 프로그래밍을
해봤다면 무엇을 말하려는지 눈치챘을 것이다. 예전에는 두 개 테이블을 조인해 사용자가 원하는 결과집합을 얻으려면, 기준 테이블을 select하고 한 건씩 fetch하면서 반대편 테이블로부터 조인 레코드를
seek하는 과정을 루핑을 통해 반복 수행하는 프로시저를 개발자가 직접 코딩해야 했다.
<br/>
<br/>
하지만 옵티마이저가 내장된 DBMS를 사용한다면 그럴 필요가 없다. 예컨대, EMP와 DEPT 테이블을 조인하는 SQL문 하나만 작성하면 결과집합을 얻기 위해 필요한 프로시저는 내부에서 자동으로 생성되기 때문이다.

우리를 대신해 프로그래밍 해 주는 존재가 DBMS에 내장돼 있는 것인데, 바로 **SQL 옵티마이저**가 그것이다.
따라서 옵티마이저가 내장된 DBMS에서 개발하고 있다면 진정한 프로그래머는 옵티마이저인 셈이다.

> **[사용자]&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;[옵티마이저]&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;[프로시저]**
>
> **&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(SQL)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(실행계획)**
<br/>

위와 같은 과정을 거쳐 **옵티마이저에 의해 생성된 처리절차를 사용자가 확인할 수 있도록 트리 구조로 표현**한 것이 **실행계획(Execution Plan)** 이다.

```
Execution Plan
----------------------------------------------------------------------
0     SELECT STATEMENT Optimizer=CHOOSE (Cost=209 Card=5 Bytes=175)
1   0   TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (Cost=2 Card=5 Bytes=85)
2   1     NESTED LOOPS (Cost=209 Card=5 Bytes=175)
3   2       TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (Cost=207 Card=1 Bytes=18)
4   3         INDEX (RANGE SCAN) OF 'DEPT_LOC_IDX' (NON-UNIQUE) (Cost=7 Card=1)
5   2       INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (NON-UNIQUE) (Cost=1 Card=5)
```

SQL 옵티마이저는 **최소비용, 최적의 경로를 선택해서 사용자가 원하는 작업을 가장 효율적으로 수행할 수 있는 프로시저를 자동으로 생성해 주는 DBMS의 핵심기능**이라고 이해하면 된다.
옵티마이저의 최적화 수행단계를 요약하면 아래와 같다.

- **[1] 사용자가 던진 쿼리수행을 위해, 후보군이 될만한 실행계획들을 찾아낸다.**
- **[2] 데이터 딕셔너리(Data Dictionary)에 미리 수집해 놓은(Dynamic Sampling 기능은 논외로 함) 오브젝트 통계 및 시스템 통계정보를 이용해 각 실행계획의 예상비용을 산정한다.**
- **[3] 각 실행계획의 비용을 비교해서 최소비용을 갖는 하나를 선택한다.**
  <br/>

참고로, 옵티마이저를 설명하고 나면 SQL 파싱과 최적화를 담당하는 별도의 백그라운드 프로세스가 떠 있냐는 질문을 가끔 받는데, 그렇지 않다.
SQL을 파싱하고, 필요하면 최적화를 수행하며, 커서를 열어 SQL을 실행하면서 블록을 읽고, 읽은 데이터를 정렬해서 클라이언트가 요청한 결과집합을 만들어 네트워크를 통해 전송하는 일련의 작업들을 모두
**서버 프로세스**가 처리해 준다.