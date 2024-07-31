# 05. Undo

#### ✅ Automatic Undo Management
- 오라클 8i 이전에는 Undo 세그먼트를 데이터베이스 관리자가 수동으로 관리했고, rollback_segments 파라미터에 의해 그 개수도 고정적이었다.
- 9i에서는 AUM(Automatic Undo Management)이 도입되어 Undo 세그먼트마다 하나의 트랜잭션이 할당되는 것을 목표로 세그먼트 개수를 오라클이 자동 관리한다.
  트랜잭션에 독립적으로 할당해 줄 Undo 세그먼트가 없을 때는(Online으로 전환할 수 있는 Offline 세그먼트가 없고 새로운 Undo 세그먼트를 생성할 공간도 부족할 때) 8i에서 가장 적게 사용되는
  Undo 세그먼트 중 하나를 할당한다.
- 예전에는 할당받은 Undo 세그먼트를 더 이상 확장할 수 없을 때 곧바로 ORA-01562 에러를 만났지만 AUM에서는 다른 Undo 세그먼트로부터 Free Undo Space를 가져올 수 있으며
  (Dynamic Extent Transfer), Undo 테이블스페이스 내에 있는 모든 Undo Space를 소진했을 때 비로소 에러를 발생시킨다.
#### ✅ Undo 세그먼트에 저장된 정보는 아래 3가지 목적을 위해 사용한다
- (1) Transaction Rollback
    - 트랜잭션에 의한 변경사항을 최종 커밋하지 않고 롤백하고자 할 때 Undo 데이터를 이용한다.
- (2) Transaction Recovery(Instance Recovery 시 rollback 단계)
    - 앞서 Redo에서 설명했듯이 Instance Crash 발생 후 Redo를 이용해 Roll forward 단계가 완료되면 최종 커밋되지 않은 변경사항까지 모두 복구된다.
      따라서 시스템이 셧다운된 시점에 아직 커밋되지 않았던 트랜잭션들을 모두 롤백해야 하는데, 이때 Undo 세그먼트에 저장된 Undo 데이터를 사용한다.
- (3) Read Consistency
    - Undo 데이터는 읽기 일관성(Read Consistency)을 위해 사용되는데, SQL 튜닝 관점에서 이 부분이 가장 중요한 관심사항이다.
      DB2, SQL Server, Sybase 등은 Lock을 통해 읽기 일관성을 구현하지만, 오라클은 Undo 데이터를 이용해 읽기 일관성을 구현한다.
      <br/>

## (1) Undo 세그먼트 트랜잭션 테이블 슬롯
#### ✅ Undo 세그먼트 헤더에는 트랜잭션 테이블 슬롯(Transaction Table Slot)이 위치하는데, 각 슬롯에 기록되는 사항들은 다음과 같다.
> (1) 트랜잭션 ID
>
> (2) 트랜잭션 상태정보 (Transaction Status)
>
> (3) 커밋 SCN (→ 트랜잭션이 커밋된 경우)
>
> (4) Last UBA (Undo Block Address)
>
> (5) 기타

- Undo 세그먼트 중 첫 번째 익스텐트, 그중에서도 첫 번째 블록에는 Undo 세그먼트 헤더(Header) 정보가 담긴다.
- 트랜잭션 ID는 [USN# + Slot# + Wrap#]으로 구성된다. USN은 Undo Segment Number의 약자다.
  트랜잭션 ID 구성을 보면 알 수 있듯이, 트랜잭션을 시작하려면 먼저 Undo 세그먼트에 있는 트랜잭션 테이블로부터 슬롯(Slot)을 할당받아야 하며, 할당받은 슬롯에 자신이 현재 Active 상태임을
  표시하고서 갱신을 시작한다.
  참고로, 트랜잭션 슬롯을 곧바로 얻지 못해 이용 가능한 슬롯이 생기기를 기다릴 때 발생하는 대기 이벤트가 undo segment tx slot 이다.
- 트랜잭션이 발생시키는 데이터 또는 인덱스 블록에 대한 변경사항은 Undo 블록에 Undo 레코드로서 하나씩 차례대로 기록된다. 각 DML 오퍼레이션 별로 Undo 레코드에 기록되는 내용은 아래와 같다.
    - Insert : 추가된 레코드의 rowid
    - Update : 변경되는 컬럼에 대한 before image
    - Delete : 지워지는 로우의 모든 컬럼에 대한 before image
- (4)번 Last UBA(Undo Block Address)는 트랜잭션의 기록사항들을 가장 마지막 Undo 레코드 뒤에 계속 추가해 나가려고 유지하는 일종의 포인터다.
  그리고 각 Undo 레코드 간에는 체인 형태로 연결돼 있어 데이터를 롤백하고자 할 때 이 체인을 따라 거슬러 올라가며 작업을 수행하게 된다.
- 하나의 Undo 블록에 쓰기가 완료되면 새로운 Undo 블록을 할당받아 쓰기 작업을 계속해 나간다.
  v$transaction 뷰에 있는 used_ublk와 used_urec 컬럼을 통해 현재 사용 중인 Undo 블록 개수와 현재까지 기록한 Undo 레코드 양을 확인할 수 있다.

  ```
  SQL> select s.sid, s.serial#, t.tidusn, t.used_ublk, t.used_urec
    2  from  v$session s, v$transaction t
    3  where t.addr = s.taddr
    4  and   s.sid = 144;
      SID    SERIAL#     XIDUSN  USED_UBLK  USED_UREC
  ------- ---------- ---------- ---------- ----------
      144          8          8          1         14
  
  1 개의 행이 선택되었습니다.
  ```
    - 인덱스가 전혀 없는 테이블이라면 한 건을 갱신할 때마다 used_urec 값이 하나씩 증가하지만 인덱스가 딸려 있다면 인덱스 엔트리에 대한 갱신 내용까지 값에 포함된다.
      예를 들어, insert 또는 delete 시에는 인덱스 하나당 하나의 Undo 레코드가 추가되고, update 시에는 인덱스 하나당 두 개의 Undo 레코드가 추가된다.
      인덱스 엔트리를 update 할 때는 내부적으로 delete & insert 방식으로 수행되기 때문이다.
- 아직 커밋되지 않은, Active 상태의 트랜잭션이 사용하는 Undo 블록과 트랜잭션 테이블 슬롯은 절대 다른 트랜잭션에 의해 재사용되지 않는다.
  하지만 가장 먼저 커밋된 트랜잭션 슬롯부터 순차적으로(in a circular fashion) 재사용되기 때문에 Undo 데이터는 커밋 후에도 상당기간 남아있게 된다.

#### ✅ Undo Retention
- 9i에서 AUM이 도입되면서 새롭게 생긴 파라미터가 undo_retention인데, 이는 트랜잭션이 완료되었어도 지정한 시간 동안은 "가급적" Undo 데이터를 재사용하지 말라고 오라클에 힌트를 주는 것이다.
- 오라클은 Undo Extent에 대한 상태정보(active, unexpired, expired)를 주기적으로 관리하며, 이를 위해 Undo 테이블스페이스에 할당된 모든 Extent의 Commit Time을 목록(List)으로 관리한다.
  undo_retention으로 지정된 시간을 기준으로 unexpired 상태의 Extent들을 expired 상태로 변경하며, Undo Extent가 필요할 때면 expired 상태의 Extent부터 재사용한다.
- 하지만 undo_retention은 강제성을 갖지 않기 때문에 expired Extent가 없고 새로 할당할 공간까지 부족해지면 unexpired 상태의 Extent라도 언제든 재사용할 수 있다.
  그래서 10g부터 새롭게 도입된 것이 retention guarantee 기능이다.

  ```
  alter tablespace undotbs1 retention guarantee;
  ```
    - Undo 테이블스페이스에 guarantee 옵션을 설정하면, 공간이 부족해 에러를 발생시키는 한이 있더라도 undo_retention으로 지정된 시간 이내에 커밋된 Undo 정보는 재사용하지 않는다.
- 10g에서 guarantee 기능과 함께 도입된 것이 Automatic Undo Retention Tuning 기능인데, 시스템 상황에 따라 tuned_undo_retention 값을 오라클이 자동으로 계산하며
  (내부적으로 수집한 통계정보 이용), 이를 기준으로 Undo Extent의 상태정보를 관리한다.
    - 이때 사용자가 설정한 undo_retention은, tuned_undo_retention이 그 수치 이하로 내려오지 못하도록 최소값(low threshold)을 지정하는 역할을 한다.
      다만, Undo 공간이 충분할 때 가급적 그렇게 하도록 요청하는 것이고, guarantee 옵션을 주지 않는 한 공간이 부족하면 언제든 재사용될 수 있다.


## (2) 블록 헤더 ITL 슬롯

#### ✅ 각 데이터 블록과 인덱스 블록 헤더에는 ITL(Interested Transaction List) 슬롯이 위치하는데, ITL 슬롯에 기록되는 내용은 다음과 같다.
> (1) ITL 슬롯 번호
>
> (2) 트랜잭션 ID
>
> (3) UBA (Undo Block Address)
>
> (4) 커밋 Flag
>
> (5) Locking 정보
>
> (6) 커밋 SCN (→ 트랜잭션이 커밋된 경우)

- 다음은 블록 Dump를 통해 ITL에 실제 어떤 정보들이 담기는지 들여다본 것이다.

  ```
  Itl    Xid                    Uba                  Flag  Lck    Scn/Fsc
  0x01   0x0004.00c.000035c3    0x008005a2.042f.1c   --U-    1    fsc 0x0003.03536d7f         
  0x02   0xffff.000.00000000    0x00000000.0000.00   C---    0    scn 0x0000.0353689e
  0x03   0x0005.00d.000003c4    0x00801a0b.01d1.1d   C-U-    0    scn 0x0000.006770c0
  ```
    - 트랜잭션을 시작하려면 Undo 세그먼트 트랜잭션 테이블로부터 슬롯을 먼저 확보하듯이, 특정 블록에 속한 레코드를 갱신하려면 먼저 블록 헤더로부터 ITL 슬롯을 확보해야 한다.
      거기에 트랜잭션 ID를 기록하고 현재 해당 트랜잭션이 Active 상태임을 표시한 후라야 블록 갱신이 가능하다.
      ITL 슬롯을 할당받지 못하면 트랜잭션은 계속 진행하지 못하고 블로킹(Blocking) 되었다가, 해당 블록을 갱신하던 앞선 트랜잭션 중 하나가 커밋 또는 롤백될 때 비로소 ITL 슬롯을 얻어
      작업을 계속 진행할 수 있게 된다.
    - 오라클은 ITL 슬롯 부족 때문에 트랜잭션이 블로킹 되는 현상을 최소화할 수 있도록 3가지 옵션을 제공한다.
      테이블과 인덱스를 생성할 때 initrans, maxtrans, pctfree 파라미터를 지정하는데, initrans 블록을 사용하려고 처음 포맷할 때 블록 헤더에 ITL 슬롯을 몇 개 할당할지를 결졍하는 파라미터다.
      블록 헤더에 미리 할당해 둔 ITL 슬롯이 모두 사용 중이라면 maxtrans로 지정된 개수만큼 데이터 영역에 추가 ITL 슬롯을 할당할 수 있다.
      단, pctfree에 의해 예약된 공간이 update(인덱스는 insert)에 의해 모두 사용되고 없다면 ITL을 할당받지 못해 Lock 경합이 발생하게 된다.
      참고로, ITL 슬롯이 부족할 때 발생하는 대기 이벤트(wait event)가 enq: TX - allocate ITL entry이다.


## (3) Lock Byte
#### ✅ 오라클은 레코드가 저장되는 로우마다 그 헤더에 Lock Byte를 할당해 해당 로우를 갱신 중인 트랜잭션의 ITL 슬롯 번호를 기록해 둔다
- 이것이 로우 단위(row-level) Lock이며, 오라클은 로우 단위 Lock과 트랜잭션 Lock(= TX Lock)을 조합해서 로우 Lock을 구현하였다.
- 레코드를 갱신하려고 할 때 대상 레코드의 Lock Byte가 활성화(turn-on)돼 있으면 ITL 슬롯을 찾아가고, 다시 그 ITL 슬롯이 가리키는 트랜잭션 테이블 슬롯을 찾아가 그 트랜잭션이 아직 active
  상태면 트랜잭션이 완료될 때까지 대기(→ 트랜잭션 Lock)한다.
#### ✅ 오라클은 로우 단위 Lock을 별도의 Lock 리소스 사용 없이 레코드의 속성으로서 관리하기 때문에 Lock 에스컬레이션 메커니즘이 전혀 불필요하다
- 다른 DBMS는 Lock 매니저를 통해 현재 갱신 중인 레코드 정보를 관리하는데, Lock 매니저가 사용할 수 있는 리소스가 유한하기 때문에 대량의 갱신이 발생할 때는 로우 단위 Lock 정보를
  모두 관리할 수 없어 블록 단위 또는 테이블 단위로 Lock 에스컬레이션(Escalation)이 발생하기도 하며, 그 순간 동시성이 현저히 저하된다.