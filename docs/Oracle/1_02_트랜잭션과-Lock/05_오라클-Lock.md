# 05. 오라클 Lock

오라클은 공유 리소스와 사용자 데이터를 보호할 목적으로 DML Lock, DDL Lock, 래치(Latch), 버퍼 Lock, 라이브러리 캐시 Lock/Pin 등 다양한 종류의 Lock을 사용한다.
미처 다 열거하지 못했지만, 이 외에도 내부적으로 더 많은 종류의 Lock이 존재한다,

이 중 래치와 버퍼 Lock에 대해서는 1장에서 다루었다.

- 래치 : SGA에 공유돼 있는 갖가지 자료구조를 보호할 목적으로 사용하는 가벼운 Lock
- 버퍼 Lock : 버퍼 블록에 대한 액세스를 직렬화

라이브러리 캐시 Lock과 라이브러리 캐시 Pin에 대해서는 이후 자세히 다루지만 간단히 설명하면 라이브러리 캐시에 공유된 오브젝트 정의, 커서・PL/SQL 프로그램 같은 실행가능 오브젝트에 대한 정의 및
실행계획을 보호하는 Lock이다.

- 라이브러리 캐시 Lock : 라이브러리 캐시 오브젝트에 대한 핸들을 보호
- 라이브러리 캐시 Pin : 라이브러리 캐시 오브젝트의 실제 내용이 담긴 힙(Heap)을 보호
  <br/>

애플리케이션 개발 측면에서 가장 중요하게 다루어야 할 Lock은 무엇보다 **DML Lock**이다.
본 절에서 집중적으로 설명하려고 하는 DML Lock은, 다중 사용자에 의해 동시에 액세스되는 사용자 데이터의 무결성을 보호해 준다. DML Lock에는 테이블 Lock과 로우 Lock이 있다.

- **DML 테이블 Lock : Enqueue Lock으로 구현함**
- **DML 로우 Lock : 로우 단위 Lock과 트랜잭션 Lock을 조합해서 구현함(트랜잭션 Lock은 Enqueue Lock으로 구현)**

DML Lock을 이해하려면 Enqueue Lock 구조와 트랜잭션 Lock(TX Lock) 개념을 먼저 이해해야 한다. 본 절에서 설명할 내용을 미리 열거하면 다음과 같다.

- [1] Enqueue Lock
- [2] TX Lock (= 트랜잭션 Lock)
- [3] TX Lock ▶︎ 무결성 제약 위배 가능성 또는 비트맵 인덱스 엔트리 갱신
- [4] TX Lock ▶︎ ITL 슬롯 부족
- [5] TX Lock ▶︎ 인덱스 분할
- [6] TX Lock ▶︎ 기타 트랜잭션 Lock
- [7] TX Lock ▶︎ DML 로우 Lock
- [8] TX Lock ▶︎ DML 테이블 Lock
- [9] Lock을 푸는 열쇠, 커밋
  <br/>

## (1) Enqueue Lock
Enqueue는 공유 리소스에 대한 액세스를 관리하는 Lock 메커니즘이다. Enqueue에 의해 보호되는 공유 리소스로는 테이블, 트랜잭션, 테이블스페이스, 시퀀스, Temp 세그먼트 같은 것들이 있다.
Enqueue Lock은 래치와 달리 순서가 보장되는 큐(Queue) 구조를 사용한다. 따라서 대기자 큐(Queue)에 가장 먼저 Lock 요청을 등록한 세션이 가장 먼저 Lock을 획득한다.

Enqueue Lock으로 관리되는 공유 리소스에 대해 Lock을 획득하려면 먼저 'Enqueue 리소스'를 할당받아야 한다.(현재 할당된 Enqueue 리소스는 v$resource 뷰를 통해 확인할 수 있다.)
Enqueue 리소스는 소유자(owner), 대기자(waiter) 목록을 관리할 수 있는 구조체(Structure)를 말한다. 각 Enqueue 리소스에는 고유한 식별자가 부여되며, 식별자는 <Type-ID1-ID2>로 구성된다.
Type은 'TX', 'TM', 'TS'처럼 2개 문자열로 이루어지며, ID1, ID2에는 Lock 종류에 따라 다른 정보를 갖는다. 예를 들어, TM Lock 식별자에는 다음과 같은 정보를 포함한다.

- TYPE : TM
- ID1 : 오브젝트 ID
- ID2 : 0

TX Lock 식별자에는 다음과 같은 정보를 포함한다.

- TYPE : TX
- ID1 : Undo 세그먼트 번호 + 트랜잭션 슬롯번호
- ID2 : 트랜잭션 슬롯 Sequence 번호
  <br/>

오라클은 Enqueue 리소스 구조체를 통합 관리하는 리소스 테이블(일종의 Array)을 갖고 있으며, 리소스 테이블에서 관리되는 각 리소스를 찾을 때는 해싱 알고리즘을 사용한다.
(통합 관리하지 않고 버퍼 Lock이나 라이브러리 캐시 오브젝트처럼 개별 공유 리소스에서 소유자, 대기자 목록을 관리하는 경우도 있다.)
물론, 해싱을 위한 해시 키(Hash Key)로는 리소스 식별자가 사용된다. 각 해시 버킷에는 연결 리스트(Linked List)로 연결된 해시 체인을 가지며, 여기에 리소스 구조체가 연결된다.

Enqueue 방식으로 관리되는 특정 리소스(테이블, 트랜잭션)에 대해 Lock을 획득하려면, 먼저 리소스 테이블에서 해당 리소스 구조체를 찾는다.
리소스 구조체를 찾지 못하면, 새로운 리소스 구조체를 할당 받아 해시 체인 연결 리스트에 연결한다. 그런 후, 리소스 구조체의 소유자 목록에 자신을 등록하면 된다.
호환되지 않는 모드로 먼저 Lock을 획득한(즉, 소유자 목록에 등록된) 세션이 있다면 Lock 요청을 대기자 목록에 등록하고 대기해야 한다.(또는 작업을 포기하는 선택을 할 수도 있다.)

소유자가 Exclusive 모드일 때는 한 순간에 하나의 세션만 Lock을 획득할 수 있지만, Shared 모드일 때는 여러 세션이 동시에 Lock을 획득할 수 있다.
즉, 여러 세션이 동시에 소유자 목록에 등록될 수 있다.
소유자 목록에 Shared 또는 Exclusive 모드 Lock이 등록된 상태에서 Exclusive 모드로 Lock을 획득하려는 세션은 대기자 목록에서 대기해야 하며, 하나의 리소스 구조체 대기자 목록에 동시에 여러
세션이 등록된 상태로 대기할 수도 있다. Enqueue Lock의 작동 메커니즘은 아래와 같다.

- [1] A 세션이 Shared 모드로 Lock을 획득한다.
- [2] B 세션이 Shared 모드로 Lock을 획득하려고 한다. 먼저 Lock을 소유한 A 세션과 호환되므로 정상적으로 Lock을 획득한다. 이제 소유자 목록에는 두 개 세션이 달려 있다.
- [3] C 세션이 Exclusive 모드로 Lock을 획득하려고 한다. Shared 모드와 Exclusive 모드 간에 호환성이 없으므로 대기자 목록에 자신을 등록하고 대기한다.
- [4] 소유자 목록에 Shared 모드로 달려 있던 A, B 두 세션이 모두 Lock을 해제하면 C 세션이 Exclusive 모드로 소유자 목록에 등록된다.
- [5] A 세션이 Exclusive 모드로 다시 Lock을 획득하려고 하면, Exclusive 모드와 호환되지 않으므로 대기자 목록에 자신을 등록하고 대기한다.
- [6] B 세션이 다시 Shared 모드로 Lock을 획득하려고 할 때도 Exclusive 모드와 호환되지 않으므로 대기자 목록에 자신을 등록하고 대기한다.
- [7] Enqueue Lock은 순서가 보장되므로 C 세션이 Lock을 해제하면 A 세션이 가장 먼저 Exclusive 모드로 Lock을 획득한다.
  <br/>

## (2) TX Lock (= 트랜잭션 Lock)
트랜잭션을 시작하려면 먼저 Undo 세그먼트 헤더에 위치한 트랜잭션 테이블로부터 슬롯(Slot)을 하나 할당받아야 한다고 설명했다.
이 트랜잭션이 변경을 가한 블록에 대한 Consistent 버전을 얻으려는 다른 트랜잭션은, 트랜잭션 슬롯에 기록된 상태 정보를 확인하고, 필요하다면 CR 블록을 생성해서 읽는다.
그렇게 함으로써 오라클은, 레코드가 갱신 중이더라도 읽기 작업에 대해서는 블로킹 없이 작업을 진행할 수 있도록 구현하였다.

하지만 변경 중인 레코드(또는 기타 리소스)를 동시에 변경하려는 트랜잭션에 대해서는 액세스를 직렬화해야 하며, 그 목적으로 사용하는 Lock 메커니즘이 트랜잭션 Lock(이하 TX Lock)이다.
TX Lock은 트랜잭션이 첫 번째 변경을 시작할 때 얻고, 커밋 또는 롤백할 때 해제한다.

TX Lock도 Enqueue Lock으로 구현되었다. 앞에서 설명했듯이 TX Lock을 위한 Enqueue 리소스 구조체의 식별자는 다음과 같은 정보를 포함한다.

- TYPE : TX
- ID1 : Undo 세그먼트 + 트랜잭션 슬롯번호
- ID2 : 트랜잭션 슬롯 Sequence 번호

이 식별자를 갖는 리소스 구조체를 Enqueue 리소스 테이블 해시 체인에 연결하고, 소유자 목록에 트랜잭션을 등록함으로써 Lock을 획득한다.
이제 TX Lock을 획득했으므로 트랜잭션을 위한 일련의 작업들을 수행할 수 있다.

TX Lock 메커니즘을 다음 예시로 이해해보자.
- [1] TX1 트랜잭션은 Undo 세그먼트에서 트랜잭션 슬롯을 할당 받고, Enqueue 리소스를 통해 TX Lock을 설정한다. 이 상태에서 r1부터 r5까지 5개 레코드를 변경하고, 아직 커밋은 하지 않았다.
- [2] TX2 트랜잭션도 트랜잭션 테이블에서 하나의 슬롯을 할당 받고, Enqueue 리소스를 통해 TX Lock을 설정한 후 r6 레코드를 변경한다.
- [3] 이제 TX2가 r3 레코드를 액세스하려는 순간 호환되지 않는 모드로 Lock이 걸려 있음을 인지하고 TX1의 트랜잭션 슬롯 상태를 확인한다.
- [4] TX1이 아직 커밋되지 않은 Active 상태이므로 TX2는, TX1이 Lock을 설정한 Enqueue 리소스 구조체 대기자 목록에 자신을 등록하고, 대기 상태로 들어간다.
- [5] TX2는 대기하면서 3초마다 한번씩 TX1이 설정한 TX Lock의 상태를 확인한다. 교착상태(Deadlock) 발생 여부를 확인하기 위함이다.
- [6] TX1이 커밋 또는 롤백하면, TX1이 설정한 TX Lock의 대기자 목록에서 가장 우선순위가 높은 TX2 트랜잭션을 깨워 트랜잭션을 재개하도록 한다.
- [7] TX2는 r3 레코드를 변경한다.

v$lock을 통해 TX Lock을 조회할 수 있다.

```
SQL> select sid, id1, id2, lmode, request, block
  2       , to_char(trunc(id1/power(2,16))) USN
  3       , bitand(id1, to_number('ffff', 'xxxx')) + 0 SLOT
  4       , id2 SQN
  5  from   v$lock
  6  where  TYPE = 'TX' ;

  SID TY     ID1   ID2  LMODE   REQUEST   BLOCK USN   SLOT    SQN
----- -- ------- ----- ------ --------- ------- --- ------ ------
  145 TX  655401  1601      0         6       0  10     41   1601
  150 TX  655401  1601      6         0       1  10     41   1601
```

현재 150번 세션이 145번 세션의 진행을 블로킹하고 있다(block=1).
145번 세션은 150번 세션이 Exclusive 모드(lmode=6)로 획득한 TX Lock을 Exclusive 모드(request=6)로 요청한 채 대기하고 있다.
현재 경합이 발생한 TX Lock 식별자는 <TX-655401-1601>이고, id1과 id2 값을 이용해 TX Lock을 소유한 트랜잭션의 Undo 세그먼트와 트랜잭션 슬롯번호, 시퀀스 번호까지 식별해 낼 수 있다.

v$lock 뷰를 이용해 TX Lock 경합 상황을 모니터링 할 수 있지만, 발생 원인까지 알 수는 없다.
원인을 파악하려면 v$session_wait 또는 이벤트 트레이스(레벨 8)를 통해 대기 이벤트 발생 현황을 관찰해야 한다.

```
SQL> select sid, seq#, event, state, seconds_in_wait, p1, p2, p3
  2  from   v$session_wait
  3  where  event like 'enq: TX%';

                                                SECONDS
SID SEQ# EVENT                           STATE _IN_WAIT         P1     P2    P3
--- ---- ----------------------------- ------- -------- ---------- ------ -----
158  137 enq: TX - row lock contention WAITING      108 1415053318 589824  7754
```

관찰된 대기 이벤트명에 따라 TX Lock을 아래와 같이 구분할 수 있다.
- 오라클 9i까지는 TX Lock을 포함해 Enqueue Lock 대기가 발생하면 무조건 enqueue 대기 이벤트 하나로 표시해 주었기 때문에 원인을 분석하는 데에 어려움이 많았었다.

|대기 이벤트명|Lock 모드|원인|
|:---|:---|:---|
|enq: TX - row lock contention|Exclusive(6)<br/>Shared(4)<br/>Shared(4)|DML 로우 Lock<br/>무결성 제약 위배 가능성<br/>비트맵 인덱스 엔트리 갱신|
|enq: TX - allocate ITL entry|Shared(4)|ITL 부족|
|enq: TX - index contention|Shared(4)|인덱스 분할|
|enq: TX - contention|Shared(4)|읽기 전용 테이블스페이스,<br/>PREPARED TxN(2PC),<br/>Free Lists 등등|

특히, 이벤트명이 enq: TX - row lock contention 일 때는 Lock 모드에 따라 그 발생원인을 판단해야 한다. Lock 모드는 이벤트 발생 시 함께 기록되는 p1 파라미터를 통해 확인할 수 있다.

```
# Lock 타입 - TM, TX, TS, SQ, HW, HV 등
  chr(bitand(:p1, -16777216)/16777215) || chr(bitand(:p1, 16711680)/65535)

# Lock 모드
  decode(to_char(bitand(:p1, 65535)), 0, 'None'
                                    , 1, 'Null'
                                    , 2, 'RS'    -- Row-Shared
                                    , 3, 'RX'    -- Row-Exclusive
                                    , 4, 'S'     -- Shared
                                    , 5, 'SRX'   -- Shared-Row-Exclusive
                                    , 6, 'X'     -- Exclusive
      )
```

참고로, p2, p3 파라미터를 통해 Undo 세그먼트, 트랜잭션 슬롯번호, 그리고 Wrap 시퀀스 번호를 식별해 낼 수 있다.

```
# Undo 세그먼트 번호
  trunc(:p2/power(2,16))

# 트랜잭션 테이블 슬롯번호
  bitand(:p2, to_number('ffff', 'xxxx')) + 0

# 트랜잭션 슬롯 Wrap 시퀀스
  :p3
```

지금부터 아래 6가지 TX Lock 발생원인에 대해 자세히 살펴보자.

- DML 로우 Lock
- 무결성 제약 위배 가능성
- 비트맵 인덱스 엔트리 갱신
- ITL 슬롯 부족
- 인덱스 분할

이 중 가장 중요한 DML 로우 Lock은 맨 뒤에서 테이블 Lock과 함께 설명한다.
<br/>
<br/>
## (3) TX Lock ▶︎ 무결성 제약 위배 가능성 또는 비트맵 인덱스 엔트리 갱신
로우 Lock 경합은 일반적으로 update나 delete 시에만 발생한다. insert는 새로운 레코드를 삽입하는 것이므로 로우 Lock 경합이 발생하지 않는다.
하지만 테이블에 Unique 인덱스가 정의되어 있을 때는 insert에 의한 로우 Lock 경합이 생길 수 있다.
두 개 이상 트랜잭션이 같은 값을 입력하려 할 때, 선행 트랜잭션이 아직 진행 중이라면 값의 중복 여부가 확정되지 않았으므로 후행 트랜잭션은 진행을 멈추고 대기해야만 하는 것이다.

dept 테이블 deptno 컬럼에 PK 인덱스가 생성돼 있는 상황에서 두 트랜잭션이 다음과 같이 진행하면, enq: TX - row lock contention 대기 이벤트가 Shared 모드로 발생한다.

- [1] 트랜잭션 TX1이 dept 테이블에 deptno = 40인 레코드를 입력한다.
- [2] 트랜잭션 TX2도 dept 테이블에 deptno = 40인 레코드를 입력하면, TX1이 커밋 또는 롤백할 때까지 Shared 모드로 enq: TX - row lock contention 대기 이벤트가 발생한다.
- [3] TX1이 커밋하면 TX2는 ORA-00001 에러를 만나게 된다.

  ```
  "ORA-00001: 무결성 제약조건(PK_DEPT)에 위배됩니다"
  ```

- [4] TX1이 롤백하면 TX2는 정상적으로 입력이 완료된다.

이번에는 dept와 emp 테이블이 1:M 관계고, deptno 컬럼으로 dept.deptno를 참조하도록 emp 테이블에 FK 제약이 설정돼 있다고 가정하자.
이런 상황에서 두 트랜잭션이 아래와 같이 진행하면, 마찬가지로 enq: TX - row lock contention 대기 이벤트가 Shared 모드로 발생한다.

- [1] 트랜잭션 TX1이 dept 테이블에 deptno = 40인 레코드를 지운다.
- [2] 트랜잭션 TX2가 emp 테이블에 deptno = 40인 레코드를 입력하면, TX1이 커밋 또는 롤백할 때까지 Shared 모드로 enq: TX - row lock contention 대기 이벤트가 발생한다.
- [3] TX1이 커밋하면 TX2는 ORA-02291 에러를 만나게 된다.

  ```
  "ORA-02291: 무결성 제약조건(FK_EMP_DEPT)이 위배되었습니다- 부모 키가 없습니다"
  ```

- [4] TX1이 롤백하면 TX2는 정상적으로 입력이 완료된다.


비트맵 인덱스 엔트리에 대한 갱신을 수행할 때도 Shared 모드로 enq: TX - row lock contention 이벤트가 발생할 수 있다. 비트맵 인덱스는 그 구조상 하나의 엔트리가 여러 개 레코드와 매핑된다.
하나의 엔트리에 Lock을 설정하면 매핑되는 레코드 전체에 Lock이 설정되므로, 비트맵 인덱스 엔트리를 두 개 이상 트랜잭션이 동시에 갱신할 때 이 이벤트가 자주 발생한다.

예를 들어, TX1 트랜잭션이 1번 레코드를 갱신하는 동안 TX2 트랜잭션이 2번 레코드를 갱신하려고 할 수 있는데, 이 때 Shared 모드로 enq: TX - row lock contention 대기 이벤트가 발생한다.
<br/>
<br/>
## (4) TX Lock ▶︎ ITL 슬롯 부족
블록에 레코드를 추가/갱신/삭제하려면, ITL 슬롯을 먼저 할당 받고 그 곳에 트랜잭션 ID를 먼저 기록해야 한다.
비어 있는 ITL 슬롯이 없다면, ITL 슬롯을 사용 중인 트랜잭션 중 하나가 커밋 또는 롤백할 때까지 기다려야 하며, 이때 Shared 모드 enq: TX - allocate ITL entry 대기 이벤트가 발생한다.
결국, 한 블록을 동시에 갱신할 수 있는 트랜잭션 개수는 ITL 슬롯에 의해 결정된다. 참고로, ITL 슬롯당 24 바이트 공간을 차지한다.

블록에 기본적으로 할당할 ITL 슬롯 개수는 INITRANS 파라미터로 설정한다.

```
create table t ( ... ) INITRANS 5 MAXTRANS 255 PCTFREE 30;
```

PCTFREE는 원래 컬럼 update를 위해 예약된 공간이다.
하지만 INITRANS에 의해 미리 할당된 ITL 슬롯이 모두 사용 중일 때, 새로운 트랜잭션이 ITL 슬롯을 요청하면 PCTFREE 설정에 의해 비워둔 공간을 활용하게 된다.
이 공간까지 활용해 최대한 생성할 수 있는 ITL 슬롯 개수는 MAXTRANS에 의해 결정된다. ITL 슬롯 부족에 의한 대기 현상이 발생했다면, 아래 둘 중 하나에 해당한다.

- 동시에 블록을 갱신하려는 트랜잭션 개수가 MAXTRANS 값을 초과
- PCTFREE를 0으로 지정했거나 PCTFREE 예약 공간을 모두 사용한 상태에서, 새로운 트랜잭션을 위한 ITL 슬롯이 부족

테이블에 insert 할 때는, ITL 슬롯이 부족하더라도 굳이 그 블록에 insert 하려고 대기할 필요가 없다. 새 블록을 할당해 그 곳에 insert하면 되기 때문이며, 오라클 9i부터 그렇게 동작하기 시작했다.
따라서 9i부터는 insert 시 테이블 블록에 대한 ITL 경합이 발생하지 않는다. 하지만 인덱스에 값을 삽입할 때는 정렬 상태를 유지해야 하므로 여전히 ITL 경합이 발생한다.
update, delete 일 때는 테이블, 인덱스를 불문하고 ITL 경합이 나타날 수 있다.

INITRANS, MAXTRANS 설정과 관련해, 오라클 버전이 올라가면서 조금씩 변화가 있었다. 9i부터는, 테이블에 INITRANS를 3보다 작게 설정하더라도 오라클이 기본적으로 3개의 ITL 슬롯을 할당한다.
10g에서는 MAXTRANS를 위해 사용자가 지정한 값은 무시되며 항상 255개로 고정된다. 이 값이 255이므로 ITL 경합을 더는 염려하지 않아도 된다고 생각하면 오산이다.
PCTFREE에 의해 예약된 공간이 update에 의해 모두 사용되고 나면 여전히 ITL 경합이 발생할 수 있기 때문이다.

ITL 경합에 의한 대기 현상이 자주 발생하는 세그먼트(테이블, 인덱스, 파티션)에 대해서는 INITRANS를 늘려주어야 하며, 그런 세그먼트 목록은 v$segstat(10g 이상)을 통해 확인할 수 있다.

```
select ts#, obj#, dataobj#, sum(value) itl_waits
from   v$segstat
where  statistic_name = 'ITL waits'
group by ts#, obj#, dataobj#
having sum(value) > 0
order by sum(value) desc
```

INITRANS 값을 변경하더라도 기존에 할당된 블록의 ITL 슬롯 개수에는 변함이 없고, 새로 할당되는 블록에만 적용된다.
따라서 기존 블록에서 ITL 경합이 빈번하게 발생한다면, 테이블 또는 인덱스 전체를 재생성해 줘야만 한다.

```
alter table t move INITRANS 5;  → 인덱스가 모두 unusable 되므로 주의 !!
alter index t_idx rebuild INITRANS 5;
```
<br/>

## (5) TX Lock ▶︎ 인덱스 분할
테이블은 레코드 간 정렬 상태를 유지하지 않기 때문에 입력할 공간이 부족할 때 새로운 블록을 할당 받아 입력하면 된다. 하지만 인덱스는 정렬된 상태를 유지해야 하므로 아무 블록에나 값을 입력할 수 없다.
따라서 값을 입력할 위치에 빈 공간이 없으면 인덱스 분할(Split)을 실시해 새 값을 입력할 공간을 확보하게 되며, 이 과정에서 Lock 경합이 발생할 수 있다.

#### (p.138, 139 [그림 2-6], [그림 2-7], [그림 2-8] 참고)
[그림 2-6]에서 현재 5번과 9번 리프 블록이 꽉 찬 모습을 볼 수 있다. 이때 5번 블록에 새로운 값을 입력하려는 트랜잭션은, 먼저 인덱스 분할을 실시해야 한다.

[그림 2-7]은 인덱스 분할이 완료된 후 모습이다. 5번과 6번 블록 사이에 10번 블록이 삽입되었고, 5번 블록에 있던 레코드 절반이 10번 블록으로 이동하였다.
이제 값을 입력하고, 계속 트랜잭션을 진행할 수 있다.

이번에는 9번 블록에 새로운 값을 추가하려는 트랜잭션이 생겼다. 9번 블록도 꽉 찬 상태이므로 먼저 입력할 공간을 확보해야 한다.
맨 우측에 값을 추가하려는 것이므로 레코드가 이동할 필요는 없으며, 새 블록을 추가해 주기만 하면 된다.

[그림 2-8]은 인덱스 분할이 완료된 후 모습이며, 9번 블록 뒤쪽에 11번 블록이 추가된 것을 볼 수 있다.

문제는, 인덱스 분할이 진행되는 동안 그 블록에 새로운 값을 입력하려는 또 다른 트랜잭션이 생길 수 있다는 점이다.
그러면 두 번째 트랜잭션은 선행 트랜잭션이 인덱스 분할을 완료할 때까지 대기해야 하며, Shared 모드에서 enq: TX - index contention 이벤트를 만나게 된다.

여기서 한 가지 의문이 생긴다.
TX Lock은 선행 트랜잭션이 커밋 또는 롤백할 때 비로소 해제되는데, 만약 인덱스 분할을 진행한 트랜잭션이 커밋하지 않은 채 계속 다른 갱신 작업을 진행한다면 TX Lock을 대기하던 트랜잭션은 어떻게 될까?
계속 대기해야만 하는 것일까?

오라클은 이 문제를 해결하려고 autonomous 트랜잭션을 사용하며, 이에 대한 개념은 동시성 구현 사례에서 이미 살펴보았다.

- [1] TX1 트랜잭션이 인덱스에 로우를 삽입하려는 순간 빈 공간을 찾지 못했다. 인덱스 분할이 필요하다.
- [2] TX1 트랜잭션은 autonomous 트랜잭션 TX2를 생성해 인덱스 분할을 진행토록 한다.
- [3] 인덱스 분할이 진행 중인 블록에 TX3 트랜잭션이 로우를 삽입하려고 한다. 이 트랜잭션은 enq: TX - index contention 이벤트를 만나게 되고, TX2 트랜잭션이 커밋할 때까지 대기한다.
- [4] 인덱스 분할이 완료되면 TX2 트랜잭션은 커밋한다. autonomous 트랜잭션이므로 TX1은 커밋되지 않은 상태로 계속 트랜잭션을 진행할 수 있다.
- [5] TX3 트랜잭션도 작업을 재개한다.

인덱스 분할을 최소화하는 방안으로, PCTFREE를 증가시키면 된다고 흔히 알려졌다.
그런데 이 방법은 대개 효과가 없거나(데이터가 없는 상태에서 인덱스를 만들면서 PCTFREE를 크게 설정한 경우) 일시적이라고 할 수 있다.
PCTFREE 설정은 인덱스를 처음 생성하거나 재생성하는 시점에만 적용되기 때문이다.
인덱스를 생성하는 시점에, 나중에 발생할 insert를 위해 공간을 남겨두는 것이므로 인덱스 분할을 최소화할 목적으로 제공되는 기능임은 틀림없다.
하지만 공간을 남겨두더라도 언젠가 다시 채워지므로, 인덱스를 주기적으로 재생성하지 않는 한 근본적인 해결책이 되지는 못한다.

테이블에서의 PCTFREE 공간이 나중에 발생할 update를 위해 남겨두는 공간인데 반해, 인덱스에서의 PCTFREE는 insert를 위해 남겨두는 공간임을 기억할 필요가 있다.
인덱스에는 update 개념이 없기 때문에(인덱스 레코드는 정렬된 상태를 유지해야 하기 때문에 update 시 delete & insert 방식으로 작동한다.) 테이블처럼 update를 위해 공간을 남겨둔다면 영영
사용되지 않는 죽은 공간이 될 것이다.
참고로, 우측 맨 끝으로만 값이 입력되는 Right Growing(예를 들어, 순차적으로 증가하는 일련번호 컬럼에 인덱스를 생성하는 경우) 인덱스라면, PCTFREE를 0으로 설정하는 것이 인덱스 크기를 줄이는
데에 도움이 된다.
<br/>
<br/>
## (6) TX Lock ▶︎ 기타 트랜잭션 Lock
Shared 모드 enq: TX - contention 대기 이벤트에 대해 오라클 매뉴얼을 찾아보면, 분산 트랜잭션에서 2-Phase 커밋을 위한 PREPARED TX Lock을 대기할 때 발생한다고 설명돼 있을 뿐 더 자세한
설명을 얻을 수 없다. 추측건대, 앞에서 열거한 중요한 TX Lock 이외의 트랜잭션 대기 상황을 모두 여기에 포함한 것 같다.

매뉴얼에는 없지만 읽기 전용 테이블스페이스로 전환할 때도, 이 TX Lock 경합이 발견된다.
예를 들어, USERS 테이블스페이스에 DML을 수행하는 트랜잭션이 아직 남아있는 상태에서 아래 명령을 수행하면 Shared Mode로 TX Lock을 대기한다.

```
alter tablespace USERS read only;
```
<br/>

## (7) TM Lock ▶︎ DML 로우 Lock
DML Lock은, 다중 사용자에 의해 동시에 액세스되는 사용자 데이터의 무결성을 보호해 준다. DML 수행 중에 호환되지 않는 다른 DML 또는 DDL 오퍼레이션의 수행을 방지시켜 주는 것이다.

그 중 로우 Lock은, 두 개의 동시 트랜잭션이 같은 로우를 변경하는 것을 방지한다. 하나의 로우를 변경하려면 로우 Lock을 먼저 획득해야 한다.
오라클은 로우 Lock을, **[1] 로우 단위(row-level) Lock**과 **[2] TX Lock**을 조합해서 구현하였다.
즉, 로우를 갱신하려면 Undo 세그먼트에서 트랜잭션 슬롯을 먼저 할당받고, Enqueue 리소스를 통해 TX Lock을 획득한다.
그런 후 insert, update, delete, merge 문장을 통해 갱신하는 각 로우마다 Exclusive 모드로 로우 단위 Lock을 획득한다. TX Lock은 트랜잭션을 시작할 때 한 번만 획득한다.

[1] 로우 단위 Lock에 대해서는 아주 자세히 설명하였다.
간단히 요약하면, TX1 트랜잭션이 로우 정보를 갱신할 때는, 블록 헤더 ITL 슬롯에 트랜잭션 ID를 기록하고, 로우 헤더에 이를 가리키는 Lock Byte를 설정한다.
이 레코드를 액세스하려는 다른 트랜잭션은, 로우 헤더에 설정한 Lock Byte를 통해 ITL 슬롯을 찾고, ITL 슬롯이 가리키는 Undo 세그먼트 헤더의 트랜잭션 슬롯에서 트랜잭션 상태 정보를 확인함으로써
해당 레코드에 대한 액세스 가능 여부를 결정한다.

TX1 트랜잭션이 진행 중일 때, 이 레코드를 읽으려는 TX2 트랜잭션은 TX1 트랜잭션의 상태를 확인하고 CR 블록을 생성해서 읽기 작업을 완료한다.
오라클은 이처럼 로우 단위 Lock과 다중 버전 읽기 일관성 메커니즘을 이용함으로써 (select for update 문이 아닌 한) 읽기 작업에 대해서는 절대 Lock에 의한 대기 현상이 발생하지 않도록 구현하였다.
(분산트랜잭션에서는 예외 존재)

[2] TX Lock에 대해 계속 이어서 설명하면, TX1이 갱신 중인 레코드를 같이 갱신하려는 TX2 트랜잭션은 TX1 트랜잭션이 완료될 때까지 대기해야 한다.
이를 위해 TX Lock이 필요하며, 이에 대해서는 앞에서 자세히 설명하였다.

로우 Lock을 로우 단위 Lock과 TX Lock을 조합해서 구현했다는 의미를 여기서 정확히 이해하기 바란다.

- 로우 단위 Lock : 블록 헤더 ITL과 로우 헤더 Lock Byte 설정을 의미함. 이를 통해 로우를 갱신 중인 트랜잭션 상태를 확인하고 액세스 가능 여부를 결정함
- TX Lock : Enqueue 리소스를 통해 TX Lock을 설정하는 것을 의미함. Lock이 설정된 레코드를 갱신하고자 할 때 Enqueue 리소스에서 대기함

DML 로우 Lock에 의한 TX Lock 때문에 블로킹된 세션을 관찰해 보면, Exclusive 모드의 enq: TX - row lock contention 대기 이벤트가 지속적으로 나타난다.
<br/>
<br/>
## (8) TM Lock ▶︎ DML 테이블 Lock
오라클은 로우 Lock 획득 시, 해당 테이블에 대한 테이블 Lock도 동시에 획득한다. 그럼으로써 현재 트랜잭션이 갱신 중인 테이블에 대한 호환되지 않는 DDL 오퍼레이션을 방지한다.
테이블 구조를 변경하지 못하도록 막는 것이다. 테이블 Lock은 주로 DDL과 관련 있지만 DML문 간에도 테이블 Lock을 이용해 동시성을 제어할 때가 있다.

테이블 Lock을 걸려면 아래처럼 명시적으로 Lock Table 명령어를 이용할 수도 있다.

- lock table emp in row share mode
- lock table emp in row exclusive mode
- lock table emp in share mode
- lock table emp in share row exclusive mode
- lock table emp in exclusive mode

하지만 지금 설명하려는 DML 테이블 Lock은, DML 문장 수행 시 자동으로 테이블 Lock까지 함께 획득하는 메커니즘을 말한다.

오라클은 변경 작업에만 로우 Lock을 사용하므로, 로우 Lock은 항상 Exclusive 모드다.
하지만 테이블 Lock에는 여러 가지 Lock 모드가 사용되며, Lock 모드간 호환성(Compatibility)을 정리하면 아래 표와 같다. ('○' 표시는 두 모드 간에 호환성이 있음을 의미한다.)

||Null|RS|RX|S|SRX|X|
|:---|:---|:---|:---|:---|:---|:---|
|Null|○|○|○|○|○|○|
|RS|○|○|○|○|○||
|RX|○|○|○||||
|S|○|○||○|||
|SRX|○|○|||||
|X|○||||||

- RS : row share (또는 SS : sub share)
- RX : row exclusive (또는 SX : sub exclusive)
- S : share
- SRX : share row exclusive (또는 SSX : share/sub exclusive)
- X : exclusive

선행 트랜잭션과 호환되지 않는 모드로 테이블 Lock을 설정하려는 후행 트랜잭션은 대기하거나 작업을 포기해야 한다.

insert, update, delete, merge 문을 위해 로우 Lock을 설정하려면 해당 테이블에 RX(=SX) 모드 테이블 Lock을 먼저 획득해야 한다.
select for update 문을 위해 로우 Lock을 설정하려면 RS(=SS) 모드 테이블 Lock을 먼저 획득해야 한다.
RS, RX 간에는 어떤 조합으로도 호환이 되므로 select for update나 DML문 수행 시 이들 간에 테이블 Lock에 의한 경합은 절대 발생하지 않는다.
다만, 같은 로우를 갱신하려 할 때 로우 Lock에 의한 경합은 발생한다.

오라클은 테이블 Lock도 Enqueue로 구현하였으며, 'TM Enqueue'라고 부른다. TM Enqueue를 이용하기 때문에 테이블 Lock을 'TM Lock'이라고 부르기도 한다.
앞에서 설명했듯이 TM Enqueue 리소스 구조체의 식별자는 아래와 같은 정보를 포함한다.

- TYPE : TM
- ID1 : 오브젝트 ID
- ID2 : 0

선행 트랜잭션이 TM Lock을 해제하기를 기다리는 트랜잭션에서 발생하는 대기 이벤트를 모니터링해 보면, enq: TM - contention 이벤트가 지속적으로 나타난다.

'테이블 Lock'이라고 하면, 테이블 전체에 Lock이 걸린다고 생각하기 쉽다. 그래서 다른 트랜잭션이 더는 레코드를 추가하거나 갱신하지 못하도록 막는다고 생각하는 경우가 많다.
하지만 앞서 설명했듯이 DML 수행 시 항상 테이블 Lock이 함께 설정되므로 그렇게 이해하는 것은 맞지 않는 개념이다.

오라클에서 말하는 테이블 Lock은, Lock을 획득한 선행 트랜잭션이 해당 테이블에서 현재 어떤 작업을 수행 중인지를 알리는 일종의 푯말(Flag)이다.
(참고로, SQL Server에서는 이런 종류의 테이블 Lock을 'Intent Lock'이라고 부른다.)
그리고 위에서 본 것처럼 테이블 Lock에는 여러 가지 모드가 있고, 각 모드에 따라 후행 트랜잭션이 수행할 수 있는 작업의 범위가 결정된다.
그 푯말에 기록된 Lock 모드와 후행 트랜잭션이 현재 하려는 작업 내용에 따라 진행 여부가 결정된다.
진행할 수 없다면 기다릴지, 아니면 작업을 포기할지 진로를 결정(내부적으로 하드코딩 돼 있거나 사용자가 지정한 옵션에 따라 결정)해야 한다.
기다려야 한다면, TM Enqueue 리소스 대기자 목록에 Lock 요청을 등록하고 대기한다.

> #### [대상 리소스가 사용 중일 때, 진로 선택]
> Lock을 얻고자 하는 리소스가 사용 중일 때, 프로세스는 아래 3가지 방법 중 하나를 택한다. 보통은 내부적으로 진로가 결정돼 있지만, 사용자가 선택할 수 있는 경우도 있다.
> 사용자가 이 3가지 옵션을 모두 선택할 수 있는 문장이 select for update 문이다.
>
> - [1] Lock이 해제될 때까지 기다린다. (예 : select * from t for update)
> - [2] 일정 시간만 기다리다 포기한다. (예 : select * from t for update wait 3)
> - [3] 기다리지 않고 작업을 포기한다. (예 : select * from t for update nowait)
>
> DML문을 수행할 때 묵시적으로 테이블 Lock을 얻게 되는데 이때는 1번, 기다리는 방법을 선택한다.
> Lock Table 명령을 이용해 명시적으로 테이블 Lock을 얻을 때도 기본적으로 기다리는 방법을 택하지만, NOWAIT 옵션을 이용해 곧바로 작업을 포기하도록 사용자가 지정해 줄 수 있다.
>
> ```
> lock table emp in exclusive mode NOWAIT;
> ```
>
> DDL문을 수행할 떄도 내부적으로 테이블 Lock을 얻는데, 이때는 NOWAIT 옵션이 자동으로 지정된다.
>
> 오라클은 2번 옵션(→ wait)에 의해 작업을 포기할 때, 아래 메시지를 던진다.
>
> ```
> ORA-30006: resource busy: acquire with WAIT timeout expired
> ```
>
> 3번 옵션(→ nowait)에 의해 작업을 포기할 때는 아래 메시지를 던진다.
>
> ```
> ORA-00054: resource busy and acquire with NOWAIT specified
> ```

예를 들어, DDL문을 이용해 테이블 구조를 변경하려는 세션은 해당 테이블에 TM Lock이 설정돼 있는지를 먼저 확인한다.
TM Lock을 Row Exclusive(=SX) 모드로 설정한 트랜잭션이 하나라도 있으면, 현재 테이블을 갱신 중인 트랜잭션이 있다는 신호다. 따라서 ORA-00054 메시지를 던지고 작업을 멈춘다.

DDL문이 먼저 수행 중일 때는, DML문을 수행하려는 세션이 TX Lock을 얻으려고 대기할 것이다. 이때, enq: TM - contention 이벤트가 발생한다.

테이블 Lock은 DDL과의 동시 진행을 막으려고 사용될 뿐만 아니라 DML 간 동시성을 제어하려고 사용되기도 한다. 병렬 DML 또는 Direct Path Insert 방식으로 작업을 수행할 때가 그렇다.
직접 테스트하기에 앞서 아래 스크립트 내용을 먼저 확인하기 바란다. Lock을 모니터링할 때 주로 사용하는 스크립트다.

```
select l.session_id SID
    , (case when lock_type = 'Transaction' then 'TX'
            when lock_type = 'DML' then 'TM' end) TYPE
    , mode_held
    , mode_requested mode_reqd
    , (case when lock_type = 'Transaction' then
                 to_char(trunc(lock_id1/power(2,16)))
            when lock_type = 'DML' then
                (select object_name from dba_objects
                 where object_id = l.lock_id1)
       end) "USN/Table"
    , (case when lock_type = 'Transaction' then
                 bitand(lock_id1, to_number('ffff', 'xxxx')) + 0
       end) "SLOT"
    , (case when lock_type = 'Transaction' then
                 to_number(lock_id2) end) "SQN"
    , (case when blocking_others = 'Blocking' then ' <<<<<' end) Blocking
from   dba_lock l
where  lock_type in ('Transaction', 'DML')
order by session_id, lock_type, lock_id1, lock_id2
```

이제 테스트를 시작하기 전에 모든 트랜잭션을 완료(커밋 또는 롤백)하고, 1번 세션(SID=144)에서 아래처럼 T 테이블에 Append 모드로 Insert를 수행해 보자.

```
insert /*+ append */ into t select ... from ...
```

이 상황에서 Lock을 모니터링 해 보면 아래와 같은 내용을 볼 수 있다.

```
  SID TY MODE_HELD     MODE_REQD  USN/Table    SLOT    SQN BLOCKING
----- -- ------------- ---------- ---------- ------ ------ --------
  144 TM Exclusive     None       T
  144 TX Exclusive     None       7              28   1661
```

앞에서 설명했다시피 DML문을 수행하니까 TX Lock과 TM Lock을 동시에 획득했다.
일반적인 DML문에서는 테이블 Lock(=TM Lock)을 Row Exclusive(=SX) 모드로 설정하지만 여기서는 Append 모드 Insert를 수행했기 때문에 Exclusive 모드인 것을 볼 수 있다.

이제 2번 세션(SID=149)에서 T 테이블에 있는 레코드 하나를 갱신하는 update문을 수행해 보자.

```
  SID TY MODE_HELD     MODE_REQD  USN/Table    SLOT    SQN BLOCKING
----- -- ------------- ---------- ---------- ------ ------ --------
  144 TM Exclusive     None       T                         <<<<<
  144 TX Exclusive     None       7              28   1661
  149 TM None          Row-X(SX)  T
```

일반 DML 문을 사용했으므로 Row Exclusive(=SX) 모드로 테이블 Lock을 요청했음을 알 수 있다.
Row Exclusive 모드 Lock은 Exclusive 모드와 호환성이 없으므로 2번 세션(SID=149)은 블로킹된다. 로우 Lock이 아니라 테이블 Lock 때문에 블로킹된 점을 주목하기 바란다.
로우 Lock 호환성을 확인하기 전에 테이블 Lock 호환성을 먼저 확인한다는 사실을 알 수 있다.

이 상황에서, v$session_wait 뷰를 통해 149번 세션의 이벤트 발생 상황을 조회해 보면, 아래와 같이 enq: TM - contention 대기 이벤트가 발생 중인 것을 관찰할 수 있다.

```
SQL> select event, wait_time, seconds_in_wait, state
  2  from   v$session_wait
  3  where  sid = 149 ;

EVENT                   WAIT_TIME   SECONDS_IN_WAIT STATE
--------------------- ----------- ----------------- ---------
enq: TM - contention            0                48 WAITING
```

이제, Exclusive 모드 테이블 Lock을 먼저 획득한 144번 세션을 커밋하고, 다시 Lock을 모니터링 해 보자.

```
  SID TY MODE_HELD     MODE_REQD  USN/Table    SLOT    SQN BLOCKING
----- -- ------------- ---------- ---------- ------ ------ --------
  149 TM Row-X (SX)    None       T                         
  149 TX Exclusive     None       10             32   1607
```

144번 세션은 Lock을 모두 해제했으므로 쿼리 결과에서 사라졌다. 그 대신, 149번 세션이 TX Lock과 TM Lock을 동시에 획득한 것을 볼 수 있다.

이번에는, 149번 세션이 현재 Lock을 걸고 있는 레코드를 다른 세션에서 변경하도록 해 보자.

```
  SID TY MODE_HELD     MODE_REQD  USN/Table    SLOT    SQN BLOCKING
----- -- ------------- ---------- ---------- ------ ------ --------
  149 TM Row-X (SX)    None       T                         
  149 TX Exclusive     None       10             44   1607  <<<<<
  144 TM Row-X (SX)    None       T                         
  144 TX None          Exclusive  10             44   1607
```

이번에는 테이블 Lock에 의한 블로킹이 아니라 로우 Lock 때문에 블로킹이 발생했음을 알 수 있다. Exclusive 모드 로우 Lock 간에는 호환성이 없기 때문이다.
Row-Exclusive 모드 테이블 Lock 간에는 호환성이 있으므로 Lock 경합이 발생하지 않았다.

마지막으로 한 가지 기억할 것은, TX Lock은 트랜잭션마다 오직 한 개씩 획득하는 반면, TM Lock은 트랜잭션에 의해 변경이 가해진 오브젝트 수만큼 획득한다.
톰 카이트(Thomas Kyte)가 오래 전에 저술한 'Expert One-On-One Oracle'에 나오는 아래 예시를 통해 위 사실을 쉽게 확인해 볼 수 있다.

```
SQL> set feedback off
SQL> create table t1 ( x int );
SQL> create table t2 ( x int );
SQL> insert into t1 values(1);
SQL> insert into t2 values(1);
SQL> select username, v$lock.sid, id1, id2
  2       , lmode, request, block, v$lock.type
  3  from   v$lock, v$session
  4  where  v$lock.sid = v$session.sid
  5  and    v$session.username = USER ;

USERNAME     SID        ID1     ID2    LMODE    REQUEST    BLOCK TY
---------- ----- ---------- ------- -------- ---------- -------- --
ORAKING        8      32230       0        3          0        0 TM
ORAKING        8      32231       0        3          0        0 TM
ORAKING        8     524291     558        6          0        0 TX

SQL> select object_name, object_id from user_objects;

OBJECT_NAME                    OBJECT_ID
------------------------------ ---------
T1                                 32230
T2                                 32231
```
<br/>

## (9) Lock을 푸는 열쇠, 커밋
가끔 블로킹과 교착상태를 구분 못하는 경우가 있다. 블로킹(Blocking)은, 우리가 흔히 알고 있는 Lock 경합이 발생해 특정 세션이 작업을 진행하지 못하고 멈춰 선 경우를 말한다.
이것을 해소하는 방법은 커밋(또는 롤백)뿐이다.

교착상태(Deadlock)는, 두 세션이 각각 Lock을 설정한 리소스를, 서로 액세스하려고 마주보고 진행하는 상황을 말하며, 둘 중 하나가 뒤로 물러나지 않으면 영영 풀릴 수 없다.
흔히 좁은 골목길에 두 대의 차량이 마주 선 것에 비유하곤 한다.

오라클에서 교착상태가 발생하면, 이를 먼저 인지(Lock 경합 때문에 대기 상태에 빠질 때, 3초의 타임아웃(timeout)을 설정하는 이유가 여기에 있다.)한 세션이 문장 수준 롤백을 진행한 후에 아래 에러
메시지를 던진다. 교착상태를 발생시킨 문장 하나만 롤백하는 것이다.

```
ORA-00060: deadlock detected while waiting for resource
```

이제 교착상태는 해소됐지만 블로킹 상태에 놓이게 된다. 따라서 이 메시지를 받은 세션은 커밋 또는 롤백을 결정해야만 한다.
만약 프로그램 내에서 이 에러에 대한 예외 처리(커밋 또는 롤백)를 하지 않는다면 대기 상태를 지속하게 되므로 주의가 필요하다.

오라클은 데이터를 읽을 때 Lock을 사용하지 않으므로 다른 DBMS에 비해 상대적으로 Lock 경합이 적게 발생한다.
읽는 세션의 진행을 막는 부담감이 없으므로 필요한 만큼 트랜잭션을 충분히 길게 가져갈 수 있다.

그렇더라도 '불필요하게' 트랜잭션을 길게 정의하지 않도록 주의해야 한다. 트랜잭션이 너무 길면, 트랜잭션을 롤백해야 할 때 너무 많은 시간이 걸려 고생할 수 있다.
Undo 세그먼트가 고갈되거나 Undo 세그먼트 경합을 유발할 수도 있다.

같은 데이터를 갱신하는 트랜잭션이 동시에 수행되지 않도록 설계해야 하고, DML Lock 때문에 동시성이 저하되지 않도록 적절한 시점에 커밋해야 한다.

반대로, 불필요하게 커밋을 너무 자주 수행하면 Snapshot too old(ORA-01555) 에러를 유발할 가능성이 높아지고, 그에 앞서 LGWR가 로그 버퍼를 비우는 동안 발생하는 log file sync 대기
이벤트 때문에 성능 저하 현상을 겪을 수 있다.

잦은 커밋 때문에 성능이 매우 느리다면, 오라클 10gR2부터 제공되는 비동기식 커밋 기능을 활용하는 방안을 검토할 수 있다.

- WAIT(Default) : LGWR가 로그버퍼를 파일에 기록했다는 완료 메시지를 받을 때까지 대기하며, 그 동안 log file sync 대기 이벤트가 발생한다(동기식 커밋).
- NOWAIT : LGWR의 완료 메시지를 기다리지 않고 바로 다음 트랜잭션을 진행하므로, log file sync 대기 이벤트가 발생하지 않는다(비동기식 커밋).
- IMMEDIATE(Default) : 커밋 명령을 받을 때마다 LGWR가 로그 버퍼를 파일에 기록한다.
- BATCH : 세션 내부에 트랜잭션 데이터를 일정량 버퍼링했다가 일괄 처리한다.

이들 옵션을 조합해 아래 4가지 커밋 명령을 사용할 수 있게 되었다.

```
SQL> COMMIT WRITE IMMEDIATE WAIT;    --------- [1]
SQL> COMMIT WRITE IMMEDIATE NOWAIT;  --------- [2]
SQL> COMMIT WRITE BATCH WAIT;        --------- [3]
SQL> COMMIT WRITE BATCH NOWAIT;      --------- [4]
```

아래 스크립트를 가지고 [1]에서 [4]까지 테스트해 본 결과, 각각 68초, 9초, 66초, 6초의 수행 속도를 보였다.

```
create table t ( a number );

begin
  for item in 1..100000
  loop
    insert into t values(item);
    commit write [immediate | batch] [wait | nowait];
  end loop;
end;
/
```

Nowait 옵션에 의한 성능 개선 효과([2],[4])는 크게 두드러지지만 Batch 옵션의 영향력은 미미해 보인다.
현재 이 옵션에 대한 정확한 설명을 찾을 수 없는데(9i 버전부터 있던 'group 커밋' 개념을 가지고 설명하는 문서를 볼 수 있는데, 정확한 설명은 아닌 것 같다.), 아래 표를 통해 IMU(In-Memory
Undo) 기능과 관련 있음을 추정해 볼 수 있다. 아래 표는 위 테스트를 수행하는 도중에 v$sesstat 각 항목의 변화량(Delta)을 측정한 후, 그 중 가장 큰 차이를 보인 항목만 선별한 것이다.

|Statistics Name|immediate<br/>(wait, nowait)|batch<br/>(wait, nowait)|
|:---|---:|---:|
|commit immediate requested|100,000|0|
|commit immediate performed|100,000|0|
|commit batch requested|0|100,000|
|commit batch performed|0|100,000|
|session pga memory|0|262,144|
|IMU Flushes|94,005|435|
|IMU commits|0|93,892|
|IMU Redo allocation size|36,605,152|118,440|
|IMU undo allocation size|34,549,388|34,667,516|

IMU commit에 대한 설명을 찾을 수 없지만 Batch 옵션을 사용했을 때 PGA 메모리 할당량이 늘어나는 것을 통해, PGA 영역에 트랜잭션 데이터를 일정량 버퍼링했다가 일괄 처리한다는 것을 추정해 볼 수 있다.

지금까지 사용해 오던 커밋(immediate wait)은, 트랜잭션 데이터가 데이터베이스에 안전하게 저장됨을 보장한다.
하지만 비동기식 커밋 옵션을 사용하면, 트랜잭션 커밋 직후 인스턴스에 문제가 생기거나, Redo 로그가 위치한 파일 시스템에 문제가 생겨 쓰기 작업을 진행할 수 없게 되면 커밋이 정상적으로 완료되지 못할
수도 있다. 트랜잭션에 의해 생성되는 데이터 중요도에 따라 이 신기능의 활용 여부를 결정해야 한다.

참고로, commit_write 파라미터를 이용해 시스템 또는 세션 레벨에서 기본 설정을 변경할 수 있다.