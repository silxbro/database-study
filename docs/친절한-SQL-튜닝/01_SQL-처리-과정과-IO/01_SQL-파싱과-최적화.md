# 1.1 | SQL 파싱과 최적화

SQL 튜닝을 본격적으로 시작하기에 앞서 옵티마이저가 SQL을 어떻게 처리하는지, 서버 프로세스는 데이터를 어떻게 읽고 저장하는지 살펴보자.

<br/>

## (1) 구조적, 집합적, 선언적 질의 언어
SQL은 'Structured Query Language'의 줄임말이다. 말 그대로 구조적 질의 언어다.

오라클 PL/SQL, SQL Server T-SQL처럼 절차적(procedural) 프로그래밍 기능을 구현할 수 있는 확장 언어도 제공하지만, SQL은 기본적으로 **구조적**(structured)이고 **집합적**(set-based)이고
**선언적**(declarative)인 질의 언어다.

원하는 결과집합을 구조적, 집합적으로 선언하지만, 그 결과집합을 만드는 과정은 절차적일 수밖에 없다. 즉, 프로시저가 필요한데, 그런 프로시저를 만들어 내는 DBMS 내부 엔진이 바로 **SQL 옵티마이저**이다.
옵티마이저가 프로그래밍을 대신해 주는 셈이다.

DBMS 내부에서 프로시저를 작성하고 컴파일해서 실행 가능한 상태로 만드는 전 과정을 '**SQL 최적화**'라고 한다.

<br/>

## (2) SQL 최적화
SQL을 실행하기 전 최적화 과정을 세분화하면 아래와 같다.

### [1] SQL 파싱
사용자로부터 SQL을 전달받으면 가장 먼저 **SQL 파서**(Parser)가 파싱을 진행한다. SQL 파싱을 요약하면 아래와 같다.
- **파싱 트리 생성** : SQL 문을 이루는 개별 구성요소를 분석해서 파싱 트리 생성
- **Syntax 체크** : 문법적 오류가 없는지 확인. 예를 들어, 사용할 수 없는 키워드를 사용했거나 순서가 바르지 않거나 누락된 키워드가 있는지 확인
- **Semantic 체크** : 의미상 오류가 없는지 확인. 예를 들어, 존재하지 않는 테이블 또는 컬럼을 사용했는지, 사용한 오브젝트에 대한 권한이 있는지 확인

### [2] SQL 최적화
그다음 단계가 SQL 최적화이고, **옵티마이저**(Optimizer)가 그 역할을 맡는다.
SQL 옵티마이저는 미리 수집한 시스템 및 오브젝트 통계정보를 바탕으로 다양한 실행경로를 생성해서 비교한 후 가장 효율적인 하나를 선택한다.
데이터베이스 성능을 결정하는 가장 핵심적인 엔진이다.

### [3] 로우 소스 생성
SQL 옵티마이저가 선택한 실행경로를 실제 실행 가능한 코드 또는 프로시저 형태로 포맷팅하는 단계다. 로우 소스 생성기(Row-Source Generator)가 그 역할을 맡는다.

<br/>

## (3) SQL 옵티마이저
SQL 옵티마이저는 사용자가 원하는 작업을 가장 효율적으로 수행할 수 있는 최적의 데이터 액세스 경로를 선택해 주는 DBMS의 핵심 엔진이다. 옵티마이저의 최적화 단계를 요약하면 다음과 같다.
- (1) 사용자로부터 전달받은 쿼리를 수행하는 데 후보군이 될만한 실행계획들을 찾아낸다.
- (2) 데이터 딕셔너리(Data Dictioinary)에 미리 수집해 둔 오브젝트 통계 및 시스템 통계정보를 이용해 각 실행계획의 예상비용을 산정한다.
- (3) 최저 비용을 나타내는 실행계획을 선택한다.

SQL 옵티마이저를 DBWR, LGWR, PMON, SMON 같은 백그라운드 프로세스로 이해하기 쉽다. 서버 프로세스가 SQL을 전달하면, 옵티마이저가 최적화해서 실행계획을 돌려준다고 생각하는 것이다.
하지만, 옵티마이저는 별도 프로세스가 아니라 서버 프로세스가 가진 기능(Function)일 뿐이다. SQL 파서와 로우 소스 생성기도 마찬가지다.

<br/>

## (4) 실행계획과 비용
DBMS에도 'SQL 실행경로 미리보기' 기능이 있다. 실행계획(Execution Plan)이 바로 그것이다. SQL 옵티마이저가 생성한 처리절차를 사용자가 확인할 수 있게 아래와 같이 트리 구조로 표현한 것이 실행계획이다.
실행계획을 확인하는 방법은 [부록]()을 참고하기 바란다.

```
Execution Plan
------------------------------------------------------------------
0      SELECT STATEMENT Optimaizer=ALL_ROWS (Cost=209 Card=5 Bytes=175)
1  0    TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (Cost=2 Card=5 Bytes=85)
2  1      NESTED LOOPS (Cost=209 Card=5 Bytes=175)
3  2        TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (Cost=207 Card=1 Bytes=18)
4  3          INDEX (RANGE SCAN) OF 'DEPT_LOC_IDX'(NON-UNIQUE) (Cost=7 Card=1)
5  2        INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX'(NON-UNIQUE) (Cost=1 Card=5)
```
미리보기 기능을 통해 자신이 작성한 SQL이 테이블을 스캔하는지 인덱스를 스캔하는지, 인덱스를 스캔한다면 어떤 인덱스인지를 확인할 수 있고, 예상과 다른 방식으로 처리된다면 실행경로를 변경할 수 있다.

옵티마이저가 특정 실행계획을 선택하는 근거는 무엇일까? 이를 설명하기 위해 아래와 같이 테스트용 테이블을 생성해 보자.
```
SQL> create table t
  2  as
  3  select d.no, e.*
  4  from   scott.emp e
  5       , (select rownum no from dual connect by level <= 1000) d;
```
아래와 같이 인덱스도 생성하자.
```
SQL> create index t_x01 on t(deptno, no);
SQL> create index t_x02 on t(deptno, job, no);
```

아래 명령어는 방금 생성한 T 테이블에 통계정보를 수집하는 명령어다.
```
SQL> exec dbms_stats.gather_table_stats( user, 't' );
```

SQL*Plus에서 아래와 같이 AutoTrace를 활성화하고 SQL을 실행하면 실행계획을 확인할 수 있다. 토드나 오렌지 같은 쿼리 툴을 사용하고 있다면, SQL을 선택하고 Ctrl-E 키를 누르면 된다.
```
SQL> set autotrace traceonly exp;

SQL> select * from t
  2  where  deptno = 10
  3  and    no = 1;

----------------------------------------------------------------------------------
| Id | Operation                            | Name  | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------------------------
|  0 | SELECT STATEMENT                     |       |     5 |   210 |     2   (0)|
|  1 |   TABLE ACCESS BY INDEX ROWID BATCHED| T     |     5 |   210 |     2   (0)|
|  2 |     INDEX RANGE SCAN                 | T_X01 |     5 |       |     1   (0)|
----------------------------------------------------------------------------------
```
옵티마이저가 T_X01 인덱스를 선택했다. T_X02 인덱스를 선택할 수 있고, 테이블을 Full Scan할 수도 있는데, T_X01 인덱스를 선택한 근거는 무엇일까?

위 실행계획에서 맨 우측에 Cost가 2로 표시된 것을 확인하기 바란다. T_X02 인덱스를 사용하도록 index 힌트를 지정하고 실행계획을 확인해 보면, Cost가 아래와 같이 19로 표시된다.
```
SQL> set autotrace traceonly exp;

SQL> select /*+ index(t t_x02) */ * from t
  2  where  deptno = 10
  3  and    no = 1;

----------------------------------------------------------------------------------
| Id | Operation                            | Name  | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------------------------
|  0 | SELECT STATEMENT                     |       |     5 |   210 |    19   (0)|
|  1 |   TABLE ACCESS BY INDEX ROWID BATCHED| T     |     5 |   210 |    19   (0)|
|  2 |     INDEX RANGE SCAN                 | T_X02 |     5 |       |    18   (0)|
----------------------------------------------------------------------------------
```

Table Full Scan 하도록 full 힌트를 지정하고 실행계획을 확인해 보면, Cost가 아래와 같이 29로 표시된다.
```
SQL> set autotrace traceonly exp;

SQL> select /*+ full(t) */ * from t
  2  where  deptno = 10
  3  and    no = 1;

----------------------------------------------------------------
| Id | Operation          | Name  | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------
|  0 | SELECT STATEMENT   |       |     5 |   210 |    29   (0)|
|  1 |   TABLE ACCESS FULL| T     |     5 |   210 |    29   (0)|
----------------------------------------------------------------
```
옵티마이저가 T_X01 인덱스를 선택한 근거가 비용임을 알 수 있다. **비용(Cost)** 은 쿼리를 수행하는 동안 발생할 것으로 예상되는 I/O 횟수 또는 예상 소요시간을 표현한 값이다.

SQL 실행계획에 표시되는 Cost도 어디까지나 **예상치**다. 실행경로를 선택하기 위해 옵티마이저가 여러 통계정보를 활용해서 계산해 낸 값이다.
실측치가 아니므로 실제 수행할 때 발생하는 I/O 또는 시간과 많은 차이가 난다.

<br/>

## (5) 옵티마이저 힌트
SQL 옵티마이저는 대부분 좋은 선택을 하지만, 완벽하진 않다. SQL이 복잡할수록 실수할 가능성도 크다.
통계정보에 담을 수 없는 데이터 또는 업무 특성을 활용해 개발자가 직접 더 효율적인 액세스 경로를 찾아낼 수 있다. 이럴 때 옵티마이저 힌트를 이용해 데이터 액세스 경로를 바꿀 수 있다.

힌트 사용법은 아래와 같다. 주석 기호에 '+'를 붙이면 된다.
```
SELECT /* INDEX(A 고객_PK) */
       고객명, 연락처, 주소, 가입일시
FROM   고객 A
WHERE  고객ID = '000000008'
```

### [주의사항]
- 힌트 안에 인자를 나열할 땐 ','(콤마)를 사용할 수 있지만, 힌트와 힌트 사이에 사용하면 안 된다.

  ```
  /* INDEX(A A_X01) INDEX(B B_X03) */  → 모두 유효
  /* INDEX(C), FULL(D) */              → 첫 번째 힌트만 유효
  ```

- 테이블을 지정할 때 아래와 같이 스키마명까지 명시하면 안 된다.

  ```
  SELECT /* FULL(SCOTT.EMP) */ → 무효
  FROM   EMP
  ```

- FROM 절 테이블명 옆에 ALIAS를 지정했다면, 힌트에도 반드시 ALIAS를 사용해야 한다. FROM 절에 ALIAS를 지정했는데 힌트에는 아래와 같이 테이블명을 사용하면, 그 힌트는 무시된다.

  ```
  SELECT /* FULL(EMP) */ → 무효
  FROM EMP E
  ```

### [자주 사용하는 힌트 목록]
|분류|힌트|설명|
|:---:|:---|:---|
|최적화 목표|ALL_ROWS|전체 처리속도 최적화|
|최적화 목표|FIRST_ROWS(N)|최초 N건 응답속도 최적화|
|액세스 방식|FULL|Table Full Scan으로 유도|
|액세스 방식|INDEX|Index Scan으로 유도|
|액세스 방식|INDEX_DESC|Index를 역순으로 스캔하도록 유도|
|액세스 방식|INDEX_FFS|Index Fast Full Scan으로 유도|
|액세스 방식|INDEX_SS|Index Skip Scan으로 유도|
|조인 순서|ORDERED|From 절에 나열된 순서대로 조인|
|조인 순서|LEADING|LEADING 힌트 괄호에 기술한 순서대로 조인<br/>(예) LEADING(T1 T2)|
|조인 순서|SWAP_JOIN_INPUTS|해시 조인 시, BUILD INPUT을 명시적으로 선택<br/>(예) SWAP_JOIN_INPUTS(T1)|
|조인 방식|USE_NL|NL 조인으로 유도|
|조인 방식|USE_MERGE|소트 머지 조인으로 유도|
|조인 방식|USE_HASH|해시 조인으로 유도|
|조인 방식|NL_SJ|NL 세미조인으로 유도|
|조인 방식|MERGE_SJ|소트 머지 세미조인으로 유도|
|조인 방식|HASH_SJ|해시 세미조인으로 유도|
|서브쿼리<br/>팩토링|MATERIALIZE|WITH 문으로 정의한 집합을 물리적으로 생성하도록 유도<br/>(예) WITH /*+ MATERIALIZE */ T AS ( SELECT ... )|
|서브쿼리<br/>팩토링|INLINE|WITH 문으로 정의한 집합을 물리적으로 생성하지 않고 INLINE 처리하도록 유도<br/>(예) WITH /*+ INLINE */ T AS ( SELECT ... )|
|쿼리 변환|MERGE|뷰 머징 유도|
|쿼리 변환|NO_MERGE|뷰 머징 방지|
|쿼리 변환|UNNEST|서브쿼리 Unnesting 유도|
|쿼리 변환|NO_UNNEST|서브쿼리 Unnesting 방지|
|쿼리 변환|PUSH_PRED|조인조건 Pushdown 유도|
|쿼리 변환|NO_PUSH_PRED|조인조건 Pushdown 방지|
|쿼리 변환|USE_CONCAT|OR 또는 IN-List 조건을 OR-Expansion으로 유도|
|쿼리 변환|NO_EXPAND|OR 또는 IN-List 조건에 대한 OR-Expansion 방지|
|병렬 처리|PARALLEL|테이블 스캔 또는 DML을 병렬방식으로 처리하도록 유도<br/>(예) PARALLEL(T1 2)  PARALLEL(T2 2)|
|병렬 처리|PARALLEL_INDEX|인덱스 스캔을 병렬방식으로 처리하도록 유도|
|병렬 처리|PQ_DISTRIBUTE|병렬 수행 시 데이터 분배 방식 결정<br/>(예) PQ_DISTRIBUTE(T1 HASH HASH)|
|기타|APPEND|Direct-Path Insert 로 유도|
|기타|DRIVING_SITE|DB Link Remote 쿼리에 대한 최적화 및 실행 주체 지정(Local 또는 Remote)|
|기타|PUSH_SUBQ|서브쿼리를 가급적 빨리 필터링하도록 유도|
|기타|NO_PUSH_SUBQ|서브쿼리를 가급적 늦게 필터링하도록 유도|