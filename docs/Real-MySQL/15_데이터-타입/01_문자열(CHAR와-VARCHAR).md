# [CH 15-1] 문자열(CHAR와 VARCHAR)

문자열 칼럼을 사용할 때는 우선 CHAR 타입과 VARCHAR 타입 중 어떤 타입을 사용할지 결정해야 한다.
그래서 CHAR과 VARCHAR 타입의 차이가 무엇이고 어떤 타입을 사용하는 것이 좋은가에 관한 질문도 많은 편이다.
처음 데이터베이스를 사용할 때는 둘 중에서 뭘 선택해야 할지 고민하다가 결국 VARCHAR만 쭉 사용하는 사람들도 있다.
하지만 지금까지 모든 DBMS에서 CHAR나 VARCHAR 타입을 구분해서 제공하는 것을 보면 그만큼의 장단점을 가지고 있음을 짐작할 수 있다.
우선 저장 공간과 비교 방식의 관점에서 CHAR와 VARCHAR를 비교해 보고, MySQL 내부적으로 어떤 차이가 있는지도 한 번 살펴보자.

---
<br/>

## (1) 저장 공간
우선 CHAR와 VARCHAR의 공통점은 문자열을 저장할 수 있는 데이터 타입이라는 점이고, 가장 큰 차이는 고정 길이냐 가변 길이냐다.

- 고정 길이는 실제 입력되는 칼럼값의 길이에 따라 사용하는 저장 공간의 크기가 변하지 않는다. CHAR 타입은 이미 저장 공간의 크기가 고정적이다.
  실제 저장된 값의 유효 크기가 얼마인지 별도로 저장할 필요가 없으므로 추가로 공간이 필요하지 않다.

- 가변 길이는 최대로 저장할 수 있는 값의 길이는 제한돼 있지만, 그 이하 크기의 값이 저장되면 그만큼 저장 공간이 줄어든다.
  하지만 VARCHAR 타입은 저장된 값의 유효 크기가 얼마인지를 별도로 저장해 둬야 하므로 1\~2바이트의 저장 공간이 추가로 더 필요하다.

하나의 글자를 저장하기 위해 CHAR(1)와 VARCHAR(1) 타입을 사용할 때 실제 사용되는 저장 공간의 크기를 한번 살펴보자.
우선 두 문자열 타입 모두 한 글자를 저장할 때 사용하는 문자 집합에 따라 실제 저장 공간을 1\~4바이트가지 사용한다.
여기서 하나의 글자가 CHAR 타입에 저장될 때는 추가 공간이 더 필요하지 않지만 VARCHAR 타입에 저장될 때는 문자열의 길이를 관리하기 위한 1\~2바이트의 공간을 추가로 더 사용한다.
VARCHAR 타입의 길이가 255바이트 이하이면 1바이트만 사용하고, 256바이트 이상으로 설정되면 2바이트를 사용한다. VARCHAR 타입의 최대 길이는 2바이트로 표현할 수 있는 이상은 사용할 수 없다.
즉, VARCHAR 타입의 최대 길이는 65,536 바이트 이상으로 설정할 수 없다.

> MySQL에서는 하나의 레코드에서 TEXT와 BLOB 타입을 제외한 칼럼의 전체 크기가 64KB를 초과할 수 없다.
> 테이블에 VARCHAR 타입의 칼럼 하나만 있다면 이 VARCHAR 타입은 최대 64KB 크기의 데이터를 저장할 수 있다.
> 하지만 이미 다른 칼럼에서 40KB의 크기를 사용하고 있다면 VARCHAR 타입은 24KB만 사용할 수 있다.
> 이때 24KB를 초과하는 크기의 VARCHAR 타입을 생성하려고 하면 에러가 발생하거나 자동으로 VARCHAR 타입이 TEXT 타입으로 대체된다.
> 그래서 칼럼을 새로 추가할 때는 VARCHAR 타입이 TEXT 타입으로 자동으로 변환되지 않았는지 확인해 보는 것이 좋다.
>
> 문자열 타입의 저장 공간을 언급할 때는 1문자와 1바이트를 구분해서 사용한다. 1문자는 실제 저장되는 값의 문자 집합에 따라 1\~4바이트까지 공간을 사용할 수 있기 때문이다.
> 위의 VARCHAR 타입의 칼럼 하나만 가지는 테이블의 예에서 VARCHAR 타입은 최대 64KB 크기의 데이터를 저장할 수 있다고 했는데, 이 수치는 바이트 수를 의미하므로 실제 65,536개의 글자를 저장할
> 수 있다는 것은 아니다. 실제 저장되는 문자가 아시아권의 언어라면 저장 가능한 글자 수는 반으로 줄고, UTF-8 문자를 저장한다면 실제 저장 가능한 글자 수는 1/4로 줄어들 것이다.

문자열 값의 길이가 항상 일정하다면 CHAR를 사용하고 가변적이라면 VARCHAR를 사용하는 것이 일반적이다. 왜 길이가 고정적일 때 CHAR를 사용하면 좋을까?
VARCHAR 타입을 선택해도 기껏 디스크에서 1바이트만 더 사용할 뿐인데, 이렇게 고민해가면서 시간을 투자할 가치가 있는 것일까?
실제 문자열 값의 길이가 정적이나 가변적이냐만으로 CHAR와 VARCHAR 타입을 결정하는 것은 적절하지 않다. CHAR 타입과 VARCHAR 타입을 결정할 때 중요한 판단 기준은 다음과 같다.

- 저장되는 문자열의 길이가 대개 비슷한가?
- 칼럼의 값이 자주 변경되는가?

CHAR와 VARCHAR 타입의 선택 기준은 값의 길이도 중요하지만, 해당 칼럼의 값이 얼마나 자주 변경되느냐가 기준이 돼야 한다.
칼럼의 값이 얼마나 자주 변경되느냐가 왜 중요한지 그림으로 한번 살펴보자. 우선 다음과 같이 테스트용 테이블이 있고, 그 테이블에 레코드 1건이 저장된다고 가정해보자.

```
mysql> CREATE TABLE tb_test (
         fd1 INT NOT NULL,
         fd2 CHAR(10) NOT NULL,
         fd3 DATETIME NOT NULL
       );

mysql> INSERT INTO tb_test (fd1, fd2, fd3) VALUES (1, 'ABCD', '2011-06-27 11:02:11');
```

tb_test 테이블에 레코드 1건을 저장하면 내부적으로 디스크에는 아래 그림과 같이 저장될 것이다.

#### [그림 15.1] CHAR 타입이 저장된 상태
<img src="https://github.com/user-attachments/assets/476b3fef-5921-4371-9bad-0d8692a6ea8b" width="400"/><br/>

fd1 칼럼은 INTEGER 타입이므로 고정 길이로 4바이트를 사용하며, fd3 또한 DATETIME이므로 고정 길이로 8바이트를 사용한다.
지금 여기서 관심사는 fd1과 fd3 칼럼이 아니라 그 사이에 위치한 fd2 칼럼이다. fd2 칼럼이 사용하는 공간을 눈여겨보자.
위 그림에서 fd2 칼럼은 정확히 10바이트를 사용하면서 앞쪽의 4바이트만 유효한 값으로 채워졌고 나머지는 공백 문자로 채워져 있다(fd2 칼럼의 빈 공간은 공백 문자(Space character)를 의미한다).

그러면 이번에는 tb_test 테이블의 fd2 칼럼만 CHAR(10) 대신 VARCHAR(10)으로 변경해서 똑같은 데이터를 저장했을 때 디스크에 어떻게 저장되는지 살펴보자.

#### [그림 15.2] VARCHAR 타입의 칼럼이 저장된 상태
<img src="https://github.com/user-attachments/assets/4f0a87a4-8f59-4686-8d9f-cad38dd6610a" width="300"/><br/>

fd1 칼럼과 fd3 칼럼 사이에서 fd2 칼럼은 5바이트의 공간을 차지하는데, 첫 번째 바이트에는 저장된 칼럼값의 유효한 바이트 수인 숫자 4(문자 '4'가 아님)가 저장되고 두 번째 바이트부터 다섯 번째
바이트까지 실제 칼럼값이 저장된다.

위 그림들은 이미 대략 예측하고 있는 사실일 것이다. 하지만 중요한 것은 레코드 한 건이 저장된 상태가 아니라 fd2 칼럼의 값의 변경될 때 어떤 현상이 발생하느냐다.
fd2 칼럼의 값을 "ABCDE"로 UPDATE했다고 가정해 보자.

- CHAR(10) 타입을 사용하는 경우 fd2 칼럼을 위해 공간이 10바이트가 준비돼 있으므로 그냥 변경되는 칼럼의 값을 업데이트만 하면 된다.
- VARCHAR(10) 타입을 사용하는 경우 fd2 칼럼에 4바이트밖에 저장할 수 없는 구조로 만들어져 있다.
  그래서 "ABCDE"와 같이 길이가 더 큰 값으로 변경될 때는 레코드 자체를 다른 공간으로 옮겨서(Row migration) 저장해야 한다.

물론 주민등록번호처럼 항상 값의 길이가 고정적일 때는 당연히 CHAR 타입을 사용해야 한다.
또한 값이 2\~3바이트씩 차이가 나더라도 자주 변경될 수 있는 부서 번호나 게시물의 상태 값 등은 CHAR 타입을 사용하는 것이 좋다.
자주 변경돼도 레코드가 물리적으로 다른 위치로 이동하거나 분리되지 않아도 되기 때문이다.
레코드의 이동이나 분리는 CHAR 타입으로 인해 발생하는 2\~3바이트 공간 낭비보다 더 큰 공간이나 자원을 낭비하게 만든다.

문자열 데이터 타입을 사용할 때 또 하나 주의할 사항이 있다. CHAR나 VARCHAR 키워드 뒤에 인자로 전달하는 숫자 값의 의미를 알아야 한다는 점이다.
다른 DBMS에 익숙한 사용자에게는 상당히 혼란스러울 수 있는데, MySQL에서 CHAR나 VARCHAR 뒤에 지정하는 숫자는 그 칼럼의 바이트 크기가 아니라 문자의 수를 의미한다.
즉, CHAR(10) 또는 VARCHAR(10)으로 칼럼을 정의하면 이 칼럼은 10바이트를 저장할 수 있는 공간이 아니라 10글자(문자)를 저장할 수 있는 공간을 의미한다.
그래서 CHAR(10) 타입을 사용하더라도 이 칼럼이 실제로 디스크나 메모리에서 사용하는 공간은 각각 달라진다.

- 일반적으로 영어를 포함한 서구권 언어는 각 문자가 1바이트를 사용하므로 10바이트를 사용한다.
- 한국어나 일본어와 같은 아시아권 언어는 각 문자가 최대 2바이트를 사용하므로 20바이트를 사용한다.
- UTF-8과 같은 유니코드는 최대 4바이트까지 사용하므로 40바이트까지 사용할 수 있다.

> MySQL 서버에서는 uff8과 utf8mb4 문자 집합이 별도로 존재한다.
> utf8 문자 집합은 MySQL 5.5 이전 버전까지 UTF-8 인코딩을 저장하기 위해 사용됐으며, utf8mb4는 MySQL 5.5 버전부터 지원되기 시작했다.
> MySQL 서버의 utf8 문자 셋은 한 글자당 최대 3바이트까지만 지원됐다. UTF-8 인코딩에서는 각 문자가 저장 공간에 따라 대략 다음과 같이 구분된다.
>
> - 1바이트 저장 공간 사용: 아스키(ASCII) 문자
> - 2바이트 저장 공간 사용: 추가 알파벳 문자
> - 3바이트 저장 공간 사용: BMP(Basic Multilingual Plane) 문자
> - 4바이트 저장 공간 사용: SMP(Supplementary Multilingual Plane) & SIP(Supplementary Ideographic Plane) & ...
>
> MySQL 5.5 이전 버전에서 주로 사용했던 utf8 문자 집합은 1\~3바이트까지만 저장을 지원했기 때문에 SMP와 SIP, 그리고 그 이후 플레인 문자는 저장할 수가 없었다.
> 하지만 이모티콘의 발전으로 이는 예상외로 심각한 문제가 되어버렸다. 그래서 MySQL 서버는 이 문제를 해결하기 위해 utf8mb4라는 새로운 문자 집합을 도입했다.
> utf8mb4 문자 집합은 최대 4바이트까지의 문자를 저장할 수 있기 때문에 유니코드에서 지원하는 대부분의 문자를 지원했다.
> utf8mb4는 utf8 문자 집합의 수퍼 셋(상위 셋)이기 때문에 utf8 문자 집합을 사용하던 테이블을 utf8mb4 문자 집합으로 전환(CONVERT)하는 것은 아무런 문제를 유발하지 않는다.
>
> utf8mb4 문자 집합이 만들어지면서 utf8bmb3라는 문자 집합도 만들어졌으며, MySQL 8.0 버전에서는 utf8 문자 집합은 utf8mb3를 가리키는 별칭으로 정의돼 있다.
> utf8mb3 문자 집합은 곧 MySQL 서버에서 제거될 예정(Deprecated)이며, MySQL 서버에서 utf8mb3 문자 집합이 제거되면 utf8 별칭은 utf8mb4를 가리킬 예정이다.
<br/>

## (2) 저장 공간과 스키마 변경(Online DDL)
MySQL 서버에서는 데이터가 변경되는 도중에도 스키마 변경을 할 수 있도록 "Online DDL"이라는 기능을 제공한다.
하지만 모든 스키마 변경이 온라인으로 가능한 것은 아니며, 변경 작업의 특성에 따라 SELECT는 가능하지만 INSERT나 UPDATE 같은 데이터 변경은 허용되지 않을 수도 있다.
VARCHAR 데이터 타입을 사용하는 칼럼의 길이를 늘리는 작업은 길이에 따라 매우 빠르게 처리될 수도 있지만 어떤 경우에는 테이블에 대해 읽기 잠금을 걸고 레코드를 복사하는 작업이 필요할 수도 있다.

다음과 같이 길이가 60으로 정의된 VARCHAR 타입의 칼럼을 가진 테이블에서 확장하는 길이에 따른 ALTER TABLE 명령의 결과를 살펴보자.

```
mysql> CREATE TABLE test (
         id INT PRIMARY KEY,
         value VARCHAR(60)
       ) DEFAULT CHARSET=utf8mb4;
```

value 칼럼의 길이를 63으로 늘리는 경우와 64로 늘리는 경우 Online DDL 명령의 결과는 다음과 같다.

```
mysql> ALTER TABLE test MODIFY value VARCHAR(63), ALGORITHM=INPLACE, LOCK=NONE;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER TABLE test MODIFY value VARCHAR(64), ALGORITHM=INPLACE, LOCK=NONE;
ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Cannot change column type
INPLACE. Try ALGORITHM=COPY.
```

칼럼의 타입을 VARCHAR(63)으로 늘리는 경우는 잠금 없이(LOCK=NONE) 매우 빠르게 변경된 것을 확인할 수 있다.
하지만 칼럼의 타입을 VARCHAR(64)로 늘리는 경우는 INPLACE 알고리즘으로 스키마 변경이 허용되지 않는다는 것을 알 수 있다.
그래서 VARCHAR(64)로 변경하는 경우에는 다음과 같이 COPY 알고리즘으로 스키마 변경을 실행했으며, 스키마 변경 시간도 상당히 많이 걸리게 된다.
그뿐만 아니라 INPLACE 알고리즘의 스키마 변경은 잠금 없이 실행되지만 COPY 알고리즘의 스키마 변경은 읽기 잠금(LOCK=SHARED)까지 필요로 한다.
즉, 스키마 변경을 하는 동안 test 테이블에는 INSERT나 UPDATE, DELETE를 실행할 수 없게 된다.

```
mysql> ALTER TABLE test MODIFY value VARCHAR(64), ALGORITHM=COPY, LOCK=SHARED;
Query OK, 1000000 rows affected (36.12 sec)
```

이러한 차이가 발생하는 이유는 VARCHAR 타입의 칼럼이 가지는 길이 저장 공간의 크기 때문이다.
utf8mb4 문자 집합을 사용하는 VARCHAR(60) 칼럼은 최대 길이가 240(60 * 4)바이트이기 때문에 문자열 값의 길이를 저장하는 공간은 1바이트면 된다.
하지만 VARCHAR(64) 타입은 저장할 수 있는 문자열의 크기가 최대 256바이트까지 가능하기 때문에 문자열 길이를 저장하는 공간의 크기가 2바이트로 바뀌어야 한다.
이처럼 문자열 길이를 저장하는 공간의 크기가 바뀌게 되면 MySQL 서버는 스키마 변경을 하는 동안 읽기 잠금(LOCK=SHARED)을 걸어서 아무도 데이터를 변경하지 못하도록 막고 테이블의 레코드를 복사하는
방식으로 처리한다.

이러한 이유로 문자열 타입의 칼럼을 설계할 때는 앞으로 요건이 바뀌어서 VARCHAR 타입의 길이가 크게 변경될 것으로 예상된다면 길이 저장 공간의 크기가 바뀌지 않도록 미리 조금 크게 설계하는 것이 좋다.
레코드 건수가 많은 테이블에서 읽기 잠금을 필요로 하는 스키마 변경을 실행하기 위해서는 스키마를 변경할 때마다 서비스를 점검 모드로 바꿔야 할 수도 있으며, 이는 서비스의 가용성을 훼손하게 된다.
<br/>
<br/>
## (3) 문자 집합(캐릭터 셋)
MySQL 서버에서 각 테이블의 칼럼은 모두 서로 다른 문자 집합을 사용해 문자열 값을 저장할 수 있다. 문자 집합은 문자열을 저장하는 CHAR, VARCHAR, TEXT 타입의 칼럼에만 설정할 수 있다.
MySQL에서 최종적으로는 칼럼 단위로 문자 집합을 관리하지만 관리의 편의를 위해 MySQL 서버와 DB, 그리고 테이블 단위로 기본 문자 집합을 설정할 수 있는 기능을 제공한다.
즉 테이블의 문자 집합을 UTF-8로 설정하면 칼럼의 문자 집합을 별도로 지정하지 않아도 해당 테이블에 속한 칼럼은 UTF-8 문자 집합을 사용한다.
물론 테이블의 기본 문자 집합이 UTF-8이라고 하더라도 각 칼럼에 대해 문자 집합을 EUC-KR이나 ASCII 등으로 별도 지정할 수 있다.

한글 기반의 서비스에서는 euckr 또는 utf9mb4 문자 집합을 사용하며, 일본어인 경우에는 cp932 또는 utf8mb4를 적용하는 것이 일반적이다.
한글 윈도우에서 기본적으로 사용되는 MS949(MSWIN949) 문자 집합은 EUC-KR보다는 조금 더 확장된 형태의 문자 집합으로, 유닉스 계열의 운영체제에서 사용하는 CP949와 똑같은 문자 집합이다.
MySQL 서버에서는 별도로 CP949라는 이름의 문자 집합은 지원하지 않고, EUC-KR만 지원한다. 실제로 CP949는 EUC-KR보다 더 많은 문자를 표현할 수 있는 문자 집합이다.
하지만 MySQL 5.5 버전부터는 euckr 문자 집합이 보완되어 CP949가 표현하는 모든 문자 집합을 지원하므로 CP949 대신 euckr을 사용해도 아무런 문제없이 사용할 수 있다.

최근의 웹 서비스나 스마트폰 애플리케이션(앱, App)은 여러 나라의 언어를 동시에 지원하기 위해 기본적으로 UTF-8 문자 집합(utf8mb4)을 사용하는 추세다.
ANSI 표준에서는 하나의 문자 집합만 기본으로 사용할 수 있는 DB에서 다국어를 지원할 수 있게끔 NCHAR 또는 NATIONAL CHAR와 같은 칼럼 타입을 정의하고 있다.
MySQL에서도 NCHAR 타입을 지원하지만 기본적으로 MySQL에서는 칼럼 단위로 문자 집합을 선택할 수 있기 때문에 NCHAR 타입을 사용할 필요는 없다.
MySQL에서 NCHAR 타입을 사용하면 UTF-8 문자 집합을 사용하는 CHAR 타입으로 생성된다.

MySQL 서버에서 사용 가능한 문자 집합은 다음과 같이 "SHOW CHARACTER SET" 명령으로 확인해 볼 수 있다.

```
mysql> SHOW CHARACTER SET;
+----------+---------------------------+---------------------+--------+
| Charset  | Description               | Default collation   | Maxlen |
+----------+---------------------------+---------------------+--------+
| ascii    | US ASCII                  | ascii_general_ci    |      1 |
| binary   | Binary pseudo charset     | binary              |      1 |
| cp932    | SJIS for Windows Japanese | cp932_japanese_ci   |      2 |
| eucjpms  | UJIS for Windows Japanese | eucjpms_japanese_ci |      3 |
| euckr    | EUC-KR Korean             | euckr_korean_ci     |      2 |
| latin1   | cp1252 West European      | latin1_swedish_ci   |      1 |
| utf8     | UTF-8 Unicode             | utf8_general_ci     |      3 |
| utf8mb4  | UTF-8 Unicode             | utf8mb4_0900_ai_ci  |      4 |
...
+----------+---------------------------+---------------------+--------+
```

출력 내용을 확인해보면 여러 가지 문자 집합이 다양하게 사용되고 있다. 하지만 한국에서 MySQL을 사용한다면 대부분 euckr이나 utf8mb4만으로도 충분할 것이다.

- latin 계열의 문자 집합은 알파벳이나 숫자, 그리고 키보드의 특수 문자로만 구성된 문자열만 저장해도 될 때 저장 공간을 절약하면서 사용할 수 있는 문자 집합이다(대부분 해시 값이나 16진수로 구성된
  "Hex String" 또는 단순한 코드 값을 저장하는 용도로 사용한다).
- euckr은 한국어 전용으로 사용되는 문자 집합이며, 모든 글자는 1\~2바이트를 사용한다.
- utf8mb4은 다국어 문자를 포함할 수 있는 칼럼에 사용하기에 적합하다. 칼럼의 문자 집합이 utf8mb4로 생성되면 일반적으로 디스크에 저장할 때는 한 글자를 저장하기 위해 1\~4바이트까지 사용한다.
  하지만 utf8mb4 문자 집합을 사용하는 문자열 값이 메모리에 기록(MEMORY 테이블이나 정렬 버퍼 등과 같은 용도에서)될 때는 실제 문자열의 길이와 관계없이 문자당 4바이트로 공간이 할당되는 경우도 있다.
- Utf8은 utf8mb4의 부분 집합인데, MySQL 서버에서 utf8mb4가 도입되기 이전에 주로 사용됐다. utf8 문자 집합은 한 글자를 저장하기 위해 1\~3바이트까지 사용한다.
  한 글자를 저장하기 위한 공간이 최대 3바이트이기 때문에 utf8mb4보다는 저장할 수 있는 문자의 범위가 좁다.

SHOW CHARACTER SET 명령의 결과에서 "Default collation" 칼럼에는 해당 문자 집합의 기본 콜레이션이 무엇인지 표시해준다.
기본 콜레이션이란 칼럼에 콜레이션은 명시하지 않고 문자 집합만 지정했을 때 설정되는 콜레이션을 의미한다.

MySQL에서는 문자 집합을 설정하는 시스템 변수가 여러 가지가 있는데, 모두 제각기 목적이 다르므로 주의해야 한다.
간략하게 MySQL에서 설정 가능한 문자 집합 관련 변수를 살펴보고, 서로 어떻게 상호 작용하는지 살펴보자.

- #### character_set_system
  MySQL 서버가 식별자(Identifier, 테이블명이나 칼럼명 등)를 저장할 때 사용하는 문자 집합이다. 이 값은 기본적으로 utf8로 설정되며, 사용자가 설정하거나 변경할 필요가 없다.

- #### character_set_server
  MySQL 서버의 기본 문자 집합이다. DB나 테이블 또는 칼럼에 아무런 문자 집합이 설정되지 않을 때 이 시스템 변수에 명시된 문자 집합이 기본으로 사용된다. 이 시스템 변수의 기본값은 utf8mb4다.

- #### character_set_database
  MySQL DB의 기본 문자 집합이다. DB를 생성할 때 아무런 문자 집합이 명시되지 않았다면 이 시스템 변수에 명시된 문자 집합이 기본값으로 사용된다.
  이 변수가 정의되지 않으면 character_set_server 시스템 변수에 명시된 문자 집합이 기본으로 사용된다. 이 시스템 변수의 기본값은 utf8mb4이다.

- #### character_set_filesystem
  LOAD DATA INFILE ... 또는 SELECT ... INTO OUTFILE 문장을 실행할 때 인자로 지정되는 파일의 이름을 해석할 때 사용되는 문자 집합이다.
  여기서 주의해야 할 것은 데이터 파일의 내용을 읽을 때 사용하는 문자 집합이 아니라 파일의 이름을 찾을 때 사용하는 문자 집합이라는 점이다.
  이 설정값은 각 커넥션에서 임의의 문자 집합으로 변경해서 사용할 수 있다.
  기본값은 binary인데, LOAD DATA INFILE 명령이나 SELECT INTO OUTFILE 명령에서 파일명을 제대로 인식하지 못한다면 character_set_filesystem 시스템 변수를 utf8mb4로 변경하는 것이
  좋다.

- #### character_set_client
  MySQL 클라이언트가 보낸 SQL 문장은 character_set_client에 설정된 문자 집합으로 인코딩해서 MySQL 서버로 전송한다. 이 값은 각 커넥션에서 임의의 문자 집합으로 변경해서 사용할 수 있다.
  기본값은 utf8mb4이다.
  SQL 문장에서 리터럴에 대해 인트로듀서("_utf8mb4 'string_value'" 형태로 문자열 리터럴에 문자 셋을 설정하는 방법)가 사용된 경우에는 character_set_client 시스템 변수와 무관하게
  개별적으로 설정된 문자 셋이 적용된다.

- #### character_set_connection
  MySQL 서버가 클라이언트로부터 전달받은 SQL 문장을 처리하기 위해 character_set_connection 문자 집합으로 변환한다.
  또한 클라이언트로부터 전달받은 숫자 값을 문자열로 변환할 때도 character_set_connection에 설정된 문자 집합이 사용된다.
  이 변숫값 또한 각 커넥션에서 임의의 문자 집합으로 변경해서 사용할 수 있다. 기본값은 utf8mb4다.

- #### character_set_results
  MySQL 서버가 쿼리의 처리 결과를 클라이언트로 보낼 때 사용하는 문자 집합을 설정하는 시스템 변수다. 이 시스템 변수도 각 커넥션에서 임의의 문자 집합으로 변경해서 사용할 수 있다.
  기본값은 utf8mb4다.

#### [그림 15.3] 문자 집합의 적용 범위 및 클라이언트와 서버 간의 문자 집합 변환
<img src="https://github.com/user-attachments/assets/4b17b292-7805-4cf9-bd0a-7b5eea339465" width="750"/><br/>

### [1] 클라이언트로부터 쿼리를 요청했을 때의 문자 집합 변환
MySQL 서버는 클라이언트로부터 받은 메시지(SQL 문장과 변숫값)가 character_set_client에 지정된 문자 집합으로 인코딩돼 있다고 판단하고, 받은 문자열 데이터를 character_set_connection에
정의된 문자 집합으로 변환한다. 하지만 SQL 문장에 별도의 문자 집합이 지정된 리터럴(문자열)은 변환 대상에 포함하지 않는다.

SQL 문장에서 별도로 문자 집합을 설정하는 지정자를 "인트로듀서"라고 하며, 사용법은 다음과 같다.

```
mysql> SELECT emp_no, first_name FROM employees WHERE first_name='Matt';
mysql> SELECT emp_no, first_name FROM employees WHERE first_name=_latin1'Matt';
```

첫 번째 쿼리에서 first_name 칼럼의 비교 조건으로 사용한 "Matt" 문자열은 character_set_connection으로 문자 집합이 변환된 이후 처리될 것이다.
하지만 두 번째 쿼리는 인트로듀서(_latin1)가 사용됐으므로 "Matt" 문자열은 character_set_connection이 아니라 latin1 문자 집합으로 first_name 칼럼의 값과의 비교가 실행된다.
일반적으로 인트로듀서는 "_utf8mb4" 또는 "_latin1"과 같이 문자열 리터럴 앞에 언더스코어 기호("\_")와 문자 집합의 이름을 붙여서 표현한다.

### [2] 처리 결과를 클라이언트로 전송할 때의 문자 집합 변환
character_set_connection에 정의된 문자 집합으로 변환해 SQL을 실행한 다음, MySQL 서버는 쿼리의 결과(결과 셋이나 에러 메시지)를 character_set_results 변수에 설정된 문자 집합으로
변환해 클라리언트로 전송한다. 이때 결과 셋에 포함된 칼럼의 값이나 칼럼명과 같은 메타데이터도 모두 character_set_reulsts로 인코딩되어 클라이언트로 전송된다.

전체 과정에서 변환 전의 문자 집합과 변환해야 할 문자 집합이 똑같다면 별도의 문자 집합 변환 작업은 모두 생략한다.
예를 들어, 쿼리를 MySQL 서버로 전송할 때 character_set_client와 character_set_connection의 문자 집합이 똑같이 utf8mb4라면 MySQL 서버가 클라이언트로부터 쿼리 요청을 받아도 문자
집합은 변환되지 않는다. 결과를 클라이언트로 전송할 때도 character_set_results와 칼럼의 문자 집합이 똑같다면 별도의 변환이 필요하지 않다.
여기서 문자 집합이라고만 표현했지만 문자 집합은 물론이고 콜레이션까지 포함해서 같은지 다른지를 비교한다.

character_set_client와 character_set_results, character_set_connection이라는 3개의 시스템 설정 변수에 대해서는 클라이언트 프로그램이나 클라이언트 GUI 도구에서 마음대로 변경할
수 있다. 즉, 이 시스템 설정 변수는 모두 세션 변수이면서 동적 변수다. 다음과 같이 이들 변수의 값을 한 번에 설정하거나 개별적으로 변경할 수 있다.

```
mysql> SET character_set_client = 'utf8mb4';
mysql> SET character_set_results = 'utf8mb4';
mysql> SET character_set_connection = 'utf8mb4';

mysql> SET NAMES utf8mb4;
mysql> CHARSET utf8mb4;
```

위의 예제에서 처음 3개의 SET 명령은 각 설정값을 개별적으로 변경하는 명령이며, 나머지 2개의 명령은 3개의 설정값을 한 번에 변경할 수 있는 명령이다. 예제 아래 쪽의 명령 2개도 조금 차이가 있다.
SET NAMES 명령은 현재 접속된 커넥션에서만 유효하지만 CHARSET 명령은 MySQL 클라이언트 프로그램이 재시작되지 않은 상태에서 재접속할 때도 문자 집합 설정이 유효하게 만들어 준다.
<br/>
<br/>
## (4) 콜레이션(Collation)
콜레이션은 문자열 칼럼의 값에 대한 비교나 정렬 순서를 위한 규칙을 의미한다.
즉, 비교나 정렬 작업에서 영문 대소문자를 같은 것으로 처리할지, 아니면 더 크거나 작은 것으로 판단할지에 대한 규칙을 정의하는 것이다.

MySQL의 모든 문자열 타입의 칼럼은 독립적인 문자 집합과 콜레이션을 가진다. 각 칼럼에 대해 독립적으로 문자 집합이나 콜레이션을 지정하든 그렇지 않든 독립적인 문자 집합과 콜레이션을 가지는 것이다.
각 칼럼에 대해 독립적으로 지정하지 않으면 MySQL 서버나 DB의 기본 문자 집합과 콜레이션이 자동으로 설정된다. 콜레이션이란 문자열 칼럼의 값을 비교하거나 정렬하는 기준이 된다.
그래서 각 문자열 칼럼의 값을 비교하거나 정렬할 때는 항상 문자 집합뿐 아니라 콜레이션의 일치 여부에 따라 결과가 달라지며, 쿼리의 성능 또한 상당한 영향을 받는다.

### [1] 콜레이션 이해
문자 집합은 2개 이상의 콜레이션을 갖고 있는데, 하나의 문자 집합에 속한 콜레이션은 다른 문자 집합과 공유해서 사용할 수 없다.
또한 테이블이나 칼럼에 문자 집합만 지정하면 해당 문자 집합의 디폴트 콜레이션이 해당 칼럼의 콜레이션으로 지정된다.
반대로 칼럼의 문자 집합은 지정하지 않고 콜레이션만 지정하면 해당 콜레이션이 소속된 문자 집합이 묵시적으로 그 칼럼의 문자 집합으로 사용된다.
MySQL 서버에서 사용 가능한 콜레이션의 목록은 "SHOW COLLATIONS" 명령을 이용해 다음과 같이 확인할 수 있다.

```
mysql> SHOW COLLATIONS;
+-------------------------+----------+-----+---------+----------+---------+---------------+
| Collation               | Charset  | Id  | Default | Compiled | Sortlen | Pad_attribute |
+-------------------------+----------+-----+---------+----------+---------+---------------+
| ascii_bin               | ascii    |  65 |         | Yes      |       1 | PAD SPACE     |
| ascii_general_ci        | ascii    |  11 | Yes     | Yes      |       1 | PAD SPACE     |
| euckr_bin               | euckr    |  85 |         | Yes      |       1 | PAD SPACE     |
| euckr_korean_ci         | euckr    |  19 | Yes     | Yes      |       1 | PAD SPACE     |
| latin1_bin              | latin1   |  47 |         | Yes      |       1 | PAD SPACE     |
| latin1_general_ci       | latin1   |  48 |         | Yes      |       1 | PAD SPACE     |
| latin1_general_cs       | latin1   |  49 |         | Yes      |       1 | PAD SPACE     |
| utf8mb4_0900_ai_ci      | utf8mb4  | 255 | Yes     | Yes      |       0 | NO PAD        |
| utf9mb4_0900_as_ci      | utf8mb4  | 305 |         | Yes      |       0 | NO PAD        |
| utf8mb4_0900_as_cs      | utf8mb4  | 278 |         | Yes      |       0 | NO PAD        |
| utf8mb4_0900_bin        | utf8mb4  | 309 |         | Yes      |       1 | NO PAD        |
| utf8mb4_bin             | utf8mb4  |  46 |         | Yes      |       1 | PAD SPACE     |
| utf8mb4_general_ci      | utf8mb4  |  45 |         | Yes      |       1 | PAD SPACE     |
| utf8mb4_unicode_ci      | utf8mb4  | 224 |         | Yes      |       8 | PAD SPACE     |
| utf8_bin                | utf8     |  83 |         | Yes      |       1 | PAD SPACE     |
| utf8_general_ci         | utf8     |  33 | Yes     | Yes      |       1 | PAD SPACE     |
| utf8_unicode_520_ci     | utf8     | 214 |         | Yes      |       8 | PAD SPACE     |
| utf8_unicode_ci         | utf8     | 192 |         | Yes      |       8 | PAD SPACE     |
...
+-------------------------+----------+-----+---------+----------+---------+---------------+
```

일반적으로 콜레이션의 이름은 2개 또는 3개의 파트로 구분돼 있으며, 각 파트는 다음과 같은 의미로 사용된다.

- 3개의 파트로 구성된 콜레이션 이름
  - 첫 번째 파트는 문자 집합의 이름이다.
  - 두 번째 파트는 해당 문자 집합의 하위 분류를 나타낸다.
  - 세 번째 파트는 대문자나 소문자의 구분 여부를 나타낸다.
    즉, 세 번째 파트가 "ci"이면 대소문자를 구분하지 않는 콜레이션(Case Insensitive)을 의미하며, "cs"이면 대소문자를 별도의 문자로 구분하는 콜레이션(Case Sensitive)이다.

- 2개의 파트로 구성된 콜레이션 이름
  - 첫 번째 파트는 마찬가지로 문자 집합의 이름이다.
  - 두 번째 파트는 항상 "bin"이라는 키워드가 사용된다. 여기서 "bin"은 이진 데이터(binary)를 의미하며, 이진 데이터로 관리되는 문자열 칼럼은 별도의 콜레이션을 가지지 않는다.
    콜레이션이 "xxx_bin"이라면 비교 및 정렬은 실제 문자 데이터의 바이트 값을 기준으로 수행된다.

대부분 문자 집합의 콜레이션 이름은 2\~3개의 파트로 구분돼 있었는데, utf8mb4 문자 집합의 콜레이션은 이름이 더 많이 복잡해졌다.
utf8mb4 문자 집합의 콜레이션 중에서 "utf8mb4_0900"으로 시작하는 콜레이션에서 "0900"은 UCA(Unicode Collation Algorithm)의 버전을 의미한다.
UCA는 문자 비교 규칙 정도로 이해하면 된다. 예를 들어, utf8mb4_unicode_520_ci 콜레이션에서 "520"은 UCA 5.2.0을 의미한다. 참고로 2020년 기준 UCA의 최종 버전은 13.0.0이다.
UCA의 버전이 올라갈수록 문자 간의 정렬 순서와 비교 규칙은 더 정교해지고 언어별 특성이 더 반영된 버전이라고 볼 수 있다.

utf8mb4 문자 집합에서는 액센트 문자의 구분 여부가 콜레이션의 이름에 포함됐다.
예를 들어, "utf8mb4_0900_ai_ci"와 같이 "ai" 또는 "as"를 포함하는 경우가 있는데, 이는 액센트를 가진 문자(Accent Sensitive)와 그렇지 않은 문자(Accent Insensitive)들을 정렬 순서상
동일 문자로 판단할지 여부를 나타낸다. "ai"라면 다음의 5개 문자는 정렬 순서상 동일하게 취급된다.
여기서 한 가지 주의할 것은 콜레이션이 정렬 순서에만 영향을 미치는 것이 아니라 동일 문자인지 아닌지의 검색 결과에도 영향을 미친다는 점이다. 정렬의 크다 작다는 개념 자체가 비교의 결과이기 때문이다.

```
e, è, é, ê, ë
```

> 콜레이션이 대소문자나 액센트 문자를 구분하지 않는다고 해서 실제 칼럼에 저장되는 값이 모두 소문자나 대문자로 변환되어 저장되는 것은 아니며, 콜레이션과 관계없이 입력된 데이터의 대소문자는 별도의 변환
> 없이 그대로 저장된다.
>
> MySQL 서버에서 문자열의 정렬이나 검색을 위한 비교 작업이 단순히 저장된 문자열 값의 인코딩된 바이트 값(이를 Code Point라고 한다)으로 비교한다고 생각하는 사용자가 많은데, 이는 사실이 아니다.
> MySQL 서버는 인코딩된 상태로 저장된 문자열을 가져와 각 인코딩된 바이트 값에 해당하는 콜레이션 값으로 매칭시킨 다음 비교를 수행하게 된다.
> 즉, 저장된 문자열의 바이트 값은 직접적인 비교 대상이 아니다.

자주 사용하는 latin1이나 euckr, utf8mb4 문자 집합의 디폴트 콜레이션은 각각 latin1_swedish_ci, euckr_korean_ci, utf8mb4_0900_ai_ci다.
이들은 모두 대소문자를 구분하지 않는 콜레이션이라서 대소문자를 구분해서 비교하거나 정렬해야 하는 칼럼에서는 "_cs" 계열의 콜레이션을 명시적으로 지정해야 한다.
하지만 utf8mb4 문자 집합이나 euckr과 같이 별도로 "_cs" 계열의 콜레이션을 가지지 않는 문자 집합도 있는데, 이때는 utf8mb4_bin 또는 euckr_bin과 같이 "_bin" 계열의 콜레이션을 사용하면
된다. 일반적으로 각 국가의 언어는 그 나라 국민에게 익숙한 순서대로 문자 코드 값이 부여돼 있으므로 대소문자를 구분할 때는 "_bin" 계열의 콜레이션을 적용해도 특별히 문제되지는 않는다.

MySQL의 문자열 칼럼은 콜레이션 없이 문자 집합만 가질 수는 없다. 콜레이션을 명시적으로 지정하지 않았다면 지정된 문자 집합의 기본 콜레이션이 묵시적으로 적용된다.
그리고 문자열 칼럼의 정렬이나 비교는 항상 해당 문자열 칼럼의 콜레이션에 의해 판단하므로 문자열 칼럼에서는 CHAR나 VARCHAR 같은 타입의 이름과 길이만 같다고 해서 똑같은 타입이라고 판단해서는 안 된다.
타입의 이름과 문자열의 길이, 문자 집합과 콜레이션까지 일치해야 똑같은 타입이라고 할 수 있다.
문자열 칼럼에서는 문자 집합과 콜레이션이 모두 일치해야만 조인이나 WHERE 조건이 인덱스를 효율적으로 사용할 수 있다.
조인을 수행하는 양쪽 테이블의 칼럼이 문자 집합이나 콜레이션이 다르다면 비교 작업에서 콜레이션의 변환이 필요하기 때문에 인덱스를 효율적으로 이용하지 못할 때가 많으므로 주의해야 한다.

테이블을 생성할 때 문자 집합이나 콜레이션을 적용하는 방법을 한 번 살펴보자.

```
mysql> CREATE DATABASE db_test CHARACTER SET=utf8mb4;

mysql> CREATE TABLE tb_member (
         member_id VARCHAR(20) NOT NULL COLLATE latin1_general_cs,
         member_name VARCHAR(20) NOT NULL COLLATE utf8_bin,
         member_email VARCHAR(100) NOT NULL,
         ...
       );
```

문자 집합이나 콜레이션은 DB 수준에서 설정할 수도 있으며, 테이블 수준으로 설정할 수도 있다. 그리고 마지막으로 칼럼 수준에서도 개별적으로 설정할 수 있다.

- 첫 번째 "CREATE DATABASE" 명령으로 기본 문자 집합이 utf8mb4인 DB를 생성한다.
  이 명령에서 콜레이션은 명시적으로 정의하지 않았지만 utf8mb4의 기본 콜레이션인 utf8mb4_0900_ai_ci가 기본적인 콜레이션이 된다.
  db_test DB 내에서 생성되는 테이블이나 칼럼 중에서 별도로 문자 집합과 콜레이션을 정의하지 않으면 모두 utf8mb4 문자 집합과 utf8mb4_0900_ai_ci 콜레이션을 사용하도록 자동 설정된다.

- 두 번째 "CREATE TABLE" 명령에서는 각 칼럼이 서로 다른 문자 집합이나 콜레이션을 사용하게 정의했다. 각 칼럼의 비교나 정렬 특성을 살펴보자.

  - tb_member 테이블을 생성하면서 member_id 칼럼의 콜레이션을 latin1_general_cs로 설정했다.
    그래서 member_id 칼럼은 숫자나 영문 알파벳, 키보드의 특수 문자 위주로만 저장할 수 있고, "_cs" 계열의 콜레이션이므로 대소문자 구분을 하는 정렬이나 비교를 수행한다.
  - member_name 칼럼은 콜레이션이 utf8_bin으로 설정됐으므로 한글이나 다른 나라의 언어를 사용할 수 있지만 "_bin" 계열의 콜레이션이 사용됐으므로 대소문자를 구분하는 정렬과 비교를 수행한다.
  - member_email 칼럼은 아무런 문자 집합이나 콜레이션을 정의하지 않았으므로 DB의 기본 문자 집합과 콜레이션을 그대로 사용한다.
    그래서 member_email 칼럼은 utf8mb4_0900_ai_ci 콜레이션을 사용하고, 비교나 정렬 시 대소문자를 구분하지 않는다.
    <br/>

대표적으로 latin 계열의 문자 집합에 대해 _ci, _cs, _bin 콜레이션의 정렬 규칙을 테스트해 보기 위해 tb_collate 테이블에 여러 종류의 콜레이션을 섞어서 테이블을 생성해봤다.
칼럼명은 이해를 위해 콜레이션의 이름을 포함해서 생성했다.

```
mysql> CREATE TABLE tb_collate (
         fd_latin1_general_ci VARCHAR(10) COLLATE latin1_general_ci,
         fd_latin1_general_cs VARCHAR(10) COLLATE latin1_general_cs,
         fd_latin1_bin VARCHAR(10) COLLATE latin1_bin,
         fd_latin7_general_ci VARCHAR(10) COLLATE latin7_general_ci
       );

mysql> INSERT INTO tb_collate VALUES
         ('a','a','a','a'), ('A','A','A','A'), ('b','b','b','b'), ('B','B','B','B'),
         ('_','_','_','_'), ('-','-','-','-'), ('.','.','.','.'), ('~','~','~','~');
```

테이블에서 칼럼별로 정렬한 결과를 한 번 확인해 보자.

```
mysql> SELECT fd_latin1_general_ci FROM tb_collate ORDER BY fd_latin1_general_ci;
+----------------------+
| fd_latin1_general_ci |
+----------------------+
| -                    |
| .                    |
| a                    |
| A                    |
| b                    |
| B                    |
| _                    |
| ~                    |
+----------------------+

mysql> SELECT fd_latin1_general_cs FROM tb_collate ORDER BY fd_latin1_general_cs;
+----------------------+
| fd_latin1_general_cs |
+----------------------+
| -                    |
| .                    |
| A                    |
| a                    |
| B                    |
| b                    |
| _                    |
| ~                    |
+----------------------+

mysql> SELECT fd_latin1_bin FROM tb_collate ORDER BY fd_latin1_bin;
+---------------+
| fd_latin1_bin |
+---------------+
| -             |
| .             |
| A             |
| B             |
| _             |
| a             |
| b             |
| ~             |
+---------------+

mysql> SELECT fd_latin7_general_ci FROM tb_collate ORDER BY fd_latin7_general_ci;
+----------------------+
| fd_latin7_general_ci |
+----------------------+
| -                    |
| .                    |
| _                    |
| ~                    |
| a                    |
| A                    |
| b                    |
| B                    |
+----------------------+
```

위의 예제 쿼리에서 각 정렬이 어떻게 수행됐고 각 콜레이션에서 주의할 사항으로 어떤 것이 있는지 살펴보자.

- 첫 번째 예제는 latin1_general_ci 콜레이션을 사용하는 칼럼을 기준으로 정렬했다.
  출력된 정렬 순서로 보면 'a'와 'A' 중에서 소문자가 먼저인 것처럼 보이지만 사실 대소문자 구분이 없이 정렬된 것이다.
- 두 번째 예제는 latin1_general_cs 콜레이션으로 정렬한 것인데, 대문자 'A'와 소문자 'a'는 모두 대문자 'B'보다 먼저 정렬됐다. 그런데 같은 알파벳에서는 대문자가 소문자보다 먼저 정렬됐다.
- 세 번째 예제는 latin1_bin 콜레이션으로 정렬한 예제로 대문자만 먼저 정렬되고 그다음으로 소문자가 정렬됐다.
- 네 번째 예제는 조금 다른 성격의 정렬인데, 첫 번째부터 세 번째 정렬은 모두 특수 문자의 정렬 위치가 알파벳의 앞뒤로 분산돼 있다.
  그런데 특수문자만 먼저 정렬하고 알파벳을 그다음으로 정렬하기를 원할 수도 있다. 이때는 latin1이 아니라 latin7 문자셋을 사용하면 특수문자가 알파벳보다 먼저 정렬된다.

때로는 WHERE 조건의 검색은 대소문자를 구분하지 않고 실행하되 정렬은 대소문자를 구분해서 해야 할 때도 있는데, 이때는 검색과 정렬 작업 중 하나는 인덱스를 이용하는 것을 포기할 수밖에 없다.
주로 이때는 칼럼의 콜레이션을 "_ci"로 만들어 검색은 인덱스를 충분히 이용할 수 있게 하고, 정렬 작업은 인덱스를 사용하지 않는 명시적인 정렬(Using filesort) 형태로 처리하는 것이 일반적이다.
검색과 정렬 모두 인덱스를 이용하려면 정렬을 위한 콜레이션을 사용하는 칼럼을 하나 더 추가하고 검색은 원본 칼럼을, 그리고 정렬은 복사된 추출 칼럼을 이용하는 방법도 생각해볼 수 있다.
데이터의 양이나 업무의 중요도를 적절히 반영해 방법을 선택하면 될 것이다.

테이블의 구조는 "SHOW CREATE TABLE" 명령으로 확인할 수 있다. 어떤 칼럼이 디폴트 문자 집합이나 콜레이션을 사용할 때는 별도로 표시해주지 않으므로 조금 분석하기가 어려울 수 있다.
각 칼럼의 문자 집합이나 콜레이션을 정확히 확인하려면 information_schema DB의 COLUMNS 뷰를 확인해 보면 된다.

```
mysql> SELECT table_name, column_name,
              column_type, character_set_name, collate_name
       FROM information_schema.columns
       WHERE table_schema='test' AND table_name='tb_collate';

+------------+----------------------+-------------+--------------------+-------------------+
| TABLE_NAME | COLUMN_NAME          | COLUMN_TYPE | CHARACTER_SET_NAME | COLLATION_NAME    |
+------------+----------------------+-------------+--------------------+-------------------+
| tb_collate | fd_latin1_bin        | varchar(10) | latin1             | latin1_bin        |
| tb_collate | fd_latin1_general_ci | varchar(10) | latin1             | latin1_general_ci |
| tb_collate | fd_latin1_general_cs | varchar(10) | latin1             | latin1_general_cs |
| tb_collate | fd_latin7_general_ci | varchar(10) | latin7             | latin7_general_ci |
+------------+----------------------+-------------+--------------------+-------------------+
```

### [2] utf8mb4 문자 집합의 콜레이션
지금까지는 문자 집합의 콜레이션을 이해하기 위해 주로 latin 계열의 콜레이션을 살펴봤는데, 실제 응용 프로그램에서는 latin 계열 문자 집합은 아주 특별한 경우 이외에는 거의 사용되지 않는다.
그리고 최근에는 응용 프로그램의 다국어 지원이 필수적이어서 대부분은 utf8bm4 문자 집합을 사용할 것이다. 이제 utf8mb4 문자 집합에 대해 자세히 살펴보자.

우선 다음 4개의 콜레이션은 utf8mb3(utf8) 또는 utf8mb4 문자 집합의 콜레이션 중 하나다.
여기서 숫자 값이 포함된 콜레이션과 그렇지 않은 콜레이션을 볼 수 있는데, 콜레이션 이름의 숫자 값은 모두 콜레이션의 비교 알고리즘 버전이다.
첫 번째 utf8_unicode_ci와 같이 별도의 숫자 값이 명시돼 있지 않은 콜레이션은 UCA 버전 4.0.0을 의미한다.

|콜레이션|UCA 버전|
|:---|:---|
|utf8_unicode_ci|4.0.0|
|utf8_unicode_520_ci|5.2.0|
|utf8mb4_unicode_520_ci|5.2.0|
|utf8mb4_0900_ai_ci|9.0.0|

그리고 콜레이션의 이름에 로캘(Locale)이 포함돼 있는지 여부로 언어에 종속적인 콜레이션과 언어에 비종속적인 콜레이션으로 구분할 수 있다.

|콜레이션|언어|표기|
|:---|:---|:---|
|utf8mb4_0900_ai_ci|N/A|없음|
|utf8mb4_zh_0900_as_cs|중국어|zh|
|utf8mb4_la_0900_ai_ci|클래식 라틴|la 또는 roman|
|utf8mb4_de_pb_0900_ai_ci|독일 전화번호 안내 책자 순서|de_pb 또는 german2|
|utf8mb4_ja_0900_as_cs|일본어|ja|
|utf8mb4_ro_0900_ai_ci|로마어|ro 또는 romanian|
|utf8mb4_ru_0900_ai_ci|러시아어|ru|
|utf8mb4_es_0900_ai_ci|현대 스페인어|es 또는 spanish|
|utf8mb4_vi_0900_ai_ci|베트남|vi 또는 vietnamese|

utf8mb4_0900_ai_ci와 같이 언어 비종속적인 콜레이션은 문자 셋의 기본 정렬 순서에 의해 정렬 및 비교가 수행되며, 언어 종속적인 콜레이션은 해당 언어에서 정의한 정렬 순서에 의해 정렬 및 비교가
수행된다. 특정 언어에 종속적인 정렬 순서가 필요하다면 그 언어에 맞는 콜레이션을 선택해야 한다. 하지만 범용 응용 프로그램이라면 "utf8mb4_0900_ai_ci" 콜레이션으로도 충분할 것이다.

UCA 9.0.0 버전은 그 이전의 콜레이션보다 빠르다고 MySQL 매뉴얼에서 소개하고 있다. 하지만 실제로 간단히 테스트만 해봐도 그렇지 않다는 것을 확인할 수 있다.
또한 UCA 9.0.0 버전을 사용하는 콜레이션은 모두 "NO PAD" 옵션으로 문자열 비교 작업이 처리되기 때문에 특정 케이스에는 UCA 9.0.0 이전 버전의 콜레이션보다 더 빠르게 작동한다고 소개돼 있지만
이 부분도 크게 성능 영향은 없는 것으로 보인다. 다음의 테스트 결과를 보면 오히려 일반적으로 많이 사용되는 2개의 콜레이션 기준으로 볼 때 이전 버전의 콜레이션이 더 빠른 결과를 보였다.

```
mysql> SET NAMES utf8mb4 COLLATE utf8mb4_general_ci;
mysql> SELECT BENCHMARK(10000000, '한글입니까'='한글입니다');
1 row in set (0.58 sec)

mysql> SET NAMES utf8mb4 COLLATE utf8mb4_0900_ai_ci;
mysql> SELECT BENCHMARK(10000000, '한글입니까'='한글입니다');
1 row in set (1.71 sec)
```

위의 테스트 결과를 보고 단순히 성능 때문에 utf8mb4_general_ci 콜레이션을 선택하지는 말자.
위 테스트는 비교 횟수가 많아서 큰 차이를 보이는 것 같지만 실제로는 문자열 비교 한 번에 대략 0.1 마이크로초 차이밖에 되지 않는다.
콜레이션의 필요에 따라 결정해야 할 부분이지, 성능을 기준으로 콜레이션을 선택하지는 않도록 하자.

UCA 9.0.0 콜레이션은 최신 정렬 순서를 반영하고 있으므로 응용 프로그램의 요건에 따라 "utf8mb4_0900" 콜레이션을 선택해야 할 수도 있다.
하지만 "utf8mb4_0900" 콜레이션은 "NO PAD" 옵션으로 인해 문자열 뒤에 존재하는 공백도 유효 문자로 취급되어 비교되고, 이로 인해 기존과는 다른 비교 결과를 보일 수도 있으므로 주의해야 한다.
기존 UCA 9.0.0 이전 콜레이션을 사용 중인 응용 프로그램에 대해서는 문자 집합과 콜레이션을 변경하는 작업이 위험할 수 있으니 필요한 테스트를 진행한 후 콜레이션 변경을 진행하는 것이 좋다.
새로운 서비스를 개발하고 있다면 성능과 관계없이 UCA 9.0.0 기반의 콜레이션을 사용할 것을 권장한다.

MySQL 8.0 이전 버전에서 MySQL 8.0으로 업그레이드하는 경우 utf8mb4 문자 집합의 콜레이션에 주의해야 한다. "utf8mb4_0900" 콜레이션은 MySQL 8.0 버전에서 처음 도입됐다.
MySQL 5.7 버전에서는 utf8mb4 문자 집합의 기본 콜레이션이 "utf8mb4_general_ci"였는데, MySQL 8.0 버전부터는 utf8mb4 문자 집합의 기본 콜레이션이 "utf8mb4_0900_ai_ci"로 변경됐다.
그래서 MySQL 8.0 버전의 설정 파일(my.cnf)에 콜레이션에 대한 설정 없이 문자 집합에 대한 설정만 추가된 경우 다음과 같이 콜레이션이 utf8mb4_0900_ai_ci로 초기화된다.
다음은 MySQL 5.7과 MySQL 8.0에서 어떻게 차이가 나는지를 보여준다.

#### [MySQL 5.7]
```
mysql> SHOW GLOBAL VARIABLES LIKE '%character%';
+--------------------------+---------+
| Variable_name            | Value   |
+--------------------------+---------+
| character_set_client     | utf8mb4 |
| character_set_connection | utf8mb4 |
| character_set_database   | utf8mb4 |
| character_set_filesystem | utf8mb4 |
| character_set_results    | utf8mb4 |
| character_set_server     | utf8mb4 |
| character_set_system     | utf8    |
+--------------------------+---------+

mysql> SHOW GLOBAL VARIABLES LIKE 'collation%';
+-------------------------------+--------------------+
| Variable_name                 | Value              |
+-------------------------------+--------------------+
| collation_connection          | utf8mb4_general_ci |
| collation_database            | utf8mb4_general_ci |
| collation_server              | utf8mb4_general_ci |
+-------------------------------+--------------------+
```

#### [MySQL 8.0]
```
mysql> SHOW GLOBAL VARIABLES LIKE '%character%';
==> MySQL 5.7과 동일

mysql> SHOW GLOBAL VARIABLES LIKE 'collation%';
+-------------------------------+--------------------+
| Variable_name                 | Value              |
+-------------------------------+--------------------+
| collation_connection          | utf8mb4_0900_ai_ci |
| collation_database            | utf8mb4_0900_ai_ci |
| collation_server              | utf8mb4_0900_ai_ci |
+-------------------------------+--------------------+
```

이렇게 기본 콜레이션이 변경되면 이후부터 새로 생성되는 데이터베이스와 테이블들은 모두 utf8mb4_0900_ai_ci 콜레이션을 사용한다.
하지만 MySQL 5.7 버전부터 존재하던 테이블은 이미 utf8mb4_general_ci 콜레이션을 사용하고 있기 때문에 이 두 테이블은 조인을 할 때 에러가 발생하거나 성능이 심각하게 떨어진다.
이러한 문제를 해결할 수 있게 MySQL 서버는 `default_collation_for_utf8mb4` 시스템 변수를 제공한다.
default_collation_for_utf8mb4 시스템 변수에 "utf8mb4_general_ci"를 설정하면 문자 집합이 utf8mb4로 설정될 경우 콜레이션들도 utf8mb4_general_ci로 초기화된다.
하지만 default_collation_for_utf8mb4 시스템 변수는 일시적으로 제공되는 기능이므로 영구적으로 사용하기에는 불안할 수 있다.

당분간 utf8mb4_0900_ai_ci로 콜레이션을 변경할 예정이 없다면 MySQL 서버의 설정 파일(my.cnf)에 콜레이션 관련 시스테메 변수를 utf8mb4_general_ci로 고정해두는 것이 좋다.
그리고 MySQL 서버에 접속하는 응용 프로그램의 연결 문자열(Connection String)에도 connectionCollation 파라미터를 추가하는 것을 권장한다.
다음은 JDBC 드라이버의 연결 문자열에서 connectionCollation 파라미터를 설정한 예제다.

```
jdbc:mysql://dbms_server:3306/DB?connectionCollation=utf8mb4_general_ci
```

MySQL 서버의 콜레이션 관련 시스템 변수들이 모두 utf8mb4_general_ci로 고정됐다면 연결 문자열에서 문자 셋이나 콜레이션 관련 파라미터(useUnicode와 characterEncoding 같은 파라미터)를
모두 제거해주면 자동으로 클라이언트 드라이버가 MySQL 서버에 설정된 문자 집합과 콜레이션을 가져와서 사용한다.

MySQL 8.0으로 새로 시작하는 응용 프로그램이라면 utf8mb4_0900_ai_ci 콜레이션을 기본으로 사용할 것을 권장한다.
여러 설정 중에서 하나라도 잘못된 설정이 있다면 MySQL 서버는 다음과 같이 콜레이션이 달라서 비교할 수 없다는 에러를 발생시킬 것이다.

```
Error Code: 1267. Illegal mix of collations
  (utf8mb4_general_ci,IMPLICIT) and (utf8mb4_0900_ai_ci,IMPLICIT)
```
<br/>

## (5) 비교 방식
MySQL에서 문자열 칼럼을 비교하는 방식은 CHAR와 VARCHAR가 거의 같다. CHAR 타입의 칼럼에 SELECT를 실행했을 때 다른 DBMS처럼 사용되지 않는 공간에 공백 문자가 채워져서 나오지 않는다.
그리고 MySQL 서버에서 지원하는 대부분의 문자 집합과 콜레이션에서 CHAR 타입이나 VARCHAR 타입을 비교할 때 공백 문자를 뒤에 붙여서 두 문자열의 길이를 동일하게 만든 후 비교를 수행한다.
다음의 간단한 예제를 보면 더 쉽게 이해할 수 있을 것이다.

```
mysql> SELECT 'ABC'='ABC    ' AS is_equal;
+----------+
| is_equal |
+----------+
|        1 | ==> TRUE
+----------+

mysql> SELECT 'ABC'='    ABC' AS is_equal;
+----------+
| is_equal |
+----------+
|        0 | ==> FALSE
+----------+
```

첫 번째 예제 쿼리의 결과를 보면 "ABC"라는 문자열 뒤에 붙어 있는 3개의 공백은 있어도 없는 것처럼 비교했다는 것을 알 수 있다.
그리고 두 번째 쿼리의 결과를 보면 "ABC"라는 문자열의 앞쪽에 위치하는 공백 문자는 유효한 문자로 비교된다는 사실을 알 수 있다.
이러한 문자열 비교 방식은 문자열의 크다(>) 작다(<) 비교와 문자열 비교 함수인 STRCMP()에서도 똑같이 적용된다.

하지만 utf8mb4 문자 집합이 UCA 버전 9.0.0을 지원하면서 문자열 뒤에 붙어있는 공백 문자들에 대한 비교 방식이 달라졌다.
다음 예제 쿼리를 보면 utf8mb4_bin 콜레이션을 사용하는 경우 문자열 뒤에 붙어 있는 공백은 비교에 영향을 미치지 않는다.
하지만 utf8mb4_0900_bin 콜레이션을 사용하는 경우 문자열 뒤의 공백이 비교 결과에 영향을 미치는 것을 알 수 있다.

```
mysql> SET NAMES utf8mb4 COLLATE utf8mb4_bin;
mysql> SELECT 'a ' = 'a';
+------------+
| 'a ' = 'a' |
+------------+
|          1 |
+------------+

mysql> SET NAMES utf8mb4 COLLATE utf8mb4_0900_bin;
mysql> SELECT 'a ' = 'a';
+------------+
| 'a ' = 'a' |
+------------+
|          0 |
+------------+
```

이는 지금까지 MySQL 서버의 문자열 비교 규칙에 큰 영향을 미치는 변경이므로 utf8mb4 문자 집합을 사용하는 경우 주의해야 한다.
문자열 뒤의 공백이 비교 결과에 영향을 미치는지 아닌지는 다음과 같이 information_schema 데이터베이스의 COLLATIONS 뷰에서 PAD_ATTRIBUTE 칼럼의 값으로 판단할 수 있다.

```
mysql> SELECT collation_name, pad_attribute
       FROM information_schema.COLLATIONS
       WHERE collation_name LIKE 'utf8mb4%';
+----------------------------+---------------+
| collation_name             | pad_attribute |
+----------------------------+---------------+
| utf8mb4_general_ci         | PAD SPACE     |
| utf8mb4_bin                | PAD SPACE     |
| utf8mb4_unicode_ci         | PAD SPACE     |
| utf8mb4_0900_ai_ci         | NO PAD        |
| utf8mb4_0900_as_cs         | NO PAD        |
| utf8mb4_0900_as_ci         | NO PAD        |
| utf8mb4_0900_bin           | NO PAD        |
...
+----------------------------+---------------+
```

pad_attribute 칼럼의 값이 "PAD SPACE"라고 표시된 콜레이션에서는 비교 대상 문자열의 길이가 같아지도록 문자열 뒤에 공백을 추가해서 비교를 수행한다.
그리고 "NO PAD"로 표시된 콜레이션에서는 별도로 문자열의 길이를 일치시키지 않고 그대로 비교한다.
MySQL 서버에서 지원하는 대부분의 콜레이션은 "PAD SPACE"이며, "utf8mb4_0900"으로 시작하는 콜레이션만 "NO PAD"다.
이 같은 이유로 "utf8mb4_0900"으로 시작하는 콜레이션은 비교 대상 문자열의 길이가 많이 차이 나는 경우 더 빠른 비교 성능을 낸다.

문자열 비교의 경우 예외적으로 LIKE를 사용한 문자열 패턴 비교에서는 공백 문자가 유효 문자로 취급된다. LIKE 조건으로 비교하는 예제를 한 번 살펴보자.

```
mysql> SELECT 'ABC    ' LIKE 'ABC' AS is_same_pattern;
+-----------------+
| is_same_pattern |
+-----------------+
|               0 | ==> FALSE
+-----------------+

mysql> SELECT '    ABC' LIKE 'ABC' AS is_same_pattern;
+-----------------+
| is_same_pattern |
+-----------------+
|               0 | ==> FALSE
+-----------------+

mysql> SELECT 'ABC    ' LIKE 'ABC%' AS is_same_pattern;
+-----------------+
| is_same_pattern |
+-----------------+
|               1 | ==> TRUE
+-----------------+
```

위의 비교 예제를 보면 첫 번째와 두 번째 쿼리에서 문자열의 앞뒤에 있는 공백이 모두 유효한 문자 값으로 인식됐음을 알 수 있다.
그리고 실제 이런 값을 비교하려면 세 번째 쿼리와 같이 검색어 앞뒤로 와일드 카드("%") 문자를 사용해야 한다는 것을 알 수 있다.
MySQL의 독특한 문자열 비교 방식은 주로 회원의 아이디나 닉네임과 같이 다른 DBMS와 연동해야 하는 서비스에서 문제가 되곤 하므로 주의해야 한다.
<br/>
<br/>
## (6) 문자열 이스케이프 처리
MySQL에서 SQL 문장에 사용하는 문자열은 프로그래밍 언어처럼 "\"를 이용해 이스케이프 처리를 하는 것이 가능하다. 즉, "\t"나 "\n"으로 탭이나 개행문자를 표시할 수 있다.
각 특수문자를 어떻게 이스케이프 처리하는지 살펴보자.

|이스케이프 표기|의미|
|:---|:---|
|\0|아스키(ASCII) NULL 문자(0x00)|
|\\'|홑따옴표(')|
|\\"|쌍따옴표(")|
|\b|백스페이스 문자|
|\n|개행문자(라인 피드)|
|\r|캐리지 리턴 문자<br/>유닉스 계열 운영체제에서는 "\n"만 개행문자로 사용되며, 윈도우 계열 운영체제에서는 "\r\n"의 조합<br/>으로 개행문자를 사용한다.|
|\t|탭 문자|
|\\\\ |백 슬래시(\\) 문자|
|\\%|퍼센트(%) 문자(LIKE의 패턴에서만 사용함)|
|\\_|언더 스코어(_) 문자(LIKE의 패턴에서만 사용함)|

마지막의 "\\%"와 "\\_"는 LIKE를 사용하는 패턴 검색 쿼리의 검색어에서만 사용할 수 있다.
LIKE 패턴 검색에서는 "%"와 "\_"를 와일드 카드를 표현하기 위한 패턴 문자로 사용하므로 실제 "%" 문자나 "\_" 문자를 검색하려면 "\\"를 이용해 이스케이프 처리를 해야 한다.

MySQL에서는 다른 DBMS에서와 같이 홑따옴표와 쌍따옴표의 경우에는 홑따옴표나 쌍따옴표를 두번 연속으로 표기해서 이스케이프 처리할 수도 있다.
MySQL에서는 문자열을 표시하기 위해 홑따옴표와 쌍따옴표를 모두 사용할 수 있는데, 홑따옴표를 문자열로 표현할 때는 홑따옴표를 두 번 연속으로 표기해서 이스케이프 처리할 수 있다.
그리고 홑따옴표로 문자열로 감쌀 때는 쌍따옴표는 두 번 연속으로 표기해도 이스케이프 용도로 해석되지 않는다. 그 반대로도 똑같이 적용된다.
간단하게 따옴표를 두 번 연속해서 이스케이프 처리하는 예제를 한번 살펴보자.

```
mysql> CREATE TABLE tb_char_escape (fd1 VARCHAR(100));

mysql> INSERT INTO tb_char_escape VALUES ('ab''ba');
mysql> INSERT INTO tb_char_escape VALUES ("ab""ba");
mysql> INSERT INTO tb_char_escape VALUES ("ab\'ba");
mysql> INSERT INTO tb_char_escape VALUES ('ab\"ba');
mysql> INSERT INTO tb_char_escape VALUES ('ab""ba');
mysql> INSERT INTO tb_char_escape VALUES ("ab''ba");

mysql> SELECT * FROM tb_char_escape;
+--------+
|   fd1  |
+--------+
| ab'ba  |
| ab"ba  |
| ab'ba  |
| ab"ba  |
| ab""ba |
| ab''ba |
+--------+
```

위의 예제에서 하단의 마지막 두 INSERT는 홑따옴표와 쌍따옴표를 연속 두 번 표기해도 이스케이프 처리되지 않는 예를 보여준다.