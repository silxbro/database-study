# 6.2 | Direct Path I/O 활용

온라인 트랜잭션은 기준성 데이터, 특정 고객, 특정 상품, 최근 거래 등을 반복적으로 읽기 때문에 버퍼캐시가 성능 향상에 큰 도움을 준다.
반면, 정보계 시스템(DW/OLAP 등)이나 배치 프로그램에서 사용하는 SQL은 주로 대량 데이터를 처리하기 때문에 버퍼캐시를 경유하는 I/O 메커니즘이 오히려 성능을 떨어뜨릴 수 있다.
그래서 오라클은 버퍼캐시를 경유하지 않고 곧바로 데이터 블록을 읽고 쓸 수 있는 Direct Path I/O 기능을 제공하는데, 지금부터 살펴보자.

<br/>

## (1) Direct Path I/O
일반적인 블록 I/O는 DB 버퍼캐시를 경유한다. 즉, 읽고자 하는 블록을 먼저 버퍼캐시에서 찾아보고, 찾지 못할 때만 디스크에서 읽는다. 데이터를 변경할 때도 먼저 블록을 버퍼캐시에서 찾는다.
찾은 버퍼블록에 변경을 가하고 나면, DBWR 프로세스가 변경된 블록(Dirty 블록)들을 주기적으로 찾아 데이터파일에 반영해 준다.

자주 읽는 블록에 대한 반복적인 I/O Call을 줄임으로써 시스템 전반적인 성능을 높이려고 버퍼캐시를 이용하지만, 대량 데이터를 읽고 쓸 때 건건이 버퍼캐시를 탐색한다면 개별 프로그램 성능에는 오히려 안 좋다.
버퍼캐시에서 블록을 찾을 가능성이 거의 없기 때문이다.

대량 블록을 건건이 디스크로부터 버퍼캐시에 적재하고서 읽어야 하는 부담도 크다.
그렇게 적재한 블록을 재사용할 가능성이 있느냐도 중요한데, Full Scan 위주로 가끔 수행되는 대용량 처리 프로그램이 읽어 들인 데이터는 대개 재사용성이 낮다.
그런 데이터 블록들이 버퍼캐시를 점유한다면 다른 프로그램 성능에도 나쁜 영향을 미친다.

그래서 오라클은 버퍼캐시를 경유하지 않고 곧바로 데이터 블록을 읽고 쓸 수 있는 Direct Path I/O 기능을 제공한다. 아래는 그 기능이 작동하는 경우다.
- [1] 병렬 쿼리로 Full Scan을 수행할 때
- [2] 병렬 DML을 수행할 때
- [3] Direct Path Insert를 수행할 때
- [4] Temp 세그먼트 블록들을 읽고 쓸 때
- [5] direct 옵션을 지정하고 export를 수행할 때
- [6] nocache 옵션을 지정한 LOB 컬럼을 읽을 때

- #### [병렬 쿼리]

  1 ~ 3번이 가장 중요하고 활용도가 높은데, 2번과 3번은 잠시 후 살펴본다. 반면, 1번 병렬 쿼리는 본서 어디이ㅔ서도 설명하지 않으므로 DML을 다루는 챕터임에도 짧게 소개하려고 한다.

  쿼리문에 아래처럼 parallel 또는 parallel_index 힌트를 사용하면, 지정한 병렬도(Parallel Degree)만큼 병렬 프로세스가 떠서 동시에 작업을 진행한다.

  ```
  select /*+ full(t) parallel(t 4) */ * from big_table t;

  select /*+ index_ffs(t big_table_x1) parallel_index(t big_table_x1 4) */ count(*)
  from  big_table t;
  ```

  놀랍게도, 위처럼 병렬도를 4로 지정하면, 성능이 네 배 빨라지는 게 아니라 수십 배 빨라진다. 바로 Direct Path I/O 때문에 나타나는 효과다.
  버퍼캐시를 탐색하지 않고, 디스크로부터 버퍼캐시에 적재하는 부담도 없으니 빠른 것이다.

  참고로, Order by, Group By, 해시 조인, 소트 머지 조인 등을 처리할 때는 힌트로 지정한 병렬도보다 두 배 많은 프로세스가 사용된다.

<br/>

## (2) Direct Path Insert
일반적인 INSERT가 느린 이유는 다음과 같다.

- [1] 데이터를 입력할 수 있는 블록을 Freelist에서 찾는다.
  테이블 HWM(High-Water Mark) 아래쪽에 있는 블록 중 데이터 입력이 가능한(여유 공간이 있는) 블록을 목록으로 관리하는데, 이를 'Freelist'라고 한다.
- [2] Freelist에서 할당받은 블록을 버퍼캐시에서 찾는다.
- [3] 버퍼캐시에 없으면, 데이터파일에서 읽어 버퍼캐시에 적재한다.
- [4] INSERT 내용을 Undo 세그먼트에 기록한다.
- [5] INSERT 내용을 Redo 로그에 기록한다.

Direct Path Insert 방식을 사용하면, 대량 데이터를 일반적인 INSERT보다 훨씬 더 빠르게 입력할 수 있다. 데이터를 Direct Path Insert 방식으로 입력하는 방법은 다음과 같다.

- INSERT ... SELECT 문에 append 힌트 사용
- parallel 힌트를 이용해 병렬 모드로 INSERT
- direct 옵션을 지정하고 SQL*Loader(sqlldr)로 데이터 적재
- CTAS(create table ... as select) 문 수행

Direct path Insert 방식이 빠른 이유는 다음과 같다.

- [1] Freelist를 참조하지 않고 HWM 바깥 영역에 데이터를 순차적으로 입력한다.
- [2] 블록을 버퍼캐시에서 탐색하지 않는다.
- [3] 버퍼캐시에 적재하지 않고, 데이터파일에 직접 기록한다.
- [4] Undo 로깅을 안 한다.
    - Undo 로깅을 최소화할 수 있다. Undo의 용도는 이전에 설명하였다.
      HWM 뒤쪽에 입력한 데이터는 커밋하기 전까지 다른 세션이 읽지 않으므로 [1] Read Consistency를 위해 Undo 데이터를 남기지 않아도 된다.
      INSERT 작업을 커밋하면 HWM을 이동하고, 롤백하면 할당된 익스텐트에 대한 딕셔너리 정보만 롤백하면 된다.
      따라서 [2] Transaction Rollback 또는 [3] Transaction Recovery를 위한 Undo는 남길 필요가 없다. 할당된 익스텐트 정보만 로깅하면 된다.
- [5] Redo 로깅을 안 하게 할 수 있다. 테이블을 아래와 같이 nologging 모드로 전환한 상태에서 Direct Path Insert 하면 된다.
    - 정확히 표현하면, Redo 로깅을 '최소화'할 수 있다. 데이터 딕셔너리 변경사항만 로깅하기 때문이다.

  ```
  alter table t NOLOGGING;
  ```

  참고로, Direct Path insert가 아닌 일반 INSERT 문을 로깅하지 않게 하는 방법은 없다.
    - Redo 로깅은 데이터베이스 영속성(Durability)을 위한 필수 기능이다.

- #### [작동하지 않는 nologging Insert]

  가끔 아래와 같이 코딩하는 개발자를 볼 수 있는데, 여기서 nologging은 T 테이블에 대한 별칭(Alias)일 뿐 nologging 기능과는 무관하다.

  ```
  insert into t NOLOGGING select * from test;
  ```
  아래와 같이 코딩하는 개발자도 볼 수 있는데, 오라클은 nologging 힌트를 제공하지 않는다.
  ```
  insert /*+ APPEND NOLOGGING */ into t select * from test;


Array Processing도 Direct Path Insert 방식으로 처리할 수 있다. append_values 힌트를 사용하면 된다.
'Array Processing 활용'에서 사용한 PL/SQL 코드를 예로 들면, 아래와 같이 하면 된다.
```
  7  ...
  8  procedure insert_target( p_source  in typ_source) is
  9  begin
 10    forall i in p_source.first..p_source.last
 11      insert /*+ append_values */ into target values p_source(i);
 12  end insert target;
 13  ...
```
Direct Path Insert를 사용할 때 주의할 점이 두 가지 있다.

첫째, 이 방식을 사용하면 성능은 비교할 수 없이 빨라지지만 Exclusive 모드 TM Lock이 걸린다는 사실이다. 따라서 커밋하기 전까지 다른 트랜잭션은 해당 테이블에 DML을 수행하지 못한다.
트랜잭셔닝 빈번한 주간에 이 옵션을 사용하는 것은 절대 금물이다. TM Lock은 '(6.4.1) DML 테이블 Lock'에서 설명한다.

둘째, Freelist를 조회하지 않고 HWM 바깥 영역에 입력하므로 테이블에 여유 공간이 있어도 재활용하지 않는다는 사실이다.
과거 데이터를 주기적으로 DELETE 해서 여유 공간이 생겨도 이 방식으로만 계속 INSERT하는 테이블은 사이즈가 줄지 않고 계속 늘어만 간다.
Range 파티션 테이블이면 과거 데이터를 DELETE가 아닌 파티션 DROP 방식으로 지워야 공간 반환이 제대로 이루어진다. 비파티션 테이블이면 주기적으로 Reorg 작업을 수행해 줘야 한다.
- DELETE 방식으로 지운 공간은 자동으로 반환되지 않는다. INSERT에 의해 재활용될 수 있을 뿐이다.
  따라서 Range 파티션 테이블은 일반 INSERT로 입력하더라도 과거 데이터를 DELETE가 아닌 파티션 DROP(또는 Truncate) 방식으로 지워야 한다.
  테이블을 시계열 컬럼 기준으로 Range 파티셔닝하면 INSERT 작업이 신규 파티션에서만 이루어지기 때문이다.

<br/>

## (3) 병렬 DML
INSERT는 append 힌트를 이용해 Direct Path Write 방식으로 유도할 수 있지만, UPDATE, DELETE는 기본적으로 Direct Path Write가 불가능하다.
유일한 방법은 병렬 DML로 처리하는 것이다. 병렬 처리는 대용량 데이터가 전제이므로 오라클은 병렬 DML에 항상 Direct Path Write 방식을 사용한다.

DML을 병렬로 처리하려면, 먼저 아래와 같이 병렬 DML을 활성화해야 한다.
```
alter session enable parallel dml;
```
그러고 나서 각 DML 문에 아래와 같이 힌트를 사용하면, 대상 레코드를 찾는 작업(INSERT는 SELECT 쿼리, UPDATE/DELETE는 조건절 검색)은 물론 데이터 추가/변경/삭제도 병렬로 진행한다.
```
insert /*+ parallel(c 4) */ into 고객 c
select /*+ full(o) parallel(o 4) */ * from 외부가입고객 o;

update /*+ full(c) parallel(c 4) */ 고객 c set 고객상태코드 = 'WD'
where  최종거래일시 < '20100101';

delete /*+ full(c) parallel(c 4) */ from 고객 c
where  탈퇴일시 <  '20100101';
```
힌트를 제대로 기술했는데, 만약 실수로 병렬 DML을 활성화하지 않으면 어떻게 될까? 대상 레코드를 찾는 작업은 병렬로 진행하지만, 추가/변경/삭제는 QC가 혼자 담당하므로 병목이 생긴다.
- 'Query Coordinator'의 줄임말이다.
  SQL을 병렬로 실행하면 병렬도로 지정한 만큼 또는 두 배로 병렬 프로세스를 띄워 동시에 작업을 진행하는데, 이때 최초 DB에 접속해서 SQL을 수행한 프로세스는 Query Coordinator 역할을 맡는다.
  단, 병렬로 처리할 수 없거나 병렬로 처리하도록 지정하지 않은 작업은 Query Coordinator가 직접 처리한다.

- #### [두 단계 전략]

  'Undo 로깅과 DML 성능'을 설명하면서 'Undo의 용도와 MVCC 모델'을 부연해서 설명하였다.
  MVCC 모델을 상기하면 '병렬 DML을 활성화해야 대상 레코드를 찾는 작업과 추가/변경/삭제 작업 모두 병렬로 처리할 수 있다'는 내용을 이해하는 데 도움이 된다.
  오라클은 DML 문에 두 단계 전략을 사용한다. 즉, Consistent 모드로 대상 레코드를 찾고 Current 모드로 추가/변경/삭제한다.

병렬 INSERT는 append 힌트를 지정하지 않아도 Direct Path Insert 방식을 사용한다. 하지만, 병렬 DML이 작동하지 않을 경우를 대비해 아래와 같이 append 힌트를 같이 사용하는 게 좋다.
혹시라도 병렬 DML이 작동하지 않더라도 QC가 Direct Path Insert를 사용하면 어느 정도 만족할 만한 성능을 낼 수 있기 때문이다.
```
insert /*+ append parallel(c 4) */ into 고객 c
select /*+ full(o) parallel(o 4) */ * from 외부가입고객 o;
```
12c부터는 아래와 같이 enable_parallel_dml 힌트도 지원한다.
```
insert /*+ enable_parallel_dml parallel(c 4) */ into 고객 c
select /*+ full(o) parallel(o 4) */ * from 외부가입고객 o;

update /*+ enable_parallel_dml full(c) parallel(c 4) */ 고객 c
set    고객상태코드 = 'WD'
where  최종거래일시 < '20100101';

delete /*+ enable_parallel_dml full(c) parallel(c 4) */ from 고객 c
where  탈퇴일시 < '20100101';
```
병렬 DML도 Direct Path Write 방식을 사용하므로 데이터를 입력/수정/삭제할 때 Exclusive 모드 TM Lock이 걸린다는 사실을 꼭 기억하자.
다시 강조하지만, 트랜잭션이 빈번한 주간에 이 옵션을 사용하는 것은 절대 금물이다.

### [병렬 DML이 잘 작동하는지 확인하는 방법]
DML 작업을 각 병렬 프로세스가 처리하는지, 아니면 QC가 처리하는지를 실행계획에서 확인할 수 있다.
아래와 같이 UPDATE(또는 DELETE/INSERT)가 'PX COORDINATOR' 아래쪽에 나타나면 UPDATE를 각 병렬 프로세스가 처리한다.
```
---------------------------------------------------------------------------------------------
| Id  | Operation                  | Name     | Psstart| Pstop |    TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT           |          |        |       |        |      |            |
|   1 |   PX COORDINATOR           |          |        |       |        |      |            |
|   2 |     PX SEND QC (RANDOM)    | :TQ10000 |        |       |  Q1,00 | P->S | QC (RAND)  |
|   3 |       UPDATE               | 고객      |        |       |  Q1,00 | PCWP |            |
|   4 |         PX BLOCK ITERATOR  |          |      1 |     4 |  Q1,00 | PCWC |            |
|   5 |           TABLE ACCESS FULL| 고객      |      1 |     4 |  Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------------
```
반면, 아래와 같이 UPDATE(또는 DELETE/INSERT)가 'PX COORDINATOR' 위쪽에 나타나면 UPDATE를 QC가 처리한다.
```
---------------------------------------------------------------------------------------------
| Id  | Operation                  | Name     | Psstart| Pstop |    TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT           |          |        |       |        |      |            |
|   1 |   UPDATE                   | 고객      |        |       |  Q1,00 | PCWP |            |
|   2 |     PX COORDINATOR         |          |        |       |        |      |            |
|   3 |       PX SEND QC (RANDOM)  | :TQ10000 |        |       |  Q1,00 | P->S | QC (RAND)  |
|   4 |         PX BLOCK ITERATOR  |          |      1 |     4 |  Q1,00 | PCWC |            |
|   5 |           TABLE ACCESS FULL| 고객      |      1 |     4 |  Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------------
```