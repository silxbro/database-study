# 05. Direct Path I/O

일반적인 블록 I/O는 DB 버퍼 캐시를 경유한다. 즉, 읽고자 하는 블록을 먼저 버퍼 캐시에서 찾아보고, 찾지 못할 때만 디스크에서 읽는다.
데이터 변경도 버퍼 캐시에 적재된 블록에서 이루어지며, DBWR 프로세스가 주기적으로 변경된 블록(Dirty 버퍼 블록)들을 데이터파일에 기록한다.

시스템 전반의 I/O 성능을 향상시키려고 버퍼 캐시를 이용하지만 개별 프로세스 입장에서 대용량 데이터를 읽고 쓸 때 건건이 버퍼 캐시를 경유한다면 오히려 성능이 나빠질 수 있다.
재사용 가능성이 없는 임시 세그먼트 블록들을 읽고 쓸 때도 버퍼 캐시를 경유하지 않는 것이 유리하다.
오라클은 이럴 때 버퍼 캐시를 경유하지 않고 곧바로 데이터 블록을 읽고 쓸 수 있는 Direct Path I/O 기능을 제공한다. 아래는 Direct Path I/O가 작동하는 경우다.

- Temp 세그먼트 블록들을 읽고 쓸 때
- 병렬 쿼리로 Full Scan을 수행할 때
- nocache 옵션을 지정한 LOB 컬럼을 읽을 때
- direct 옵션을 지정하고 export를 수행할 때
- parallel DML을 수행할 때
- Direct Path Insert를 수행할 때
  <br/>

## (1) Direct Path Read/Write Temp
데이터를 정렬할 때는, PGA 메모리에 할당되는 Sort Area를 이용한다.
정렬할 데이터가 많아 Sort Area가 부족해지면 Temp 테이블스페이스를 이용하는데, Sort Area에 정렬된 데이터를 Temp 테이블스페이스에 쓰고 이를 다시 읽을 때 Direct Part I/O 방식을 사용한다.
이 과정에서 I/O Call이 완료될 때까지 대기가 발생하는데, direct path write temp와 direct path read temp 이벤트로 측정된다.

```
create table t as select * from all_objects;

alter session set workarea_size_policy = manual;

alter session set sort_area_size = 524288;

select *
from (
  select a.*, rownum no
  from ( select * from t order by object_name ) a
)
where no <= 10

call     count       cpu    elapsed       disk      query    current       rows
------- ------  -------- ---------- ---------- ---------- ---------- ----------
Parse        1      0.00       0.00          0          0          0          0
Execute      1      0.00       0.00          0          0          0          0
Fetch        2      0.34       0.74        698        691         14         10
------- ------  -------- ---------- ---------- ---------- ---------- ----------
total        4      0.34       0.74        698        691         14         10

Rows     Row Source Operation
-------  --------------------------------------------------
    10   VIEW  (cr=691 pr=698 pw=698 time=281516 us)
 49867    COUNT  (cr=691 pr=698 pw=698 time=1727596 us)
 49867     VIEW  (cr=691 pr=698 pw=698 time=1328642 us)
 49867      SORT ORDER BY (cr=691 pr=698 pw=698 time=879823 us)
 49867       TABLE ACCESS FULL T (cr=691 pr=0 pw=0 time=249381 us)

Elapsed times include waiting on following events:
  Event waited on                        Times   Max. Wait  Total Waited
  -----------------------------------   Waited  ----------  ------------
  SQL*Net message to client                  2        0.00          0.00
  direct path write temp                   384        0.01          0.02
  direct path read temp                    486        0.03          0.36
  SQL*Net message from client                2        0.27          0.27
```
<br/>

## (2) Direct Path Read
병렬 쿼리로 Full Scan을 수행할 때도 Direct Path Read 방식을 사용한다.
병렬도(DOP)를 2로 주고 병렬쿼리를 수행하면 쿼리 수행 속도가 2배만 빨라지는 게 아니라 그 이상의 빠른 수행속도를 보이는 이유가 바로 여기에 있다.
따라서 대용량 데이터를 읽을 때는 Full Scan과 병렬 옵션을 적절히 사용함으로써 시스템 리소스를 적게 사용하도록 하는 것이 좋다.
Direct Path Read 과정에서 읽기 Call이 완료될 때까지 대기가 발생하는데, direct path read 이벤트로 측정된다.

버퍼 캐시에만 기록된 변경사항이 아직 데이터파일에 기록되지 않은 상태에서 데이터파일을 직접 읽으면 정합성에 문제가 생긴다.
따라서 병렬로 Direct Path Read를 수행하려면 메모리와 디스크간 동기화를 먼저 수행함으로써 Dirty 버퍼를 해소해야 한다.
예전에는 병렬 쿼리 수행 시 체크포인트를 통해 버퍼 캐시 전체를 데이터파일에 기록했지만 10gR2부터는 병렬 쿼리와 관련된 세그먼트만 동기화를 수행한다.

> #### [Adaptive Direct Path Reads]
> 사실 병렬 쿼리가 아니더라도 오라클 8.1.5 이후 버전부터는 Hidden 파라미터 _serial_direct_read를 true로 변경해 Direct Path Read 방식으로 읽도록 할 수 있다.
>
> 게다가 11g부터는 이 파라미터가 false인 상태에서도 Serial Direct Path Read 방식이 작동할 수 있으며, 이미 캐싱돼 있는 블록 개수, 디스크에 기록해야 할 Dirty 블록 개수(→ 세그먼트 단위
> 체크포인터 일량을 결정) 등에 따라 오라클이 결정한다.
<br/>

## (3) Direct Path Write
Direct Path Write에 대해서도 간단히 살펴보자. Direct Path Write는 병렬로 DML을 수행하거나 Direct Path Insert 방식으로 데이터를 insert 할 때 사용된다.
이 과정에서 I/O Call이 발생할 때마다 direct path write 이벤트가 나타난다.

아래는 Direct Path Insert 방식으로 데이터를 입력하는 방법이다.

- insert ... select 문장에 /*+ append */ 힌트 사용
- 병렬 모드로 insert
- direct 옵션을 지정하고 SQL*Loader(sqlldr)로 데이터를 로드
- CTAS(create table ... as select) 문장을 수행

일반적인(conventional) insert 시에는 Freelist를 통해 데이터를 삽입할 블록을 할당받는다.
Freelist를 조회하면서 Random 액세스 방식으로 버퍼 캐시에서 해당 블록을 찾고, 없으면 데이터파일에서 읽어 캐시에 적재한 후에 데이터를 삽입하므로 대량의 데이터를 insert 할 때 매우 느리다.

Direct Path Insert 시에는 Freelist를 참조하지 않고 테이블 세그먼트 또는 각 파티션 세그먼트의 HWM(High-Water Mark) 바깥 영역에 데이터를 순차적으로 입력한다.
Freelist로부터 블록을 할당받는 작업이 생략될 뿐 아니라 insert 할 블록을 버퍼 캐시에 적재하지 않고 데이터파일에 직접 insert 하므로 일반적인 insert와는 비교할 수 없을 정도로 빠르다.
HWM 바깥 영역에 데이터를 입력하므로 Undo 발생량도 최소화된다
(HWM 뒤쪽에 입력한 데이터는 커밋하기 전까지 다른 세션에 읽히지 않으므로 Undo 데이터를 제공하지 않아도 되고, 롤백할 때는 할당된 익스텐트에 대한 딕셔너리 정보만 롤백하면 되기 때문).

게다가 Direct Path Insert에서는 Redo 로그까지 최소화(데이터 딕셔너리 변경사항만 로깅)하도록 옵션을 줄 수도 있어 더 빠른 insert가 가능하다.
이 기능을 활성화하려면 테이블 속성을 nologging으로 바꿔주면 된다.

```
alter table t NOLOGGING;
```

가끔 아래처럼 코딩하는 개발자를 볼 수 있는데, 여기서 nologging은 T 테이블에 대한 별칭(Alias)일 뿐 nologging 기능과는 무관하다.

```
insert into t NOLOGGING select * from  test;
```

참고로, Direct Path Insert가 아닌 일반 insert문을 로깅하지 않도록 하는 방법은 없다.

주의할 것은, Direct Path Insert 방식으로 데이터를 입력하면 Exclusive 모드 테이블 Lock이 걸린다는 사실이다. 아래처럼 병렬 방식으로 DML을 수행할 때도 마찬가지다.

```
alter session enable parallel dml;
delete /*+ parallel(b 4) */ from big_table b ;  → Exclusive 모드 TM Lock
```

성능은 비교할 수 없을 정도로 빨라지겠지만 Exclusive 모드 테이블 Lock을 사용하면 해당 테이블에 다른 트랜잭션이 DML을 수행하지 못하도록 막는다.
(일반적인 DML 수행 시에는 Row Exclusive 모드 테이블 Lock을 사용하므로 다른 트랜잭션의 DML을 막지 않는다.) 따라서 트랜잭션이 빈번한 주간에 이 옵션을 사용하는 것은 절대 금물이다.