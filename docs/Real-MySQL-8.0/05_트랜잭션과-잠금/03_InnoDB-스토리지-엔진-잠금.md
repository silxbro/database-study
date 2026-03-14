# [CH 5-3] InnoDB 스토리지 엔진 잠금

InnoDB 스토리지 엔진은 MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다.
InnoDB는 레코드 기반의 잠금 방식 때문에 MyISAM보다는 훨씬 뛰어난 동시성 처리를 제공할 수 있다.
하지만 이원화된 잠금 처리 탓에 InnoDB 스토리지 엔진에서 사용되는 잠금에 대한 정보는 MySQL 명령을 이용해 접근하기가 상당히 까다롭다.
예전 버전의 MySQL 서버에서는 InnoDB의 잠금 정보를 진단할 수 있는 도구라고는 lock_monitor(innodb_lock_monitor라는 이름의 InnoDB 테이블을 생성해서 InnoDB의 잠금 정보를 덤프하는
방법)와 SHOW ENGINE INNODB STATUS 명령이 전부였다. 하지만 이 내용도 거의 어셈블리 코드를 보는 것 같아서 이해하기가 상당히 어려웠다.

하지만 최근 버전에서는 InnoDB의 트랜잭션과 잠금, 그릐고 잠금 대기 중인 트랜잭션의 목록을 조회할 수 있는 방법이 도입됐다.
MySQL 서버의 information_schema 데이터베이스에 존재하는 INNODB_TRX, INNODB_LOCKS, INNODB_LOCK_WAITS라는 테이블을 조인해서 조회하면 현재 어떤 트랜잭션이 어떤 잠금을 대기하고
있고 해당 잠금을 어떤 트랜잭션이 가지고 있는지 확인할 수 있으며, 또한 장시간 잠금을 가지고 있는 클라이언트를 찾아서 종료시킬 수도 있다.
그리고 조금씩 업그레이드되면서 InnoDB의 중요도가 높아졌고, InnoDB의 잠금에 대한 모니터링도 더 강화되면서 Performance Schema를 이용해 InnoDB 스토리지 엔진의 내부 잠금(세마포어)에 대한
모니터링 방법도 추가됐다.

---
<br/>

## (1) InnoDB 스토리지 엔진의 잠금
InnoDB 스토리지 엔진은 레코드 기반의 잠금 기능을 제공하며, 잠금 정보가 상당히 작은 공간으로 관리되기 때문에 레코드 락이 페이지 락으로, 또는 테이블 락으로 레벨업되는 경우(락 에스컬레이션)는 없다.
일반 상용 DBMS와는 조금 다르게 InnoDB 스토리지 엔진에서는 레코드 락뿐 아니라 레코드와 레코드 사이의 간격을 잠그는 갭(GAP) 락이라는 것이 존재하는데, 아래 그림은 InnoDB 스토리지 엔진의 레코드
락과 레코드 간의 간격을 잠그는 갭 락을 보여준다.

#### [그림 5.1] InnoDB 잠금의 종류(점선의 레코드는 실제 존재하지 않는 레코드를 가정한 것임)
<img src="https://github.com/user-attachments/assets/76b6dc9e-d5e8-4883-8793-b8f20b53568c" width="350"/><br/>

### [1] 레코드 락
레코드 자체만을 잠그는 것을 레코드 락(Record lock, Record only lock)이라고 하며, 다른 상용 DBMS의 레코드 락과 동일한 역할을 한다.
한 가지 중요한 차이는 InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다는 점이다.
인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.
많은 사용자가 간과하는 부분이지만 레코드 자체를 잠그느냐, 아니면 인덱스를 잠그느냐는 상당히 크고 중요한 차이를 만들어 내기 때문에 다음에 다시 잠깐 예제로 다루겠다.

InnoDB에서는 대부분 보조 인덱스를 이용한 변경 작업은 넥스트 키 락(Next key lock) 또는 갭 락(Gap lock)을 사용하지만 프라이머리 키 또는 유니크 인덱스에 의한 변경 작업에서는 갭(Gap, 간격)에
대해서는 잠그지 않고 레코드 자체에 대해서만 락을 건다.

### [2] 갭 락
다른 DBMS와의 또 다른 차이가 바로 갭 락(Gap lock)이다. 갭 락은 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미한다.
갭 락의 역할은 레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT)되는 것을 제어하는 것이다. 갭 락은 그 자체보다는 넥스트 키 락의 일부로 자주 사용된다.

### [3] 넥스트 키 락
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락(Next key lock)이라고 한다. STATEMENT 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는 REPEATABLE READ 격리 수준을 사용해야 한다.
또한 `innodb_locks_unsafe_for_binlog` 시스템 변수가 비활성화되면(0으로 설정되면) 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다.
InnoDB의 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다.
그런데 의외로 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다.
가능하다면 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락이나 갭 락을 줄이는 것이 좋다.

> MySQL 5.5 버전까지는 ROW 포맷의 바이너리 로그가 도입된 지 오래되지 않아서 그다지 널리 사용되지 않았다.
> 하지만 MySQL 5.7 버전과 8.0 버전으로 업그레이드되면서 ROW 포맷의 바이너리 로그에 대한 안정성도 높아졌으며 STATEMENT 포맷의 바이너리 로그가 가지는 단점을 많이 해결해줄 수 있기 때문에
> MySQL 8.0에서는 ROW 포맷의 바이너리 로그가 기본 설정으로 변경됐다.

### [4] 자동 증가 락
MySQL에서는 자동 증가하는 숫자 값을 추출(채번)하기 위해 AUTO_INCREMENT라는 칼럼 속성을 제공한다.
AUTO_INCREMENT 칼럼이 사용된 테이블에 동시에 여러 레코드가 INSERT되는 경우, 저장되는 각 레코드는 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가져야 한다.
InnoDB 스토리지 엔진에서는 이를 내부적으로 AUTO_INCREMENT 락(Auto increment lock)이라고 하는 테이블 수준의 잠금을 사용한다.

AUTO_INCREMENT 락은 INSERT와 REPLACE 쿼리 문장과 같이 새로운 레코드를 저장하는 쿼리에서만 필요하며, UPDATE나 DELETE 등의 쿼리에서는 걸리지 않는다.
InnoDB의 다른 잠금(레코드 락이나 넥스트 키 락)과는 달리 AUTO_INCREMENT 락은 트랜잭션과 관계없이 INSERT나 REPLACE 문장에서 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시
해제된다.
AUTO_INCREMENT 락은 테이블에 단 하나만 존재하기 떄문에 두 개의 INSERT 쿼리가 동시에 실행되는 경우 하나의 쿼리가 AUTO_INCREMENT 락을 걸면 나머지 쿼리는 AUTO_INCREMENT 락을 기다려야
한다(AUTO_INCREMENT 칼럼에 명시적으로 값을 설정하더라도 자동 증가 락을 걸게 된다).

AUTO_INCREMENT 락을 명시적으로 획득하고 해제하는 방법은 없다. AUTO_INCREMENT 락은 아주 짧은 시간 동안 걸렸다가 해제되는 잠금이라서 대부분의 경우 문제가 되지 않는다.
자동 증가 락에 대한 지금까지의 설명은 MySQL 5.0 이하 버전에서 사용되던 방식이다.
MySQL 5.1 이상부터는 `innodb_autoinc_lock_mode`라는 시스템 변수를 이용해 자동 증가 락의 작동 방식을 변경할 수 있다.

- #### innodb_autoinc_lock_mode=0
  MySQL 5.0과 동일한 잠금 방식으로 모든 INSERT 문장은 자동 증가 락을 사용한다.
- #### innodb_autoinc_lock_mode=1
  단순히 한 건 또는 여러 건의 레코드를 INSERT하는 SQL 중에서 MySQL 서버가 INSERT되는 레코드의 건수를 정확히 예측할 수 있을 때는 자동 증가 락(Auto increment lock)을 사용하지 않고,
  훨씬 가볍고 빠른 래치(뮤텍스)를 이용해 처리한다. 개선된 래치는 자동 증가 락과 달리 아주 짧은 시간 동안만 잠금을 걸고 필요한 자동 증가 값을 가져오면 즉시 잠금이 해제된다.
  하지만 INSERT ... SELECT와 같이 MySQL 서버가 건수를 (쿼리를 실행하기 전에) 예측할 수 없을 때는 MySQL 5.0에서와 같이 자동 증가 락을 사용한다.
  이때는 INSERT 문장이 완료되기 전까지는 자동 증가 락은 해제되지 않기 때문에 다른 커넥션에서는 INSERT를 실행하지 못하고 대기하게 된다.
  이렇게 대량 INSERT가 수행될 때는 InnoDB 스토리지 엔진은 여러 개의 자동 증가 값을 한 번에 할당받아서 INSERT되는 레코드에 사용한다.
  그래서 대량 INSERT되는 레코드는 자동 증가 값이 누락되지 않고 연속되게 INSERT된다.
  하지만 한 번에 할당받은 자동 증가 값이 남아서 사용되지 못하면 폐기하므로 대량 INSERT 문장의 실행 이후에 INSERT되는 레코드의 자동 증가 값은 연속되지 않고 누락된 값이 발생할 수 있다.
  이 설정에서는 최소한 하나의 INSERT 문장으로 INSERT되는 레코드는 연속된 자동 증가 값을 가지게 된다. 그래서 이 설정 모드를 연속 모드(Consecutive mode)라고도 한다.
- #### innodb_autoinc_lock_mode=2
  innodb_autoinc_lock_mode가 2로 설정되면 InnoDB 스토리지 엔진은 절대 자동 증가 락을 걸지 않고 경량화된 래치(뮤텍스)를 사용한다.
  하지만 이 설정에서는 하나의 INSERT 문장으로 INSERT되는 레코드라고 하더라도 연속된 자동 증가 값을 보장하지는 않는다. 그래서 이 설정 모드를 인터리빙 모드(Interleaved mode)라고도 한다.
  이 설정 모드에서는 INSERT ... SELECT와 같은 대량 INSERT 문장이 실행되는 중에도 다른 커넥션에서 INSERT를 수행할 수 있으므로 동시 처리 성능이 높아진다.
  하지만 이 설정에서는 자동 증가 기능은 유니크한 값이 생성된다는 것만 보장한다.
  STATEMENT 포맷의 바이너리 로그를 사용하는 복제에서는 소스 서버와 레플리카 서버의 자동 증가 값이 달라질 수도 있기 때문에 주의해야 한다.

더 자세한 내용은 [MySQL 매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)의 내용을 참조하길 바란다.
별로 관계없는 것 같지만, 자동 증가 값이 한 번 증가하면 절대 줄어들지 않는 이유가 AUTO_INCREMENT 잠금을 최소화하기 위해서다.
설령 INSERT 쿼리가 실패했더라도 한 번 증가된 AUTO_INCREMENT 값은 다시 줄어들지 않고 그대로 남는다.

> MySQL 5.7 버전까지는 innodb_autoinc_lock_mode의 기본값이 1이었지만, MySQL 8.0 버전부터는 innodb_autoinc_lock_mode의 기본값이 2로 바뀌었다.
> 이는 MySQL 8.0부터 바이너리 로그 포맷이 STATEMENT가 아니라 ROW 포맷이 기본값이 됐기 때문이다.
> MySQL 8.0에서 ROW 포맷이 아니라 STATEMENT 포맷의 바이너리 로그를 사용한다면 innodb_autoinc_lock_mode를 2가 아닌 1로 변경해서 사용할 것을 권장한다.
<br/>

## (2) 인덱스와 잠금
InnoDB의 잠금과 인덱스는 상당히 중요한 연관 관계가 있기 때문에 다시 한번 더 자세히 살펴보자.
"레코드 락"을 소개하면서 잠깐 언급했듯이 InnoDB의 잠금은 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리된다.
즉, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다. 정확한 이해를 위해 다음 UDPATE 문장을 한 번 살펴보자.

```
-- // 예제 데이터베이스의 employees 테이블에는 아래와 같이 first_name 칼럼만
-- // 멤버로 담긴 ix_firstname이라는 인덱스가 준비돼 있다.
-- //   KEY ix_firstname (first_name)
-- // employees 테이블에서 first_name='Georgi'인 사원은 전체 253명이 있으며,
-- // first_name='Georgi'이고 last_name='Klassen'인 사원은 딱 1명만 있는 것을 아래 쿼리로
-- // 확인할 수 있다.
mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
+----------+
|      253 |
+----------+

mysql> SELECT COUNT(*) FROM employees WHERE first_name='Georgi' AND last_name='Klassen';
+----------+
|        1 |
+----------+

-- // employees 테이블에서 first_name='Georgi'이고 last_name='Klassen'인 사원의
-- // 입사 일자를 오늘로 변경하는 쿼리를 실행해보자.
mysql> UPDATE employees SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen';
```

UPDATE 문장이 실행되면 1건의 레코드가 업데이트될 것이다. 하지만 이 1건의 업데이트를 위해 몇 개의 레코드에 락을 걸어야 할까?
이 UPDATE 문장의 조건에서 인덱스를 이용할 수 있는 조건은 first_name='Georgi'이며, last_name 칼럼은 인덱스에 없기 때문에 first_name='Georgi'인 레코드 253건의 레코드가 모두 잠긴다.
아래 그림은 예제의 UPDATE 문장이 어떻게 변경 대상 레코드를 검색하고, 실제 변경이 수행되는지를 보여준다.
아마 MySQL에 익숙하지 않은 사용자라면 상당히 이상하게 생각될 것이며, 이러한 부분을 잘 모르고 개발하면 MySQL 서버를 제대로 이용하지 못할 것이다.
또한 이러한 MySQL의 특성을 알지 못하면 "MySQL은 정말 이상한 데이터베이스군"이라고 생각하게 될 것이다.
이 예제에서는 몇 건 안 되는 레코드만 잠그지만 UPDATE 문장을 위해 적절히 인덱스가 준비돼 있지 않다면 각 클라이언트 간의 동시성이 상당히 떨어져서 한 세션에서 UPDATE 작업을 하는 중에는 다른
클라이언트는 그 테이블을 업데이트하지 못하고 기다려야 하는 상황이 발생할 것이다.

#### [그림 5.2] 업데이트를 위해 잠긴 레코드와 실제 업데이트된 레코드
<img src="https://github.com/user-attachments/assets/ba32f5e1-fd48-4896-839d-32d64c791cd9" width="470"/><br/>

이 테이블에 인덱스가 하나도 없다면 어떻게 될까? 이러한 경우에는 테이블을 풀 스캔하면서 UPDATE 작업을 하는데, 이 과정에서 테이블에 있는 30여만 건의 모든 레코드를 잠그게 된다.
이것이 MySQL의 방식이며, MySQL의 InnoDB에서 인덱스 설계가 중요한 이유 또한 이것이다.
<br/>
<br/>
## (3) 레코드 수준의 잠금 확인 및 해제
InnoDB 스토리지 엔진을 사용하는 테이블의 레코드 수준 잠금은 테이블 수준의 잠금보다는 조금 더 복잡하다. 테이블 잠금에서는 잠금의 대상이 테이블 자체이므로 쉽게 문제의 원인이 발견되고 해결될 수 있다.
하지만 레코드 수준의 잠금은 테이블의 레코드 각각에 잠금이 걸리므로 그 레코드가 자주 사용되지 않는다면 오랜 시간 동안 잠겨진 상태로 남아 있어도 잘 발견되지 않는다.

예전 버전의 MySQL 서버에서는 레코드 잠금에 대한 메타 정보(딕셔너리 테이블)를 제공하지 않기 때문에 더더욱 어려운 부분이다.
하지만 MySQL 5.1부터는 레코드 잠금과 잠금 대기에 대한 조회가 가능하므로 쿼리 하나만 실행해 보면 잠금과 잠금 대기를 바로 확인할 수 있다.
그럼 버전별로 레코드 잠금과 잠금을 대기하는 클라이언트의 정보를 확인하는 방법을 알아보자. 강제로 잠금을 해제하려면 KILL 명령을 이용해 MySQL 서버의 프로세스를 강제로 종료하면 된다.

우선 다음과 같은 잠금 시나리오를 가정해보자.

<img src="https://github.com/user-attachments/assets/a4d3995c-e69f-4895-b4bd-771ef7d03153" width="580"/><br/>

각 트랜잭션이 어떤 잠금을 기다리고 있는지, 기다리고 있는 잠금을 어떤 트랜잭션이 가지고 있는지를 쉽게 메타 정보를 통해 조회할 수 있다.
우선 MySQL 5.1부터는 information_schema라는 DB에 INNODB_TRX라는 테이블과 INNODB_LOCKS, INNODB_LOCK_WAITS라는 테이블을 통해 확인이 가능했다.
하지만 MySQL 8.0 버전부터는 information_schema의 정보들은 조금씩 제거(Deprecated)되고 있으며, 그 대신 performance_schema의 data_locks와 data_lock_waits 테이블로 대체되고
있다. 여기서는 performance_schema의 테이블을 이용해 잠금과 잠금 대기 순서를 확인하는 방법을 살펴보자.

우선 아래 내용은 MySQL 서버에서 앞의 UPDATE 명령 3개가 실행된 상태의 프로세스 목록을 조회한 것이다(가독성을 위해서 꼭 필요한 부분만 캡처했다).
17번 스레드는 지금 아무것도 하지 않고 있지만 트랜잭션을 시작하고 UPDATE 명령이 실행 완료된 것이다.
하지만 아직 17번 스레드는 COMMIT을 실행하지는 않은 상태이므로 업데이트한 레코드의 잠금을 그대로 가지고 있는 상태다.
18번 스레드가 그다음으로 UPDATE 명령을 실행했으며, 그 이후 19번 스레드에서 UPDATE 명령을 실행했다.
그래서 프로세스 목록에서 18번과 19번 스레드는 잠금 대기로 인해 아직 UPDATE 명령을 실행 중인 것으로 표시된 것이다.
```
mysql> SHOW PROCESSLIST;
+----+-------+----------+-----------------------------------------------------------+
| Id | Time  | State    | Info                                                      |
+----+-------+----------+-----------------------------------------------------------+
| 17 |   607 |          | NULL                                                      |
| 18 |    22 | updating | UPDATE employees SET birth_date=NOW() WHERE emp_no=100001 |
| 19 |    21 | updating | UPDATE employees SET birth_date=NOW() WHERE emp_no=100001 |
+----+-------+----------+-----------------------------------------------------------+
```

이제 performance_schema의 data_locks 테이블과 data_lock_waits 테이블을 조인해서 잠금 대기 순서를 한 번 살펴보자. 다음 내용 또한 가독성을 위해 조금 편집한 결과다.
```
mysql> SELECT
         r.trx_id waiting_trx_id,
         r.trx_mysql_thread_id waiting_thread,
         r.trx_query waiting_query,
         b.trx_id blocking_trx_id,
         b.trx_mysql_thread_id blocking_thread,
         b.trx_query blocking_query
       FROM performance_schema.data_lock_waits w
       INNER JOIN information_schema.innodb_trx b
          ON b.trx_id = w.blocking_engine_transaction_id
       INNER JOIN information_schema.innodb_trx r
          ON r.trx_id = w.requesting_engine_transaction_id;

+---------+---------+-------------------+----------+----------+-------------------+
| waiting | waiting | waiting_query     | blocking | blocking | blocking_query    |
| _trx_id | _thread |                   |  _trx_id |  _thread |                   |
+---------+---------+-------------------+----------+----------+-------------------+
|   11990 |      19 | UPDATE employees..|    11989 |       18 | UPDATE employees..|
|   11990 |      19 | UPDATE employees..|    11984 |       17 | NULL              |
|   11989 |      18 | UPDATE employees..|    11984 |       17 | NULL              |
+---------+---------+-------------------+----------+----------+-------------------+
```

쿼리의 실행 결과를 보면 현재 대기 중인 스레드는 18번과 19번인 것을 알 수 있다.
18번 스레드는 17번 스레드를 기다리고 있고, 19번 스레드는 17번 스레드와 18번 스레드를 기다리고 있다는 것을 알 수 있다. 이는 잠금 대기 큐의 내용을 그대로 보여주기 때문에 이렇게 표시되는 것이다.
즉 17번 스레드가 가지고 있는 잠금을 해제하고, 18번 스레드가 그 잠금을 획득하고 UPDATE를 완료한 후 잠금을 풀어야만 비로소 19번 스레드가 UPDATE를 실행할 수 있음을 의미한다.
여기서 17번 스레드가 어떤 잠금을 가지고 있는지 더 상세히 확인하고 싶다면 다음과 같이 performance_schema의 data_locks 테이블이 가진 칼럼을 모두 살펴보면 된다.

```
mysql> SELECT * FROM performance_schema.data_locks\G
************************** 1. row **************************
                ENGINE: INNODB
        ENGINE_LOCK_ID: 4828335432:1157:140695376728800
 ENGINE_TRANSACTION_ID: 11984
             THREAD_ID: 61
              EVENT_ID: 16028
         OBJECT_SCHEMA: employees
           OBJECT_NAME: employees
        PARTITION_NAME: NULL
     SUBPARTITION_NAME: NULL
            INDEX_NAME: NULL
 OBJECT_INSTANCE_BEGIN: 140695376728800
             LOCK_TYPE: TABLE
             LOCK_MODE: IX
           LOCK_STATUS: GRANTED
             LOCK_DATA: NULL
************************** 2. row **************************
                ENGINE: INNODB
        ENGINE_LOCK_ID: 4828335432:8:298:25:140695394434080
 ENGINE_TRANSACTION_ID: 11984
             THREAD_ID: 61
              EVENT_ID: 16048
         OBJECT_SCHEMA: employees
           OBJECT_NAME: employees
        PARTITION_NAME: NULL
     SUBPARTITION_NAME: NULL
            INDEX_NAME: PRIMARY
 OBJECT_INSTANCE_BEGIN: 140695394434080
             LOCK_TYPE: RECORD
             LOCK_MODE: X,REC_NOT_GAP
           LOCK_STATUS: GRANTED
             LOCK_DATA: 10001
```

결과를 보면 employees 테이블에 대해 IX 잠금(Intentional Exclusive)을 가지고 있으며, employees 테이블의 특정 레코드에 대해서 쓰기 잠금을 가지고 있다는 것을 확인할 수 있다.
이때 REC_NOT_GAP 표시가 있으므로 레코드 잠금은 갭이 포함되지 않은 순수 레코드에 대해서만 잠금을 가지고 있음을 알 수 있다.

만약 이 상황에서 17번 스레드가 잠금을 가진 상태에서 상당히 오랜 시간 멈춰 있다면 다음과 같이 17번 스레드를 강제 종료하면 나머지 UPDATE 명령들이 진행되면서 잠금 경합이 끝날 것이다.

```
mysql> KILL 17;
```