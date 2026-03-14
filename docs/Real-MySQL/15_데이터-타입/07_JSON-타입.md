# [CH 15-7] JSON 타입

MySQL 5.7 버전부터 JSON 데이터를 저장할 수 있는 JSON 타입이 지원되기 시작했으며, MySQL 8.0 버전으로 업그레이드되면서 많은 기능과 성능 개선 사항이 추가됐다.
물론 MySQL 서버에서 TEXT 칼럼이나 BLOB 칼럼에 JSON 데이터를 저장할 수도 있었다.
하지만 MySQL 5.7 버전부터 지원되는 JSON 타입의 칼럼은 JSON 데이터를 문자열로 저장하는 것이 아니라 MongoDB와 같이 바이너리 포맷의 BSON(Binary JSON)으로 변환해서 저장한다.

---
<br/>

## (1) 저장 방식
MySQL 서버는 내부적으로 JSON 타입의 값을 BLOB 타입에 저장한다. 하지만 JSON 칼럼에 저장되는 값은 사용자가 입력한 값 그대로 저장하는 것이 아니라 바이너리 포맷인 BSON 타입으로 변환해서 저장한다.
그래서 JSON 데이터를 BLOB이나 TEXT 타입의 칼럼에 저장하는 것보다 공간 효율이 높은 편이다.

다음은 JSON 칼럼의 값이 이진 포맷으로 변환됐을 때 길이가 몇 바이트인지 확인하는 예제다.

```
mysql> CREATE TABLE tb_json (id INT, fd JSON);
mysql> INSERT INTO tb_json VALUES
         (1, '{"user_id":1234567890}'),
         (2, '{"user_id":"1234567890"}');

mysql> SELECT id, fd,
         JSON_TYPE(fd->"$.user_id") AS field_type,
         JSON_STORAGE_SIZE(fd) AS byte_size
       FROM tb_json;
+------+---------------------------+------------+-----------+
| id   | fd                        | field_type | byte_size |
+------+---------------------------+------------+-----------+
|    1 | {"user_id": 1234567890}   | INTEGER    |        23 |
|    2 | {"user_id": "1234567890"} | STRING     |        30 |
+------+---------------------------+------------+-----------+
```

첫 번째 레코드는 user_id 필드의 값을 정수 타입으로 저장했으며, 두 번째 레코드는 user_id 필드의 값을 문자열 타입으로 저장했다.
그 결과 두 레코드의 JSON 값을 이진 포맷으로 변환하면 7바이트의 공간 차이가 발생했다.

이제 다음 예제를 통해 JSON 도큐먼트가 어떻게 이진 데이터로 변환되어 저장되는지를 한 번 살펴보자.

```
-- // JSON 도큐먼트 { "a": "x", "b": "y", "c": "z" }
mysql> SELECT JSON_STORAGE_SIZE('{ "a": "x", "b": "y", "c": "z" }') AS binary_length;
+---------------+
| binary_length |
+---------------+
|            35 |
+---------------+
```

위의 JSON 도큐먼트가 MySQL 서버의 JSON 칼럼에 저장될 때는 다음과 같이 35바이트의 이진 데이터로 저장된다.

```
00 03 00 22 00 19 00 01 00 1A
00 01 00 1B 00 01 00 0C 1C 00
0C 1E 00 0C 20 00 61 62 63 01
78 01 79 01 7A
```

이진 데이터는 다음과 같이 24개의 필드로 구성되는데, 각 필드는 저장되는 값의 특성에 맞게 1개 이상의 바이트를 차지한다.
위의 예제는 단순한 JSON 도큐먼트이며, 각 필드의 키나 값의 길이가 짧아서 모든 필드가 1\~2바이트만 사용하고 있다는 것을 알 수 있다.

#### [이진 포맷 JSON 데이터 필드]
|필드 순서|바이트 수|주소(Offset)|이진값|문자열|십진 숫자값|설명|
|---:|---:|---:|:---:|:---:|:---:|:---|
|1|1||00 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||0|type(JSONB_TYPE_SMALL_OBJECT)|
|2|2|0|03 \| 00||3|JSON 어트리뷰트 개수|
|3|2|2|22 \| 00||34|JSON 도큐먼트 길이(바이트 수)|
|4|2|4|19 \| 00||25|첫 번째 키 주소(Offset)|
|5|2|6|01 \| 00||1|첫 번째 키 길이(바이트 수)|
|6|2|8|1A \| 00||26|두 번째 키 주소(Offset)|
|7|2|10|01 \| 00||1|두 번째 키 길이(바이트 수)|
|8|2|12|1B \| 00||27|세 번째 키 주소(Offset)|
|9|2|14|01 \| 00||1|세 번째 키 길이(바이트 수)|
|10|1|16|0C \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||12|첫 번째 값 타입(JSONB_TYPE_STRING)|
|11|2|17|1C \| 00||28|첫 번째 값 주소(Offset)|
|12|1|19|0C \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||12|두 번째 값 타입(JSONB_TYPE_STRING)|
|13|2|20|1E \| 00||30|두 번째 값 주소(Offset)|
|14|1|22|0C \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||12|세 번째 값 타입(JSONB_TYPE_STRING)|
|15|2|23|20 \| 00||32|세 번째 값 주소(Offset)|
|16|1|25|61 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||a||첫 번째 키|
|17|1|26|62 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||b||두 번째 키|
|18|1|27|63 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||c||세 번째 키|
|19|1|28|01 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||1|첫 번째 값 길이(바이트 수)|
|20|1|29|78 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|x||첫 번째 값|
|21|1|30|01 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||1|두 번째 값 길이(바이트 수)|
|22|1|31|79 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|y||두 번째 값|
|23|1|32|01 \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;||1|세 번째 값 길이(바이트 수)|
|24|1|33|7A \|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|z||세 번째 값|

위 표에서 "이진값"으로 표시된 항목이 실제 바이너리 필드 값이며, 각 필드 값의 순서와 길이, 주소(Offset)를 표시했다. 그리고 각 필드가 어떤 값을 저장하고 있는지도 함께 설명으로 추가해뒀다.
"주소"는 JSON 도큐먼트에서 첫 번째 1바이트를 제외하고 난 다음, 이진 데이터에서 각 필드의 오프셋(Offset)을 의미한다.
중요한 것은 JSON 도큐먼트를 구성하는 모든 키의 위치와 키의 이름이 각 JSON 필드의 값보다 먼저 나열돼 있다는 것이다.
그래서 JSON 칼럼의 특정 필드만 참조하거나 특정 필드의 값만 업데이트(길이가 변경되지 않은 부분 업데이트)하는 경우 JSON 칼럼의 값을 모두 읽어보지 않고도 즉시 원하는 필드의 이름을 읽거나 변경할 수
있다.

MySQL 서버에서 매우 큰 용량의 JSON 도큐먼트가 저장되면 MySQL 서버는 16KB 단위로 여러 개의 데이터 페이지로 나뉘어 저장된다.
이때 대용량 BLOB 데이터(MySQL 서버의 JSON 칼럼은 내부적으로 BLOB 타입을 사용하므로 BLOB 타입의 모든 기능을 활용하게 된다.)는 여러 개의 BLOB 페이지로 나뉘어 저장되는데, MySQL 5.7
버전까지는 BLOB 페이지들이 단순 링크드 리스트(Linked List)처럼 관리됐다.
하지만 이러한 형태는 JSON 필드의 부분 업데이트를 효율적으로 처리할 수 없기 때문에 MySQL 8.0 버전부터는 BLOB 페이지들의 인덱스를 관리하고 각 인덱스는 실제 BLOB 데이터를 가진 페이지들의 링크를
갖도록 개선됐다. JSON 필드의 부분 업데이트가 필요한 경우 MySQL 서버는 BLOB 페이지 인덱스와 JSON 칼럼의 각 필드 주소 정보를 이용해 변경이 필요한 부분만 업데이트할 수 있게 됐다.
<br/>
<br/>
## (2) 부분 업데이트 성능
MySQL 8.0 버전부터는 JSON 타입에 대해 부분 업데이트(Partial Update) 기능을 제공한다.
JSON 칼럼의 부분 업데이트 기능은 JSON_SET()과 JSON_REPLACE(), JSON_REMOVE() 함수를 이용해 JSON 도큐먼트의 특정 필드 값을 변경하거나 삭제하는 경우에만 작동한다.
다음은 두 번째 레코드의 JSON 칼럼의 값에서 "1234567890"이었던 "user_id" 필드의 값을 "12345"로 변경하는 예제다.

```
mysql> UPDATE tb_json
         SET fd=JSON_SET(fd, '$.user_id', "12345")
       WHERE id=2;

mysql> SELECT id, fd, JSON_STORAGE_SIZE(fd), JSON_STORAGE_FREE(fd)
       FROM tb_json;
+------+-------------------------+-----------------------+-----------------------+
| id   | fd                      | JSON_STORAGE_SIZE(fd) | JSON_STORAGE_FREE(fd) |
+------+-------------------------+-----------------------+-----------------------+
|    1 | {"user_id": 1234567890} |                    23 |                     0 |
|    2 | {"user_id": "12345"}    |                    30 |                     5 |
+------+-------------------------+-----------------------+-----------------------+
```

JSON_SET() 함수를 이용한 JSON 칼럼의 "user_id" 필드 값 변경 작업이 "부분 업데이트"로 처리됐는지 확인할 수 있는 명확한 방법은 없다.
하지만 JSON_STORAGE_SIZE() 함수와 JSON_STORAGE_FREE() 함수를 이용하면 대략 예측할 수 있다.
위의 예제에서 첫 번째 레코드의 JSON_STORAGE_FREE() 함수 결괏값은 0인 반면, 두 번째 레코드는 JSON_STORAGE_FREE() 함수의 결괏값이 5로 표시됐다.
이는 "user_id" 필드의 값이 10바이트를 차지하고 있다가 "12345"로 변경되면서 앞 부분 5바이트만 사용하고 나머지 5바이트는 비워 뒀기 때문이다.
그래서 JSON_STORAGE_SIZE() 함수의 결괏값은 변하지 않았지만 JSON_STORAGE_FREE() 값은 5가 표시된 것이다.
예제의 UPDATE 명령 실행 후 실제 JSON 칼럼이 사용하는 디스크의 저장 공간 자체는 줄어들지 않았지만 기존 JSON 칼럼이 사용하던 전체 공간에서 빈 공간이 5바이트가 생긴 것이다.

그러면 이제 "user_id" 필드의 값을 10바이트 이상인 값으로 변경한 후 JSON_STORAGE_SIZE()와 JSON_STORAGE_FREE() 값을 확인해보자.

```
mysql> UPDATE tb_json SET fd=JSON_SET(fd, '$.user_id', "12345678901") WHERE id=2;

mysql> SELECT id, fd, JSON_STORAGE_SIZE(fd), JSON_STORAGE_FREE(fd) FROM tb_json;
+------+----------------------------+-----------------------+-----------------------+
| id   | fd                         | JSON_STORAGE_SIZE(fd) | JSON_STORAGE_FREE(fd) |
+------+----------------------------+-----------------------+-----------------------+
|    1 | {"user_id": 1234567890}    |                    23 |                     0 |
|    2 | {"user_id": "12345678901"} |                    31 |                     0 |
+------+----------------------------+-----------------------+-----------------------+
```

최초 두 번째 레코드가 INSERT되던 시점에 JSON 칼럼에서 "user_id" 필드가 사용했던 공간의 크기가 10바이트인데, "user_id" 필드의 값을 11바이트 문자열로 업데이트했다.
JSON_SET() 함수를 이용해 업데이트했지만 이번에는 부분 업데이트 방식으로 처리되지 못했다.
이는 최초 할당됐던 10바이트 공간으로 부족하기 때문에 MySQL 서버가 JSON 칼럼 또는 두 번째 레코드를 통째로 다른 위치로 복사해서 저장한 것이다.
그러면서 JSON_STORAGE_FREE() 함수의 결괏값도 다시 0으로 초기화됐다.

JSON 칼럼의 전체 업데이트와 부분 업데이트 기능의 성능 차이가 크게 느껴지지 않을 수 있는데, 사실 부분 업데이트 기능은 특정 조건에서는 매우 빠른 업데이트 성능을 보여준다.
참고로 JSON 칼럼의 값은 MySQL 내부적으로 BLOB 타입으로 저장(더 정확히는 LONGBLOB 타입)되는데, 실제 JSON 칼럼은 최대 4GB까지의 값을 가질 수 있다.
물론 일반적으로는 이 정도 공간을 채우지는 않겠지만 1MB JSON 데이터를 저장해도 MySQL 서버는 16KB 데이터 페이지를 64개나 사용하게 된다.
JSON 부분 업데이트의 경우 64개 페이지 중에서 단 하나만 변경하면 되지만, 부분 업데이트를 사용할 수 없다면 MySQL 서버는 64개 데이터 페이지를 다시 디스크로 기록해야 한다.
이제 이 경우의 JSON 칼럼 업데이트의 성능을 비교해보자.
다음 예제에서는 대략 10MB 정도의 JSON 데이터를 가지는 레코드 16건을 만들고, 부분 업데이트를 사용하는 경우와 그렇지 못한 경우의 성능을 보여준다.

```
-- // 테스트 데이터 준비
mysql> CREATE TABLE tb_json(id INT, fd JSON, PRIMARY KEY (id));
mysql> INSERT INTO tb_json(id, fd) VALUES
         (1, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('a', 10 * 1000 * 1000))),
         (2, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('b', 10 * 1000 * 1000))),
         (3, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('c', 10 * 1000 * 1000))),
         (4, JSON_OBJECT('name', 'Matt', 'visits', 0, 'data', REPEAT('d', 10 * 1000 * 1000)));

mysql> INSERT INTO tb_json(id, fd) SELECT id+5,  fd FROM tb_json;
mysql> INSERT INTO tb_json(id, fd) SELECT id+10, fd FROM tb_json;

-- // 부분 업데이트를 사용하지 못하는 경우
mysql> UPDATE tb_json SET fd=JSON_SET(fd, '$.name', "Matt Lee");
Query OK, 16 rows affected (2.74 sec)

-- // 부분 업데이트를 사용하는 경우
mysql> UPDATE tb_json SET fd=JSON_SET(fd, '$.name', "Kit");
Query OK, 16 rows affected (1.34 sec)
```

16건의 레코드를 업데이트하는데, 부분 업데이트를 사용하는 경우에는 1.34초가 걸렸지만 부분 업데이트를 사용하지 못하는 쿼리는 2.74초가 걸렸다.
MySQL 서버에서는 일반적으로 복제를 사용하기 때문에 MySQL 서버는 JSON 변경 내용을 바이너리 로그에 기록해야 한다. 이때 MySQL 서버의 바이너리 로그에는 여전히 JSON의 데이터를 모두 기록한다.
하지만 변경된 내용들만 바이너리 로그에 기록되도록 `binlog_row_value_options` 시스템 변수와 `binlog_row_image` 시스템 변수의 설정값을 변경하면 JSON 칼럼의 부분 업데이트의 성능을 훨씬
더 빠르게 만들 수 있다.

```
mysql> SET binlog_format = ROW;
mysql> SET binlog_row_value_options = PARTIAL_JSON;
mysql> SET binlog_row_image = MINIMAL;

mysql> UPDATE tb_json SET fd=JSON_SET(fd, '$.name', "Matt Lee");
Query OK, 16 rows affected (2.30 sec)

mysql> UPDATE tb_json SET fd=JSON_SET(fd, '$.name', "Kit");
Query OK, 16 rows affected (0.18 sec)
```

대략 13배 정도 UPDATE 성능이 개선됐다. 바이너리 로그의 포맷을 STATEMENT 타입으로 변경해도 거의 동일한 성능 향상 효과를 얻을 수 있다.

```
mysql> SET binlog_format=STATEMENT;

mysql> UPDATE tb_json SET fd = JSON_SET(fd, '$.name', "Kit");
Query OK, 16 rows affected (0.15 sec)
```

> JSON 칼럼의 부분 업데이트 최적화 효과를 얻기 위해서는 binlog_row_value_options 시스템 변수와 binlog_row_image 시스템 변수의 변경도 필요하지만 JSON 칼럼을 가진 테이블의 프라이머리
> 키가 필수적이다. 테이블의 프라이머리 키가 없다면 MySQL 복제에서 레플리카 서버는 업데이트할 레코드를 식별하기 위해 레코드의 모든 칼럼을 필요로 한다.
> 그래서 테이블의 프라이머리 키가 없는 경우, 위의 두 시스템 변수와 관계없이 JSON 칼럼을 포함해서 레코드의 모든 칼럼을 바이너리 로그에 기록해야 하므로 부분 업데이트의 성능은 느려진다.

단순히 정수 필드의 값을 변경하는 UPDATE는 항상 부분 업데이트 기능이 적용될 것이다. 하지만 문자열 타입의 필드라면 저장되는 문자열의 길이에 따라 부분 업데이트가 사용되지 못할 수도 있다.
특정 필드의 값이 작은 용량을 가지면서 자주 길이가 다른 값으로 변경된다면 해당 필드가 가질 수 있는 최대 길이의 값으로 초기화해 두거나 애플리케이션에서 추가로 패딩해서 고정 길이의 문자열로 만들어서
저장하는 방법도 부분 업데이트 기능을 활용할 수 있는 좋은 방법이다.
<br/>
<br/>
## (3) JSON 타입 콜레이션과 비교
JSON 칼럼에 저장되는 데이터와 JSON 칼럼으로부터 가공되어 나온 결괏값은 모두 utf8mb4 문자 집합과 utf8mb4_bin 콜레이션을 가진다.
utf8mb4_bin 콜레이션은 바이너리 콜레이션이기 때문에 JSON 칼럼의 비교와 JSON 칼럼으로부터 가공된 문자열은 대소문자 구분은 물론 액센트 문자 등도 구분해서 비교한다.
다음 예제를 보면 대문자를 포함한 JSON 오브젝트와 소문자로만 된 JSON 오브젝트의 값은 서로 다르다는 것을 알 수 있다.

```
mysql> SET @user1 = JSON_OBJECT('name', 'Matt');
mysql> SELECT CHARSET(@user1), COLLATION(@user1);
+-----------------+-------------------+
| CHARSET(@user1) | COLLATION(@user1) |
+-----------------+-------------------+
| utf8mb4         | utf8mb4_bin       |
+-----------------+-------------------+

mysql> SET @user2 = JSON_OBJECCT('name', 'matt');
mysql> SELECT @user1=@user2;
+---------------+
| @user1=@user2 |
+---------------+
|             0 | ==> FALSE
+---------------+
```
<br/>

## (4) JSON 칼럼 선택
BLOB 타입이나 TEXT 타입에 JSON 문자열을 저장하는 경우 아무런 변환 없이 입력된 값을 그대로 디스크에 저장한다.
하지만 JSON 타입은 JSON 데이터를 이진 포맷으로 컴팩션해서 저장할뿐마나 아니라 필요한 경우 부분 업데이트를 통한 빠른 변경 기능을 제공하며, JSON 데이터 가공에 필요한 여러 가지 기능을 제공한다.
그래서 JSON 데이터를 저장해야 한다면 당연히 BLOB이나 TEXT 칼럼보다는 JSON 칼럼을 선택하는 것이 좋다.

그렇다면 일반적으로 정규화한 칼럼과 JSON 칼럼 중에서는 어떤 것을 선택해야 할까? 우선 간단히 JSON 칼럼만으로 구성된 테이블과 정규화된 칼럼만 사용하는 테이블의 예시를 살펴보자.

#### [JSON 칼럼만으로 구성된 테이블]
```
mysql> CREATE TABLE tb_json (
         doc JSON NOT NULL,
         id BIGINT AS (doc->>'$.id') STORED NOT NULL,
         PRIMARY KEY (id)
       );

mysql> INSERT INTO tb_json (doc) VALUES
         ('{"id":1, "name":"Matt"}'),
         ('{"id":2, "name":"Esther"}');
```

#### [정규화된 칼럼만으로 구성된 테이블]
```
mysql> CREATE TABLE tb_column (
         id BIGINT NOT NULL,
         name VARCHAR(50) NOT NULL,
         PRIMARY KEY (id)
       );

mysql> INSERT INTO tb_column VALUES
         (1, 'Matt'),
         (2, 'Esther');
```

위의 예제처럼 JSON 칼럼만 유지하는 경우에도 필요한 인덱스를 모두 생성할 수 있다.
그리고 MySQL 8.0 버전부터는 멀티 밸류 인덱스 기능이 지원되기 때문에 JSON 도큐먼트에서 배열(Array) 타입의 필드에도 인덱스를 생성할 수 있게 됐다.
굳이 양쪽의 장단점을 모두 언급하지 않더라도 JSON 칼럼과 정규화된 칼럼 모두 장점과 단점을 알고 있을 것이다. 그리고 개발자의 취향에 따라 주관적인 선호도도 있을 것이다.

MySQL 서버의 성능에 가장 큰 무게를 두고 볼 때, JSON 칼럼과 정규화된 칼럼의 선택 문제에서도 성능을 중심으로 판단한다면 JSON 칼럼보다는 성능적인 이점을 가지고 있는 정규화된 칼럼을 추천한다.
정규화된 칼럼은 칼럼의 이름을 메타 정보로만 저장하기 때문에 칼럼의 이름이 별도의 데이터 파일의 공간을 차지하지 않는다. 하지만 JSON 칼럼은 각 필드의 이름이 데이터 파일에 매번 저장돼야 한다.
JSON 필드의 이름을 얼마나 컴팩트하게 저장할 수 있을지 모르지만 레코드 건수가 많아지면 많아질수록 JSON 필드의 이름들이 차지하는 디스크의 공간은 더욱더 커질 것이다.

MySQL 서버의 압축을 사용하면 공간을 줄일 수 있을 것으로 생각할 수 있지만 압축은 디스크의 공간만 줄이는 수준이지 메모리의 사용 효율까지 높여주진 못한다.
또한 MySQL 서버의 데이터 압축은 다른 DBMS와는 달리 메모리에 압축된 페이지와 압축 해제된 페이지가 공존해야 하기 때문에 메모리 효율과 CPU 효율 모두를 떨어뜨릴 수도 있다.

또한 MySQL 서버에서 정규화된 칼럼을 사용하는 경우 BLOB이나 TEXT와 같이 대용량 데이터의 경우 외부 페이지로 관리된다.
이러한 장점을 응용 프로그램의 요건에 맞게 적절히 활용하면 메모리 효율이나 쿼리의 성능을 훨씬 더 끌어올릴 수 있다.
하지만 모든 데이터를 하나의 JSON 칼럼에 저장하면 응용 프로그램의 요건이나 쿼리가 필요한 데이터 등을 선별적으로 접근함으로써 얻을 수 있는 성능 효과는 기대하기 어렵다.
레코드를 통째로 하나의 JSON 칼럼에 저장한다면 정숫값 하나만 참조하더라도 JSON 칼럼에 저장된 도큐먼트를 모두 읽어봐야 하기 때문이다.

그렇다고 해서 JSON 칼럼의 장점이 전혀 없는 것은 아니다.
예를 들어, 각 레코드가 가지는 속성들이 너무 상이하고 다양하지만 레코드별로 선택적으로 값을 가지는 경우(이렇게 많은 레코드들이 칼럼의 값을 가지지 않는 경우를 Sparse Column이라고 한다.)라면 가능한
모든 속성에 대한 칼럼을 생성하는 것보다는 JSON 칼럼을 만들어서 저장하는 것이 좋다. 물론 이렇게 JSON 칼럼에 저장되는 속성들은 가능하면 중요도가 낮은 것일수록 좋다.
중요도가 낮다는 것은 그만큼 검색 조건으로 사용될 가능성도 낮고, 쿼리에서 자주 접근될 가능성도 낮다는 것을 의미하기 때문이다.

또한 너무 정규화된 테이블 구조를 유지하면 테이블의 개수가 많아지고 응용 프로그램의 코드도 길어지는 경우가 많다.
이런 경우에도 중요도가 낮은 데이터라면 JSON 칼럼에 비정규화된 형태로 데이터를 저장할 수도 있다.

> MySQL 서버를 포함한 대부분의 RDBMS는 작은 크기의 데이터 처리에 적합하도록 설계됐다. 그래서 JSON 같은 칼럼에 큰 값을 저장하게 되면 예상했던 것보다 훨씬 느린 성능을 보이기도 한다.
> 테이블의 JSON 칼럼에 2MB 정도의 값을 저장하고, JSON 칼럼을 빼고 SELECT하는 쿼리와 JSON 칼럼을 함께 조회하는 SELECT 쿼리의 성능을 간단히 비교해보자.
>
> ```
> mysql> CREATE TABLE test (id INT NOT NULL PRIMARY KEY, value JSON);
>
> mysql> SELECT id, value FROM test WHERE id=1;
> 1 row in set (0.16 sec)
>
> mysql> SELECT id FROM test WHERE id=1;
> 1 row in set (0.01 sec)
> ```
>
> 아무것도 처리하지 않는 한가한 서버에서 테스트해본 결과로도 상당한 시간 차이가 발생하는 것을 확인할 수 있다. 많은 커넥션의 쿼리 요청으로 바쁜 MySQL 서버였다면 시간 차이는 더 커질 것이다.
> MySQL 서버와 같은 RDBMS에서 JSON 칼럼을 지원한다고 해서 너무 JSON 칼럼을 남용하거나 너무 큰 데이터를 저장하는 것은 권장하지 않는다.