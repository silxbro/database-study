# [CH 15-5] TEXT와 BLOB

MySQL에서 대량의 데이터를 저장하려면 TEXT나 BLOB 타입을 사용해야 하는데, 이 두 타입은 많은 부분에서 거의 똑같은 설정이나 방식으로 작동한다.
TEXT 타입과 BLOB 타입의 유일한 차이점은 TEXT 타입은 문자열을 저장하는 대용량 칼럼이라서 문자 집합이나 콜레이션을 가진다는 것이고, BLOB 타입은 이진 데이터 타입이라서 별도의 문자 집합이나
콜레이션을 가지지 않는다는 것이다. 다음 표와 같이 TEXT와 BLOB 타입 모두 다시 내부적으로 저장 가능한 최대 길이에 따라 4가지 타입으로 구분한다.

|데이터 타입|필요 저장 공간<br/>(L = 저장하고자 하는 데이터의 바이트 수)|저장 가능한 최대 바이트 수|
|:---|:---|:---|
|**TINYTEXT, TINYBLOB**|L + 1바이트|2^8-1(255)|
|**TEXT, BLOB**|L + 2바이트|2^16-1(65,535)|
|**MEDIUMTEXT, MEDIUMBLOB**|L + 3바이트|2^24-1(16,777,215)|
|**LONGTEXT, LONGBLOB**|L + 4바이트|2^32-1(4,294,967,295)|

LONG이나 LONG VARCHAR라는 타입도 있는데, MEDIUMTEXT의 동의어이므로 특별히 기억할 필요는 없다.
이진 데이터를 저장하기 위한 데이터 타입과 문자열을 저장하기 위한 데이터 타입은 다음과 같이 고정 길이나 가변 길이 타입이 정확하게 매핑된다.

||고정길이|가변길이|대용량|
|:---|:---|:---|:---|
|**문자 데이터**|CHAR|VARCHAR|TEXT|
|**이진 데이터**|BINARY|VARBINARY|BLOB|

오라클 DBMS의 영향인지 많은 사람이 BLOB 타입에 대해서는 대용량 칼럼이라는 인식을 가지고 주의하는 데 반해, TEXT 타입은 그다지 부담을 가지지 않고 사용하는 경향도 있다.
MySQL의 TEXT 타입은 오라클에서 CLOB이라고 하는 대용량 타입과 동일한 역할을 하는 데이터 타입이므로 TEXT와 BLOB 칼럼 모두 사용할 때 주의하고 너무 남용해서는 안 된다.
TEXT나 BLOB 타입은 주로 다음과 같은 상황에서 사용하는 것이 좋다.

- 칼럼 하나에 저장되는 문자열이나 이진 값의 길이가 예측할 수 없이 클 때 TEXT나 BLOB을 사용한다.
  하지만 다른 DBMS와는 달리 MySQL에서는 값의 크기가 4000바이트를 넘을 때 반드시 BLOB이나 TEXT를 사용해야 하는 것은 아니다.
  MySQL에서는 레코드의 전체 크기가 64KB를 넘지 않는 한도 내에서는 VARCHAR나 VARBINARY의 길이는 제한이 없다.
  그래서 용도에 따라서는 다음 예제와 같이 4000바이트 이상의 값을 저장하는 칼럼도 VARCHAR나 VARBINARY 타입을 이용할 수 있다.

- MySQL에서는 버전에 따라 조금씩 차이는 있지만 일반적으로 하나의 레코드는 전체 크기가 64KB를 넘어설 수 없다.
  VARCHAR나 VARBINARY와 같은 가변 길이 칼럼은 최대 저장 가능 크기를 포함해 64KB로 크기가 제한된다.
  레코드의 전체 크기가 64KB를 넘어서서 더 큰 칼럼을 추가할 수 없다면 일부 칼럼을 TEXT나 BLOB 타입으로 전환해야 할 수도 있다.

```
mysql> CREATE TABLE tb_varchar (
         id INT NOT NULL,
         body VARCHAR(6000),
         PRIMARY KEY (id)
       );

mysql> SHOW CREATE TABLE tb_varchar \G
*************************** 1. row ***************************
       Table: tb_varchar
Create Table: CREATE TABLE `tb_varchar` (
  `id` int NOT NULL,
  `body` varchar(6000) COLLATE utf8mb4_general_ci DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```

MySQL에서 인덱스 레코드의 모든 칼럼은 최대 제한 크기(MyISAM은 1000바이트, REDUNDANT 또는 COMPACT 로우 포맷을 사용하는 InnoDB의 경우에는 767바이트, DYNAMIC 또는 COMPRESSED 로우
포맷을 사용하는 InnoDB의 경우에는 3072바이트)를 가지고 있다.
자주 사용되지는 않지만 BLOB이나 TEXT 타입의 칼럼에 인덱스를 생성할 때는 칼럼값의 몇 바이트까지 인덱스를 생성할 것인지를 명시해야 할 때도 있다.
물론 최대 제한 크기를 넘어서는 인덱스는 생성할 수 없다.
DYNAMIC 또는 COMPRESSED 로우 포맷을 사용하는 InnoDB 테이블에서 TEXT 타입의 문자 집합이 utf8mb4이라면 최대 768 글자까지만 인덱스로 생성할 수 있고, latin1 문자 셋의 TEXT 타입이라면
3072 글자까지 인덱스로 생성할 수 있다.
또한 BLOB이나 TEXT 칼럼으로 정렬을 수행할 때도 칼럼에 저장된 값이 10MB라고 하더라도 실제로는 MySQL 서버의 `max_sort_length` 시스템 변수에 설정된 길이까지만 정렬을 수행한다.
일반적으로 이 설정값은 1024바이트로 설정돼 있는데, TEXT 타입의 정렬을 더 빠르게 실행하려면 이 값을 줄여서 설정하는 것이 좋다.

MySQL에서는 쿼리의 특성에 따라 임시 테이블을 생성해야 할 때도 있다. 이때 사용하는 임시 테이블은 메모리에 저장될 수도 있고 디스크에 저장될 수도 있다.
임시 테이블을 메모리에 저장할 때는 `internal_tmp_mem_storage_engine` 시스템 변수의 설정값에 따라 MEMORY 스토리지 엔진이나 TempTable 스토리지 엔진 중 하나를 사용한다.
MySQL 8.0 버전부터 TempTable은 TEXT나 BLOB 타입을 지원하지만 MEMORY 스토리지 엔진은 TEXT나 BLOB 타입을 지원하지 않는다.
가능하면 internal_tmp_mem_storage_engine은 "TempTable"로 설정해서 BLOB 타입이나 TEXT 타입을 포함하는 결과도 메모리를 사용할 수 있게 하는 것이 좋다.

BLOB이나 TEXT 타입의 칼럼이 포함된 테이블에 실행되는 INSERT나 UPDATE 문장 중에서 BLOB이나 TEXT 칼럼을 조작하는 SQL 문장은 매우 길어질 수 있는데, MySQL 서버의 `max_allowed_packet`
시스템 변수에 설정된 값보다 큰 SQL 문장은 MySQL 서버로 전송되지 못하고 오류가 발생할 수도 있다.
대용량 BLOB이나 TEXT 칼럼을 사용하는 쿼리가 있다면 MySQL 서버의 max_allowed_packet 시스템 변수를 필요한 만큼 충분히 늘려서 설정하는 것이 좋다.

MySQL 서버에서 TEXT나 BLOB 타입 칼럼의 값이 어떻게 저장되는지는 혼란스러운 부분 중 하나다.
MySQL 서버에서 BLOB과 TEXT 타입 칼럼의 데이터가 어떻게 저장될지를 결정하는 요소는 테이블의 ROW_FORMAT 옵션이다.
테이블을 생성할 때 ROW_FORMAT 옵션이 별도로 지정되지 않으면 MySQL 서버는 `innodb_default_row_format` 시스템 변수에 설정된 값을 적용한다.
MySQL 서버에서 별도로 innodb_default_row_format 시스템 변수를 설정하지 않으면 기본으로 최신 ROW_FORMAT인 dynamic이 설정된다.

```
mysql> SHOW GLOBAL VARIABLES LIKE 'innodb_default_row_format';
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| innodb_default_row_format | dynamic |
+---------------------------+---------+
```

우선 MySQL 8.0에서는 지금까지 사용 가능한 모든 ROW_FORMAT(REDUNDANT와 COMPACT, 그리고 DYNAMIC과 COMPRESSED)에서는 가능하다면 TEXT와 BLOB 칼럼의 값을 다른 레코드와 같이 저장하려고
노력한다. 그런데 이를 불가능하게 하는 한 가지 이유가 레코드의 최대 길이 제한이다.

설명의 편의를 위해 다음과 같이 TEXT와 BLOB 타입의 칼럼을 하나씩 가지는 테이블을 가정해보자.

```
mysql> CREATE TABLE tb_large_object (
         id INT NOT NULL PRIMARY KEY,
         fd_blob BLOB,
         fd_text TEXT
       );
```

MySQL 5.6 버전에서 테이블의 기본 ROW_FORMAT은 COMPACT이며, MySQL 5.7 버전부터는 DYNAMIC이 기본 ROW_FORMAT으로 변경됐다.
MySQL 서버의 ROW_FORMAT에서 COMPACT는 나머지 모든 ROW_FORMAT의 바탕이 되는 포맷이다.
DYNAMIC은 COMPACT 포맷에 몇 가지 규칙이 추가된 버전이고, COMPRESSED는 DYNAMIC에 압축 관련 규칙이 추가된 버전이다.
여기서 COMPACT 포맷을 예시로 언급하는 이유는 COMPACT 포맷이 나머지 모든 포맷의 근간이기 때문이다.

COMPACT 포맷에서 저장할 수 있는 레코드 하나의 최대 길이는 데이터 페이지(데이터 블록) 크기 16KB의 절반인 8126바이트(정확히 8K가 아니고 8126인 것은 데이터 페이지에서 관리용으로 사용되는
공간들을 빼고 사용할 수 있는 최대 공간의 절반이기 때문이다.)다. 이 경우 MySQL 서버는 BLOB이나 TEXT 타입의 칼럼을 가능한 레코드에 같이 포함해서 저장하려고 할 것이다.
레코드의 전체 길이가 8126바이트를 넘어선다면 용량이 큰 칼럼 순서대로 외부 페이지(Off-page 또는 External-page)로 옮기면서 레코드의 길이를 8126바이트 이하로 맞추려고 할 것이다.
대략 몇 가지 예제를 살펴보면 다음과 같이 레코드의 칼럼을 프라이머리 키 페이지와 외부 페이지로 나눠서 저장할 것이다.

|fd_blob의 길이|fd_text의 길이|fd_blob의 저장 위치|fd_text의 저장 위치|
|---:|---:|:---|:---|
|3000|3000|프라이머리 키 페이지|프라이머리 키 페이지|
|3000|10000|프라이머리 키 페이지|외부 페이지|
|10000|10000|외부 페이지|외부 페이지|

첫 번째 경우는 TEXT와 BLOB을 합해도 6000바이트이므로 모두 프라이머리 키 페이지에 같이 저장(인라인(inline)으로 저장)할 수 있다.
하지만 두 번째 경우는 TEXT 칼럼의 길이가 너무 길어서 TEXT 칼럼은 외부 페이지로 저장하고 BLOB 칼럼은 프라이머리 키 페이지에 같이 저장한다.
그리고 세 번째 경우는 둘 모두 너무 길이가 길어서 TEXT와 BLOB 칼럼 모두 외부 페이지로 저장한다.
BLOB이나 TEXT 칼럼이 외부 페이지로 저장될 때 길이가 16KB를 넘는 경우 MySQL 서버는 칼럼의 값을 나눠서 여러 개의 외부 페이지에 저장하고 각 페이지는 체인으로 연결한다.
아래 그림은 이렇게 칼럼의 값이 여러 개의 외부 페이지에 저장된 형태를 보여준다. 하나의 테이블에 여러 개의 BLOB이나 TEXT 칼럼이 있다면 하나의 레코드는 여러 개의 외부 페이지 체인을 가질 수도 있다.

#### [그림 15.4] BLOB이나 TEXT 칼럼의 값이 여러 개의 외부 페이지로 저장된 형태
<img src="https://github.com/user-attachments/assets/8a80aff7-677c-44d6-b1f3-a92888479b0c" width="600"/><br/>

BLOB이나 TEXT 칼럼을 외부 페이지로 저장하는 경우 MySQL 서버는 COMPACT와 REDUNDANT 레코드 포맷을 사용하는 테이블에서는 외부 페이지로 저장된 TEXT나 BLOB 칼럼의 앞쪽 768바이트(BLOB
프리픽스)만 잘라서 프라이머리 키 페이지에 같이 저장한다. DYNAMIC이나 COMPRESSED 레코드 포맷에서는 BLOB 프리픽스를 프라이머리 키 페이지에 저장하지 않는다.
COMPACT나 REDUNDANT 레코드 포맷의 BLOB 프리픽스는 인덱스를 생성할 때 도움이 되기도 하지만 BLOB이나 TEXT 칼럼을 가진 테이블의 저장 효율을 낮추게 될 수도 있다.
BLOB 프리픽스는 프라이머리 키 페이지에 저장할 수 있는 레코드의 건수를 줄이는데, BLOB이나 TEXT 칼럼을 거의 참조하지 않는 쿼리는 성능이 더 떨어진다.