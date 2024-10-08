# 6.4 | Lock과 트랜잭션 동시성 제어

Lock은 데이터베이스의 특징을 결정짓는 가장 핵심적인 메커니즘이다. 자신이 사용하는 데이터베이스의 고유한 Lock 메커니즘을 이해하지 못한다면, 고품질, 고성능 애플리케이션을 구축하기 어렵다
트랜잭션 동시성 제어도 데이터베이스 개발자라면 반드시 학습해야 할 주제다.

본 절은 오라클 Lock을 간단히 소개하고, 개발자가 알아야 할 기본적인 동시성 제어 기법을 설명한다.
마지막으로, 여러 채번 방식의 성능을 비교함으로써 테이블 식별자 설계 및 채번 방식 선택의 기준을 제시하고자 한다.

<br/>

## (1) 오라클 Lock
오라클은 공유 리소스와 사용자 데이터를 보호할 목적으로 DML Lock, DDL Lock, 래치, 버퍼 Lock, 라이브러리 캐시 Lock/Pin 등 다양한 종류의 Lock을 사용한다.
이 외에도 내부에 더 많은 종류의 Lock이 존재한다.
- DDL Lock을 'Dictionary Lock'이라고도 부르며, Exclusive DDL Lock, Share DDL Lock, (Breakable) Parse Lock 세 가지가 있다.
  그런데 DDL Lock은 오라클 문서 상 분류일 뿐, 실제로는 TM Lock(=DML 테이블 Lock), 라이브러리 캐시 Lock/Pin을 이용해 구현한 가상의 Lock이다.
  말하자면 DDL Lock은, TM Lock, 라이브러리 캐시 Lock/Pin의 작용으로 나타나는 현상을 또 하나의 Lock으로 분류한 것에 불과하다.

이 중 래치와 버퍼 Lock은 '(1.3.8) 캐시 탐색 메커니즘'에서 짧게 소개했다.
요약하면, 래치는 SGA에 공유된 각종 자료구조를 보호하기 위해 사용하며, 버퍼 Lock은 버퍼 블록에 대한 액세스를 직렬화하기 위해 사용한다.
라이브러리 캐시 Lock과 Pin은 라이브러리 캐시에 공유된 SQL 커서와 PL/SQL 프로그램을 보호하기 위해 사용한다.

애플리케이션 개발 측면에서 가장 중요하기 다루어야 할 Lock은 무엇보다 DML Lock이다. DML Lock은 다중 트랜잭션이 동시에 액세스하는 사용자 데이터의 무결성을 보호해 준다.
DML Lock에는 테이블 Lock과 로우 Lock이 있다.

### [DML 로우 Lock]
DML 로우 Lock은, 두 개의 동시 트랜잭션이 같은 로우를 변경하는 것을 방지한다. 하나의 로우를 변경하려면 로우 Lock을 먼저 설정해야 한다.

어떤 DBMS든지 DML 로우 Lock에는 배타적 모드를 사용하므로 UPDATE 또는 DELETE를 진행 중인(아직 커밋하지 않은) 로우를 다른 트랜잭션이 UPDATE하거나 DELETE 할 수 없다.

INSERT에 대한 로우 Lock 경합은 Unique 인덱스가 있을 때만 발생한다. 즉, Unique 인덱스가 있는 상황에서 두 트랜잭션이 같은 값을 입력하려고 할 때, 블로킹이 발생한다.
블로킹이 발생하면, 후행 트랜잭션은 기다렸다가 선행 트랜잭션이 커밋하면 INSERT에 실패하고, 롤백하면 성공한다.
두 트랜잭션이 서로 다른 값을 입력하거나 Unique 인덱스가 아예 없으면, INSERT에 대한 로우 Lock 경합은 발생하지 않는다.

MVCC 모델을 사용하는 오라클은(for update 절이 없는) SELECT 문에 로우 Lock을 사용하지 않는다. MVCC 모델은 '(6.1.1) 중 Undo 로깅과 DML 성능'에서 설명하였다.
요약하면, 오라클은 다른 트랜잭션이 변경한 로우를 읽을 때 복사본 블록을 만들어서 쿼리가 '시작된 시점'으로 되돌려서 읽는다.
변경이 진행 중인(아직 커밋하지 않은) 로우를 읽을 때도 Lock이 풀릴 때까지 기다리지 않고 복사본을 만들어서 읽는다. 따라서 SELECT 문에 Lock을 사용할 필요가 없다.

결국, 오라클에서는 DML과 SELECT는 서로 진행을 방해하지 않는다. 물론 SELECT끼리도 서로 방해하지 않는다. DML끼리는 서로 방해할 수 있는데, 이는 어떤 DBMS를 사용하더라도 마찬가지다.

참고로, MVCC 모델을 사용하지 않는 DBMS는 SELECT 문에 공유 Lock을 사용한다. 공유 Lock끼리는 호환된다. 두 트랜잭션이 같이 Lock을 설정할 수 있다는 뜻이다.
반면, 공유 Lock과 배타적 Lock은 호환되지 않기 때문에 DML과 SELECT가 서로 진행을 방해할 수 있다.
즉, 다른 트랜잭션이 읽고 있는 로우를 변경하려면 다음 레코드로 이동할 때까지 기다려야 하고, 다른 트랜잭션이 변경 중인 로우를 읽으려면 커밋할 때까지 기다려야 한다.

DML 로우 Lock에 의한 성능 저하를 방지하려면, 온라인 트랜잭션을 처리하는 주간에 Lock을 필요 이상으로 오래 유지하지 않도록 커밋 시점을 조절해야 한다.
그에 앞서 트랜잭션이 빨리 일을 마치도록, 즉 Lock이 오래 지속되지 않도록 관련 SQL을 모두 튜닝해야 한다. 1 ~ 5장에서 설명한 SQL 튜닝이 곧 Lock 튜닝인 셈이다.

### [DML 테이블 Lock]
오라클은 DML 로우 Lock을 설정하기에 앞서 테이블 Lock을 먼저 설정한다. 현재 트랜잭션이 갱신 중인 테이블 구조를 다른 트랜잭션이 변경하지 못하게 막기 위해서다.
테이블 Lock을 'TM Lock'이라고 부르기도 한다.

오라클은 로우 Lock에 항상 배타적 모드를 사용하지만, 테이블 Lock에는 여러 가지 Lock 모드를 사용한다. Lock 모드간 호환성(Compatibility)을 정리하면 다음 표와 같다.
('○' 표시는 두 모드 간에 호환성이 있음을 의미한다.)

||Null|RS|RX|S|SRX|X|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|**Null**|○|○|○|○|○|○|
|**RS**|○|○|○|○|○||
|**RX**|○|○|○||||
|**S**|○|○||○|||
|**SRX**|○|○|||||
|**X**|○||||||

- RS : row share (또는 SS : sub share)
- RX : row exclusive (또는 SX : sub exclusive)
- S : share
- SRX : share row exclusive (또는 SSX : share/sub exclusive)
- X : exclusive

선행 트랜잭션과 호환되지 않는 모드로 테이블 Lock을 설정하려는 후행 트랜잭션은 대기하거나 작업을 포기해야 한다.

INSERT, UPDATE, DELETE, MERGE 문을 위해 로우 Lock을 설정하려면 해당 테이블에 RX(=SX) 모드 테이블 Lock을 먼저 설정해야 한다.
SELECT FOR UPDATE 문을 위해 로우 Lock을 설정하려면 10gR1 이하는 RS(=SS) 모드, 10gR2 이상은 RX(=SX) 모드 테이블 Lock을 먼저 설정해야 한다.
RS, RX 간에는 어떤 조합으로도 호환이 되므로 SELECT FOR UPDATE나 DML문 수행 시 테이블 Lock에 의한 경합은 절대 발생하지 않는다.
같은 로우를 갱신하려고 할 때만 로우 Lock에 의한 경합이 발생한다.

'테이블 Lock'이라고 하면, 테이블 전체에 Lock이 걸린다고 생각하기 쉽다. 그래서 다른 트랜잭션이 더는 레코드를 추가하거나 갱신하지 못하게 막는다고 생각하는 분들이 많다.
하지만 앞서 설명했듯이 DML을 수행하기 전에 항상 테이블 Lock을 먼저 설정하므로 그렇게 이해하는 것은 맞지 않는다.
하나의 로우를 변경하기 위해 테이블 전체에 Lock을 건다면 동시성이 좋은 애플리케이션을 구현하기 어렵다.

오라클에서 말하는 테이블 Lock은, 자신(테이블 Lock을 설정한 트랜잭션)이 해당 테이블에서 현재 어떤 작업을 수행 중인지를 알리는 일종의 푯말(Flag)이다.
그리고 위에서 본 것처럼 테이블 Lock에는 여러 가지 모드가 있고, 어떤 모드를 사용했는지에 따라 후행 트랜잭션이 수행할 수 있는 작업의 범위가 결정된다.
푯말에 기록된 Lock 모드와 후행 트랜잭션이 현재 하려는 작업 내용에 따라 진행 여부가 결정된다.
진행할 수 없다면 기다릴지, 아니면 작업을 포기할지 진로를 결정(내부적으로 하드코딩 돼 있거나 사용자가 지정한 옵션에 따라 결정)해야 한다. 기다려야 한다면, 대기자 목록에 Lock 요청을 등록하고 기다린다.
- 참고로, SQL Server에서는 이런 종류의 테이블 Lock을 'Intent Lock'이라고 부른다.

예를 들어, DDL을 이용해 테이블 구조를 변경하려는 트랜잭션은 해당 테이블에 TM Lock이 설정돼 있는지를 먼저 확인한다.
TM Lock을 RX(=SX) 모드로 설정한 트랜잭션이 하나라도 있으면, 현재 테이블을 갱신 중인 트랜잭션이 있다는 신호다. 따라서 ORA-00054 메시지를 남기고 작업을 멈춘다.
반대로, DDL 문이 먼저 수행 중일 때는, DML 문을 수행하려는 트랜잭션이 기다린다.

- #### [대상 리소스가 사용 중일 때, 진로 선택]

  Lock을 얻고자 하는 리소스가 사용 중일 때, 프로세스는 아래 세 가지 방법 중 하나를 택한다. 보통은 내부적으로 진로가 결정돼 있지만, 사용자가 선택할 수 있는 경우도 있다.
  사용자가 이 세 가지 옵션을 모두 선택할 수 있는 문장이 바로 SELECT FOR UPDATE 문이다.

    - [1] Lock이 해제될 때까지 기다린다. (예 : select * from t for update)
    - [2] 일정 시간만 기다리다 포기한다. (예 : select * from t for update wait 3)
    - [3] 기다리지 않고 작업을 포기한다. (예 : select * from t for update nowait)

  DML을 수행할 때 묵시적으로 테이블 Lock을 설정하는데, 이때는 1번, 기다리는 방법을 선택한다.
  Lock Table 명령을 이용해 명시적으로 테이블 Lock을 설정할 때도 기본적으로 기다리는 방법을 택하지만, NOWAIT 옵션을 이용해 곧바로 작업을 포기하도록 사용자가 지정할 수 있다.

  ```
  lock table emp in exclusive mode NOWAIT;
  ```

  DDL을 수행할 때도 내부적으로 테이블 Lock을 설정하는데, 이때는 NOWAIT 옵션이 자동으로 지정된다.
  오라클 11g부터 ddl_lock_timeout 파라미터를 0보다 크게 설정하면, 설정한 시간(초 단위)만큼 기다리다 작업을 포기하게 할 수 있다.

### [Lock을 푸는 열쇠, 커밋]
가끔 블로킹과 교착상태를 구분 못 하는 분들을 만난다. 블로킹(Blocking)은 선행 트랜잭션이 설정한 Lock 때문에 후행 트랜잭션이 작업을 진행하지 못하고 멈춰 있는 상태를 말한다.
이것을 해소하는 방법은 커밋(또는 롤백)뿐이다.

교착상태(Deadlock)는 두 트랜잭션이 각각 특정 리소스에 Lock을 설정한 상태에서 맞은편 트랜잭션이 Lock을 설정한 리소스에 또 Lock을 설정하려고 진행하는 상황을 말한다.
교착상태가 발생하면 둘 중 하나가 뒤로 물러나지 않으면 영영 풀릴 수 없다. 좁은 골목길에 두 대의 차량이 마주 선 것에 비유할 수 있다.

오라클에서 교착상태가 발생하면, 이를 먼저 인지한 트랜잭션이 문장 수준 롤백을 진행한 후에 아래 에러 메시지를 던진다. 교착상태를 발생시킨 문장 하나만 롤백하는 것이다.
```
ORA-00060: deadlock detected while waiting for resource
```
이제 교착상태는 해소됐지만 블로킹 상태에 놓이게 된다. 따라서 이 메시지를 받은 트랜잭션은 커밋 또는 롤백을 결정해야만 한다.
만약 프로그램 내에서 이 에러에 대한 예외처리(커밋 또는 롤백)를 하지 않는다면 대기 상태를 지속하게 되므로 주의가 필요하다.

오라클은 데이터를 읽을 때 Lock을 사용하지 않으므로 다른 DBMS에 비해 상대적으로 Lock 경합이 적게 발생한다.
읽는 트랜잭션의 진행을 막는 부담감이 없으므로 필요한 만큼 트랜잭션을 충분히 길게 가져갈 수 있다.

그렇더라도 '불필요하게' 트랜잭션을 길게 정의하지 않도록 주의해야 한다. 트랜잭션이 너무 길면, 트랜잭션을 롤백해야 할 때 너무 많은 시간이 걸려 고생할 수 있다.
Undo 세그먼트가 고갈되거나 Undo 세그먼트 경합을 유발할 수도 있다.

같은 데이터를 갱신하는 트랜잭션이 동시에 수행되지 않도록 애플리케이션을 설계해야 하고, DML Lock 때문에 동시성이 저하되지 않도록 적절한 시점에 커밋해야 한다.

반대로, 불필요하게 커밋을 너무 자주 수행하면 서버 프로세스가 LGWR에게 로그 버퍼를 비우도록 요청하고 동기(sync) 방식으로 기다리는 횟수가 늘기 때문에 기본적으로 성능이 느려진다.
잦은 커밋 때문에 성능이 매우 느리다면, 오라클 10gR2부터 제공하는 비동기식 커밋과 배치 커밋을 활용하는 방안을 검토할 수 있다.
- WAIT(Default) : LGWR가 로그버퍼를 파일에 기록했다는 완료 메시지를 받을 때까지 기다린다(동기식 커밋).
- NOWAIT : LGWR의 완료 메시지를 기다리지 않고 바로 다음 트랜잭션을 진행한다(비동기식 커밋).
- IMMEDIATE(Default) : 커밋 명령을 받을 때마다 LGWR가 로그 버퍼를 파일에 기록한다.
- BATCH : 세션 내부에 트랜잭션 데이터를 일정량 버퍼링했다가 일괄 처리한다.

이들 옵션을 조합해 아래 네 가지 커밋 명령을 사용할 수 있다.
```
SQL> COMMIT WRITE IMMEDIATE WAIT;      ------ [1]
SQL> COMMIT WRITE IMMEDIATE NOWAIT;    ------ [2]
SQL> COMMIT WRITE BATCH WAIT;          ------ [3]
SQL> COMMIT WRITE BATCH NOWAIT;        ------ [4]
```

<br/>

## (2) 트랜잭션 동시성 제어
동시성 제어의 개념은 이전에 이미 설명하였다. 동시성 제어는 비관적 동시성 제어와 낙관적 동시성 제어로 나뉜다.

비관적 동시성 제어(Pessimistic Concurrency Control)는 사용자들이 같은 데이터를 동시에 수정할 것으로 가정한다.
따라서 한 사용자가 데이터를 읽는 시점에 Lock을 걸고 조회 또는 갱신처리가 완료될 때까지 이를 유지한다.
Lock은 첫 번째 사용자가 트랜잭션을 완료하기 전까지 다른 사용자들이 같은 데이터를 수정할 수 없게 만들기 때문에 비관적 동시성 제어를 잘못 사용하면 동시성이 나빠진다.
'잘못 사용하면'이라고 한 데에 주목하자. 잘 사용하면 약이 될 수도 있다는 뜻이며, 글을 계속 읽다 보면 이해하게 될 것이다.

반면, 낙관적 동시성 제어(Optimistic Concurrency Control)는 사용자들이 같은 데이터를 동시에 수정하지 않을 것으로 가정한다. 따라서 데이터를 읽을 때 Lock을 설정하지 않는다.
그런데 낙관적 입장에 섰다고 해서 동시 트랜잭션에 의한 잘못된 데이터 갱신을 신경 쓰지 않아도 된다는 것은 절대 아니다.
읽는 시점에 Lock을 사용하지 않았지만, 데이터를 수정하고자 하는 시점에 앞서 읽은 데이터가 다른 사용자에 의해 변경되었는지 반드시 검사해야 한다.

### [비관적 동시성 제어]
비관적 동시성 제어를 위한 기본 구현 패턴을 살펴보자. 우수 고객을 대상으로 적립포인트를 제공하는 이벤트를 실시한다고 가정하자.
아래 코딩 예시처럼 고객의 다양한 실적정보를 읽고 복잡한 산출공식을 이용해 적립포인트를 계산하는 동안(아래 SELECT 문 이후, UPDATE 문 이전) 다른 트랜잭션이 같은 고객의 실적정보를 변경한다면
문제가 생길 수 있다.
```
select 적립포인트, 방문횟수, 최근방문일시, 구매실적 from 고객
where  고객번호 = :cust_num;

-- 새로운 적립포인트 게산

update 고객 set 적립포인트 = :적립포인트 where 고객번호 = :cust_num
```
하지만, 아래와 같이 SELECT 문에 FOR UDPATE를 사용하면 고객 레코드에 Lock을 설정하므로 데이터가 잘못 갱신되는 문제를 방지할 수 있다.
```
select 적립포인트, 방문횟수, 최근방문일시, 구매실적 from 고객
where  고객번호 = :cust_num for update;
```
비관적 동시성 제어는 자칫 시스템 동시성을 심각하게 떨어뜨릴 우려가 있지만, FOR UPDATE에 WAIT 또는 NOWAIT 옵션을 함께 사용하면 Lock을 얻기 위해 무한정 기다리지 않아도 된다.
```
for update nowait  → 대기없이 Exception(ORA-00054)을 던짐
for update wait 3  → 3초 대기 후 Exception(ORA-30006)을 던짐
```
WAIT 또는 NOWAIT 옵션을 사용하면, 다른 트랜잭션에 의해 Lock이 걸렸을 때 Exception을 만나게 되므로 "다른 사용자에 의해 변경 중이므로 다시 시도하십시오"라는 메시지를 출력하면서 트랜잭션을
종료할 수 있다. 따라서 오히려 동시성을 증가시키게 된다.

- #### [큐(Queue) 테이블 동시성 제어]

  큐(Queue) 테이블에 쌓인 고객 입금 정보를 일정한 시간 간격으로 읽어서 입금 테이블에 반영하는 데몬 프로그램이 있다고 가정하자.

  데몬이 여러 개이므로 Lock이 걸릴 수 있는 상황이다. Lock이 걸리면 3초간 대기했다가 다음에 다시 시도하게 하려고 아래와 같이 for update wait 3 옵션을 지정했다.
  큐에 쌓인 데이터를 한 번에 다 읽어서 처리하면 Lock이 풀릴 때까지 다른 데몬이 오래 걸릴 수 있으므로 고객 정보를 100개씩만 읽도록 했다.

  ```
  select cust_id, rcpt_amt from cust_rcpt_Q
  where yn_upd = 'Y' and rownum <= 100 FOR UPDATE WAIT 3;
  ```

  이럴 때 아래와 같이 skip locked 옵션을 사용하면, Lock이 걸린 레코드는 생략하고 다음 레코드를 계속 읽도록 구현할 수 있다.
  (SQL에서 rownum 조건절을 제거하고, 클라이언트 단에서 100개를 읽으면 멈추도록 구현해야 한다. 한 건씩 Fetch하면 성능이 안 좋으니 100개 단위로 Array 처리하기 바란다.)

  ```
  select cust_id, rcpt_amt from cust_rcpt_Q
  where yn_upd = 'Y' FOR UPDATE SKIP LOCKED;
  ```


### [낙관적 동시성 제어]
아래는 SELECT-LIST에서 네 개 컬럼을 참조했을 때의 낙관적 동시성 제어 예시다.
```
select 적립포인트, 방문횟수, 최근방문일시, 구매실적 into :a, :b, :c, :d
from   고객
where  고객번호 = :cust_num;

-- 새로운 적립 포인트 계산

update 고객 set 적립포인트 = :적립포인트
where  고객번호    = :cust_num
and    적립포인트   = :a
and    방문횟수     = :b
and    최근방문일시  = :c
and    구매실적     = :d ;

if sql%rowcount = 0 then
  alert('다른 사용자에 의해 변경되었습니다.');
end if;
```
SELECT 문에서 읽은 컬럼이 매우 많다면 UPDATE 문에 조건절을 일일이 기술하는 것이 여간 귀찮은 일이 아닐 것이다.
만약 UPDATE 대상 테이블에 최종변경일시를 관리하는 컬럼이 있다면, 이를 조건절에 넣어 간단히 해당 레코드의 갱신여부를 판단할 수 있다.
```
select 적립포인트, 방문횟수, 최근방문일시, 구매실적, 변경일시(*)
into   :a, :b, :c, :d, :mod_dt
from   고객
where  고객번호 = :cust_num;

-- 새로운 적립 포인트 계산

update 고객 set 적립포인트 = :적립포인트, 변경일시 = SYSDATE
where  고객번호    = :cust_num
and    변경일시    = :mod_dt;  → 최종 변경일시가 앞서 읽은 값과 같은지 비교

if sql%rowcount = 0 then
  alert('다른 사용자에 의해 변경되었습니다.');
end if;
```
낙관적 동시성 제어에서도 UPDATE 전에 아래 SELECT 문(nowait 옵션을 사용한 것에 주목)을 한 번 더 수행함으로써 Lock에 대한 예외처리를 한다면, 다른 트랜잭션이 설정한 Lock을 기다리지 않게
구현할 수 있다.
```
select 고객번호
from   고객
where  고객번호 = :cust_num
and    변경일시 = :mod_dt
for update nowait;
```

### [동시성 제어 없는 낙관적 프로그래밍]
낙관적 동시성 제어를 사용하면 Lock이 유지되는 시간이 매우 짧아서 동시성을 높이는 데 매우 유리하다.
하지만 다른 사용자가 같은 데이터를 변경했는지 검사하고 그에 따라 처리 방향성을 결정하는 귀찮은 절차가 뒤따른다. 정말 귀찮아서일까, 아니면 동시성 제어의 필요성을 몰라서일까?
튜닝하다 보면 개발팀이 작성한 SQL을 많이 분석하게 되는데, 동시성 제어 없이 낙관적으로 프로그래밍하는 경우를 자주 본다.

예를 들어, 온라인 쇼핑몰에서 특정 상품을 조회해서 결제를 완료하는 순간까지를 하나의 트랜잭션으로 정의했다고 가정하자.

TX1이 t1 시점에 상품을 조회할 때는 가격이 1,000원이었다. 주문을 진행하는 동안 TX2에 의해 가격이 1,200원으로 수정되었다면, TX1이 최종 결제 버튼을 클릭하는 순간 어떻게 처리해야 할까?
상품 정보를 조회한 시점 기준으로 주문을 처리하는 것이 업무 규칙이라면 주문을 그대로 처리해도 상관없지만, 그렇지 않다면 아래와 같이 상품가격의 변경 여부를 체크함으로써 해당 주문을 취소시키거나
사용자에게 변경사실을 알리고 처리방향을 확인받는 프로세스를 거쳐야 한다.
```
insert into 주문
select :상품코드, :고객ID, :주문일시, :상점번호, ...
from   상품
where  상품코드 = :상품코드
and    가격 = :가격 ;      -- 주문을 시작한 시점 가격

if sql%rowcount = 0 then
  alert('상품가격이 변경되었습니다.');
end if;
```
그런데 그런 로직을 찾기 힘들다. 주문을 진행하는 동안 상품 공급업체가 가격을 변경하지 않을 것이라고 낙관적으로 생각하기 때문이다.

지금 당장 자신이 회원으로 가입해 있는 아무 사이트나 방문해서 브라우저를 좌우로 두 개 띄우고 회원정보 변경 화면으로 각각 이동해 보자.
그리고 좌측 화면에서는 전화번호를 바꾸고 커밋, 우측 화면에서는 생년월일을 고치고 커밋한 후에 다시 회원정보를 조회해 보면 전화번호가 이전으로 롤백되어 있는 것을 확인할 수 있을 것이다.
사용자가 수정한 필드값만 UPDATE하지 않고 화면에 뿌려진 전체 필드를 UPDATE하도록 구현하기 때문에 생기는 현상이다.

물론 동시에 고객정보를 갱신할 가능성이 전혀 없거나 그렇게 처리하는 것이 업무 규칙이라면 전혀 문제 삼을 일이 아니다.
하지만 그렇지 않은데도 동시성 제어를 제대로 구현하지 않아 고객 정보가 계속 잘못 갱신된다면, 언제고 고객으로부터 클레임을 받을 수도 있는 문제다.

### [데이터 품질과 동시성 향상을 위한 제언]
본서는 성능을 다루는 책이지만, 성능보다 데이터 품질이 더 중요하다는 사실을 강조하지 않을 수 없다. 동의한다면, FOR UPDATE 사용을 두려워하지 말자.
다중 트랜잭션이 존재하는 데이터베이스 환경에서 공유 자우너에 대한 액세스 직렬화는 필수다. JAVA 프로그래머라면 멀티 쓰레드 프로그래밍할 때 synchronized 키워드의 역할을 상기하기 바란다.

데이터를 변경할 목적으로 읽는다면 당연히 Lock을 걸어야 한다. 금융권에서 근무하는 개발자는 이를 아주 당연하게 여기지만, 그렇지 않은 환경에서 근무하는 개발자는 FOR UPDATE를 잘 활용하지 않는다.
FOR UPDATE 자체를 모르는 개발자도 있다. FOR UPDATE를 제거했더니 성능 문제가 해결됐다고 뿌듯해하는 개발자를 만난 적도 있다.
그렇게 성능 문제가 해결됐다면 그 순간부터 데이터 품질이 나빠지고 있다고 생각하면 틀림없다.

FOR UPDATE가 필요한 상황이면 이를 정확히 사용하고, 코딩하기 번거롭더라도 동시성이 나빠지지 않게 WAIT 또는 NOWAIT 옵션을 활용한 예외처리에 세심한 주의를 기울여야 한다.

불필요하게 Lock을 오래 유지하지 않고, 트랜잭션의 원자성을 보장하는 범위 내에서 가급적 빨리 커밋하자.
트랜잭션을 재생할 수 있는 경우(원본 데이터를 읽어 가공된 데이터를 생성하는 경우)라면, 중간에 적당한 주기로 커밋하는 방안도 고려할 수 있다.
꼭 주간에 수행할 필요가 없는 배치 프로그램은 야간 시간대에 수행하자.

낙관적, 비관적 동시성 제어를 같이 사용하는 방법도 있다.
일단 낙관적 동시성 제어를 시도했다가 다른 트랜잭션에 의해 데이터가 변경된 사실이 발견되면, 롤백하고 다시 시도할 때 비관적 동시성 제어를 사용하는 방식이다.

동시성을 향상하고자 할 때 SQL 튜닝은 기본이다. 가장 효율적인 인덱스를 구성해 주고, 데이터량에 맞는 조인 메소드를 선택해야 한다.
루프를 돌면서 절차적으로 처리하면 성능이 매우 느리고, 느린 만큼 Lock도 오래 지속된다. Array Processing을 활용하든, One SQL로 구현하든, 처리 성능이 빨라지면 Lock도 빨리 해제된다.
Lock에 대한 고민은 트랜잭션 내 모든 SQL을 완벽히 튜닝하고 나서 해도 늦지 않다.

- #### [로우 Lock 대상 테이블 지정]

  계좌마스터와 주문 테이블이 있다고 가정하자. 쿼리를 아래와 같이 작성하면 계좌마스터와 주문 테이블 양쪽 모두에 로우 Lock이 걸린다.

  ```
  select b.주문수량
  from   계좌마스터 a, 주문 b
  where  a.고객번호 = :cust_no
  and    b.계좌번호 = a.계좌번호
  and    b.주문일자 = :ord_dt
  for update
  ```

  아래와 같이 작성하면 주문수량이 있는 주문 테이블에만 로우 Lock이 걸린다.
  ```
  select b.주문수량
  from   계좌마스터 a, 주문 b
  where  a.고객번호 = :cust_no
  and    b.계좌번호 = a.계좌번호
  and    b.주문일자 = :ord_dt
  for update of b.주문수량
  ```

<br/>

## (3) 채번 방식에 따른 INSERT 성능 비교
INSERT, UPDATE, DELETE, MERGE 중 가장 중요하고 튜닝 요소가 많은 것은 INSERT다. 수행빈도가 가장 높아서 그렇기도 하지만, 채번 방식에 따른 성능 차이가 매우 크기 때문이다.

신규 데이터를 입력하려면 PK 중복을 방지하기 위한 채번이 선행되어야 하는데, 가장 많이 사용하는 아래 세 가지 채번 방식의 성능과 장단점을 비교해 보자.
- 채번 테이블
- 시퀀스 오브젝트
- MAX + 1 조회

설명의 편의를 위해 용어부터 정리할 필요가 있다.
예를 들어, PK가 [상담원ID + 상담일자 + 상담순번]처럼 복합컬럼으로 구성돼 있을 때, 순번 이외의 컬럼(상담원ID, 상담일자)을 지금부터 '구분 속성'이라고 부르기로 하자.

### [채번 테이블]
각 테이블 식별자의 단일컬럼 일련번호 또는 구분 속성별 순번을 채번하기 위해 별도 테이블을 관리하는 방식이다.
채번 레코드를 읽어서 1을 더한 값으로 변경하고, 그 값을 새로운 레코드를 입력하는 데 사용한다.
이 방식은 채번 레코드를 변경하는 과정에 자연스럽게 액세스 직렬화(트랜잭션 줄 세우기)가 이루어지므로 두 트랜잭션이 중복 값을 채번할 가능성을 원천적으로 방지해 준다.

이 방식의 장점부터 나열하면 아래와 같다.
- 범용성이 좋다.
- INSERT 과정에 중복 레코드 발생에 대비한 예외(Exception) 처리에 크게 신경쓰지 않아도 되므로 채번 함수만 잘 정의하면 편리하게 사용할 수 있다.
  (→ MAX + 1 방식과 비교)

- INSERT 과정에 결번을 방지할 수 있다. (→ 시퀀스 방식과 비교)
- PK가 복합컬럼일 때도 사용할 수 있다. (→ 시퀀스 방식과 비교)

가장 큰 단점은 다른 채번 방식에 비해 성능이 안 좋다는 데 있다. 채번 레코드를 변경하기 위한 로우 Lock 경합 때문이다.
로우 Lock은 기본적으로 (자율 트랜잭션 기능을 활용하지 않으면) 대상 테이블에 INSERT를 마치고 커밋 또는 롤백할 때까지 지속된다.

동시 INSERT가 아주 많으면 채번 레코드뿐만 아니라 채번 테이블 블록 자체에도 경합이 발생한다. 서로 다른 레코드를 변경하는 프로세스끼리도 경합할 수 있다는 뜻이다.
- 구체적으로 말하면, 버퍼 Lock과 ITL 경합을 의미한다. 버퍼 Lock에 대해서는 '(1.3.8) 캐시 탐색 메커니즘'에서 설명하였다.

PK가 복합컬럼인 경우, 즉 구분 속성별 순번을 채번하는 경우에는 Lock 경합이 줄어들지만, 구분 속성 레코드 수가 소수일 때만 이 방식을 사용하므로 Lock 경합이 발생할 가능성은 다른 채번 방식에 비해
여전히 높다. 따라서 동시 INSERT가 아주 많은 테이블에는 사실상 이 방식을 사용하기가 어렵다.
- 예를 들어, 고객별 순번에 채번 테이블을 사용하면, 고객 수만큼 채번 레코드가 필요하므로 이 방식을 사용하기에 무리수가 따른다.

- #### [자율 트랜잭션]

  PL/SQL의 자율(Autonomous) 트랜잭션 기능을 이용하면 메인 트랜잭션에 영향을 주지 않고 서브 트랜잭션에서 일부 자원만 Lock을 해제할 수 있다. 방법은 간단하다.
  PL/SQL 선언부에 아래와 같이 'pramga autonomous_transaction'이라고 선언하기만 하면 된다.

  ```
  create or replace function seq_nextval(l_gubun number) return number
  as
      pragma autonomous_transaction;  **
      l_new_seq seq_tab.seq%type;
  begin
      update seq_tab
      set    seq = seq + 1
      where  gubun = l_gubun;

      select seq into l_new_seq
      from   seq_tab
      where  gubun = 1_gubun;

      commit;
      return l_new_seq;
  end;
  ```

  PL/SQL 함수/프로시저를 자율 트랜잭션으로 선언하면, 그 내부에서 커밋을 수행해도 메인 트랜잭션은 커밋하지 않은 상태로 남는다.
  메인 트랜잭션 INSERT 문에서 아래와 같이 채번 함수를 호출하고 최종적으로 커밋하기 전까지 다른 작업을 많이 수행하더라도 채번 테이블 로우 Lock은 이미 해제한 상태이므로 다른 트랜잭션을 블록킹하지
  않는다.

  ```
  insert into target_tab values ( seq_nextval(123), :x, :y, :z );


### [시퀀스 오브젝트]
시퀀스(Sequence)의 가장 큰 장점은 성능이 빠르다는 데 있다. 채번 테이블과 마찬가지로 INSERT 과정에 중복 레코드 발생에 대비한 예외처리에 크게 신경 쓰지 않아도 된다.
테이블별로 시퀀스 오브젝트를 생성하고 관리하는 부담은 있지만, 개발팀 입장에서는 사용하기에 매우 편리하다.

시퀀스의 가장 큰 장점이 성능이지만, 성능 이슈가 없는 것은 아니다. 시퀀스 채번 과정에 발생하는 Lock 때문이다.
시퀀스의 성능 이슈를 이해하려면, 시퀀스 오브젝트가 오라클 내부에서 관리하는 채번 테이블이라는 사실을 이해해야 한다. 구체적으로 SYS.SEQ$ 테이블을 말하며, DBA_SEQUENCES 뷰를 통해 조회할 수 있다.

시퀀스 오브젝트도 결국 테이블이므로 값을 읽고 변경하는 과정에 Lock 메커니즘이 작동한다. 시퀀스 Lock에 의한 성능 이슈가 있지만, 캐시 사이즈를 적절히 설정하면 가장 빠른 성능을 제공한다.
시퀀스에는 자율 트랜잭션 기능도 기본적으로 구현돼 있다.

- #### [시퀀스 Lock]

  오라클이 시퀀스 오브젝트에 사용하는 Lock으로는 세 가지가 있다. 로우 캐시 Lock, 시퀀스 캐시 Lock, SV Lock이 그것이다.

  #### [1] 로우 캐시 Lock
  딕셔너리 정보를 매번 디스크에서 읽고 쓰면 성능이 매우 느리므로 오라클은 로우 캐시를 사용한다. 로우 캐시는 공유 캐시(SGA)의 구성요소이므로 정보를 읽고 쓸 때 액세스를 직렬화해야 한다.
  이를 위해 사용하는 Lock이 '로우 캐시 Lock'이다.
    - 테이블, 인덱스, 테이블스페이스, 데이터파일, 세그먼트, 익스텐트, 사용자, 제약, 시퀀스, DB Link 등에 관한 정보를 오라클은 '딕셔너리(Dictionary)'라고 부른다.
      또한, 딕셔너리 정보를 빠르게 읽고 쓰기 위해 캐시를 사용하는데, 이를 '딕셔너리 캐시' 또는 '로우 캐시'라고 부른다.
      '로우 캐시'라는 이름을 갖게 된 이유는, 딕셔너리 캐시는 로우 단위로 I/O 하기 때문이다. 일반적인 데이터 블록은 블록 단위로 I/O 한다는 사실을 상기하기 바란다.
    - 오라클에서 로우 캐시 Lock에 의한 경합은 'row cache lock' 대기 이벤트로 측정할 수 있다.

  로우 캐시를 사용하는 대표적인 오브젝트가 시퀀스(SYS.SEQ$)이므로 로우 캐시 Lock 경합이 나타날 수 있다.
  즉, 'nextval을 호출할 때마다' 로우 캐시에서 시퀀스 레코드(last_number 컬럼)를 변경해야 하는데, 많은 사용자가 동시에 nextval을 호출하면 로우 캐시 Lock 경합이 발생한다.

  시퀀시 채번으로 인한 로우 캐시 Lock 경합을 줄이기 위해 오라클은 기본적으로 CACHE 옵션을 사용한다.
  옵션을 명시적으로 설정하지 않았을 때 기본값은 20이다. 시퀀스 채번에 의한 로우 캐시 Lock 경합을 줄이고 싶다면, 이 값을 크게 설정하면 된다.
  반대로, 채번 빈도가 낮아 굳이 캐시를 사용하고 싶지 않다면 NOCACHE 옵션을 지정하면 된다. 아래 스크립트 결과를 참조하면 CACHE 옵션을 더 정확히 이해할 수 있다.

  ```
  SQL> create sequence MYSEQ cache 1000;

  SQL> select cache_size, last_number
    2  from   user_sequences
    3  where  sequence_name = 'MYSEQ';

  CACHE_SIZE LAST_NUMBER
  ---------- -----------
        1000           1

  SQL> select MYSEQ.NEXTVAL from dual;

      NEXTVAL
  -----------
            1

  SQL> select cache_size, last_number
    2  from   user_sequences
    3  where  sequence_name = 'MYSEQ';
  
  CACHE_SIZE LAST_NUMBER
  ---------- -----------
        1000        1001
  ```

  CACHE 크기를 1,000으로 지정한 시퀀스를 생성하고 nextval을 호출했더니 last_number 값이 1에서 1,001로 증가했다. 이제 nextval을 호출할 때마다 시퀀스 레코드를 변경하지 않아도 된다.
  값을 시퀀스 캐시에서 얻으면 되기 때문이다. 시퀀스 캐시에서 1,000개의 값을 모두 소진한 직후 nextval을 호출하면 그때 다시 로우 캐시에서 시퀀스 레코드를 2,001로 변경한다.

  #### [2] 시퀀스 캐시 Lock
  시퀀스 캐시도 공유 캐시에 위치한다. 따라서 시퀀스 캐시에서 값을 얻을 때도 액세스 직렬화가 필요하며, 이를 'SQ Lock'이라고 부른다.
    - 오라클에서 SQ Lock에 의한 경합은 'enq: SQ - contention' 대기 이벤트로 측정할 수 있다.

  #### [3] SV Lock
  시퀀스 캐시는 한 인스턴스 내에서 공유된다. nextval을 호출하는 순서대로 값을 제공하므로 인스턴스 내에서는 번호 순서를 보장한다.

  데이터베이스 하나에 인스턴스가 여러 개인 RAC 환경에서는 인스턴스마다 시퀀스 캐시를 따로 갖는다. 따라서 인스턴스 간에는 번호 순서를 기본적으로 보장하지 않는다.

  예를 들어, 첫 번째 nextval을 1번 인스턴스 A 프로세스가 호출하고, 이어서 두 번째 nextval은 2번 인스턴스 B 프로세스가 호출한다.
  그때부터 1번 인스턴스 시퀀시 캐시는 1부터 1,000까지의 값을 순서대로 반환하고, 2번 인스턴스 시퀀스 캐시는 1,001부터 2,000까지의 값을 순서대로 반환한다.
  따라서 1번과 2번 인스턴스에 있는 프로세스들이 교차로 nextval을 호출하면, 테이블에는 아래와 같은 순서로 값이 입력된다.

  > 1 → 1001 → 2 → 1002 → 3 → 1003 → 4 → 1004 → 5 → 1005 → ...

  식별자가 갖추어야 하는 기본 조건을 상기해 보자. 식별자는 값이 유일(Unique)해야 하고, 반드시 값이 있어야(Not Null) 한다. 식별자에 값을 순서대로 입력해야 한다는 조건은 어디에도 없다.
  위와 같이 입력해도 식별자로서 조건을 전혀 위배하지 않는다. 그런데 식별자 번호를 레코드 입력순으로 넣어주길 원하느나 업무 담당자가 많다. 즉, 식별자가 일련번호(Serial Number)이길 원한다.

  어떤 이유에서든 업무적으로 순서를 보장해야 한다면, 즉 어떤 인스턴스에서 nextval을 호출하더라도 순서대로 일련번호를 제공해야 한다면, ORDER 옵션을 사용하면 된다.
  그러면 시퀀스 캐시 하나를 모든 RAC 노드가 공유한다.

  그런데 자원을 공유할 때는 항상 Lock 메커니즘이 필요하다. RAC 환경에서 ORDER 옵션을 사용하면 오라클은 'SV Lock'을 통해 시퀀스 캐시에 대한 액세스를 직렬화한다.
    - 오라클에서 SV Lock에 의한 경합은 'DFS lock handle' 대기 이벤트로 측정할 수 있다.

  RAC 각 노드는 네트워크 상에 서로 분리된 서버인데, 어떻게 시퀀스 캐시를 공유할까? 네트워크를 통해 시퀀스 캐시를 서로 주고 받으면서 공유한다. 문제는 성능이다.

  1번 인스턴스에서 nextval을 연속해서 1,000번 호출하고, 이어서 2번 인스턴스에서 연속해서 1,000번을 호출하고, 이어서 다시 1번 인스턴스가 1,000번을 호출하는 트랜잭션 패턴이라면
  ORDER 옵션을 사용해도 성능이 나빠지진 않는다.
  하지만, 1번과 2번 인스턴스가 교대로 nextval을 빠르게 호출하는 트랜잭션 패턴이라면 ORDER 옵션을 사용하는 순간 INSERT 성능은 급격히 나빠진다.
  업무적으로 꼭 필요할 때만 ORDER 옵션을 사용해야 하는 이유가 여기에 있다.

  마지막으로 가장 많이 받는 질문 중 하나이다. "RAC 환경에서 ORDER 옵션을 사용했더니 성능이 급격히 나빠졌다. 해결방법은?" 방법은 단 한 가지다. ORDER 옵션을 제거하는 것뿐이다.

  그럼 다른 채번 방식을 사용하겠다고 생각하겠지만, 다른 방식을 사용해도 아래와 같이 똑같은 부작용이 나타난다.

    - 시퀀스 : 시퀀스 캐시를 인스턴스 간에 주고 받아야 함
    - 채번 테이블 : 채번 레코드가 저장된 데이터 및 인덱스 블록을 인스턴스 간에 주고 받아야 함
    - MAX + 1 : MAX 값을 찾는 데 필요한 인덱스 블록을 인스턴스 간에 주고 받아야 함

  RAC 환경에서 ORDER 옵션을 주고 안 주고에 따라 성능 차이가 많이 나지만, 그래도 시퀀스를 이용한 채번이 가장 빠르다. 결론적으로, 시퀀스의 가장 큰 장점은 성능이다.
  '동시 INSERT가 많은' 테이블에 단일속성의 일련번호 식별자를 두었다면, 시퀀스를 활용하는 것보다 나은 솔루션은 없다.

시퀀스의 가장 큰 단점은 '기본적으로' PK가 단일컬럼일 때만 사용 가능하다는 데 있다.
PK가 복합컬럼일 때도 사용할 수는 있지만, 각 레코드를 유일하게 식별하는(유일성을 보장하는) 최소 컬럼으로 PK를 구성해야 한다는 최소성(Minimalty) 요건을 위배하게 된다.

- #### [순환옵션을 가진 시퀀스 활용]

  PK가 복합컬럼인데 동시 트랜잭션이 높아 시퀀스가 꼭 필요하다면, 순환(cycle) 옵션을 가진 시퀀스 활용을 고려할 수 있다.
  하루에 도저히 도달할 수 없는 값으로 최대값(maxvalue)을 설정하고, 그 값에 도달하면 1부터 다시 시작하도록 순환옵션을 설정하는 것이다.
  순환옵션을 사용하는 이유는 값이 무한정 커지지 않게 함으로써 순번 컬럼 길이를 최소화하기 위해서다. 일반적으로 권장할 솔루션은 아니지만, 채번 성능이 문제가 될 때 고려해 볼 수 있다.

시퀀스의 또 다른 단점은, 신규 데이터를 입력하는 과정에 결번이 생길 수 있다는 점이다. 원인은 크게 두 가지다. 첫째, 시퀀스 채번 이후에 트랜잭션을 롤백하는 경우다.
둘째, CACHE 옵션을 설정한 시퀀스가 캐시에서 밀려나는 경우다.
자주 사용하지 않아(시퀀스 채번 간격이 길어) 캐시에서 밀려나거나 인스턴스를 재기동하는 순간, 캐시돼 있던 번호는 모두 사라지며 디스크에서 다시 읽을 때 그다음 번호부터 읽는다.

인스턴스 재기동에 의한 결번은 피할 방법이 없지만(NOCACHE 옵션을 지정하면 되지만 성능에 문제가 생김) 사용 빈도가 낮아서 생기는 결번은 방법이 있다.
시퀀스를 Shared Pool에 KEEP하도록 아래 명령을 수행해 주면 된다.
```
SQL> EXEC SYS.DBMS_SHARED_POOL.KEEP('SCOTT.MY_SEQ', 'Q');
```
그런데 과연 일련번호에 결번이 생기는 현상을 막을 필요가 있을까? 업무적으로 반드시 그래야 하는 경우는 많지 않다. 그리고 채번 테이블이나 MAX + 1 방식을 사용하더라도 결번을 원천적으로 막을 수는 없다.
데이터를 삭제하면서 생기는 결번은 막을 수 없기 때문이다. (자율 트랜잭션을 사용하면 채번 테이블에서 채번할 때도 롤백에 의해 결번이 생길 수 있다.)

### [MAX + 1 조회]
아래와 같이 대상 테이블의 최종 일련번호를 조회하고, 거기에 1을 더해서 INSERT하는 방식이다.
```
insert into 상품거래(거래일련번호, 계좌번호, 거래일시, 상품코드, 거래가격, 거래수량)
values  (  (select max(거래일련번호) + 1 from 상품거래
          , :acnt_no, sysdate, :prod_cd, :trd_price, :trd_qty  );
```
이 방식의 장점으로는 첫째, 시퀀스 또는 별도의 채번 테이블을 관리하는 부담이 없다. 둘째, 동시 트랜잭션에 의한 충돌이 많지 않으면, 성능이 매우 빠르다.
셋째, PK가 복합컬럼인 경우, 즉 구분 속성별 순번을 채번할 때도 사용할 수 있다.
채번 테이블은 구분 속성 값의 수(Number of Distinct Values)가 적을 때만 사용할 수 있지만, 이 방식은 값의 수가 아무리 많아도 상관없다. 오히려 값의 수가 많을수록 성능이 더 좋아진다.
입력 값 중복에 의한 로우 Lock 경합이 줄고 재실행 횟수도 줄기 때문이다.

단점으로는 첫째, 레코드 중복에 대비한 세밀한 예외처리가 필요하다. 둘째, 다중 트랜잭션에 의한 동시 채번이 심하면 시퀀스보다 성능이 많이 나빠질 수 있다. 레코드 중복에 의한 로우 Lock 경합 때문이다.
로우 Lock은 선행 트랜잭션이 커밋 또는 롤백할 때까지 지속된다. 선행 트랜잭션이 롤백하지 않는 한, INSERT는 결국 실패하게 되므로 채번과 INSERT를 다시 실행해야 한다.
이런 현상이 자주 발생하면 성능이 늘릴 수밖에 없다.

다행히 PK가 복합컬럼이고 구분 속성별 값의 수가 많으면, 구분 속성 값별로 채번이 분산된다. 따라서 동시 채번이 많아도 로우 Lock 경합 및 재실행 가능성은 현저히 줄어든다.

로우 Lock 경합 이외의 성능 이슈는 MAX 값 조회에 최적화된 인덱스를 구성해 주지 않을 때 생긴다. '(5.3.3) 최소값/최대값 구하기'에서 설명한 내용을 상기하기 바란다.

각 채번 방식에 발생하는 Lock 경합 요소를 정리하면, 다음과 같다.

|채번 방식|식별자<br/>구조|주요/부수적 경합|비고|
|:---:|:---:|:---|:---|
|**채번 테이블**|**일련번호**|- 주요 경합 : (값 변경을 위한) 로우 Lock 경합<br/>- 부수적인 경합 : (동시성이 높다면) 채번 테이블 블록 경합|- 채번 테이블 관리 부담|
|**채번 테이블**|**구분+순번**|단일 일련번호일 떄보다 Lock 경합 감소||
|**시퀀스 오브젝트**|**일련번호**|- 주요 경합 : 시퀀스 경합<br/>- 부수적인 경합 : (시퀀스 경합 해소 시)인덱스 블록 경합|- 시퀀스 관리 부담<br/>- INSERT 과정에 결번 가능성|
|**MAX + 1**|**일련번호**|- 주요 경합 : (입력 값 중복 시) 로우 Lock + 재실행<br/>- 부수적인 경합 : (동시성이 매우 높다면) 인덱스 블록 경합|- 별도 오브젝트 관리 없음<br/>- 중복 값 발생에 대비한 예외처리 필수<br/>- PK 인덱스 구성에 따른 성능 차이 발생|
|**MAX + 1**|**구분+순번**|단일 일련번호일 때보다 Lock 경합 감소<br/>(구분 속성 값의 종류가 많으면, 현저히 감소)|위와 동일|

















