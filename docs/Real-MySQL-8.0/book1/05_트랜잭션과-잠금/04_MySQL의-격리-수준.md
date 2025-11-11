# [CH 5-4] MySQL의 격리 수준

트랜잭션의 격리 수준(isolation level)이란 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것이다.
격리 수준은 크게 "READ UNCOMMITTED", "READ COMMITTED", "REPEATABLE READ", "SERIALIZABLE"의 4가지로 나뉜다.
"DIRTY READ"라고도 하는 READ UNCOMMITTED는 일반적인 데이터베이스에서는 거의 사용하지 않고, SERIALIZABLE 또한 동시성이 중요한 데이터베이스에서는 거의 사용되지 않는다.
4개의 격리 수준에서 순서대로 뒤로 갈수록 각 트랜잭션 간의 데이터 격리(고립) 정도가 높아지며, 동시 처리 성능도 떨어지는 것이 일반적이라고 볼 수 있다.
격리 수준이 높아질수록 MySQL 서버의 처리 성능이 많이 떨어질 것으로 생각하는 사용자가 많은데, 사실 SERIALIZABLE 격리 수준이 아니라면 크게 성능의 개선이나 저하는 발생하지 않는다.

데이터베이스의 격리 수준을 이야기하면 항상 함께 언급되는 세 가지 부정합의 문제점이 있다. 이 세 가지 부정합의 문제는 격리 수준의 레벨에 따라 발생할 수도 있고 발생하지 않을 수도 있다.

||DIRTY READ|NON-REPEATABLE READ|PHANTOM READ|
|:---|:---|:---|:---|
|**READ UNCOMMITTED**|발생|발생|발생|
|**READ COMMITTED**|없음|발생|발생|
|**REPEATABLE READ**|없음|없음|발생<br/>(InnoDB는 없음)|
|**SERIALIZABLE**|없음|없음|없음|

SQL-92 또는 SQL-99 표준에 따르면 REPEATABLE READ 격리 수준에서는 PHANTOM READ가 발생할 수 있지만, InnoDB에서는 독특한 특성 때문에 REPEATABLE READ 격리 수준에서도 PHANTOM
READ가 발생하지 않는다. 일반적인 온라인 서비스 용도의 데이터베이스는 READ COMMITTED와 REPEATABLE READ 중 하나를 사용한다.
오라클 같은 DBMS는 주로 READ COMMITTED 수준을 많이 사용하며, MySQL에서는 REPEATABLE READ를 주로 사용한다.
여기서 설명하는 SQL 예제는 모두 AUTOCOMMIT이 OFF인 상태(SET autocommit=OFF)에서만 테스트할 수 있다.

---
<br/>

## (1) READ UNCOMMITTED
READ UNCOMMITTED 격리 수준에서는 아래와 같이 각 트랜잭션에서의 변경 내용이 COMMIT이나 ROLLBACK 여부와 상관없이 다른 트랜잭션에서 보인다.
다른 트랜잭션이 사용자 B가 실행하는 SELECT 쿼리의 결과에 어떤 영향을 미치는지 보여주는 예제다.

#### [그림 5.3] READ UNCOMMITTED
<img src="https://github.com/user-attachments/assets/d450ff5c-2eaa-4887-83a7-038666297046" width="430"/><br/>

사용자 A는 emp_nork 500000이고 first_name이 "Lara"인 새로운 사원을 INSERT한다. 사용자 A가 변경된 내용을 커밋하기도 전에 사용자 B는 emp_no=500000인 사원을 검색하고 있다.
하지만 사용자 B는 사용자 A가 INSERT한 사원의 정보를 커밋되지 않은 상태에서도 조회할 수 있다.
그런데 문제는 사용자 A가 처리 도중 알 수 없는 문제가 발생해 INSERT된 내용을 롤백한다고 하더라도 여전히 사용자 B는 "Lara"가 정상적인 사원이라고 생각하고 계속 처리할 것이라는 점이다.

이처럼 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상을 더티 리드(Dirty read)라 하고, 더티 리드가 허용되는 격리 수준이 READ UNCOMMITTED이다.
더티 리드 현상은 데이터가 나타났다가 사라졌다 하는 현상을 초래하므로 애플리케이션 개발자와 사용자를 상당히 혼란스럽게 만들 것이다.
또한 더티 리드를 유발하는 READ UNCOMMITTED는 RDBMS 표준에서는 트랜잭션의 격리 수준으로 인정하지 않을 정도로 정합성에 문제가 많은 격리 수준이다.
MySQL을 사용한다면 최소한 READ COMMITTED 이상의 격리 수준을 사용할 것을 권장한다.
<br/>
<br/>
## (2) READ COMMITTED
READ COMMITTED는 오라클 DBMS에서 기본으로 사용되는 격리 수준이며, 온라인 서비스에서 가장 많이 선택되는 격리 수준이다.
이 레벨에서는 위에서 언급한 더티 리드(Dirty read) 같은 현상은 발생하지 않는다. 어떤 트랜잭션에서 데이터를 변경했더라도 COMMIT이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있기 때문이다.
아래는 READ COMMITTED 격리 수준에서 사용자 A가 변경한 내용이 사용자 B에게 어떻게 조회되는지 보여준다.

#### [그림 5.4] READ COMMITTED
<img src="https://github.com/user-attachments/assets/d0ce6f00-54bf-4595-b9f8-a54b826bcba1" width="430"/><br/>

사용자 A는 emp_no=500000인 사원의 first_name을 "Lara"에서 "Toto"로 변경했는데, 이 때 새로운 값인 "Toto"는 employees 테이블에 즉시 기록되고 이전 값인 "Lara"는 언두 영역으로
백업된다. 사용자 A가 커밋을 수행하기 전에 사용자 B가 emp_no=500000인 사원을 SELECT하면 조회된 결과의 first_name 칼럼의 값은 "Toto"가 아니라 "Lara"로 조회된다.
여기서 사용자 B의 SELECT 쿼리 결과는 employees 테이블이 아니라 언두 영역에 백업된 레코드에서 가져온 것이다.
READ COMMITTED 격리 수준에서는 어떤 트랜잭션에서 변경한 내용이 커밋되기 전까지는 다른 트랜잭션에서 그러한 변경 내역을 조회할 수 없기 때문이다.
최종적으로 사용자 A가 변경된 내용을 커밋하면 그때부터는 다른 트랜잭션에서도 백업된 언두 레코드("Lara")가 아니라 새롭게 변경된 "Toto"라는 값을 참조할 수 있게 된다.

READ COMMITTED 격리 수준에서도 "NON-REPEATABLE READ"("REPEATABLE READ"가 불가능하다)라는 부정합의 문제가 있다.
아래는 "NON-REPEATABLE READ"가 왜 발생하고 어떤 문제를 만들어낼 수 있는지 보여준다.

#### [그림 5.5] NON-REPEATABLE READ
<img src="https://github.com/user-attachments/assets/aff01148-e079-4ed2-8c60-4bb24d2c12cb" width="430"/><br/>

처음 사용자 B가 BEGIN 명령으로 트랜잭션을 시작하고 first_name이 "Toto"인 사용자를 검색했는데, 일치하는 결과가 없었다.
하지만 사용자 A가 사원 번호가 500000인 사원의 이름을 "Toto"로 변경하고 커밋을 실행한 후, 사용자 B가 똑같은 SELECT 쿼리로 다시 조회하면 이번에는 결과가 1건이 조회된다.
이는 별다른 문제가 없어 보이지만, 사실 사용자 B가 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때는 항상 같은 결과를 가져와야 한다는 "REPEATABLE READ" 정합성에 어긋나는 것이다.

이러한 부정합 현상은 일반적인 웹 프로그램에서는 크게 문제되지 않을 수 있지만 하나의 트랜잭션에서 동일 데이터를 여러 번 읽고 변경하는 작업이 금전적인 처리와 연결되면 문제가 될 수도 있다.
예를 들어, 다른 트랜잭션에서 입금과 출금 처리가 계속 진행될 때 다른 트랜잭션에서 오늘 입금된 금액의 총합을 조회한다고 가정해보자.
그런데 "REPEATABLE READ"가 보장되지 않기 때문에 총합을 계산하는 SELECT 쿼리는 실행될 때마다 다른 결과를 가져올 것이다.
중요한 것은 사용 중인 트랜잭션의 격리 수준에 의해 실행하는 SQL 문장이 어떤 결과를 가져오게 되는지를 정확히 예측할 수 있어야 한다는 것이다.
그리고 당연히 이를 위해서는 각 트랜잭션의 격리 수준이 어떻게 작동하는지 알아야 한다.

가끔 사용자 중에서 트랜잭션 내에서 실행되는 SELECT 문장과 트랜잭션 없이 실행되는 SELECT 문장의 차이를 혼동하는 경우가 있다.
READ COMMITTED 격리 수준에서는 트랜잭션 내에서 실행되는 SELECT 문장과 트랜잭션 외부에서 실행되는 SELECT 문장의 차이가 별로 없다.
하지만 REPEATABLE READ 격리 수준에서는 기본적으로 SELECT 쿼리 문장도 트랜잭션 범위 내에서만 작동한다.
즉, START TRANSACTION(또는 BEGIN) 명령으로 트랜잭션을 시작한 상태에서 온종일 동일한 쿼리를 반복해서 실행해봐도 동일한 결과만 보게 된다
(아무리 다른 트랜잭션에서 그 데이터를 변경하고 COMMIT을 실행한다고 하더라도 말이다).
별로 중요하지 않은 차이처럼 보이지만 이런 문제로 데이터의 정합성이 깨지고 그로 인해 애플리케이션에 버그가 발생하면 찾아내기가 쉽지 않다.
<br/>
<br/>
## (3) REPEATABLE READ
REPEATABLE READ는 MySQL의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준이다. 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.
이 격리 수준에서는 READ COMMITTED 격리 수준에서 발생하는 "NON-REPEATABLE READ" 부정합이 발생하지 않는다.
InnoDB 스토리지 엔진은 트랜잭션이 ROLLBACK될 가능성에 대비해 변경되기 전 레코드를 언두(Undo) 공간에 백업해두고 실제 레코드 값을 변경한다.
이러한 변경 방식을 MVCC라고 하며, REPEATABLE READ는 이 MVCC를 위해 언두 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있게 보장한다.
사실 READ COMMITTED도 MVCC를 이용해 COMMIT되기 전의 데이터를 보여준다.
REPEATABLE READ와 READ COMMITTED의 차이는 언두 영역에 백업된 레코드의 여러 버전 가운데 몇 번째 이전 버전까지 찾아 들어가야 하느냐에 있다.

모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는 값)를 가지며, 언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 번호가 포함돼 있다.
그리고 언두 영역의 백업된 데이터는 InnoDB 스토리지 엔진이 불필요하다고 판단하는 시점에 주기적으로 삭제한다.
REPEATABLE READ 격리 수준에서는 MVCC를 보장하기 위해 실행 중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 언두 영역의 데이터는 삭제할 수가 없다.
그렇다고 가장 오래된 트랜잭션 번호 이전의 트랜잭션에 의해 변경된 모든 언두 데이터가 필요한 것은 아니다. 더 정확하게는 특정 트랜잭션 번호의 구간 내에서 백업된 언두 데이터가 보존돼야 한다.

아래는 REPEATABLE READ 격리 수준이 작동하는 방식을 보여준다. 우선 이 시나리오가 실행되기 전에 employees 테이블은 번호가 6인 트랜잭션에 의해 INSERT됐다고 가정하자.
그래서 employees 테이블의 초기 두 레코드는 트랜잭션 번호가 6으로 표현된 것이다.
사용자 A가 emp_no가 500000인 사원의 이름을 변경하는 과정에서 사용자 B가 emp_no=500000인 사원을 SELECT할 때 어떤 과정을 거쳐서 처리되는지 보여준다.

#### [그림 5.6] REPEATABLE READ
<img src="https://github.com/user-attachments/assets/f2806c25-f87f-43e9-8338-b2f90b5c691c" width="430"/><br/>

사용자 A의 트랜잭션 번호는 12였으며 사용자 B의 트랜잭션의 번호는 10이었다. 이때 사용자 A는 사원의 이름을 "Toto"로 변경하고 커밋을 수행한다.
그런데 A 트랜잭션이 변경을 수행하고 커밋을 했지만, 사용자 B가 emp_no=500000인 사원을 A 트랜잭션의 변경 전후 각각 한 번씩 SELECT했는데 결과는 항상 "Lara"라는 값을 가져온다.
사용자 B가 BEGIN 명령으로 트랜잭션을 시작하면서 10번이라는 트랜잭션 번호를 부여받았는데, 그때부터 사용자 B의 10번 트랜잭션 안에서 실행되는 모든 SELECT 쿼리는 트랜잭션 번호가 10(자신의
트랜잭션 번호)보다 작은 트랜잭션 번호에서 변경한 것만 보게 된다.

언두 영역에 백업된 데이터가 하나만 이쓴ㄴ 것으로 표현했지만 사실 하나의 레코드에 대해 백업이 하나 이상 얼마든지 존재할 수 있다.
한 사용자가 BEGIN으로 트랜잭션을 시작하고 장시간 트랜잭션을 종료하지 않으면 언두 영역이 백업된 데이터로 무한정 커질 수도 있다.
이렇게 언두에 백업된 레코드가 많아지면 MySQL 서버의 처리 성능이 떨어질 수 있다.

REPEATABLE READ 격리 수준에서도 다음과 같은 부정합이 발생할 수 있다.
아래 그림에서 사용자 A가 employees 테이블에 INSERT를 수행하는 도중에 사용자 B가 SELECT ... FOR UPDATE 쿼리로 employees 테이블을 조회했을 때 어떤 결과를 가져오는지 보여준다.

#### [그림 5.7] PHANTOM READ(PHANTOM ROWS)
<img src="https://github.com/user-attachments/assets/4285d7e2-fd65-4edd-9af2-1a2b02d34aea" width="430"/><br/>

사용자 B는 BEGIN 명령으로 트랜잭션을 시작한 후 SELECT를 수행한다. 그러므로 REPEATABLE READ에서 배운 것처럼 두 번의 SELECT 쿼리 결과는 똑같아야 한다.
하지만 사용자 B가 실행하는 두 번의 SELECT ... FOR UPDATE 쿼리 결과는 서로 다르다.
이렇게 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상을 PHANTOM READ(또는 PHANTOM ROWS)라고 한다.
SELECT ... FOR UPDATE 쿼리는 SELECT하는 레코드에 쓰기 잠금을 걸어야 하는데, 언두 레코드에는 잠금을 걸 수 없다.
그래서 SELECT ... FOR UPDATE나 SELECT ... LOCK IN SHARE MODE로 조회되는 레코드는 언두 영역의 변경 전 데이터를 가져오는 것이 아니라 현재 레코드의 값을 가져오게 되는 것이다.
<br/>
<br/>
## (4) SERIALIZABLE
가장 단순한 격리 수준이면서 동시에 가장 엄격한 격리 수준이다. 그만큼 동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어진다.
InnoDB 테이블에서 기본적으로 순수한 SELECT 작업(INSERT ... SELECT ... 또는 CREATE TABLE ... AS SELECT ...가 아닌)은 아무런 레코드 잠금도 설정하지 않고 실행된다.
InnoDB 매뉴얼에서 자주 나타나는 "Non-locking consistent read(잠금이 필요 없는 일관된 읽기)"라는 말이 이를 의미하는 것이다.
하지만 트랜잭션의 격리 수준이 SERIALIZABLE로 설정되면 읽기 작업도 공유 잠금(읽기 잠금)을 획득해야만 하며, 동시에 다른 트랜잭션은 그러한 레코드를 변경하지 못하게 된다.
즉, 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없는 것이다. SERIALIZABLE 격리 수준에서는 일반적인 DBMS에서 일어나는 "PHANTOM READ"라는 문제가 발생하지 않는다.
하지만 InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 이미 "PHANTOM READ"가 발생하지 않기 때문에 굳이 SERIALIZABLE을 사용할 필요성은 없어
보인다.(엄밀하게는 "SELECT ... FOR UPDATE" 또는 "SELECT ... FOR SHARE" 쿼리의 경우 REPEATABLE READ 격리 수준에서도 PHANTOM READ 현상이 발생할 수 있다.
하지만 레코드의 변경 이력(언두 레코드)에 잠금을 걸 수는 없기 때문에 이러한 잠금을 동반한 SELECT 쿼리는 예외적인 상황으로 볼 수 있다.)