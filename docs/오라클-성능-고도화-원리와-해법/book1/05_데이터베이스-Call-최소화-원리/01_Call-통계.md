# 01. Call 통계

아래는 SQL 트레이스 레포트에서 Call 통계(Statistics) 부분만을 발췌한 것이다. 이 레포트는 커서의 활동상태를 Parse, Execute, Fetch 세 단계로 나누어 각각에 대한 수행통계를 보여준다.

```
select cust_nm, birthday from customer where cust_id = :cust_id

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute   5000   0.18       0.14       0        0          0       0
Fetch     5000   0.21       0.25       0    20000          0   50000
------- ------ ------ ---------- ------- -------- ---------- -------
total    10001   0.39       0.40       0    20000          0   50000

Misses in library cache during parse: 1
```

**Parse Call**은 커서를 파싱하는 과정에 대한 통계로서, 실행계획을 생성하거나 찾는 과정에 관한 정보를 포함한다. **Execute Call**은 말 그대로 커서를 실행하는 단계에 대한 통계를 보여준다.
**Fetch Call**은 select문에서 실제 레코드를 읽어 사용자가 요구한 결과집합을 반환하는 과정에 대한 통계를 보여준다.

Parse Call 최소화 및 최적화 원리에 대해서는 앞 장에서 자세히 살펴보았다.
바인드 변수를 사용해서 Parse Call을 가볍게 하는 방법, 세션 커서 캐싱을 통해 Parse Call을 더 가볍게 하는 방법, 애플리케이션 커서 캐싱을 통해 Parse Call이 아예 발생하지 않게 하는 방법 등을
기억할 것이다. 위 트레이스 결과에서도 애플리케이션 커서 캐싱 기법을 사용했음을 알 수 있다(Parse Call = 1).

insert, update, delete, merge 등 DML문은 Execute Call 시점에 모든 처리과정을 서버 내에서 완료하고 처리결과만 리턴하므로 Fetch Call이 전혀 발생하지 않는다.
insert...select문도 마찬가지다. 클라이언트로부터 명시적인 Fetch Call을 받지 않으며 서버 내에서 묵시적으로 Fetch가 이루어진다.

```
insert into MEMEBER_BACKUP
select * from MEMBER

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.07       0.15       0        3          0       0
Execute      1   6.27      43.08   13001    18410     481251  449604
Fetch        0   0.00       0.00       0        0          0       0
------- ------ ------ ---------- ------- -------- ---------- -------
total        2   6.34      43.23   13001    18413     481251  449604

Misses in library caache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 64

Rows     Row Source Operation
-------  ----------------------------------------------------
 449604  TABLE ACCESS FULL MEMBER (cr=14443 pr=13001 pw=0 time=64765167 us)
```

select문일 때 Execute Call 단계에서는 커서만 오픈하고, 실제 데이터를 처리하는 과정은 모두 Fetch 단계에서 일어난다.
예를 들어, 아래는 100만 건짜리 CUST 테이블을 group by하는 쿼리인데, sort group by는 Execute 단계에서 처리할 것으로 예상되지만 실제로는 Fetch 시점에 모든 처리가 일어난다.
실제 데이터를 액세스하면서 일을 시작하는 시점은 첫 번째 Fetch Call인 것을 짐작할 수 있다.

```
select REGION, count(*)
from   CUST
group by REGION

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.06       0.16       0        3          0       0
Execute      1   0.00       0.00       0        0          0       0
Fetch        3   2.26      41.20   14427    14443          0      26
------- ------ ------ ---------- ------- -------- ---------- -------
total        5   2.32      41.37   14427    14446          0      26

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 64

Rows     Row Source Operation
-------  ----------------------------------------------------
     26  SORT GROUP BY (cr=14443 pr=14427 pw=0 time=41208794 us)
1000000   TABLE ACCESS FULL CUST (cr=14443 pr=14427 pw=0 time=34065961 us)
```

for update 구문을 사용하면 Execute Call 단계에서 모든 레코드를 읽어 Lock을 설정한다.
아래는 10,000개 레코드를 갖는 테이블을 for update 구문을 사용해 쿼리한 결과인데, 사용자는 11번의 Fetch Call을 통해 101개 레코드만 Fetch하고 멈췄지만 Execute 단계에서 이미 Current
모드로 읽어 10,000개 레코드 전체에 대해 Lock을 설정했음을 알 수 있다.

```
select *
from  SUPPLIER for update

call     count    cpu    elapsed    disk    query    current    rows
------- ------ ------ ---------- ------- -------- ---------- -------
Parse        1   0.00       0.00       0        0          0       0
Execute      1   0.20       0.24       0       74      10178       0
Fetch       11   0.00       0.00       0       14          0     101
------- ------ ------ ---------- ------- -------- ---------- -------
total        5   2.32      41.37   14427    14446          0      26

Misses in library cache during parse: 0
Optimizer mode: ALL_ROWS
Parsing user id: 61

Rows     Row Source Operation
-------  ----------------------------------------------------
    101  FOR UPDATE  (cr=92 pr=0 pw=0 time=246704 us)
  10101   TABLE ACCESS FULL SUPPLIER (cr=87 pr=0 pw=0 time=70852 us)
```