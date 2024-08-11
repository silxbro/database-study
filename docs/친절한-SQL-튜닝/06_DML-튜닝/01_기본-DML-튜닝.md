# 6.1 | 기본 DML 튜닝

지금까지 배운 인덱스와 조인 튜닝을 DML 문에도 그대로 적용할 수 있지만, 본 절은 DML 성능에 영향을 주는 다른 요소와 튜닝 방법들을 모아서 따로 설명하고자 한다.
DML 성능에 영향을 미치는 요소에 어떤 것들이 있는지부터 살펴보자.

<br/>

## (1) DML 성능에 영향을 미치는 요소
DML 성능에 영향을 미치는 요소는 다음과 같다.
- 인덱스
- 무결성 제약
- 조건절
- 서브쿼리
- Redo 로깅
- Undo 로깅
- Lock
- 커밋

### [인덱스와 DML 성능]
테이블에 레코드를 입력하면, 인덱스에도 입력해야 한다. 테이블은 Freelist를 통해 입력할 블록을 할당받지만, 인덱스는 정렬된 자료구조이므로 수직적 탐색을 통해 입력할 블록을 찾아야 한다.
인덱스에 입력하는 과정이 더 복잡하므로 DML 성능에 미치는 영향도 더 크다.
- 테이블마다 데이터 입력이 가능한(여유 공간이 있는) 블록 목록을 관리하는데, 이를 'Freelist'라고 한다.

DELETE 할 때도 마찬가지다. 테이블에서 레코드 하나를 삭제하면, 인덱스 레코드를 모두 찾아서 삭제해 줘야 한다. UPDATE 할 때는 변경된 컬럼을 참조하는 인덱스만 찾아서 변경해 주면 된다.
그 대신, 테이블에서 한 건 변경할 때마다 인덱스에는 두 개 오퍼레이션이 발생한다. 인덱스는 정렬된 자료구조이기 때문이다.
예를 들어, 'A'를 'K'로 변경하면 저장 위치도 달라지므로 **삭제 후 삽입**하는 방식으로 처리한다.

인덱스 개수가 DML 성능에 미치는 영향이 매우 큰 만큼, 인덱스 설계에 심혈을 기울여야 한다. 핵심 트랜잭션 테이블에서 인덱스 하나라도 줄이면 TPS(Transaction Per Second)는 그만큼 향상된다.

인덱스 개수가 DML 성능에 미치는 영향을 직접 테스트해 보자.
```
SQL> create table source
  2  as
  3  select b.no, a.*
  4  from   (select * from emp where rownum <= 10) a
  5       , (select rownum as no from dual connect by level <= 100000) b;

SQL> create table target
  2  as
  3  select * from source where 1 = 2;

SQL> alter table target add
  2  constraint target_pk primary key(no, empno);
```
방금 생성한 SOURCE 테이블에는 레코드 100만 개가 입력돼 있다. TARGET 테이블은 현재 비어 있다.
TARGET 테이블에 PK 인덱스 하나만 생성한 상태에서 SOURCE 테이블을 읽어 레코드 100만 개를 입력해 보자.
```
SQL> set timing on;
SQL> insert into target
  2  select * from source;

1000000 개의 행이 만들어졌습니다.

경    과: 00:00:04.95
```
4.95초만에 수행을 마쳤다. 인덱스를 두 개 더 생성하고 다시 100만 건을 입력해 보자.
```
SQL> truncate table target;

SQL> create index target_x1 on target(ename);

SQL> create index target_x2 on target(deptno, mgr);

SQL> insert into target
  2  select * from source;

1000000 개의 행이 만들어졌습니다.

경    과: 00:00:38.98
```
38.98초로 무려 여덟 배나 느려졌다. 인덱스 두 개의 영향력이 이 정도로 크다.

### [무결성 제약과 DML 성능]
데이터베이스에 논리적으로 의미 있는 자료만 저장되게 하는 데이터 무결성 규칙으로는 아래 네 가지가 있다.

- 개체 무결성 (Entity Integrity)
- 참조 무결성 (Referential Integrity)
- 도메인 무결성 (Domain Integrity)
- 사용자 정의 무결성 (또는 업무 제약 조건)

이들 규칙을 애플리케이션으로 구현할 수도 있지만, DBMS에서 PK, FK, Check, Not Null 같은 제약(Constraint)을 설정하면 더 완벽하게 데이터 무결성을 지켜낼 수 있다.

PK, FK 제약은 Check, Not Null 제약보다 성능에 더 큰 영향을 미친다.
Check, Not Null은 정의한 제약 조건을 준수하는지만 확인하면 되지만, PK, FK 제약은 실제 데이터를 조회해 봐야 하기 때문이다.

앞서 진행한 테스트에 이어 이번에는 일반 인덱스와 PK 제약을 모두 제거한 상태에서 100만건 입력하는 데 걸리는 시간을 확인해 보자.
```
SQL> drop index target_x1;

SQL> drop index target_x2;

SQL> alter table target drop primary key;

SQL> truncate table target;

SQL> insert into target
  2  select * from source;

1000000 개의 행이 만들어졌습니다.

경    과: 00:00:01.95
```

테스트 결과를 요약하면, 다음 표와 같다.
|PK 제약/인덱스|일반 인덱스(2개)|소요시간|
|:---:|:---:|:---:|
|○|○|38.98초|
|○|✕|4.95초|
|✕|✕|1.32초|

### [조건절과 DML 성능]
아래는 조건절만 포함하는 가장 기본적인 DML 문과 실행계획이다.
```
SQL> set autotrace traceonly exp

SQL> update emp set sal = sal * 1.1 where deptno = 40;

-------------------------------------------------------------------------------
| Id  | Operation           | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT    |         |     1 |     7 |     2   (0)| 00:00:01 |
|   1 |   UPDATE            | EMP     |       |       |            |          |
|   2 |     INDEX RANGE SCAN| EMP_X01 |     1 |     7 |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------

SQL> delete from emp where deptno = 40;

-------------------------------------------------------------------------------
| Id  | Operation           | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------
|   0 | DELETE STATEMENT    |         |     1 |    13 |     1   (0)| 00:00:01 |
|   1 |   DELETE            | EMP     |       |       |            |          |
|   2 |     INDEX RANGE SCAN| EMP_X01 |     1 |    13 |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------
```
SELECT 문과 실행계획이 다르지 않으므로 이들 DML 문에는 인덱스 튜닝 원리를 그대로 적용할 수 있다.

### [서브쿼리와 DML 성능]
아래는 서브쿼리를 포함하는 DML 문과 실행계획이다.
```
SQL> update emp e set sal = sal * 1.1
  2  where  exists
  3          (select 'x' from dept where deptno = e.deptno and loc = 'CHICAGO');

------------------------------------------------------------------------------------
| Id  | Operation                          | Name     | Rows  | Bytes | Cost (%CPU)|
------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT                   |          |     5 |    90 |     5  (20)|
|   1 |   UPDATE                           | EMP      |       |       |            |
|   2 |     NESTED LOOPS                   |          |     5 |    90 |     5   (0)|
|   3 |       SORT UNIQUE                  |          |     1 |    11 |     2   (0)|
|   4 |         TABLE ACCESS BY INDEX ROWID| DEPT     |     1 |    11 |     2   (0)|
|   5 |           INDEX RANGE SCAN         | DEPT_X01 |     1 |       |     1   (0)|
|   6 |       INDEX RANGE SCAN             | EMP_X01  |     5 |    35 |     1   (0)|
------------------------------------------------------------------------------------

SQL> delete from e
  2  where  exists
  3          (select 'x' from dept where deptno = e.deptno and loc = 'CHICAGO');

----------------------------------------------------------------------------------
| Id  | Operation                        | Name     | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------------------------
|   0 | DELETE STATEMENT                 |          |     5 |    90 |     4  (20)|
|   1 |   DELETE                         | EMP      |       |       |            |
|   2 |     HASH JOIN SEMI               |          |     5 |   120 |     4   (0)|
|   3 |       INDEX FULL SCAN            | EMP_X01  |    14 |   182 |     1   (0)|
|   4 |       TABLE ACCESS BY INDEX ROWID| DEPT     |     1 |    11 |     2   (0)|
|   5 |         INDEX RANGE SCAN         | DEPT_X01 |     1 |       |     1   (0)|
----------------------------------------------------------------------------------

SQL> insert into emp
  2  select e.*
  3  from   emp_t e
  4  where  exists
  5          (select 'x' from dept where deptno = e.deptno and loc = 'CHICAGO');    

----------------------------------------------------------------------------------
| Id  | Operation                        | Name     | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------------------------
|   0 | INSERT STATEMENT                 |          |     5 |   490 |     6  (17)|
|   1 |   LOAD TABLE CONVENTIONAL        | EMP      |       |       |            |
|   2 |     HASH JOIN SEMI               |          |     5 |   490 |     6  (17)|
|   3 |       TABLE ACCESS FULL          | EMP_T    |    14 |  1218 |     3   (0)|
|   4 |       TABLE ACCESS BY INDEX ROWID| DEPT     |     1 |    11 |     2   (0)|
|   5 |         INDEX RANGE SCAN         | DEPT_X01 |     1 |       |     1   (0)|
----------------------------------------------------------------------------------
```

SELECT 문과 실행계획이 다르지 않으므로 이들 DML 문에는 조인 튜닝 원리를 그대로 적용할 수 있다. 특히, 서브쿼리 조인과 밀접한 관련이 있다.

### [Redo 로깅과 DML 성능]
오라클은 데이터파일과 컨트롤 파일에 가해지는 모든 변경사항을 Redo 로그에 기록한다.
Redo 로그는 트랜잭션 데이터가 어떤 이유에서건 유실됐을 때, 트랜잭션을 재현함으로써 유실 이전 상태로 복구하는 데 사용된다.
- 데이터파일에 대한 변경은 캐시된 버퍼블록을 통해 이루어진다.

DML을 수행할 때마다 Redo 로그를 생성해야 하므로 Redo 로깅은 DML 성능에 영향을 미친다. INSERT 작업에 대해 Redo 로깅 생략 기능을 제공하는 이유가 여기에 있다.

- #### [Redo 로그의 용도]

  Redo 로그는 아래 세 가지 목적에 사용된다.

  - [1] Database Recovery
  - [2] Cache Recovery (Instance Recovery 시 roll forward 단계)
  - [3] Fast Commit

  첫째, Redo 로그는 물리적으로 디스크가 깨지는 등의 Media Fail 발생 시 데이터베이스를 복구하기 위해 사용된다. 이때는 온라인 Redo 로그를 백업해 둔 Archived Redo로 그를 이용하게 된다.
  'Media Recovery'라고도 한다.

  둘째, Redo 로그는 'Cache Recovery'를 위해 사용하며 다른 말로 'Instance Recovery'라고도 한다. 모든 DBMS가 버퍼캐시를 도입하는 이유가 I/O 성능을 높이기 위해서인데, 버퍼캐시는 휘발성이다.
  캐시에 저장된 변경사항이 디스크 상의 데이터 블록에 아직 기억되지 않은 상태에서 정전 등이 발생해 인스턴스가 비정상적으로 종료되면, 그때까지의 작업내용을 모두 잃게 된다는 뜻이다.
  이러한 트랜잭션 데이터 유실에 대비하기 위해 Redo 로그를 남긴다.

  마지막으로, Redo 로그는 'Fast Commit'을 위해 사용한다. 변경된 메모리 버퍼블록을 디스크 상의 데이터 블록에 반영하는 작업은 랜덤 액세스 방식으로 이루어지므로 매우 느리다.
  반면 로그는 Append 방식으로 기록하므로 상대적으로 빠르다.
  따라서 트랜잭션에 의한 변경사항을 우선 Append 방식으로 빠르게 로그 파일에 기록하고, 변경된 메모리 버퍼블록과 데이터파일 블록 간 동기화는 적절한 수단(DBWR, Checkpoint)을 이용해 나중에
  배치(Batch) 방식으로 일괄 수행한다.

  사용자의 갱신내용이 메모리상의 버퍼블록에만 기록된 채 아직 디스크에 기록되지 않았지만 Redo 로그를 믿고 빠르게 커밋을 완료한다는 의미에서 이를 'Fast Commit'이라고 부른다.
  커밋 정보까지 Redo 로그 파일에 안전하게 기록했다면, 인스턴스 Crash가 발생해도 언제든 복구할 수 있으므로 오라클은 안심하고 커밋을 완료할 수 있다.

### [Undo 로깅과 DML 성능]
과거에는 롤백(Rollback)이라는 용어를 주로 사용했지만, 9i부터 오라클은 Undo라는 용어를 사용하고 있다.
Redo는 트랜잭션을 재현함으로써 과거를 현재 상태로 되돌리는 데 사용하고, Undo는 트랜잭션을 롤백함으로써 현재를 과거 상태로 되돌리는 데 사용한다.

따라서 Redo에는 트랜잭션을 재현하는 데 필요한 정보를 로깅하고, Undo에는 변경된 블록을 이전 상태로 되돌리는 데 필요한 정보를 로깅한다.

DML을 수행할 때마다 Undo를 생성해야 하므로 Undo 로깅은 DML 성능에 영향을 미친다. 그렇다고 Undo를 안 남길 순 없다. 오라클은 그런 방법을 아예 제공하지 않는다.

- #### [Undo의 용도와 MVCC 모델]

  오라클은 데이터를 입력, 수정, 삭제할 때마다 Undo 세그먼트에 기록을 남긴다. Undo 데이터를 기록한 공간은 해당 트랜잭션이 커밋하는 순간, 다른 트랜잭션이 재사용할 수 있는 상태로 바뀐다.
  가장 오래 전에 커밋한 Undo 공간부터 재사용하므로 Undo 데이터가 곧바로 사라지진 않겠지만, 언젠가 다른 트랜잭션 데이터로 덮어쓰이면서 (overwritten) 사라질 수 밖에 없다.

  Undo에 기록한 데이터는 아래 세 가지 목적에 사용된다.
  - [1] Transaction Rollback
  - [2] Transaction Recovery (Instance Recovery 시 rollback 단계)
  - [3] Read Consistency

  첫째, 트랜잭션에 의한 변경사항을 최종 커밋하지 않고 롤백하고자 할 때 Undo 데이터를 이용한다.

  둘째, Instance Crash 발생 후 Redo를 이용하여 roll forward 단계가 완료되면 최종 커밋되지 않은 변경사항까지 모두 복구된다.
  따라서 시스템이 셧다운된 시점에 아직 커밋되지 않았던 트랜잭션들을 모두 롤백해야 하는데, 이때 Undo 데이터를 사용한다.

  마지막으로, Undo 데이터는 '읽기 일관성(Read Consistency)'을 위해 사용한다. SQL 튜닝 관점에서 주목할 내용은 읽기 일관성이다.
  읽기 일관성을 위해 Consistent 모드로 데이터를 읽는 오라클에선 동시 트랜잭션이 많을수록 블록 I/O가 증가하면서 성능 저하로 이어지기 때문이다.

  #### MVCC(Multi-Version Concurrency Control) 모델
  MVCC 모델을 사용하는 오라클은 데이터를 두 가지 모드로 읽는다. 하나는 Current 모드, 다른 하나는 Consistent 모드다.
  Current 모드는 디스크에서 캐시로 적재된 원본(Current) 블록을 현재 상태 그대로 읽는 방식을 말한다.

  Consistent 모드는 쿼리가 시작된 이후에 다른 트랜잭션에 의해 변경된 블록을 만나면 원본 블록으로부터 복사본(CR Copy) 블록을 만들고, 거기에 Undo 데이터를 적용함으로써 쿼리가 '시작된 시점'으로
  되돌려서 읽는 방식을 말한다. 따라서 원본 블록 하나에 여러 복사본이 캐시에 존재할 수 있다.

  Consistent 모드를 정확히 이해하려면 SCN에 대한 이해가 필요하다. 오라클은 시스템에서 마지막 커밋이 발생한 시점정보를 'SCN(System Commit Number'이라는 Global 변수값으로 관리한다.
  이 값은 기본적으로 각 트랜잭션이 커밋할 때마다 1씩 증가하지만, 오라클 백그라운드 프로세서에 의해서도 조금씩 증가한다.

  또한, 오라클은 각 블록이 마지막으로 변경된 시점을 관리하기 위해 모든 블록 헤더에 SCN을 기록하는데, 이를 '블록 SCN'이라과 한다.
  그리고 모든 쿼리는 Global 변수인 SCN 값을 먼저 확인하고서 읽기 작업을 시작하는데, 이를 '쿼리 SCN'이라고 한다.

  SCN 개념을 이용해 Consistent 모드를 다시 설명하면, Consistent 모드는 쿼리 SCN과 블록 SCN을 비교함으로써 쿼리 수행 도중에 블록이 변경됐는지를 확인하면서 데이터를 읽는 방식이다.
  데이터를 읽다가 블록 SCN이 쿼리 SCN보다 더 큰 블록을 만나면 복사본 블록을 만들고 Undo 데이터를 적용함으로써 쿼리가 시작된 시점으로 되돌려서 읽는다.
  참고로, Undo 데이터가 다른 트랜잭션에 의해 재사용됨으로써 쿼리 시작 시점으로 되돌리는 작업에 실패할 때 Snapshot too old(ORA-01555) 에러가 발생한다.

  SELECT 문은 (몇몇 예외 케이스를 제외하곤) 항상 Consistent 모드로 데이터를 읽는다. 반면, DML 문은 Consistent 모드로 대상 레코드를 찾고, Current 모드로 추가/변경/삭제한다.
  즉, Consistent 모드로 DML 문이 '시작된 시점'에 존재했던 데이터 블록을 찾고, 다시 Current 모드로 원본 블록을 찾아서 갱신한다.
  읽기 일관성을 위해 Consistent 모드로 읽지만, 변경 작업을 복사본 블록에 할 수는 없지 않은가. 데이터 변경은 원본 블록에 해야 한다.
  - Consistent 모드로 읽은 레코드의 ROWID를 이용한다.

  더 자세한 내용은 '오라클 성능 고도화 원리와 해법' 1권을 통해 학습하기 바란다.
  MVCC 모델을 설명하는 데 1장 지면 대부분을 할애할 정도로 다룰 내용이 많기 때문에 여기서는 서문에 밝힌 본서 취지에 맞게 핵심만 간추려 설명하였다.


### [Lock과 DML 성능]
Lock은 DML 성능에 매우 크고 직접적인 영향을 미친다. Lock을 필요 이상으로 자주, 길게 사용하거나 레벨을 높일수록 DML 성능은 느려진다.
그렇다고 Lock을 너무 적게, 짧게 사용하거나 필요한 레벨 이하로 낮추면 데이터 품질이 나빠진다. 성능과 데이터 품질이 모두 중요한데, 이 둘은 트레이드 오프(Trade-off) 관계여서 어렵다.
두 마리 토끼를 다 잡으려면 매우 세심한 동시성 제어가 필요하다.
- 네 가지 트랜잭션 격리성 수준(Transaction Isolation Level)이 있다. Read Uncommitted, Read Committed, Repeatable Read, Serializable이 그것이다.
  기본은 Read Committed이며, 격리성 수준을 올릴수록 Lock을 오래 유지한다. 가장 높은 Serializable은 테이블 레벨 Lock을 사용하며, 트랜잭션을 완료할 때까지 유지한다.
  참고로, 오라클은 Read Committed와 Serializable만 지원하며, Serializable로 상향 조정해도 Lock 레벨이나 지속기간에 변함이 없다.
  다만, 데이터를 읽는 기준 시점이 'SQL 시작 시점'에서 '트랜잭션 시작 시점'으로 바뀔 뿐이다.

- Lock 레벨로는 로우 레벨, 블록 레벨, 익스텐트 레벨, 테이블 레벨 등이 있다.
  처음에는 로우 레벨을 사용하다가 Lock 설정 대상이 많아지면 점점 레벨을 올리기도 하는데, 이를 'Lock 에스컬레이션(Escalation)'이라고 부른다. 오라클은 Lock 에스컬레이션을 사용하지 않는다.

동시성 제어(Concurrency Control)란, 동시에 실행되는 트랜잭션 수를 최대화(고성능)하면서도 입력, 수정, 삭제, 검색 시 데이터 무결성을 유지(고품질)하기 위해 노력하는 것을 말한다.
Lock과 트랜잭션 동시성 제어에 대해서는 4절에서 좀 더 자세히 다룬다.


### [커밋과 DML 성능]
커밋은 DML과 별개로 실행하지만, DML을 끝내려면 커밋까지 완료해야 하므로 서로 밀접한 관련이 있다. 특히 DML이 Lock에 의해 블로킹(Blocking)된 경우, 커밋은 DML 성능과 직결된다.
DML을 완료할 수 있게 Lock을 풀 수 있는 열쇠가 바로 커밋이기 때문이다. Lock을 푸는 열쇠로서 커밋의 기능은 4절에서 설명하므로 여기서는 커밋 자체의 내부 메커니즘과 성능 이슈를 살펴보고자 한다.

모든 DBMS가 Fast Commit을 구현하고 있다. 구현방식은 서로 다르지만, 갱신한 데이터가 아무리 많아도 커밋만큼은 빠르게 처리한다는 점은 같다.
Fast Commit의 도움으로 커밋을 순간적으로 처리하긴 하지만, 커밋은 결코 가벼운 작업이 아니다. 커밋의 내부 메커니즘을 통해 그 이유를 살펴보자.

#### [1] DB 버퍼캐시
DB에 접속한 사용자를 대신해 모든 일을 처리하는 서버 프로세스는 버퍼캐시를 통해 데이터를 읽고 쓴다.
버퍼캐시에서 변경된 블록(Dirty 블록)을 모아 주기적으로 데이터파일에 일괄 기록하는 작업은 DBWR(Database Writer) 프로세스가 맡는다.
일을 건건이 처리하지 않고 모았다가 한 번에 일괄(Batch) 처리하는 방식은 컴퓨터 세계뿐만 아니라 우리 일상생활에서도 흔히 찾아볼 수 있다.

#### [2] Redo 로그버퍼
버퍼캐시는 휘발성이므로 DBWR 프로세스가 Dirty 블록들을 데이터파일에 반영할 때까지 불안한 상태라고 생각할 수 있다.
하지만, 버퍼캐시에 가한 변경사항을 Redo 로그에도 기록해 두었으므로 안심해도 된다. 버퍼캐시 데이터가 유실되더라도 Redo 로그를 이용해 언제든 복구할 수 있기 때문이다.

그런데 Redo 로그도 파일이다. Append 방식으로 기록하더라도 디스크 I/O는 느리다. Redo 로깅 성능 문제를 해결하기 위해 오라클은 로그버퍼를 이용한다.
Redo 로그 파일에 기록하기 전에 먼저 로그버퍼에 기록하는 방식이다. 로그버퍼에 기록한 내용은 나중에 LGWR(Log Writer) 프로세스가 Redo 로그 파일에 일괄(Batch) 기록한다.

#### [3] 트랜잭션 데이터 저장 과정
한 트랜잭션이 데이터를 변경하고 커밋하는 과정, 그리고 변경된 블록을 데이터파일에 기록하는 과정은 다음과 같다.
- [1] DML 문을 실행하면 Redo 로그버퍼에 변경사항을 기록한다.
- [2] 버퍼블록에서 데이터를 변경(레코드 추가/수정/삭제)한다. 물론, 버퍼캐시에서 블록을 찾지 못하면, 데이터파일에서 읽는 작업부터 한다.
- [3] 커밋한다.
- [4] LGWR 프로세스가 Redo 로그버퍼 내용을 로그파일에 일괄 저장한다.
- [5] DBWR 프로세스가 변경된 버퍼블록들은 데이터파일에 일괄 저장한다.

오라클은 데이터를 변경하기 전에 항상 로그부터 기록한다. 서버 프로세스가 버퍼블록에서 데이터를 변경([2])하기 전에 Redo 로그버퍼에 로그를 먼저 기록([1])하는 이유다.
DBWR 프로세스가 Dirty 블록을 디스크에 기록([5])하기 전에 LGWR 프로세스가 Redo 로그파일에 로그를 먼저 기록([4])하는 이유이기도 하다. 이를 'Write Ahead Logging'이라고 부른다.

여기서 한가지 의문이 생긴다. 메모리 버퍼캐시가 휘발성이어서 Redo 로그를 남기는데, Redo 로그마저도 휘발성 로그버퍼에 기록한다면 트랜잭션 데이터를 안전하게 지킬 수 있느냐는 것이다.
문제가 원점으로 되돌아간다. 커밋한 트랜잭션의 영속성을 어떻게 보장할 것인가? 오라클은 이 문제를 어떻게 해결할까?

잠자던 DBWR와 LGWR 프로세스는 '주기적으로' 깨어나 각각 Dirty 블록과 Redo 로그퍼버를 파일에 기록한다. LGWR 프로세스는 서버 프로세스가 커밋을 발행했다고 '신호를 보낼 때도' 깨어나서 활동을 시작한다.
'적어도 커밋 시점에는' Redo 로그버퍼 내용을 로그파일에 기록한다는 뜻이다. 이를 'Log Force at Commit'이라고 부른다.

서버 프로세스가 변경한 버퍼블록들을 디스크에 기록하지 않았더라도 커밋 시점에 Redo로 그를 디스크에 안전하게 기록했다면, 그 순간부터 트랜잭션의 영속성은 보장된다.

#### [4] '커밋 = 저장 버튼'
문서를 작성할 때 워드프로세서는 사용자가 입력한 내용을 메모리에 기록하며, 저장 버튼을 눌러야 비로소 디스크 파일에 저장한다. 워드프로세서가 저장을 완료할 때까지 사용자는 작업을 계속할 수 없다.
즉, Sync 방식이다. 문서 저장과 관련해 안 좋은 습관 몇 가지를 나열하면 아래와 같다.

- 문서 작성르 모두 완료할 때까지 저장 버튼을 한 번도 누르지 않는다.
- 너무 자주, 수시로 저장 버튼을 누른다.
- 습관적으로 저장 버튼을 연속해서 두 번씩 누른다.

데이터베이스 트랜잭션을 문서 작업에 비유하면, 커밋은 문서 작업 도중에 '저장' 버튼을 누르는 것과 같다. 서버 프로세스가 그때까지 했던 작업을 디스크에 기록하라는 명령어인 셈이다.
저장을 완료할 때까지 서버 프로세스는 다음 작업을 진행할 수 없다.
Redo 로그버퍼에 기록된 내용을 디스크에 기록하도록 LGWR 프로세스에 신호를 보낸 후 작업을 완료했다는 신호를 받아야 다음 작업을 진행할 수 있다. Sync 방식이다.
LGWR 프로세스가 Redo 로그를 기록하는 작업은 디스크 I/O 작업이다. 커밋은 그래서 생각보다 느리다.
- 서버 프로세스는 커밋 레코드를 로그버퍼에 기록하고 LGWR 프로세스에 신호를 보낸 후 대기(Wait 큐에서 Sleep) 상태로 전환한다.
  LGWWR 프로세스가 로그 버퍼를 디스크에 기록하는 작업을 마치면 Wait 큐에서 대기 중인 서버 프로세스에 완료 메시지를 전송한다.
  신호를 받은 서버 프로세스는 Runnable 큐로 옮겨진 후 CPU를 할당받으면 다음 작업을 계속 진행한다.

트랜잭션을 필요 이상으로 길게 정의함으로써 오랫동안 오랫동안 커밋하지 않는 것도 문제지만, 너무 자주 커밋하는 것도 문제다.
오랫동안 커밋하지 않는 채 데이터를 계속 갱신하면 Undo 공간이 부족해져 시스템 장애 상황을 유발할 수 있다. 루프를 돌면서 건건이 커밋한다면, 프로그램 자체 성능이 매우 느려진다.
문서 작업할 때 단어마다 저장 버튼을 누르는 것과 같다. 트랜잭션을 논리적으로 잘 정의함으로써 불필요한 커밋이 발생하지 않도록 구현해야 한다.

<br/>

## (2) 데이터베이스 Call과 성능

### [데이터베이스 Call]

```
select cust_nm, birthday from customer where cust_id = :cust_id

call     count    cpu     elapsed    disk  query   current   rows
------- ------ ------ ----------- ------- ------ --------- ------
Parse        1   0.00        0.00       0      0         0      0
Execute   5000   0.18        0.14       0      0         0      0
Fetch     5000   0.21        0.25       0  20000         0  50000
------- ------ ------ ----------- ------- ------ --------- ------
Total    10001   0.39        0.40       0  20000         0  50000

Misses in library cache during parse: 1
```
위는 SQL 트레이스 리포트에서 Call 통계 부분만을 발췌한 것이다. SQL 트레이스 Call 통계가 보여주듯, SQL은 아래 세 단계로 나누어 실행된다.
- Parse Call : SQL 파싱과 최적화를 수행하는 단계다. SQL과 실행계획을 라이브러리 캐시에서 찾으면, 최적화 단계는 생략할 수 있다.
- Execute Call : 말 그대로 SQL을 실행하는 단계다. DML은 이 단계에서 모든 과정이 끝나지만, SELECT문은 Fetch 단계를 거친다.
- Fetch Call : 데이터를 읽어서 사용자에게 결과집합을 전송하는 과정으로 SELECT 문에서만 나타난다. 전송할 데이터가 많을 때는 Fetch Call이 여러 번 발생한다.

Call이 어디서 발생하느냐에 따라 User Call과 Recursive Call로 나눌 수도 있다.

User Call은 네트워크를 경유해 DBMS 외부로부터 인입되는 Call이다. 최종 사용자(User)는 클라이언트 단에 위치한다. 하지만, DBMS 입장에서 사용자는 WAS(또는 AP 서버)다.
3-Tier 아키텍처에서 User Call은 WAS(또는 AP서버) 서버에서 발생하는 Call이다.

Recursive Call은 DBMS 내부에서 발생하는 Call이다.
SQL 파싱과 최적화 과정에서 발생하는 데이터 딕셔너리 조회, PL/SQL로 작성한 사용자 정의 함수/프로시저/트리거에 내장된 SQL을 실행할 때 발생하는 Call이 여기에 해당한다.

User Call이든 Recursive Call이든, SQL을 실행할 때마다 Parse, Execute, Fetch Call 단계를 거친다.
데이터베이스 Call이 많으면 성능은 느릴 수 밖에 없다. 특히, 네트워크를 경유하는 User Call이 성능에 미치는 영향은 매우 크다.

### [절차적 루프 처리]
데이터베이스 Call이 성능에 미치는 영향을 테스트를 통해 확인해 보자.
```
SQL> create table source
  2  as
  3  select b.no, a.*
  4  from   (select * from emp where rownum <= 10) a
  5       , (select rownum as no from dual connect by level <= 100000) b;

SQL> create table target
  2  as
  3  select * from source where 1 = 2;
```
방금 생성한 SOURCE 테이블에 레코드가 100만 개가 입력돼 있다. PL/SQL 프로그램에서 SOURCE 테이블을 읽어 100만 번 루프를 돌면서 건건이 TARGET 테이블에 입력해 보자.
```
SQL> set timing on;

SQL> begin
  2  for s in (select * from source)
  3  loop
  4    insert into target values ( s.no, s.empno, s.ename, s.job, s.mgr
  5                              , s.hiredate, s.sal, s.comm, s.deptno);
  6  end loop;
  7
  8  commit;
  9  end;
 10  /

경   과: 00:00:29.31
```
루프를 돌면서 건건이 Call이 발생했지만, 네트워크를 경유하지 않는 Recursive Call이므로 그나마 29초 만에 수행을 마쳤다.

- #### [커밋과 성능]

  다음 테스트를 진행하기에 앞서 커밋이 성능에 미치는 영향을 확인해 보자. 조금 전 테스트에선 모든 루프 처리하기를 완료하고서 커밋했는데, 커밋을 루프 안쪽으로 옮겨서 실행해 보자.

  ```
  SQL> begin
    2    for s in (select * from source)
    3    loop
    4
    5      insert into target values ( s.no, s.empno, s.ename, s.job, s.mgr
    6                              , s.hiredate, s.sal, s.comm, s.deptno);
    7
    8      commit;
    9
   10    end loop;
   11
   12  end;
   13  /
  경   과: 00:01:00.50
  ```
  29초 걸리던 프로그램 수행시간이 1분으로 늘었다. 성능도 문제이지만, 이런 식으로 커밋을 자주 발행하면 트랜잭션 원자성(Atomicity)에도 문제가 생긴다.

  반대로, 매우 오래 걸리는 트랜잭션을 한 번도 커밋하지 않고 진행하면 Undo 공간 부족으로 인해 시스템에 여러 부작용을 초래할 수 있다.
  트랜잭션의 원자성을 위해 반드시 그렇게 처리해야 한다면 Undo 공간을 늘려야 하지만, 그렇지 않다면 적당한 주기로 커밋하는 방안을 고려할 수 있다. 루프 안쪽에 아래와 같은 코드를 삽입하면 된다.

  ```
  if mod(i, 100000) = 0 then  -- 10만 번에 한 번씩 커밋
    commit;
  end if;
  ```
  이렇게 처리하면 맨 마지막에 한 번 커밋하는 것과 비교해 성능 차이가 크지 않다.

아래와 같이 JAVA 프로그램으로 수행하면 네트워크를 경유하는 User Call이므로 성능이 급격히 나빠진다.
```
public class JavaLoopQuery {
  public void execute() throws Exception {
    String SQLStmt = "select no, empno, ename, job, mgr"
                  + ", to_char(hiredate, 'yyyymmdd hh24miss'), sal, comm, deptno "
                  + "from    source";

    PreparedStatement stmt = con.prepareStatement(SQLStmt);
    ResultSet rs = stmt.executeQuery();
    while(rs.next()) {
      long    no        = rs.getLong(1);
      long    empno     = rs.getLong(2);
      String  ename     = rs.getString(3);
      String  job       = rs.getString(4);
      Integer mgr       = rs.getInt(5);
      String  hiredate  = rs.getString(6);
      long    sal       = rs.getLong(7);
      long    comm      = rs.getLong(8);
      long    deptno    = rs.getLong(9);

      insertTarget(con, no, empno, ename, job, mgr, hiredate, sal, comm, deptno);
    }
    rs.close();
    stmt.close();
  }

  public void insertTarget( long    p_no
                          , long    p_empno
                          , String  p_ename
                          , String  p_job
                          , int     p_mgr
                          , String  p_hiredate
                          , long    p_sal
                          , long    p_comm
                          , int     p_deptno) throws Exception {
    String SQLStmt = "insert into target "
            + "(no, empno, ename, job, mgr, hiredate, sal, comm, deptno) "
            + "values (?, ?, ?, ?, ?, to_date(?, 'yyyymmdd hh24miss'), ?, ?, ?)";

    PreparedStatement st = con.prepareStatement(SQLStmt);
    st.setLong      (1, p_no      );
    st.setLong      (2, p_empno   );
    st.setString    (3, p_ename   );
    st.setString    (4, p_job     );
    st.setInt       (5, p_mgr     );
    st.setsetString (6, p_hiredate);
    st.setLong      (7, p_sal     );
    st.setLong      (8, p_comm    );
    st.setInt       (9, p_deptno  );
    st.execute();
    st.close();
  }
...
...
}
```
아래는 위 JAVA 프로그램을 컴파일하고 실행한 결과다.
```
c:\Test>javac JavaLoopQuery.java

c:\Test>java JavaLoopQuery.java

elapsed time : 218.392초
```
29초 걸리던 PL/SQL 프로그램을 JAVA로 구현했더니 218초나 걸렸다.

### [One SQL의 중요성]
아래와 같이 Insert Into Select 구문으로 수행해 보자.
```
SQL> insert into target
  2  select * from source;

1000000 개의 행이 만들어졌습니다.

경    과: 00:00:01.46
```
한 번의 Call로 처리하니 1.46초 만에 수행을 마쳤다. JAVA 프로그램과 비교하면 150배 빨라졌다. One SQL의 중요성이 바로 여기에 있다.
업무 로직이 복잡하면 절차적으로 처리할 수밖에 없지만, 그렇지 않다면 가급적 One SQL로 구현하려고 노력해야 한다.

절차적으로 구현된 프로그램을 One SQL로 구현하는 데 매우 유용한 아래 구문 활용법을 잘 익혀두기 바란다. 수정가능 조인 뷰와 Merge 문 활용법에 대해서는 이후에 다룰 예정이다.
- Insert Into Select
- 수정가능 조인 뷰
- Merge 문

<br/>

## (3) Array Processing 활용
실무에서 절차적 프로그램을 One SQL로 구현하는 일은 절대 쉽지 않다. 복잡한 업무 로직을 포함하는 경우가 많기 때문이다.
그럴 때 Array Processing 기능을 활용하면 One SQL로 구현하지 않고도 Call 부하를 획기적으로 줄일 수 있다.

앞서 테스트한 절차적 프로그램을 PL/SQL에서 Array Processing으로 처리하면 아래와 같다.
```
SQL> declare
  2    cursor c is select * from source;
  3    type typ_source is table of c%rowtype;
  4    l_source typ_source;
  5
  6    l_array_size number default 10000;
  7
  8    procedure insert_target( p_source   in typ_source) is
  9    begin
 10      forall i in p_source.first..p_source.last
 11      insert into target values p_source(i);
 12    end insert target;
 13
 14  begin
 15    open c;
 16    loop
 17      fetch c bulk collect into l_source limit l_array_size;
 18
 19      insert_target( l_source );
 20
 21      exit when c%notfound;
 22    end loop;
 23
 24    close c;
 25
 26    commit;
 27  end;
 28  /

경    과: 00:00:03.99
```
절차적으로 수행할 때 29.31초 걸리던 PL/SQL 프로그램이 3.99초 만에 수행을 마쳤다. JAVA 프로그램에서 Array Processing으로 처리하면 아래와 같다.
```
public class JavaArrayProcessing {
  public void execute() throws Exception {
    int      arraySize = 10000;
    long     no       [] = new long    [arraySize];
    long     empno    [] = new long    [arraySize];
    String   ename    [] = new String  [arraySize];
    String   job      [] = new String  [arraySize];
    int      mgr      [] = new int     [arraySize];
    String   hiredate [] = new String  [arraySize];
    long     sal      [] = new long    [arraySize];
    long     comm     [] = new long    [arraySize];
    int      deptno   [] = new int     [arraySize];

    String SQLStmt = "select no, empno, ename, job, mgr"
                   + ", to_char(hiredate, 'yyyymmdd hh24miss'), sal, comm, deptno "
                   + "from   source";
    PreparedStatement st = con.prepareStatement(SQLStmt);
    st.setFetchSize(arraySize);

    ResultSet rs = st.executeQuery();

    int i = 0;
    while(rs.next()){
      no      [i] = rs.getLong(1);
      empno   [i] = rs.getLong(2);
      ename   [i] = rs.getString(3);
      job     [i] = rs.getString(4);
      mgr     [i] = rs.getInt(5);
      hiredate[i] = rs.getString(6);
      sal     [i] = rs.getLong(7);
      comm    [i] = rs.getLong(8);
      deptno  [i] = rs.getInt(9);

      if(++i == arraSize) {    // 10,000번에 한 번씩 insertTarget 실행
        insertTarget(i, no, empno, ename, job, mgr, hiredate, sal, comm, deptno);
        i = 0;
      }
    }

    if (i > 0) insertTarget(i, no, empno, ename, job, mgr, hiredate, sal, comm, deptno);

    rs.close();
    st.close();
  }

  public void insertTarget( int        length
                          , long   []  p_no
                          , long   []  p_empno
                          , String []  p_ename
                          , String []  p_job
                          , int    []  p_mgr
                          , String []  p_hiredate
                          , long   []  p_sal
                          , long   []  p_comm
                          , int    []  p_deptno) throws Exception {
    String SQLStmt = "insert into target "
            + "(no, empno, ename, job, mgr, hiredate, sal, comm, deptno) "
            + "values (?, ?, ?, ?, ?, to_date(?, 'yyyymmdd hh24miss'), ?, ?, ?)";

    PreparedStatement st = con.prepareStatement(SQLStmt);

    for(int i=0; i < length; i++){
      st.setLong      (1, p_no      [i]);
      st.setLong      (2, p_empno   [i]);
      st.setString    (3, p_ename   [i]);
      st.setString    (4, p_job     [i]);
      st.setInt       (5, p_mgr     [i]);
      st.setsetString (6, p_hiredate[i]);
      st.setLong      (7, p_sal     [i]);
      st.setLong      (8, p_comm    [i]);
      st.setInt       (9, p_deptno  [i]);
      st.addBatch();      // insert할 값들을 배열에 저장
    };
    st.executeBatch();    // 배열에 저장된 값을 한 번에 insert
    st.close();
  }
...
...
}
```
아래는 위 프로그램을 컴파일하고 실행한 결과다.
```
c:\Test>javac JavaArrayProcessing.java

c:\Test>java JavaArrayProcessing.java

elapsed time : 11.813
```
절차적으로 수행할 때 218초 걸리던 JAVA 프로그램이 11.8초 만에 수행을 마쳤다. 만 번에 한 번씩 INSERT 하도록 구현함으로써 백만 번 발생할 Call을 백 번으로 줄였기 때문에 나타난 성능 향상이다.
한 번과 백만 번의 차이는 크지만, 한 번과 백 번의 차이는 크지 않다.
방금 확인한 것처럼, Call을 단 하나로 줄이지 못하더라도 Array Processing을 활용해 10 ~ 100번 수준으로 줄일 수 있다면 One SQL에 준하는 성능효과를 얻을 수 있다.

<br/>

## (4) 인덱스 및 제약 해제를 통한 대량 DML 튜닝
앞서 설명했듯 인덱스와 무결성 제약 조건은 DML 성능에 큰 영향을 끼친다. 그렇다고 온라인 트랜잭션 처리 시스템에서 이들 기능을 해제할 순 없다.
반면, 동시 트랜잭션 없이 대량 데이터를 적재하는 배치(Batch) 프로그램에서는 이들 기능을 해제함으로써 큰 성능개선 효과를 얻을 수 있다.

테스트를 위해 아래와 같이 테이블과 인덱스를 생성해 보자. 앞서 했던 테스트보다 데이터를 열 배 더 늘려 SOURCE 테이블에 1,000만 건을 입력했다.
```
SQL> create table source
  2  as
  3  select b.no, a.*
  4  from   (select * from emp where rownum <= 10) a
  5       , (select rownum as no from dual connect by level <= 1000000) b;

SQL> create table target
  2  as
  3  select * from source where 1 = 2;

SQL> alter table target add
  2  constraint target_pk primary key(no, empno);

SQL> create index target_x1 on target(ename);
```
PK 제약을 생성하면 Unique 인덱스가 자동으로 생성된다. 추가로 일반 인덱스를 하나 더 만들었으므로 인덱스는 총 두 개다. 이 상태에서 TARGET 테이블에 1,000만 건을 입력해 보자.
```
SQL> set timing on;

SQL> insert /*+ append */ into target
  2  select * from source;

10000000 개의 행이 만들어졌습니다.

경    과: 00:01:19.10
SQL> commit;
```
PK 제약과 인덱스가 있는 상태에서 1분 19초가 걸렸다.

### [PK 제약과 인덱스 해제 1 - PK 제약에 Unique 인덱스를 사용한 경우]
이번에는 PK 제약과 인덱스를 해제한 상태에서 데이터를 입력해 보자.
```
SQL> truncate table target;

SQL> alter table target modify constraint target_pk disable drop index;
```

PK 제약을 비활성화하면서 인덱스도 Drop 했다.
```
SQL> alter index target_x1 unusable;
```
일반 인덱스는 Unusable 상태로 변경했다. 인덱스가 Unusable인 상태에서 데이터를 입력하려면 skip_unusable_indexes 파라미터를 아래와 같이 true로 설정해야 한다.
기본 값이 true이므로 이전에 변경한 적이 없다면, 여기서 굳이 설정을 변경하지 않아도 된다.
```
SQL> alter session set skip_unusable_indexes = true;
```
무결성 제약과 인덱스를 해제함으로써 빠르게 INSERT 할 준비가 됐다. 다시 1,000만 건을 입력해 보자.
```
SQL> insert /*+ append */ into target
  2  select * from source;

10000000 개의 행이 만들어졌습니다.

경    과: 00:00:05.84
SQL> commit;
```

이번에는 5.84초 만에 수행을 마쳤다. 이제 PK 제약을 활성화하고 일반 인덱스를 재생성하면 모든 작업이 끝난다. PK 제약을 활성화하면 PK 인덱스는 자동으로 생성한다.
```
SQL> alter table target modify constraint target_pk enable NOVALIDATE;

경    과: 00:00:06.77
SQL> alter index target_x1 rebuild;

경    과: 00:00:08.26
```
데이터 입력 시간과 제약 활성화 및 인덱스 재생성 시간을 합쳐도 기존(1분 19초)보다 훨씬 더 빨리(21초) 작업을 마쳤다.
인덱스 및 무결성 제약이 DML 성능에 미치는 영향이 이렇게 크다.

PK 제약을 활성화하면서 NOVALIDATE 옵션을 사용한 것도 시간을 단축하는 데 한몫했다. 이는 기입력된 데이터에 대한 무결성 체크를 생략하도록 하는 옵션이다.
데이터 무결성에 확신이 없다면, 데이터를 입력하기 전에 아래 쿼리로 확인해야 한다.
```
SQL> select no, empno, count(*)
  2  from    source
  3  group by no, empno
  4  having count(*) > 1;
```

### [PK 제약과 인덱스 해제 2 - PK 제약에 Non-Unique 인덱스를 사용한 경우]
조금 전 테스트에서 X1 인덱스는 Unusable 상태로 변경했지만, PK 인덱스는 제약을 비활성화하면서 아예 Drop 해 버렸다(PK 제약을 비활성화할 때 사용한 drop index 옵션 참조).
PK 인덱스는 Unusable 상태에서 데이터를 입력할 수 없기 때문이다. 아래 실행결과를 확인하기 바란다.
```
SQL> alter index target_pk unusable;

SQL> insert into target
  2  select * from source;
1행에 오류:
ORA-01502: index 'TARGET_PK' or partition of such index is in unusable state

SQL> insert /*+ append */ into target
  2  select * from source;
1행에 오류:
ORA-26026: unique index TARGET_PK initially in unusable state
```
PK 인덱스를 Drop 하지 않고 Unusable 상태에서 데이터를 입력하고 싶다면, 아래와 같이 PK 제약에 Non-Unique 인덱스를 사용하면 된다.
```
SQL> set timing off;
SQL> truncate table target;

SQL> alter table target drop primary key drop index;

SQL> create index target_pk on target(no, empno);  -- Non-Unique 인덱스 생성

SQL> alter table target add
  2  constraint target_pk primary key (no, empno)
  3  using index target_pk;        -- PK 제약에 Non-Unique 인덱스 사용하도록 지정
```

아래와 같이 PK 제약을 비활성화하고, 인덱스를 Unusable 상태로 변경하자. PK 제약을 비활성화했지만, 인덱스는 Drop 하지 않고 남겨놨다.
```
SQL> alter table target modify constraint target_pk disable keep index;

SQL> alter index target_pk unusable;

SQL> alter index target_x1 unusable;
```
이제 아래와 같이 대량 INSERT 작업을 진행하는 데 아무런 문제가 없다.
```
SQL> set timing on;
SQL> insert /*+ append */ into target
  2  select * from source;

10000000 개의 행이 만들어졌습니다.

경    과: 00:00:05.53
SQL> commit;

경    과: 00:00:00.00
```
작업을 마쳤으면, 인덱스를 재생성하고 PK 제약을 다시 활성화한다.
```
SQL> alter index target_x1 rebuild;

경    과: 00:00:07.24
SQL> alter index target_pk rebuild;

경    과: 00:00:05.27
SQL> alter table target modify constraint target_pk enable novalidate;

경    과: 00:00:00.06
```
데이터 입력 시간과 제약 활성화 및 인덱스 재생성 시간을 합쳐도 기존(1분 19초)보다 훨씬 더 빨리(18초) 작업을 마쳤다.

<br/>

## (5) 수정가능 조인 뷰
### [전통적인 방식의 UPDATE]
튜닝을 하다 보면 아래와 같이 작성된 UPDATE 문을 종종 볼 수 있다.
```
update 고객 c
set    최종거래일시 = (select max(거래일시) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
     , 최근거래횟수 = (select count(*) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
     , 최근거래금액 = (select sum(거래금액) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```
위 UPDATE 문은 아래와 같이 고칠 수 있다.
```
update 고객 c
set    (최종거래일시, 최근거래횟수, 최근거래금액) =
       (select max(거래일시), count(*), sum(거래금액)
        from   거래
        where  고객번호 = c.고객번호
        and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```
위 방식에도 비효율이 없는 것은 아니다. 한 달 이내 고객별 거래 데이터를 두 번 조회하기 떄문이다. 총 고객 수와 한 달 이내 거래 고객 수에 따라 성능이 좌우된다.

총 고객 수가 아주 많다면 Exists 서브쿼리를 아래와 같이 해시 세미 조인으로 유도하는 것을 고려할 수 있다.
```
update 고객 c
set    (최종거래일시, 최근거래횟수, 최근거래금액) =
       (select max(거래일시), count(*), sum(거래금액)
        from   거래
        where  고객번호 = c.고객번호
        and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select /*+ unnest hash_sj */ 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```
만약 한 달 이내 거래를 발생시킨 고객이 많아 UPDATE 발생량이 많다면, 아래와 같이 변경하는 것을 고려할 수 있다.
하지만 모든 고객 레코드에 LOCK이 걸리는 것은 물론, 이전과 같은 값으로 갱신되는 비중이 높을수록 Redo 로그 발생량이 증가해 오히려 비효율적일 수 있다.
```
update 고객 c
set    (최종거래일시, 최근거래횟수, 최근거래금액) =
       (select nvl(max(거래일시), c.최종거래일시)
             , decode(count(*), 0, c.최근거래횟수, count(*))
             , nvl(sum(거래금액), c.최근거래금액)
        from   거래
        where  고객번호 = c.고객번호
        and    거래일시 >= trunc(add_months(sysdate, -1)))
```
이처럼 다른 테이블과 조인이 필요할 때 전통적인 UPDATE 문을 사용하면 비효율을 완전히 해소할 수 없다.

### [수정가능 조인 뷰]
아래와 같이 수정가능 조인 뷰를 활용하면 참조 테이블과 두 번 조인하는 비효율을 없앨 수 있다.
- 아래 쿼리는 12c 이상 버전에서만 정상적으로 실행된다. 10g 이하 버전에서는 UPDATE 옆에 bypass_ujvc 힌트를 사용하면 실행할 수 있다. 11g에서는 실행되지 않는다. 이유는 잠시 후에 살펴보자.)

```
update
( select /*+ ordered use_hash(c) no_merge(t) */
         c.최종거래일시, c.최근거래횟수, c.최근거래금액
       , t.거래일시, t.거래횟수, t.거래금액
from    (select 고객번호
              , max(거래일시) 거래일시, count(*) 거래횟수, sum(거래금액) 거래금액
         from   거래
         where  거래일시 >= trunc(add_months(sysdate, -1))
         group by 고객번호) t
       , 고객 c
where   c.고객번호 = t.고객번호
)

set 최종거래일시 = 거래일시
  , 최근거래횟수 = 거래횟수
  , 최근거래금액 = 거래금액
```
'조인 뷰'는 FROM 절에 두 개 이상 테이블을 가진 뷰를 가리키며, '수정가능 조인 뷰(Updatable/Modifiable Join View)'는 말 그대로 입력, 수정, 삭제가 허용되는 조인 뷰를 말한다.
단, 1쪽 집합과 조인하는 M쪽 집합에만 입력, 수정, 삭제가 허용된다.

아래와 같이 생성한 조인 뷰를 통해 job = 'CLERK'인 레코드의 loc을 모두 'SEOUL'로 변경하는 것을 '허용한다면' 어떤 일이 발생할까?
```
SQL> create table emp  as select * from scott.emp;
SQL> create table dept as select * from scott.dept;

SQL> create or replace view EMP_DEPT_VIEW as
  2  select e.rowid emp_rid, e.*, d.rowid dept_rid, d.dname, d.loc
  3  from emp e, dept d
  4  where  e.deptno = d.deptno;

SQL> update EMP_DEPT_VIEW set loc = 'SEOUL' where job = 'CLERK';
```
아래 쿼리 결과를 보면 job = 'CLERK'인 사원이 10, 20, 30 부서에 모두 속해 있는데, 위와 같이 UPDATE를 수행하고 나면 세 부서의 소재지(loc)가 모두 'SEOUL'로 바뀔 것이다.
세 부서의 소재지가 같다고 이상할 것이 없지만 다른 job을 가진 사원의 부서 소재지까지 바뀌는 것은 원하던 결과가 아닐 것이다.
```
SQL> select empno, ename, job, sal, deptno, dname, loc
  2  from   EMP_DEPT_VIEW
  3  order by job, deptno;

EMPNO  ENAME          JOB                   SAL        DEPTNO DNAME          LOC
------ -------------- ------------- ----------- ------------- -------------- -----------
7902   FORD           ANALYST              3000            20 RESEARCH       DALLAS
7788   SCOTT          ANALYST              3000            20 RESEARCH       DALLAS
7934   MILLER         CLERK                1300            10 ACCOUNTING     NEW YORK    ***
7369   SMITH          CLERK                 800            20 RESEARCH       DALLAS      ***
7876   ADAMS          CLERK                1100            20 RESEARCH       DALLAS      ***
7900   JAMES          CLERK                 950            30 SALES          CHICAGO     ***
7782   CLARK          MANAGER              2450            10 ACCOUNTING     NEW YORK
7566   JONES          MANAGER              2975            20 RESEARCH       DALLAS
7698   BLAKE          MANAGER              2850            30 SALES          CHICAGO
7839   KING           PRESIDENT            5000            10 ACCOUNTING     NEW YORK
7654   MARTIN         SALESMAN             1250            30 SALES          CHICAGO
7844   TURNER         SALESMAN             1500            30 SALES          CHICAGO
7521   WARD           SALESMAN             1250            30 SALES          CHICAGO
7499   ALLEN          SALESMAN             1600            30 SALES          CHICAGO

14 rows selected.
```
아래 UPDATE는 어떤가? 1쪽 집합(DEPT)과 조인하는 M쪽 집합(EMP)의 컬럼을 수정하므로 문제가 없어 보인다.
```
SQL> update EMP_DEPT_VIEW set comm = nvl(comm, 0) + (sal * 0.1) where sal <= 1500;
```
하지만, 실제 수행해 보면 아래와 같은 에러가 발생한다. (앞선 UPDATE 문에서도 똑같은 에러가 발생한다.)
```
ORA-01779: cannot modify a column which maps to a non key-preserved table
```
옵티마이저가 지금 어느 테이블이 1쪽 집합인지 알 수 없기 때문에 발생하는 에러다. 아래와 같이 지금 상태에서는 DELETE도 허용하지 않는다. INSERT도 마찬가지다.
```
SQL> delete from EMP_DEPT_VIEW where job = 'CLERK';
1행에 오류:
ORA-01752: cannot delete from view without exactly one key-preserved table
```
아래와 같이 1쪽 집합에 PK 제약을 설정하거나 Unique 인덱스를 생성해야 수정가능 조인 뷰를 통한 입력/수정/삭제가 가능하다.
```
SQL> alter table dept add constraint dept_pk primary key(deptno);

SQL> update EMP_DEPT_VIEW set comm = nvl(comm, 0) + (sal * 0.1) where sal <= 1500;

4 rows updated.
```
위와 같이 PK 제약을 설정하면 EMP 테이블은 '키-보존 테이블(Key-Preserved Table)'이 되고, DEPT 테이블은 '비 키-보존 테이블(Non Key-Preserved Table)'로 남는다.

### [키 보존 테이블이란?]
'키 보존 테이블'이란, 조인된 결과집합을 통해서도 중복 값 없이 Unique하게 식별이 가능한 테이블을 말한다.
Unique한 1쪽 집합과 조인되는 테이블이어야 조인된 결과집합을 통한 식별이 가능하다.

앞서 생성한 EMP_DEPT_VIEW 뷰에서 rowid를 함께 출력해 보자.
```
SQL> select ROWID, emp_rid, dept_rid, empno, deptno from EMP_DEPT_VIEW;

ROWID                 EMP_RID               DEPT_RID                     EMPNO       DEPTNO
--------------------- --------------------- --------------------- ------------ ------------
AAAMt4AAGAAAEWSAAA    AAAMt4AAGAAAEWSAAA    AAAMt5AAGAAAEWaAAB            7369           20
AAAMt4AAGAAAEWSAAB    AAAMt4AAGAAAEWSAAB    AAAMt5AAGAAAEWaAAC            7499           30
AAAMt4AAGAAAEWSAAC    AAAMt4AAGAAAEWSAAC    AAAMt5AAGAAAEWaAAC            7521           30
AAAMt4AAGAAAEWSAAD    AAAMt4AAGAAAEWSAAD    AAAMt5AAGAAAEWaAAB            7566           20
AAAMt4AAGAAAEWSAAE    AAAMt4AAGAAAEWSAAE    AAAMt5AAGAAAEWaAAC            7654           30
AAAMt4AAGAAAEWSAAG    AAAMt4AAGAAAEWSAAG    AAAMt5AAGAAAEWaAAA            7782           10
......                ......                ......                         ...          ...


14 rows selected.
```
dept_rid에 중복 값이 나타나고 있다. emp_rid에는 중복 값이 없으며 뷰의 rowid와 일치한다. 단적으로 말해 '키 보존 테이블'이란, 뷰에 rowid를 제공하는 테이블을 말한다.

아래와 같이 DEPT 테이블로부터 Unique 인덱스를 제거하면 키 보존 테이블이 없기 때문에 뷰에서 rowid를 출력할 수 없게 된다.
```
SQL> alter table dept drop primary key;

SQL> select rowid, emp_rid, dept_rid, empno, deptno from EMP_DEPT_VIEW;
1행에 오류:
ORA-01445: cannot select ROWID from, or sample, a join view without a key-preserved table
```

### [ORA-01779 오류 회피]
아래와 같이 DEPT 테이블에 AVG_SAL 컬럼을 추가하자.
```
SQL> alter table dept add avg_sal number(7,2);
```
아래는 EMP로부터 부서별 평균 급여를 계산해서 방금 추가한 컬럼에 반영하는 UPDATE 문이다.
```
SQL> update
  2  (select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
  3   from  (select deptno, round(avg(sal), 2) avg_sal from emp group by deptno) e
  4        , dept d
  5   where  d.deptno = e.deptno )
  6  set d_avg_sal = e_avg_sal ;
```
11g 이하 버전에서 위 UPDATE 문을 실행하면 아래와 같이 ORA-01779 에러가 발생한다.
EMP 테이블을 DEPTNO로 Group By 했으므로 DEPTNO 컬럼으로 조인한 DEPT 테이블은 키가 보존되는데도 옵티마이저가 불필요한 제약을 가한 것이다.
```
ORA-01779: cannot modify a column which maps to a non key-preserved table
```
이럴 때 10g에선 아래와 같이 bypass_ujvc 힌트를 이용해 제약을 회피할 수 있었다. Updatable Join View Check 를 생략하라고 옵티마이저에게 지시하는 힌트다.
```
SQL> update /*+ bypass_ujvc */
  2  (select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
  3   from   (select deptno, round(avg(sal), 2) avg_sal from emp group by deptno) e
  4         , dept d
  5   where   d.deptno = e.deptno )
  6  set d_avg_sal = e_avg_sal ;
```
11g부터 이 힌트를 사용할 수 없게 되었고, 따라서 위 UPDATE 문을 실행할 방법이 없다. 뒤에서 설명할 MERGE 문으로 바꿔줘야 한다.

오해하지 말 것은, bypass_ujvc 힌트 사용이 중단됐을 뿐, 수정가능 조인 뷰 사용은 중단되지 않았다는 사실이다.
11g에서도 1쪽 집합에 Unique 인덱스가 있으면, 수정가능 조인 뷰를 이용한 UPDATE가 가능하다. 과거에도 가능했고, 지금도 가능하다.

수정가능 조인 뷰는 12c에서 오히려 기능이 개선됐다. 즉, 12c부터는 힌트를 사용하지 않아도 위 UPDATE 문이 잘 실행된다.
Gropu By 한 집합과 조인한 테이블은 키가 보존된다는 사실을 오라클이 인정하기 시작한 것이다.

이런 기능 개선은 수정가능 조인 뷰의 활용성을 높여준다. 예를 들어, 고객_t 테이블 고객번호에 Unique 인덱스가 없으면, 아래 쿼리는 어떤 버전에서도 실행할 수 없다.
```
update (
  select o.주문금액, o.할인금액, c.고객등급
  from   주문_t o, 고객_t c
  where  o.고객번호 = c.고객번호
  and    o.주문금액 >= 1000000
  and    c.고객등급 = 'A')
set 할인금액 = 주문금액 * 0.2, 주문금액 = 주문금액 * 0.8
```
12c에서는 아래와 같이 고객_t 테이블을 Group By 처리함으로써 ORA-01779 에러를 회피할 수 있다.
```
update (
  select o.주문금액, o.할인금액
  from   주문_t o
       , (select 고객번호 from 고객_t where 고객등급 = 'A' group by 고객번호) c
  where  o.고객번호 = c.고객번호
  and    o.주문금액 >= 1000000)
set 할인금액 = 주문금액 * 0.2, 주문금액 = 주문금액 * 0.8
```
배치 프로그램이나 데이터 이행 프로그램에서 사용하는 중간 임시 테이블에는 일일이 PK 제약이나 인덱스를 생성하지 않으므로 이 패턴이 유용할 수 있다.

<br/>

## (6) MERGE 문 활용
DW에서 가장 흔히 발생하는 오퍼레이션은 기간계 시스템에서 가져온 신규 트랜잭션 데이터를 반영함으로써 두 시스템 간 데이터를 동기화하는 작업이다.

예를 들어, 고객(customer) 테이블에 발생한 변경분 데이터를 DW에 반영하는 프로세스는 다음과 같다. 이 중에서 3번 데이터 적재 작업을 효과적으로 지원하기 위해 오라클 9i에서 MERGE문이 도입됐다.

- [1] 전일 발생한 변경 데이터를 기간계 시스템으로부터 추출(Extraction)

  ```
  create table customer_delta
  as
  select * from customer
  where  mod_dt >= trunc(sysdate)-1
  and    mod_dt <  trunc(sysdate);
  ```

- [2] CUSTOMER_DELTA 테이블을 DW 시스템으로 전송(Transportation)
- [3] DW 시스템으로 적재(Loading)

  ```
  merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
  when matched then update
    set t.cust_nm = s.cust_nm, t.email = s.email, ...
  when not matched then insert
    (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
    (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
  ```

MERGE 문은 Source 테이블을 기준으로 Target 테이블과 Left Outer 방식으로 조인해서 조인에 성공하면 UPDATE, 실패하면 INSERT 한다.
MERGE 문을 UPSERT(=UPDATE + INSERT)라고도 부르는 이유다. 위 MERGE 문에서 Source 테이블은 Customer_Delta 테이블이고, Target 테이블은 Customer 테이블이다.

### [Optional Clauses]
아래와 같이 UPDATE와 INSERT를 선택적으로 처리할 수도 있다.
```
merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
when matched then update
  set t.cust_nm = s.cust_nm, t.email = s.email, ... ;

merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
when not matched then insert
  (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
  (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
```

이 확장 기능을 통해 아래와 같이 수정가능 조인 뷰 기능을 대체할 수 있게 되었다.
```
< 수정가능 조인 뷰 >
update
  (select d.deptno, d.avg_sal as d_avg_sal, e.avg_sal as e_avg_sal
   from   (select deptno, round(avg(sal), 2) avg_sal from emp group by deptno) e
         , dept d
   where   d.deptno = e.deptno)
set d_avg_sal = e_avg_sal ;

< Merge 문 >
merge into dept d
using (select deptno, rounc(avg(sal), 2) avg_sal from emp group by deptno) e
on (d.deptno = e.deptno)
when matched then update set d.avg_sal = e.avg_sal;
```

### [Conditional Operations]
ON 절에 기술한 조인문 외에 아래와 같이 추가로 조건절을 기술할 수도 있다.
```
merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
when matched then upate
  set t.cust_nm = s.cust_nm, t.email = s.email, ...
  where reg_dt >= to_date('20000101', 'yyyymmdd')
when not matched then insert
  (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
  (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt)
  where reg_dt < trunc(sysdate);
```

### [DELETE Clause]
이미 저장된 데이터를 조건에 따라 지우는 기능도 제공한다.
```
merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
when matched then
  update set t.cust_nm = s.cust_nm, t.email = s.email, ...
  delete where t.withdraw_dt is not null  -- 탈퇴일시가 null이 아닌 레코드 삭제
when not matched then
  (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
  (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
```
기억할 점은, 예시한 MERGE 문에서 UPDATE가 이루어진 결과로서 탈퇴일시(withdraw_dt)가 Null이 아닌 레코드만 삭제한다는 사실이다.
즉, 탈퇴일시가 Null이 아니었어도 MERGE문을 수행한 결과가 Null이면 삭제하지 않는다.

또 한가지 기억할 것은, MERGE 문 DELETE 절은 조인에 성공한 데이터만 삭제할 수 있다는 사실이다.
Source(=Customer_Delta) 테이블에서 삭제된 데이터는 Target(=Customer) 테이블에서도 지우고 싶을 텐데, MERGE 문 DELETE 절이 그 역할까지는 못한다는 뜻이다.
Source에서 삭제된 데이터는 조인에 실패하기 때문이다. 조인에 실패한 데이터는 UPDATE할 수도 없고 DELETE할 수도 없다.

결국 DELETE 절은, 조인에 성공한(Matched) 데이터를 모두 UPDATE 하고서 그 결과 값이 DELETE WHERE 조건절을 만족하면 삭제하는 기능이다.

### [Merge Into 활용 예]
저장하려는 레코드가 기존에 있던 것이면 UPDATE를 수행하고, 그렇지 않으면 INSERT 하려고 한다.
그럴 때 아래와 같이 처리하면 SQL을 '항상 두 번씩'(SELECT 한 번, INSERT 또는 UPDATE 한 번) 수행한다.
```
select count(*) into :cnt from dept where deptno = :val1;

if :cnt = 0 then
  insert into dept(deptno, dname, loc) values(:val1, :val2, :val3);
else
  update dept set dname = :val2, loc = :val3 where deptno = :val1;
end if;
```

아래와 같이 하면 SQL을 '최대 두 번' 수행한다.
```
update dept set dname = :val2, loc = :val3 where deptno = :val1;

if sql%rowcount = 0 then
  insert into dept(deptno, dname, loc) values(:val1, :val2, :val3);
end if;
```
아래와 같이 MERGE 문을 활용하면 SQL을 '한 번만' 수행한다.
```
merge into dept a
using (select :val1 deptno, :val2 dname, :val3 loc from dual) b
on    (b.deptno = a.deptno)
when matched then
  update set dname = b.dname, loc = b.loc
when not matched then
  insert (a.deptno, a.dname, d.loc) values (b.deptno, b.dname, b.loc);
```

- #### [수정가능 조인 뷰 vs. MERGE 문]

  UPDATE 문이 위기를 맞고 있다. UPDATE 대신 MERGE 문을 사용하는 개발자가 늘고 있기 때문이다. INSERT 없는 단순 UPDATE인데도 말이다.

  실행계획만 같다면 UPDATE 문을 사용하든 MERGE 문을 사용하든 상관은 없다. 그런데 개발자들이 작성한 SQL을 분석하다 보면 상관할 일이 자꾸 생긴다. 아래와 같은 패턴이 자주 보이기 때문이다.

  ```
  MERGE INTO EMP T2
  USING (SELECT T.ROWID AS RID, S.ENAME
         FROM   EMP T, EMP_SRC S
         WHERE  T.EMPNO = S.EMPNO
         AND    T.ENAME <> S.ENAME) S
  ON (T2.ROWID = S.RID)
  WHEN MATCHED THEN UPDATE SET T2.ENAME = S.ENAME;
  ```

  언제 누가 처음 사용하기 시작했는지 모르지만, 이런 패턴이 개발자들 사이에서 일반화돼 가는 느낌이 든다. 추정해 보면, UPDATE 대상 건수를 쉽게 확인할 수 있어서다.
  즉, SELECT 문을 먼저 만들어 데이터 검증을 마친 후 바깥에 MERGE 문을 씌우는 방식으로 SQL을 개발하는 것이다. MERGE 문 ON 절에는 ROWID를 사용했다.

  위 패턴이 성능에 안 좋은 이유는 자명하다. UPDATE 대상 테이블인 EMP를 두 번 액세스하기 때문이다. 실행계획을 확인해 보면 금방 알 수 있다.
  (ON절에 ROWID를 사용했으므로 성능에 문제가 없다고 생각할 수 있지만, 그렇지 않다. ROWID는 포인터가 아니다.)
  성능에 안 좋으니 아래와 같이 작성하라고 해도 개발자들이 UPDATE 대신 MERGE 문을 애용할지 궁금하다. 데이터 검증용 SELECT 문을 따로 하나 더 만드는 불편함이 있는데도 말이다.
  (복잡한 조인과 서브쿼리를 포함하는 경우, SELECT 문으로 검증할 필요가 있다.)

  ```
  MERGE INTO EMP T
  USING EMP_SRC S
  ON    (T.EMPNO = S.EMPNO)
  WHEN MATCHED THEN UPDATE SET T.ENAME = S.ENAME
  WHERE T.ENAME <> S.ENAME;
  ```

  차라리 아래 UPDATE 문(수정가능 조인 뷰)를 사용하면 편하지 않을까? SELECT 문을 먼저 만들어 데이터 검증을 마친 후, 바깥에 UPDATE 문을 씌우는 개발 패턴이다.
  ```
  UPDATE (
    SELECT S.ENAME AS S_ENAME, T.ENAME AS T_ENAME
    FROM   EMP T, EMP_SRC_S
    WHERE  T.EMPNO = S.EMPNO
    AND    T.ENAME <> S.ENAME
  )
  SET T.ENAME = S.ENAME;
  ```
  물론 EMP_SRC 테이블 EMPNO 컬럼에 Unique 인덱스가 생성돼 있어야 하는데, 대개는 있다.
  Unique 인덱스가 없으면, 10g까지는 bypass_ujvc 힌트를 통해, 12c부터는 Group By 처리를 통해 ORA-01779 에러를 회피할 수 있따. 11g에서는 사용할 수 없는 패턴이어서 문제긴 하다.
  - MERGE 문에서는 소스 집합(Using절)에 대한 Unique 인덱스를 요구하지 않지만, 따로 검증이 필요할 정도로 소스 집합이 복잡하다면 MERGE 문에서도 Group By 처리는 필요하다.
    정확히 UPDATE하기 위해서다.

  조인 UPDATE를 위해 앞으로 MERGE 문을 사용할 것인가, UPDATE 문을 사용할 것인가? 선택은 각자의 몫이다.