# [CH 15-4] ENUM과 SET

ENUM과 SET은 모두 문자열 값을 MySQL 내부적으로 숫자 값으로 매핑해서 관리하는 타입이다.
일반적으로 데이터베이스를 사용하다 보면 타입이나 상태 등과 같이 수많은 코드 형태의 칼럼을 사용하게 되는데, 실제 데이터베이스에는 이미 인코딩된 알파벳이나 숫자 값만 저장되므로 그 의미를 바로 파악하기가
쉽지 않다는 단점이 있다.

---
<br/>

## (1) ENUM
ENUM 타입은 테이블의 구조(메타 데이터)에 나열된 목록 중 하나의 값을 가질 수 있다. ENUM 타입의 가장 큰 용도는 코드화된 값을 관리하는 것이다. 다음 예제로 ENUM 타입의 특성을 한 번 살펴보자.

```
mysql> CREATE TABLE tb_enum ( fd_enum ENUM('PROCESSING', 'FAILURE', 'SUCCESS') );
mysql> INSERT INTO tb_enum VALUES ('PROCESSING'), ('FAILURE');
mysql> SELECT * FROM tb_enum;
+------------+
| fd_enum    |
+------------+
| PROCESSING |
| FAILURE    |
+------------+

-- // ENUM이나 SET 타입의 칼럼에 대해 숫자 연산을 수행하면
-- // 매핑된 문자열 값이 아닌 내부적으로 저장된 숫자 값으로 연산이 실행된다.
mysql> SELECT fd_enum*1 AS fd_enum_real_value FROM tb_enum;
+--------------------+
| fd_enum_real_value |
+--------------------+
|                  1 |
|                  2 |
+--------------------+

mysql> SELECT * FROM tb_enum WHERE fd_enum=1;
+------------+
| fd_enum    |
+------------+
| PROCESSING |
+------------+

mysql> SELECT * FROM tb_enum WHERE fd_enum='PROCESSING';
+------------+
| fd_enum    |
+------------+
| PROCESSING |
+------------+
```

ENUM 타입의 fd_enum 칼럼을 가지는 테이블을 생성하고, 예제로 2건의 레코드를 INSERT했다.
여기서 만들어진 fd_enum 칼럼은 값으로 'PROCESSING'과 'FAILURE', 'SUCCESS'를 가질 수 있게 정의됐다.
ENUM 타입은 INSERT나 UPDATE, SELECT 등의 쿼리에서 CHAR나 VARCHAR 타입과 같이 문자열처럼 비교하거나 저장할 수 있다.
하지만 MySQL 서버가 실제로 값을 디스크나 메모리에 저장할 때는 사용자로부터 요청된 문자열이 아니라 그 값에 매핑된 정숫값을 사용한다.
ENUM 타입에 사용할 수 있는 최대 아이템의 개수는 65,535개이며, 아이템의 개수가 255개 미만이면 ENUM 타입은 저장 공간으로 1바이트를 사용하고, 그 이상인 경우에는 2바이트까지 사용한다.

ENUM 타입을 사용할 때 일반적으로 특정 문자열 값이 어떤 정숫값으로 매핑됐는지는 알 필요가 없다.
하지만 필요하다면 위 예제의 두 번째 SELECT 쿼리에서와 같이 1을 곱한다거나 0을 더하는 산술 연산을 적용하는 방법으로 ENUM 타입의 실제 값을 확인할 수 있다.
ENUM 타입에서 매핑되는 정숫값은 일반적으로 테이블 정의에 나열된 문자열 순서대로 1부터 할당되며, 빈 문자열("")은 항상 0으로 매핑된다.
프로그램의 성격에 따라 다르겠지만 MySQL을 사용하는 프로그램에서는 별도의 코드 테이블을 사용하지 않을 때가 많다.
이때 실제 테이블에 저장된 코드 값이 어떤 의미인지 이해하기가 쉽지 않은데, ENUM 타입은 이러한 단점을 보완할 수 있는 상당히 유용한 타입이라고 볼 수 있다.
ENUM 타입은 저장해야 하는 아이템 값(문자열)이 길면 길수록 저장 공간을 더 많이 절약할 수 있다.

하지만 ENUM 타입의 가장 큰 단점은 칼럼에 저장되는 문자열 값이 테이블의 구조(메타 정보)가 되면서 기존 ENUM 타입에 새로운 값을 추가해야 한다면(예를 들어, 'REFUND'를 추가해야 하는 경우) 테이블의
구조를 변경해야 한다는 점이다. 예전 버전의 MySQL 서버에서는 ENUM 타입의 아이템이 새로 추가되면 항상 테이블을 리빌드해야 했다. 이러한 문제로 인해 MySQL 서버에서 ENUM 타입은 별로 사용되지 않았다.
하지만 MySQL 5.6 버전부터는 새로 추가하는 아이템이 ENUM 타입의 제일 마지막으로 추가되는 형태라면 테이블의 구조(메타데이터) 변경만으로 즉시 완료된다.

```
mysql> ALTER TABLE tb_enum
         MODIFY fd_enum ENUM('PROCESSING', 'FAILURE', 'SUCCESS', 'REFUND')
           ALGORITHM=INSTANT;

mysql> ALTER TABLE tb_enum
         MODIFY fd_enum ENUM('PROCESSING', 'FAILURE', 'REFUND', 'SUCCESS')
           ALGORITHM=COPY, LOCK=SHARED;
```

위의 예제와 같이 기존 ENUM('PROCESSING', 'FAILURE', 'SUCCESS') 타입의 마지막에 새로운 아이템 'REFUND'를 추가하는 작업은 INSTANT 알고리즘으로 메타데이터 변경만으로 완료된다는 것을 알
수 있다. 하지만 기존 ENUM 타입의 아이템들이 순서가 변경되거나 중간에 새로운 아이템이 추가되는 경우에는 COPY 알고리즘에 읽기 잠금까지 필요하다.
때로는 ENUM 타입에 저장되는 아이템의 순서상 새로운 아이템을 중간에 추가하고 싶을 때도 있다.
하지만 테이블이 매우 크다면 가독성이 좀 떨어지더라도 새로운 아이템을 ENUM 타입의 마지막에 추가하는 것이 MySQL 서버의 가용성을 높이는 방법이다.

ENUM 타입은 우리가 일반적으로 사용하는 상태나 카테고리와 같이 코드화된 칼럼을 MySQL이 자체적으로 제공하는 기능이다.
그래서 ENUM 타입의 칼럼값으로 정렬을 수행하면 매핑되기 전의 문자열 값 기준으로 정렬되는 것이 아니라 매핑된 코드 값으로 정렬이 수행된다.
ENUM 타입은 마치 CHAR나 VARCHAR와 같은 문자열 타입처럼 보이지만 사실은 정수 타입의 칼럼이기 때문이다.
가장 좋은 방법은 ENUM 타입의 칼럼에 대해서는 정렬을 수행하지 않는 것이 가장 좋겠지만 꼭 ENUM 타입의 인코딩된 값이 아니라 문자열 기준으로 정렬해야 한다면 테이블을 생성할 때 필요한 정렬 기준으로
ENUM 타입의 문자열 값을 나열하면 된다. 이미 만들어진 테이블의 ENUM 타입의 문자열 값으로 강제 정렬을 해야 한다면 다음 예제와 같이 CAST() 함수를 이용해 문자열 타입으로 변환해 정렬할 수밖에 없다.
이때 인덱스를 이용한 정렬을 사용할 수 없으므로 주의해서 사용해야 한다.

```
mysql> SELECT fd_enum*1 AS real_value, fd_enum FROM tb_enum
       ORDER BY fd_enum;
+------------+------------+
| real_value | fd_enum    |
+------------+------------+
|          1 | PROCESSING |
|          2 | FAILURE    |
+------------+------------+

mysql> SELECT fd_enum*1 AS real_value, fd_enum FROM tb_enum
       ORDER BY CAST(fd_enum AS CHAR);
+------------+------------+
| real_value | fd_enum    |
+------------+------------+
|          2 | FAILURE    |
|          1 | PROCESSING |
+------------+------------+
```

ENUM 타입은 테이블 구조에 정의된 코드 값만 사용할 수 있게 강제한다는 장점도 있지만, 더 큰 장점은 데이터베이스 서버의 디스크 저장 공간의 크기를 줄여준다는 점이다.
테이블의 레코드 건수가 많지 않다면 디스크의 사용량은 큰 장점이 아닐 수도 있다.
하지만 레코드가 억 단위를 넘어간다면 10\~20글자를 넘어서는 문자열 칼럼보다 ENUM 타입이 매우 작은 1\~2GB의 저장 공간을 줄일 수 있다.
이 칼럼이 여러 개의 인덱스에 사용된다면 용량을 몇 배로 줄이는 효과를 얻을 수 있다.

요즘 시중에 판매되는 디스크 용량이 얼마나 큰데 1\~2GB 용량 줄이는 것이 무슨 큰 효과일까라고 생각할 수도 있다. 디스크의 데이터는 InnoDB 버퍼 풀로 적재돼야 쿼리에서 비로소 사용할 수 있다.
디스크의 데이터가 크다는 것은 메모리도 그만큼 많이 필요해진다는 이야기다. 이제는 ENUM 타입의 장점이 얼마나 큰지 이해됐을 것이다.
하지만 메모리 사용량 절감 효과를 빼더라도 디스크의 사용량이 적다면 백업이나 복구 시간을 줄일 수 있다는 장점도 크다.
당장 장애가 발생했는데, 백업 파일을 복사하는 데 3\~4시간이 걸린다면 이 시간 동안은 서비스가 불가능해지는 것이다.
그뿐만 아니라 디스크의 데이터 파일 크기가 작다면 스키마를 변경하는 시간과 인덱스를 생성하는 시간도 줄어든다.

ENUM 타입을 떠나서라도 가능하면 디스크의 데이터 파일 크기는 줄이는 것이 성능과 운영에 많은 도움이 될 것이다.
<br/>
<br/>
## (2) SET
SET 타입도 테이블의 구조에 정의된 아이템을 정숫값으로 매핑해서 저장하는 방식은 똑같다. SET과 ENUM의 가장 큰 차이는 SET은 하나의 칼럼에 1개 이상의 값을 저장할 수 있다는 점이다.
MySQL 서버는 내부적으로 BIT-OR 연산을 거쳐 1개 이상의 선택된 값을 저장한다. 즉, SET 타입의 칼럼은 여러 개의 값을 저장할 수는 있지만 실제 여러 개의 값을 저장하는 공간을 가지는 것이 아니다.
그래서 각 아이템 값에 매핑되는 정숫값은 1씩 증가하는 정숫값이 아니라 2n의 값을 갖게 된다.
SET 타입은 아이템 값의 멤버 수가 8개 이하이면 1바이트의 저장 공간을 사용하며, 9개에서 16개 이하이면 2바이트를 사용하고 똑같은 방식으로 최대 8바이트까지 저장 공간을 사용한다.
간단히 SET 타입을 정의하고 사용하는 방법을 다음 예제로 살펴보자.

```
mysql> CREATE TABLE tb_set (
         fd_set SET('TENNIS','SOCCER','GOLF','TABLE-TENNIS','BASKETBALL','BILLIARD')
       );
mysql> INSERT INTO tb_set (fd_set) VALUES ('SOCCER'), ('GOLF,TENNIS');

mysql> SELECT * FROM tb_set;
+-------------+
| fd_set      |
+-------------+
| SOCCER      |
| TENNIS,GOLF |
+-------------+

mysql> SELECT * FROM tb_set WHERE FIND_IN_SET('GOLF', fd_set);
+-------------+
| fd_set      |
+-------------+
| TENNIS,GOLF |
+-------------+

mysql> SELECT * FROM tb_set WHERE fd_set LIKE '%GOLF%';
+-------------+
| fd_set      |
+-------------+
| TENNIS,GOLF |
+-------------+
```

위의 예제에서 첫 번째 INSERT 문장은 "SOCCER"라는 하나의 값만 저장하거나 "GOLF"와 "TENNIS"라는 두 개의 값을 하나의 칼럼에 저장하는 방법을 보여준다.
여러 개의 값을 하나의 SET 타입 칼럼에 저장할 때는 ","로 구분해서 문자열 값을 나열해서 입력하면 된다. 그리고 SELECT 쿼리의 결과에서도 똑같이 ","를 구분자로 해서 연결된 문자열을 반환한다.
SET 타입의 칼럼에서 "GOLF"라는 문자열 멤버를 가진 레코드를 검색해야 할 때는 두 번째나 세 번째의 SELECT 쿼리에서와 같이 FIND_IN_SET() 함수나 LIKE 검색을 이용할 수 있다.

SET 타입의 칼럼에 대해 동등 비교(Equal)를 수행하려면 칼럼에 저장된 순서대로 문자열을 나열해야만 검색할 수 있다.
또한 SET 타입의 칼럼에 인덱스가 있더라도 동등 비교 조건을 제외하고 FIND_IN_SET() 함수나 LIKE를 사용하는 쿼리는 인덱스를 사용할 수 없다.

```
mysql> SELECT * FROM tb_test WHERE fd_set='TENNIS,GOLF';
+-------------+
| fd_set      |
+-------------+
| TENNIS,GOLF |
+-------------+

mysql> SELECT * FROM tb_test WHERE fd_set='GOLF,TENNIS';
Empty set (0.00 sec)
```

동시에 여러 개의 값을 갖는 SET 타입의 칼럼에 대해 하나의 특정 값을 포함하고 있는지는 다음과 같이 FIND_IN_SET() 함수를 사용하면 된다.

```
mysql> SELECT * FROM tb_set WHERE FIND_IN_SET('TENNIS', fd_set) >= 1;
+-------------+
| fd_set      |
+-------------+
| TENNIS,GOLF |
+-------------+
```

하지만 위 예제와 같이 FIND_IN_SET() 함수의 사용은 fd_set 칼럼에 인덱스가 있어도 효율적으로 해당 인덱스를 이용할 수 없다.
이러한 형태의 검색이 빈번히 사용된다면 SET 타입의 칼럼을 정규화해서 별도로 인덱스를 가진 자식 테이블을 생성하는 것이 좋다.

ENUM과 마찬가지로 SET 타입 또한 기존 SET 타입에 정의된 아이템 중간에 새로운 아이템을 추가하는 경우 테이블의 읽기 잠금과 리빌드 작업이 필요하다.

```
mysql> ALTER TABLE tb_set
         MODIFY fd_set SET('TENNIS','SOCCER','GOLF','TABLE-TENNIS',
                           'BASKETBALL','e-SPORTS','BILLIARD'),
         ALGORITHM=COPY, LOCK=SHARED;
```

하지만 SET 타입의 마지막에 새로운 아이템을 추가하는 작업은 ENUM과 동일하게 INSTANT 알고리즘으로 메타 정보만 변경하고 즉시 완료된다.
하지만 SET 타입의 아이템 개수가 8개를 넘어서서 9개로 바뀔 때는 읽기 잠금과 테이블 리빌드 작업이 필요하다. 이는 SET 타입을 저장하기 위한 공간이 1바이트에서 2바이트로 변경돼야 하기 때문이다.

```
mysql> ALTER TABLE tb_set
         MODIFY fd_set SET('TENNIS','SOCCER','GOLF','TABLE-TENNIS',
                           'BASKETBALL','BILLIARD','e-SPORTS'),
         ALGORITHM=INSTANT;

mysql> ALTER TABLE tb_set
         MODIFY fd_set SET('TENNIS','SOCCER','GOLF','TABLE-TENNIS',
                           'BASKETBALL','BILLIARD','e-SPORTS','SCUBA-DIVING','SWIMMING'),
         ALGORITHM=INSTANT;
ERROR 1846 (0A000): ALGORITHM=INSTANT is not supported. Reason: Cannot change column type
INPLACE. Try ALGORITHM=COPY/INPLACE.
```