# [CH 11-7] 스키마 조작(DDL)

DBMS 서버의 모든 오브젝트를 생성하거나 변경하는 쿼리를 DDL(Data Definition Language)이라고 한다.
스토어드 프로시저나 함수, DB나 테이블 등을 생성하거나 변경하는 대부분의 명령이 DDL에 해당한다.
MySQL 서버가 업그레이드되면서 많은 DDL이 온라인 모드로 처리될 수 있게 개선됐지만 여전히 스키마를 변경하는 작업 중에는 상당히 오랜 시간이 걸리고 MySQL 서버에 많은 부하를 발생시키는 작업들이
있으므로 주의해야 한다. 여기서는 중요 DDL 문의 문법과 함께 어떤 DDL 문이 특히 느리고 큰 부하를 유발하는지도 함께 살펴보겠다.

각 예제에서 대괄호("[]") 표기를 사용한 곳이 있는데, 이는 MySQL 매뉴얼의 대괄호와 동일하게 선택적인 키워드임을 의미한다.

---
<br/>

## (1) 온라인 DDL
MySQL 5.5 이전 버전까지는 MySQL 서버에서 테이블의 구조를 변경하는 동안에는 다른 커넥션에서 DML을 실행할 수 없었다.
이 같은 문제점을 해결하기 위해 Percona에서 개발한 pt-online-schema-change라는 도구를 사용했다.
물론 MySQL 5.5 버전에서도 온라인 DDL의 성능이나 안정성 등의 이유로 pt-online-schema-change 도구를 많이 사용했다.
하지만 MySQL 8.0 버전으로 업그레이드되면서 대부분의 스키마 변경 작업은 MySQL 서버에 내장된 온라인 DDL 기능으로 처리가 가능해졌다.
그래서 MySQL 8.0에서는 pt-online-schema-change와 같은 도구는 이제 거의 사용되지 않는다.

### [1] 온라인 DDL 알고리즘
온라인 DDL은 스키마를 변경하는 작업 도중에도 다른 커넥션에서 해당 테이블의 데이터를 변경하거나 조회하는 작업을 가능하게 해준다.
온라인 DDL은 밑에서 살펴볼 예제에서와 같이 ALGORITHM과 LOCK 옵션을 이용해 어떤 모드로 스키마 변경을 실행할지를 결정할 수 있다.
온라인 DDL 기능은 테이블의 구조를 변경하거나 인덱스 추가와 같은 대부분의 작업에 대해 작동한다.

MySQL 서버에서는 `old_alter_table` 시스템 변수를 이용해 ALTER TABLE 명령이 온라인 DDL로 작동할지, 아니면 예전 방식(테이블의 읽고 쓰기를 막고 스키마 변경하는 방식)으로 처리할지를 결정할
수 있다. MySQL 8.0 버전에서는 old_alter_table 시스템 변수의 기본값은 OFF로 설정돼 있기 때문에 자동으로 온라인 DDL이 활성화된다.
ALTER TABLE 명령을 실행하면 MySQL 서버는 다음과 같은 순서로 스키마 변경에 적합한 알고리즘을 찾는다.

- [1] ALGORITHM=INSTANT로 스키마 변경이 가능한지 확인 후, 가능하다면 선택
- [2] ALGORITHM=INPLACE로 스키마 변경이 가능한지 확인 후, 가능하다면 선택
- [3] ALGORITHM=COPY 알고리즘 선택

스키마 변경 알고리즘의 우선순위가 낮을수록 MySQL 서버는 스키마 변경을 위해서 더 큰 잠금과 많은 작업을 필요로 하고 서버의 부하도 많이 발생시킨다.

- INSTANT: 테이블의 데이터는 전혀 변경하지 않고, 메타데이터만 변경하고 작업을 완료한다. 테이블이 가진 레코드 건수와 무관하게 작업 시간은 매우 짧다.
  스키마 변경 도중 테이블의 읽고 쓰기는 대기하게 되지만 스키마 변경 시간이 매우 짧기 때문에 다른 커넥션의 쿼리 처리에는 크게 영향을 미치지 않는다.

- INPLACE: 임시 테이블로 데이터를 복사하지 않고 스키마 변경을 실행한다. 하지만 내부적으로는 테이블의 리빌드를 실행할 수도 있다.
  레코드의 복사 작업은 없지만 테이블의 모든 레코드를 리빌드해야 하기 때문에 테이블의 크기에 따라 많은 시간이 소요될 수도 있다. 하지만 스키마 변경 중에도 테이블의 읽기와 쓰기 모두 가능하다.
  INPLACE 알고리즘으로 스키마가 변경되는 경우에도 최초 시작 시점과 마지막 종료 시점에는 테이블의 읽고 쓰기가 불가능하다.
  하지만 이 시간은 매우 짧기 떄문에 다른 커넥션의 쿼리 처리에 대한 영향도는 높지 않다.

- COPY: 변경된 스키마를 적용한 임시 테이블을 생성하고, 테이블의 레코드를 모두 임시 테이블로 복사한 후 최종적으로 임시 테이블을 RENAME해서 스키마 변경을 완료한다.
  이 방법은 테이블 읽기만 가능하고 DML(INSERT, UPDATE, DELETE)은 실행할 수 없다.

온라인 DDL 명령은 다음 예제와 같이 알고리즘과 함께 잠금 수준도 함께 명시할 수 있다. ALGORITHM과 LOCK 옵션이 명시되지 않으면 MySQL 서버가 적절한 수준의 알고리즘과 잠금 수준을 선택하게 된다.

```
mysql> ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL,
         ALGORITHM=INPLACE, LOCK=NONE;
```

온라인 DDL에서 INSTANT 알고리즘은 테이블의 메타데이터만 변경하기 때문에 매우 짧은 시간 동안의 메타데이터 잠금만 필요로 한다.
그래서 INSTANT 알고리즘을 사용하는 경우에는 LOCK 옵션은 명시할 수 없다. INPLACE나 COPY 알고리즘을 사용하는 경우 LOCK은 다음 3가지 중 하나를 명시할 수 있다.

- NONE: 아무런 잠금을 걸지 않음
- SHARED: 읽기 잠금을 걸고 스키마 변경을 실행하기 때문에 스키마 변경 중 읽기는 가능하지만 쓰기(INSERT, UPDATE, DELETE)는 불가함
- EXCLUSIVE: 쓰기 잠금을 걸고 스키마 변경을 실행하기 때문에 테이블의 읽고 쓰기가 불가함

알고리즘으로 INPLACE가 사용되는 경우 대부분 잠금은 NONE으로 설정 가능하지만, 가끔 SHARED 수준까지 설정해야 할 수도 있다.
그리고 EXCLUSIVE는 예전 MySQL 서버의 전통적인 ALTER TABLE과 동일하므로 굳이 LOCK을 명시할 필요는 없다.

온라인 스키마 변경 작업이 INPLACE 알고리즘을 사용하더라도 내부적으로는 테이블의 리빌드가 필요할 수도 있다.
대표적으로 테이블의 프라이머리 키를 추가하는 작업은 데이터 파일에서 레코드의 저장 위치가 바뀌어야 하기 때문에 테이블의 리빌드가 필요한 반면, 단순히 칼럼의 이름만 변경하는 경우 INPLACE 알고리즘을
사용해야 하지만 실제 테이블 레코드의 리빌드 작업은 필요치 않다.
프라이머리 키를 추가하는 경우와 같이 테이블의 리빌드가 필요한 경우를 MySQL 서버 매뉴얼에서는 "Data Reorganizing(데이터 재구성)"또는 "Table Rebuild(테이블 리빌드)"라고 명명한다.

결론적으로 INPLACE 알고리즘을 사용하는 경우는 다음과 같이 구분할 수 있다.

- 데이터 재구성(테이블 리빌드)이 필요한 경우: 잠금을 필요로 하지 않기 때문에 읽고 쓰기는 가능하지만 여전히 테이블의 레코드 건수에 따라 상당히 많은 시간이 소요될 수도 있다.
- 데이터 재구성(테이블 리빌드)이 필요치 않은 경우: INPLACE 알고리즘을 사용하지만 INSTANT 알고리즘과 비슷하게 매우 빨리 작업이 완료될 수 있다.

MySQL 서버의 온라인 DDL 기능은 버전별로 많은 차이가 있다.
그래서 사용 중인 MySQL 서버의 버전이 8.0이 아니라면 사용 중인 버전의 MySQL 매뉴얼을 살펴보고 테이블 리빌드(Table Rebuild)가 필요한지 확인한 후 진행하자.
스키마 변경을 실행하기 전에 이런 내용을 확인하지 않고 시작했는데 스키마 변경 작업에 시간이 오래 걸리면 그때서야 불안해하는 상황을 자주 경험했다.
스키마 변경 작업을 실행하기 전에 먼저 매뉴얼과 테스트를 진행해볼 것을 권장한다.

### [2] 온라인 처리 가능한 스키마 변경
MySQL 서버의 모든 스키마 변경 작업이 온라인으로 가능한 것이 아니기 때문에 필요한 스키마 변경 작업의 형태가 온라인으로 처리될 수 있는지, 아니면 테이블의 읽고 쓰기가 대기(Waiting)하게 되는지
확인한 후 실행하는 것이 좋다. 스키마 변경 작업 종류별로 어떤 알고리즘이 가능한지를 간략히 정리해봤다. 이는 MySQL 8.0.21 버전의 지원 사항이므로 이후 버전에서는 개선됐을 수도 있다.
사용 중인 MySQL 서버의 버전이 8.0.21보다 많이 높다면 [매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)을 참조하자.

#### [인덱스 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|프라이머리 키 추가|✕|○|○|○|✕|
|프라이머리 키 삭제|✕|✕|○|✕|✕|
|프라이머리 키 삭제 + 추가|✕|○|○|○|✕|
|세컨더리 인덱스 생성|✕|○|✕|○|✕|
|세컨더리 인덱스 삭제|✕|○|✕|○|○|
|세컨더리 인덱스 이름 변경|✕|○|✕|○|○|
|전문 검색 인덱스 생성|✕|○|✕|✕|✕|
|공간 검색 인덱스 생성|✕|○|✕|✕|✕|
|인덱스 타입 변경|○|○|✕|○|○|

#### [칼럼 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|칼럼 추가|○|○|✕|○|✕|
|칼럼 삭제|✕|○|○|○|✕|
|칼럼 이름 변경|✕|○|✕|○|○|
|칼럼 순서 변경|✕|○|○|○|✕|
|칼럼의 기본값 설정|○|○|✕|○|○|
|칼럼 데이터 타입 변경|✕|✕|○|✕|✕|
|VARCHAR 타입의 길이 확장|✕|○|✕|○|○|
|칼럼의 기본값 제거|○|○|✕|○|○|
|자동 증가(AutoIncrement) 값 변경|✕|○|✕|○|✕|
|칼럼을 NULLABLE로 변경|✕|○|○|○|✕|
|칼럼을 NOT NULL로 변경|✕|○|○|○|✕|
|ENUM이나 SET의 정의 변경|○|○|✕|○|○|

#### [가상 칼럼 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|가상칼럼(STORED) 추가|✕|✕|○|✕|✕|
|가상칼럼(STORED) 순서 변경|✕|✕|○|✕|✕|
|가상칼럼(STORED) 삭제|✕|○|○|○|✕|
|가상칼럼(VIRTUAL) 추가|○|○|✕|○|○|
|가상칼럼(VIRTUAL) 순서 변경|✕|✕|○|✕|✕|
|가상칼럼(VIRTUAL) 삭제|○|○|✕|○|○|

#### [외래키 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|외래키 생성|✕|○|✕|○|○|
|외래키 삭제|✕|○|✕|○|○|

#### [테이블 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|ROW_FORMAT 변경|✕|○|○|○|✕|
|KEY_BLOCK_SIZE 변경|✕|○|○|○|✕|
|STATS_PERSISTENT 변경|✕|○|✕|○|○|
|CHARACTER SET 설정|✕|○|○|✕|✕|
|CHARACTER SET 변경|✕|✕|○|✕|✕|
|테이블 최적화(OPTIMIZE)|✕|○|○|○|✕|
|테이블 리빌드(FORCE 옵션)|✕|○|○|○|✕|
|테이블명 변경|○|○|✕|○|○|

#### [테이블 스페이스 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|제너럴 테이블스페이스 이름 변경|✕|○|✕|○|○|
|제너럴 테이블스페이스 암호화 옵션 변경|✕|○|✕|○|✕|
|테이블별 테이블스페이스 암호화 옵션 변경|✕|✕|○|✕|✕|

#### [파티션 변경]
|변경 작업|INSTANT|INPLACE|REBUILD 테이블|DML 허용|메타데이터만 변경|
|:---|:---|:---|:---|:---|:---|
|파티션 적용(PARTITION BY)|✕|✕|○|✕|✕|
|파티션 추가(ADD PARTITION)|✕|○|○|○ (LIST, RANGE)<br/>✕ (KEY, HASH)|✕|
|파티션 삭제(DROP PARTITION)|✕|○|○|○ (LIST, RANGE)<br/>✕ (KEY, HASH)|✕|
|파티션의 테이블스페이스 삭제|✕|✕|✕|✕|✕|
|파티션의 테이블스페이스 IMPORT|✕|✕|✕|✕|✕|
|파티션 TRUNCATE|✕|○|○|○|✕|

MySQL 서버에서 사용할 수 있는 스키마 변경 작업으 매우 다양하기 때문에 모든 명령이 온라인 DDL을 지원하는지 아닌지를 기억하기는 쉽지 않다.
이러한 경우에는 다음 예제와 같이 ALTER TABLE 문장에 LOCK과 ALGORITHM 절을 명시해서 온라인 스키마 변경의 처리 알고리즘을 강제할 수 있다.
물론 이렇게 온라인 DDL 알고리즘을 강제한다고 해서 무조건 그 알고리즘으로 처리되는 것은 아니다.
하지만 명시된 알고리즘으로 온라인 DDL이 처리되지 못한다면 단순히 에러만 발생시키고 실제 스키마 변경 작업은 시작되지 않기 때문에 의도하지 않은 잠금과 대기는 발생하지 않는다.

```
mysql> ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=INSTANT;
ERROR 1846 (0A000): ALGORITHM=INSTANT is not supported. Reason: Dropping a primary key is not
allowed without also adding a new primary key. Try ALGORITHM=COPY/INPLACE.

mysql> ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=NONE;
ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Dropping a primary key is not
allowed without also adding a new primary key. Try ALGORITHM=COPY.

mysql> ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
Query OK, 300024 rows affected (6.24 sec)
Records: 300024  Dupliates: 0 Warnings: 0

mysql> ALTER TABLE employees ADD PRIMARY KEY (emp_no), ALGORITHM=INPLACE, LOCK=NONE;
Query OK, 0 rows affected (1.48 sec)
Records: 0  Duplicates: 0 Warnings: 0
```

위의 예제에서는 다음 순서로 ALGORITHM과 LOCK 옵션을 시도해보면서 해당 알고리즘이 지원되는지 여부를 판단한다.

- [1] ALGORITHM=INSTANT 옵션으로 스키마 변경을 시도
- [2] 실패하면 ALGORITHM=INPLACE, LOCK=NONE 옵션으로 스키마 변경을 시도
- [3] 실패하면 ALGORITHM=INPLACE, LOCK=SHARED 옵션으로 스키마 변경을 시도
- [4] 실패하면 ALGORITHM=COPY, LOCK=SHARED 옵션으로 스키마 변경을 시도
- [5] 실패하면 ALGORITHM=COPY, LOCK=EXCLUSIVE 옵션으로 스키마 변경을 시도

실행하고자 하는 스키마 변경 작업으로 인해 DML(INSERT, UPDATE, DELETE)이 멈춰서는 안 된다면 [1]번과 [2]번까지만 해보면 된다.
[1]번과 [2]번 옵션으로 스키마 변경이 되지 않는다면 점검(서비스를 멈추고)을 걸고 DML을 멈춘 다음 스키마 변경을 해야 한다는 것을 확인할 수 있다.
실행하고자 하는 스키마 변경이 [1]번이나 [2]번 옵션으로 가능한 작업이라면 MySQL 서버는 즉시 스키마 변경을 실행하게 된다.
하지만 온라인 DDL이라 하더라도 그만큼 MySQL 서버에 부하를 유발할 수 잇으며, 그로 인해 다른 커넥션의 쿼리들이 느려질 수도 있다.
그러므로 설령 스키마 변경 작업이 직접 다른 커넥션의 DML을 대기하게 만들지는 않더라도 주의해서 사용해야 한다.

### [3] INPLACE 알고리즘
INPLACE 알고리즘은 임시 테이블로 레코드를 복사하지는 않더라도 내부적으로 테이블의 모든 레코드를 리빌드해야 하는 경우가 많다. 이러한 경우 MySQL 서버는 다음과 같은 과정을 거치게 된다.

- [1] INPLACE 스키마 변경이 지원되는 스토리지 엔진의 테이블인지 확인
- [2] INPLACE 스키마 변경 준비(스키마 변경에 대한 정보를 준비해서 온라인 DDL 작업 동안 변경되는 데이터를 추적할 준비)
- [3] 테이블 스키마 변경 및 새로운 DML 로깅(이 작업은 실제 스키마 변경을 수행하는 과정으로, 이 작업이 수행되는 동안은 다른 커넥션의 DML 작업이 대기하지 않는다.
  이렇게 스키마를 온라인으로 변경함과 동시에 다른 스레드에서는 사용자에 의해서 발생한 DML들에 대해서 별도의 로그로 기록)
- [4] 로그 적용(온라인 DDL 작업 동안 수집된 DML 로그를 테이블에 적용)
- [5] INPLACE 스키마 변경 완료(COMMIT)

INPLACE 알고리즘으로 스키마가 변경된다고 하더라도 [2]번과 [4]번 단계에서는 잠깐의 배타적 잠금(Exclusive lock)이 필요하며, 이 시점에는 다른 커넥션의 DML들이 잠깐 대기한다.
하지만 실제 변경 작업이 실행되면서 많은 시간이 필요한 [3]번 단계는 다른 커넥션의 DML 작업이 대기 없이 즉시 처리된다.
그리고 INPLACE 알고리즘으로 온라인 스키마 변경이 진행되는 동안 새로 유입된 DML 쿼리들에 의해 변경되는 데이터를 "온라인 변경 로그(Online alter log)"라는 메모리 공간에 쌓아 두었다가
온라인 스키마 변경이 완료되면 로그의 내용을 실제 테이블로 일괄 적용하게 된다.
이때 온라인 변경 로그는 디스크가 아니라 메모리에만 생성되며, 이 메모리 공간의 크기는 `innodb_online_alter_log_max_size` 시스템 설정 변수에 의해 결정된다.
기본적으로 온라인 변경 로그의 크기는 128MB인데, 온라인 스키마 변경이 오랜 시간이 걸린다거나 온라인 스키마 변경 중에 유입되는 DML 쿼리가 많다면 이 메모리 공간을 더 크게 설정하는 것이 좋다.
innodb_online_alter_log_max_size 시스템 변수는 세션 단위의 동적 변수이므로 필요한 경우에는 언제든지 변경할 수 있다.

### [4] 온라인 DDL의 실패 케이스
온라인 DDL 명령은 다음과 같은 이유로 실패할 수도 있다. 온라인 DDL이 INSTANT 알고리즘을 사용하는 경우 거의 시작과 동시에 작업이 완료되기 때문에 작업 도중 실패할 가능성은 거의 없다.
하지만 INPLACE 알고리즘으로 실행되는 경우 내부적으로 테이블 리빌드 과정이 필요하고 최종 로그 적용 과정이 필요해서 중간 과정에서 실패할 가능성이 상대적으로 높은 편이다.
INPLACE 알고리즘으로 몇 시간 동안 실행되던 온라인 DDL이 실패하면 이는 상당한 자원과 시간 낭비가 될 것이다.

다음 실패 케이스를 살펴보고, 최대한 온라인 DDL이 실패할 가능성을 낮추는 것이 좋다.

- ALTER TABLE 명령이 장시간 실행되고 동시에 다른 커넥션에서 DML이 많이 실행되는 경우이거나 온라인 변경 로그의 공간이 부족한 경우 온라인 스키마 변경 작업은 실패

  ```
  ERROR 1799 (HY000): Creating index 'idx_col1' required more than 'innodb_online_alter_
  log_max_size' bytes of modification log. Please try again.
  ```

- ALTER TABLE 명령이 실행되는 동안 ALTER TABLE 이전 버전의 테이블 구조에서는 아무런 문제가 안 되지만 ALTER TABLE 이후의 테이블 구조에는 적합하지 않은 레코드가 INERT되거나 UPDATE됐다면
  온라인 스키마 변경 작업은 마지막 과정에서 실패

  ```
  ERROR 1062 (23000): Duplicate entry 'd005-10001' for key 'PRIMARY'
  ```

- 스키마 변경을 위해서 필요한 잠금 수준보다 낮은 잠금 옵션이 사용된 경우

  ```
  ERROR 1846 (0A000): LOCK=NONE is not supported. Reason: Adding an auto-increment column
  requires a lock. Try LOCK=SHARED.
  ```

- 온라인 스키마 변경은 LOCK=NONE으로 실행된다고 하덜다ㅗ 변경 작업의 처음과 마지막 과정에서 잠금이 필요한데, 이 잠금을 획득하지 못하고 타임 아웃이 발생하면 실패

  ```
  ERROR 1025 (HY000): Lock wait timeout exceeded; try restarting transaction
  ```

- 온라인으로 인덱스를 생성하는 작업의 경우 정렬을 위해 tmpdir 시스템 변수에 설정된 디스크의 임시 디렉터리를 사용하는데, 이 공간이 부족한 경우 또한 온라인 스키마 변경은 실패함

> LOCK=NONE으로 온라인 스키마 변경이 실행되더라도 변경 작업의 처음과 마지막에는 테이블의 메타데이터에 대한 잠금이 필요하다는 것은 이미 살펴봤다.
> 메타데이터에 대한 잠금을 획득하지 못하고 타임 아웃이 발생하면 온라인 스키마 변경이 실패하게 된다.
> MySQL 서버에는 이미 타임 아웃과 관련된 시스템 변수가 꽤 많이 있는데, 이때 타임 아웃의 기준으로 사용되는 시스템 변수는 무엇일까?
>
> InnoDB 스토리지 엔진을 사용하는 테이블이라고 하더라도 온라인 스키마 변경에서 필요한 잠금은 테이블 수준의 메타데이터 잠금이다.
> MySQL에서 메타데이터 잠금에 대한 타임 아웃은 lock_wait_timeout 시스템 변수에 의해서 결정된다.
> InnoDB의 레코드 잠금에 대한 대기 타임 아웃인 innodb_lock_wait_timeout은 온라인 스키마 변경과는 무관하다는 것도 기억해 두자.
> SHOW GLOBAL VARIABLES 명령을 이용해 lock_wait_timeout의 설정값이 얼마인지 확인해보자.
>
> ```
> mysql> SHOW GLOBAL VARIABLES LIKE 'lock_wait_timeout';
> +-------------------+----------+
> | Variable_name     | Value    |
> +-------------------+----------+
> | lock_wait_timeout | 31536000 |
> +-------------------+----------+
> ```
>
> 31536000초 정도가 lock_wait_timeout으로 설정돼 있는데, 이 값은 실제 MySQL 서버의 기본 설정값이다.
> 온라인 스키마 변경을 실행하고 있는데, 다른 커넥션에서 아주 장시간 DML이 실행되고 있다거나 트랜잭션이 정상적으로 종료되지 않아서 INSERT나 UPDATE 쿼리가 활성 트랜잭션 상태로 남아 있는 커넥션이
> 있다면 온라인 스키마 변경은 31536000초 동안 기다릴 것이다.
>
> 물론 온라인 스키마 변경이 진행되는 동안에는 주의해서 이와 같이 스키마 변경을 방해하는 트랜잭션이 발생하지 않게 해야겠지만, 이런 현상을 피할 수 없는 상황이라면 lock_wait_timeout을 적절한
> 시간으로 조정해서 일정 시간 이상 대기할 때는 온라인 스키마 변경을 취소하도록 조치하는 것도 도움이 될 수 있다.
> lock_wait_timeout은 글로벌 레벨의 시스템 변수임과 동시에 세션 레베르이 시스템 변수이기도 하다.
> 그러므로 lock_wait_timeout은 온라인 스키마 변경을 실행하는 세션에서 적정 값으로 조정하는 것이 좋다.
>
> ```
> mysql> SET SESSION lock_wait_timeout=1800;
> mysql> ALTER TABLE tab_test ADD fd2 VARCHAR(20), ALGORITHM=INPLACE, LOCK=NONE;
> ```

### [5] 온라인 DDL 진행 상황 모니터링
온라인 DDL을 포함한 모든 ALTER TABLE 명령은 MySQL 서버의 performance_schema를 통해 진행 상황을 모니터링할 수 있다.
ALTER TABLE의 진행 상황을 모니터링할 수 없었을 때는 MySQL 서버의 상태 변수를 뒤져가면서 몇 건이나 레코드를 읽고 쓰기를 했는지를 이용해 ALTER TABLE이 어느 정도 진행됐는지 대략 예측하곤 했다.
하지만 레코드를 얼마나 읽고 쓰기를 했는지로는 예측하는 데 한계가 있어 제대로 시간을 예측한 적이 없다.

우선 performance_schema를 이용해 ALTER TABLE의 진행 상황을 모니터링하려면 다음과 같이 performance_schema 옵션(Instrument와 Consumer 옵션)이 활성화돼야 한다.
물론 MySQL 서버의 performance_schema 시스템 변수가 가장 먼저 ON으로 활성화돼야 한다.

```
-- // performance_schema 시스템 변수 활성화(MySQL 서버 재시작 필요)
mysql> SET GLOBAL performance_schema=ON;

-- // "stage/innodb/alter%" instrument 활성화
mysql> UPDATE performance_schema.setup_instruments
          SET ENABLED = 'YES', TIMED = 'YES'
        WHERE NAME LIKE 'stage/innodb/alter%';

-- // "%stages%" consumer 활성화
mysql> UPDATE performance_schema.setup_consumers
          SET ENABLED = 'YES'
        WHERE NAME LIKE '%stages%';
```

스키마 변경 작업의 진행 상황은 performance_schema.events_stages_current 테이블을 통해 확인할 수 있는데, 실행 중인 스키마 변경 종류에 따라 기록되는 내용이 조금씩 달라진다.
우선 온라인 DDL이 아닌 전통적인 COPY 알고리즘으로 스키마 변경이 진행되는 경우 다음과 같이 조회된다.

```
-- // COPY 알고리즘의 스키마 변경
mysql_session1> ALTER TABLE salaries DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;

-- // performance_schema의 진행 상황
mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                FROM performance_schema.events_stages_current;
+-----------------------------+----------------+----------------+
| EVENT_NAME                  | WORK_COMPLETED | WORK_ESTIMATED |
+-----------------------------+----------------+----------------+
| stage/sql/copy to tmp table |        1562071 |        2838662 |
```

스키마 변경 작업이 온라인 DDL로 실행되는 경우 다음과 같이 다양한 상태를 보여주는데, 이는 온라인 DDL이 단계(Stage)별로 EVENT_NAME 칼럼의 값을 달리해서 보여주기 때문이다.

```
-- // INPLACE 알고리즘으로 스키마 변경을 실행
mysql_session1> ALTER TABLE salaries
                ADD INDEX ix_todate (to_date),
                ALGORITHM=INPLACE, LOCK=NONE;

-- // performance_schema를 통한 진행 상황 확인
mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                FROM performance_schema.events_stages_current;
+------------------------------------------------------+----------------+----------------+
| EVENT_NAME                                           | WORK_COMPLETED | WORK_ESTIMATED |
+------------------------------------------------------+----------------+----------------+
| stage/innodb/alter table (read PK and internal sort) |           9776 |          25281 |
+------------------------------------------------------+----------------+----------------+

mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                FROM performance_schema.events_stages_current;
+---------------------------------------+----------------+----------------+
| EVENT_NAME                            | WORK_COMPLETED | WORK_ESTIMATED |
+---------------------------------------+----------------+----------------+
| stage/innodb/alter table (merge sort) |          17641 |          27121 |
+---------------------------------------+----------------+----------------+

mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                FROM performance_schema.events_stages_current;
+--------------------------------------+----------------+----------------+
| EVENT_NAME                           | WORK_COMPLETED | WORK_ESTIMATED |
+--------------------------------------+----------------+----------------+
| stage/innodb/alter table (insert)    |          23460 |          27121 |
+--------------------------------------+----------------+----------------+

mysql_session2> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
                FROM performance_schema.events_stages_history;
+--------------------------------------+----------------+----------------+
| EVENT_NAME                           | WORK_COMPLETED | WORK_ESTIMATED |
+--------------------------------------+----------------+----------------+
| stage/innodb/alter table (end)       |          25719 |          27351 |
+--------------------------------------+----------------+----------------+
```

WORK_ESTIMATED와 WORK_COMPLETED 칼럼의 값을 비교해보면 ALTER TABLE의 진행 상황을 예측할 수 있다.
하지만 WORK_ESTIMATED 칼럼의 값은 예측치이기 때문에 ALTER TABLE이 진행되면서 조금씩 변경된다.
위의 예제에서는 WORK_ESTIMATED 값이 처음에는 25281이었지만 완료된 시점에는 27351로 변경된 것을 알 수 있다.
하지만 (WORK_COMPLETED * 100 / WORK_ESTIMATED) 값으로 대략적인 진행률을 확인할 수 있다. 위의 예제에서는 EVENT_NAME이 4개만 보이지만 이 4가지 단계가 전부는 아니다.
실제 각 단계가 매우 빨리 완료되어 여기서는 포착되지 않은 것도 있다.(performance_schema.events_stages_current 테이블은 SQL이 현재 진행 중인 경우 표시되며, SQL이 완료되면
events_stages_current 테이블에서는 사라지고 events_stages_history 테이블을 통해 확인할 수 있다.
위 예제의 마지막은 쿼리 performance_schema.events_stages_history 테이블을 조회했는데, 이는 마지막 시점을 정확하게 포착하지 못해서 events_stages_history 테이블을 참조한 것이다.)
<br/>
<br/>
## (2) 데이터베이스 변경
MySQL에서 하나의 인스턴스는 1개 이상의 데이터베이스를 가질 수 있다.
다른 RDBMS에서는 스키마(Schema)와 데이터베이스를 구분해서 관리하지만 MySQL 서버에서는 스키마와 데이터베이스는 동격의 개념이다.
그래서 MySQL 서버에서는 굳이 스키마를 명시적으로 사용하지는 않는다.
MySQL의 데이터베이스는 디스크의 물리적인 저장소를 구분하기도 하지만 여러 데이터베이스의 테이블을 묶어서 조인 쿼리를 사용할 수도 있기 때문에 단순히 논리적인 개념이기도 하다.
물론 데이터베이스는 객체에 대한 권한을 구분하는 용도로 사용되기도 하지만 그 이상의 큰 의미를 가지지는 않는다. 그래서 데이터베이스 단위로 변경하거나 설정하는 DDL 명령은 그다지 많지 않다.
데이터베이스에 설정할 수 있는 옵션은 기본 문자 집합이나 콜레이션을 설정하는 정도이므로 간단하다.

### [1] 데이터베이스 생성
```
mysql> CREATE DATABASE [IF NOT EXISTS] employees;
mysql> CREATE DATABASE [IF NOT EXISTS] employees CHARACTER SET utf8mb4;
mysql> CREATE DATABASE [IF NOT EXISTS] employees
              CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

첫 번째 명령은 기본 문자 집합과 콜레이션으로 employees라는 데이터베이스를 생성한다.
여기서 기본이라 함은 MySQL 서버의 `character_set_server` 시스템 변수에 정의된 문자 집합을 사용한다는 의미다.
두 번째와 세 번째 명령은 별도의 문자 집합과 콜레이션이 지정된 데이터베이스를 생성한다. 이미 동일 이름의 데이터베이스가 있다면 이 DDL 문장은 에러를 유발할 것이다.
하지만 "IF NOT EXISTS"라는 키워드를 사용하면 데이터베이스가 없는 경우에만 생성하고, 이미 있다면 이 DDL은 그냥 무시된다.

### [2] 데이터베이스 목록
```
mysql> SHOW DATABASES;
mysql> SHOW DATABASES LIKE '%emp%';
```

접속된 MySQL 서버가 가지고 있는 데이터베이스의 목록을 나열한다. 단, 권한을 가지고 있는 데이터베이스의 목록만 표시되며, 이 명령을 실행하려면 "SHOW DATABASES" 권한이 있어야 한다.
두 번째 명령은 "emp"라는 문자열을 포함한 데이터베이스 목록만 표시한다.

### [3] 데이터베이스 선택
```
mysql> USE employees;
```

기본 데이터베이스를 선택하는 명령이다.
SQL 문장에서 별도로 데이터베이스를 명시하지 않고 테이블 이름이나 프로시저의 이름만 명시하면 MySQL 서버는 현재 커넥션의 기본 데이터베이스에서 주어진 테이블이나 프로시저를 검색한다.
기본 데이터베이스에 존재하지 않는 테이블이나 프로시저를 사용하려면 다음과 같이 테이블이나 프로시저의 이름 앞에 데이터베이스 이름을 반드시 명시해야 한다.

```
mysql> SELECT * FROM employees.departments;
```

### [4] 데이터베이스 속성 변경
```
mysql> ALTER DATABASE employees CHARACTER SET=euckr;
mysql> ALTER DATABASE employees CHARACTER SET=euckr COLLATE=euckr_korean_ci;
```

데이터베이스를 생성할 때 지정한 문자 집합이나 콜레이션을 변경한다.

### [5] 데이터베이스 삭제
```
mysql> DROP DATABASE [IF EXISTS] employees;
```

데이터베이스를 삭제한다. 지정한 이름의 데이터베이스가 존재하지 않는다면 에러가 발생한다.
하지만 "IF EXISTS" 키워드를 사용하면 해당 데이터베이스가 존재할 때만 삭제하고, 그렇지 않으면 이 명령을 실행하지 않는다.
<br/>
<br/>
## (3) 테이블스페이스 변경
MySQL 서버에는 전통적으로 테이블별로 전용의 테이블스페이스를 사용했었다.
InnoDB 스토리지 엔진의 시스템 테이블스페이스(ibdata1 파일)만 제너럴 테이블스페이스(General Tablespace)를 사용했는데, 제너럴 테이블스페이스는 여러 테이블의 데이터를 한꺼번에 저장하는
테이블스페이스를 의미한다.

MySQL 8.0 버전이 되면서 MySQL 서버에서도 사용자 테이블을 제너럴 테이블스페이스로 저장하는 기능이 추가되고 테이블스페이스를 관리하는 DDL 명령들이 추가됐다.
그러나 MySQL 8.0에서도 제너럴 테이블스페이스는 여러 가지 제약 사항을 가진다. 그중 몇 가지 중요한 제약 사항은 다음과 같다.

- 파티션 테이블은 제너럴 테이블스페이스를 사용하지 못함
- 복제 소스와 레플리카 서버가 동일 호스트에서 실행되는 경우 ADD DATAFILE 문장은 사용 불가
- 테이블 암호화(TDE)는 테이블스페이스 단위로 설정됨
- 테이블 압축 가능 여부는 테이블스페이스의 블록 사이즈와 InnoDB 페이지 사이즈에 의해 결정됨
- 특정 테이블을 삭제(DROP TABLE)해도 디스크 공간이 운영체제로 반납되지 않음

그럼에도 불구하고, MySQL 8.0에서 사용자 테이블이 제너럴 테이블스페이스를 이용할 수 있게 개선된 것은 다음과 같은 장점이 있기 때문이다.

- 제너럴 테이블스페이스를 사용하면 파일 핸들러(Open file descriptor)를 최소화
- 테이블스페이스 관리에 필요한 메모리 공간을 최소화

제너럴 테이블스페이스가 가진 2가지 장점은 사실 테이블의 개수가 매우 많은 경우에 유용하다. 아직 일반적인 환경에서 제너럴 테이블스페이스의 장점은 취하기가 어렵다.
MySQL 서버에서 테이블이 개별 테이블스페이스를 사용할지 아니면 제너럴 테이블스페이스를 사용할지는 `innodb_file_per_table` 시스템 변수로 제어할 수 있다.
MySQL 8.0에서는 innodb_file_per_table 시스템 변수의 기본값이 ON이므로 테이블은 자동으로 개별 테이블스페이스를 사용한다.

데이터베이스에 작은 테이블이 매우 많이 필요한 응용 프로그램을 개발 중이라면 제너럴 테이블스페이스에 대한 매뉴얼을 자세히 살펴보자.
또한 제너럴 테이블스페이스를 생성하고, 각 테이블이 개별 테이블스페이스가 아니라 제너럴 테이블스페이스를 사용하게 하는 방법도 매뉴얼을 참조하자.
<br/>
<br/>
## (4) 테이블 변경
테이블은 사용자의 데이터를 가지는 주체로서, MySQL 서버의 많은 옵션과 인덱스 등의 기능이 테이블에 종속되어 사용된다.

### [1] 테이블 생성
다음 예제는 테이블을 생성하는 CREATE TABLE 문장이다. 설명을 위해 가능한 한 많은 옵션이 포함된 칼럼으로 테이블 생성 예제를 준비했다.

```
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tb_test (
  member_id BIGINT [UNSIGNED] [AUTO_INCREMENT],
  nickname CHAR(20) [CHARACTER SET 'utf8'] [COLLATE 'utf8_general_ci'] [NOT NULL],
  home_url VARCHAR(200) [COLLATE 'latin1_general_cs'],
  birth_year SMALLINT [(4)] [UNSIGNED] [ZEROFILL],
  member_point INT [NOT NULL] [DEFAULT 0],
  registered_dttm DATETIME [NOT NULL],
  modified_ts TIMESTAMP [NOT NULL] [DEFAULT CURRENT_TIMESTAMP],
  gender ENUM('Female','Male') [NOT NULL],
  hobby SET('Reading','Game','Sports'),
  profile TEXT [NOT NULL],
  session_data BLOB,
  PRIMARY KEY (member_id),
  UNIQUE INDEX ux_nickname (nickname),
  INDEX ix_registereddttm (registered_dttm)
) ENGINE=INNODB;
```

TEMPORARY 키워드를 사용하면 해당 데이터베이스 커넥션(세션)에서만 사용 가능한 임시 테이블을 생성한다.
테이블의 생성 또한 데이터베이스와 마찬가지로 이미 같은 이름의 테이블이 있으면 에러가 발생하는데, "IF NOT EXISTS" 옵션을 사용하면 에러를 무시한다.
MySQL은 테이블을 정의한 스크립트 마지막에 테이블이 사용할 스토리지 엔진을 결정하기 위해 ENGINE이라는 키워드를 사용할 수 있다.
위 쿼리에서는 ENGINE=InnoDB라고 정의했기 때문에 이 테이블은 InnoDB 스토리지 엔진을 사용하는 테이블로 생성된다.
별도로 ENGINE이 정의되지 않으면 MySQL 8.0에서는 InnoDB 스토리지 엔진이 기본으로 사용된다.

각 칼럼은 "칼럼명 + 칼럼타입 + [타입별 옵션] + [NULL 여부] + [기본값]"의 순서로 명시하고, 타입별로 다음과 같은 옵션을 추가로 사용할 수 있다.

- 모든 칼럼은 공통적으로 칼럼의 초깃값을 설정하는 DEFAULT 절과 칼럼의 NULL을 가질 수 있는지 여부를 설정하기 위해 NULL 또는 NOT NULL 제약을 명시할 수 있다.
- 문자열 타입은 타입 뒤에 반드시 칼럼에 최대한 저장할 수 있는 문자 수를 명시해야 한다.
  그리고 CHARACTER SET 절은 칼럼에 저장되는 문자열 값이 어떤 문자 집합을 사용할지를 결정하고, COLLATE로 문자열 비교나 정렬 규칙을 나타내기 위한 콜레이션을 설정할 수 있다.
  CHARACTER SET만 설정되면 해당 문자 집합의 기본 콜레이션(여기에서 "기본 콜레이션"은 문자 셋의 기본 콜레이션을 의미한다. 테이블이 생성되는 데이터베이스의 콜레이션과는 다르다는 것에 주의하자.
  예를 들어서 콜레이션이 utf8mb4_general_ci인 데이터베이스에서 테이블을 생성할 때 CHARACTER SET 절에 utf8mb4만 명시하는 경우, 테이블의 콜레이션은 utf8mb4_general_ci가 아니라
  utf8mb4_0900_ai_ci 콜레이션이 사용된다.)이 자동으로 사용된다.
- 숫자 타입은 선택적으로 길이를 가질 수 있지만, 이는 실제 칼럼에 저장될 값의 길이를 의미하는 것이 아니라 단순히 값을 표시할 때 보여줄 길이를 지정하는 것이다.(TINYINT와 SMALLINT, 그리고 INT와
  BIGINT 타입에 사용되는 길이는 MySQL 8.0 버전부터는 더이상 효력이 없으며(Deprecated), 이후 버전부터는 문법적으로도 지원하지 않을 것으로 보인다.)
  그리고 양수만 가질지 음수와 양수를 모두 저장할지에 따라 선택적으로 UNSIGNED 키워드를 명시할 수 있다.
  UNSIGNED 키워드를 명시하지 않으면 기본적으로 SIGNED가 되고, 음수와 양수를 모두 저장할 수 있다.
  숫자 타입은 ZEROFILL이라는 키워드도 선택적으로 가질 수 있는데, 이는 숫자 값의 왼쪽에 '0'을 패딩할지를 결정하는 옵션이다.
- MySQL 5.5 버전까지는 DATE나 DATETIME 타입은 기본 값(DEFAULT)을 명시할 수 없었지만, MySQL 5.6 버전부터는 DATE와 DATETIME 타입 그리고 TIMESTAMP 타입 모두 값이 자동으로 현재
  시간으로 업데이트되도록 기본 값을 명시할 수 있다.
- ENUM 또는 SET 타입은 타입의 이름 뒤에 해당 칼럼이 가질 수 있는 값을 괄호로 정의해야 한다.

### [2] 테이블 구조 조회
MySQL에서 테이블의 구조를 확인하는 방법은 SHOW CREATE TABLE 명령과 DESC 명령으로 두 가지가 있다.

SHOW CREATE TABLE 명령을 사용하면 테이블의 CREATE TABLE 문장을 표시해준다.
하지만 SHOW CREATE TABLE 명령의 결과가 최초 테이블을 생성할 때 사용자가 실행한 내용을 그대로 보여주는 것은 아니다.
MySQL 서버가 테이블의 메타 정보를 읽어서 이를 CREATE TABLE 명령으로 재작성해서 보여주는 것이다.
하지만 이 명령은 특별한 수정 없이 바로 사용할 수 있는 CREATE TABLE 명령을 만들어 주기 때문에 상당히 유용하다.

```
mysql> SHOW CREATE TABLE employees \G
*************************** 1. row ***************************
       Table: employees
Create Table: CREATE TABLE `employees` (
  `emp_no` int NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) COLLATE utf8mb4_general_ci NOT NULL,
  `last_name` varchar(16) COLLATE utf8mb4_general_ci NOT NULL,
  `gender` enum('M','F') COLLATE utf8mb4_general_ci NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_hiredate` (`hire_date`),
  KEY `ix_gender_birthdate` (`gender`,`birth_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci STATS_PERSISTENT=0
```

SHOW CREATE TABLE 명령은 칼럼의 목록과 인덱스, 외래키 정보를 동시에 보여주기 때문에 SQL을 튜닝하거나 테이블의 구조를 확인할 때 주로 이 명령을 사용한다.

DESC 명령은 DESCRIBE의 약어 형태의 명령으로 둘 모두 같은 결과를 보여준다. DESC 명령은 테이블의 칼럼 정보를 보기 편한 표 형태로 표시해준다.
하지만 인덱스 칼럼의 순서나 외래키, 테이블 자체의 속성을 보여주지는 않으므로 테이블의 전체적인 구조를 한 번에 확인하기는 어렵다.

```
mysql> DESC employees;
+------------+---------------+------+-----+---------+-------+
| Field      | Type          | Null | Key | Default | Extra |
+------------+---------------+------+-----+---------+-------+
| emp_no     | int           | NO   | PRI | NULL    |       |
| birth_date | date          | NO   |     | NULL    |       |
| first_name | varchar(14)   | NO   | MUL | NULL    |       |
| last_name  | varchar(16)   | NO   |     | NULL    |       |
| gender     | enum('M','F') | NO   | MUL | NULL    |       |
| hire_date  | date          | NO   | MUL | NULL    |       |
+------------+---------------+------+-----+---------+-------+
```

### [3] 테이블 구조 변경
테이블의 구조를 변경하려면 ALTER TABLE 명령을 사용한다. ALTER TABLE 명령은 테이블 자체의 속성을 변경할 수 있을뿐만 아니라 인덱스의 추가/삭제나 칼럼을 추가/삭제하는 용도로도 사용된다.
ALTER TABLE 명령은 테이블 자체 옵션과 칼럼, 인덱스 등 거의 대부분의 스키마를 변경하는 작업에 사용된다. ALTER TABLE 명령 중 중요한 내용 위주로 설명하겠다.

> 원하는 옵션의 ALTER TABLE 명령이 소개돼 있지 않더라도, MySQL 서버에서 그러한 기능이 제공되지 않는다고 판단하기보다 [MySQL 서버의
> 매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html)을 참조하자.

테이블 자체에 대한 속성 변경은 주로 테이블의 문자 집합이나 스토리지 엔진, 파티션 구조 등의 변경인데, 여기서는 스토리지 엔진과 문자 집합을 변경하는 예제를 한번 살펴보겠다.

```
mysql> ALTER TABLE employees
         CONVERT TO CHARACTER SET UTF8MB4 COLLATE UTF8MB4_GENERAL_CI,
         ALGORITHM=INPLACE, LOCK=NONE;

mysql> ALTER TABLE employees ENGINE=InnoDB,
       ALGORITHM=INPLACE, LOCK=NONE;
```

위 예제의 첫 번째 ALTER TABLE은 테이블의 기본 문자 집합과 콜레이션을 변경하는 명령이다.
그뿐만 아니라 테이블의 모든 칼럼과 기존 데이터까지 모두 UTF8MB4 문자 셋의 콜레이션까지 UTF8MB4_GENERAL_CI로 변경한다.

두 번째 쿼리는 테이블의 스토리지 엔진을 변경하는 명령이다. 이 명령은 내부적인 테이블의 저장소를 변경하는 것이라서 항상 테이블의 모든 레코드를 복사하는 작업이 필요하다.
ALTER TABLE 문장에 명시된 ENGINE이 기존과 동일하더라도 테이블의 데이터를 복사하는 작업은 실행되기 때문에 주의해야 한다.
이 명령은 실제 테이블의 스토리지 엔진을 변경하는 목적으로도 사용하지만 테이블 데이터를 리빌드하는 목적으로도 사용한다.
테이블 리빌드 작업은 주로 레코드의 삭제가 자주 발생하는 테이블에서 데이터가 저장되지 않은 빈 공간(프래그멘테이션, Fragmentation)을 제거해 디스크 사용 공간을 줄이는 역할을 한다.

> 테이블이 사용하는 디스크 공간의 프래그멘테이션을 최소화하고, 테이블의 구조를 최적화하기 위한 OPTIMIZE TABLE이라는 명령이 있다.
> InnoDB 스토리지 엔진을 사용하는 테이블에 대해 OPTIMIZE TABLE이라는 명령을 사용하면 InnoDB 스토리지 엔진은 내부적으로 "ALTER TABLE ... ENGINE=InnoDB" 명령과 동일한 작업을 수행한다.
> 결국 InnoDB 테이블에서 "테이블 최적화"란 테이블의 레코드를 한 건씩 새로운 테이블에 복사함으로써 테이블의 레코드 배치를 컴팩트하게 만들어주는 것이다.

### [4] 테이블 명 변경
MySQL 서버에서 테이블명을 변경하려면 다음과 같이 RENAME TABLE 명령을 이용하면 된다.
RENAME TABLE 명령은 단순히 테이블이 이름 변경뿐만 아니라 다른 데이터베이스로 테이블을 이동할 때도 사용할 수 있다.

```
mysql> RENAME TABLE table1 TO table2;
mysql> RENAME TABLE db1.table1 TO db2.table2;
```

첫 번째 명령과 같이 동일 데이터베이스 내에서 테이블의 이름만 변경하는 작업은 단순히 메타 정보만 변경하기 때문에 매우 빠르게 처리된다.
하지만 두 번째 명령과 같이 데이터베이스를 변경하는 경우에는 메타 정보뿐만 아니라 테이블이 저장된 파일까지 다른 디렉터리(데이터베이스별로 별도 디렉터리가 할당되기 때문)로 이동해야 한다.
그런데 db1과 db2 데이터베이스가 서로 다른 파티션에 만들어졌다고 가정해보자.
일반적으로 유닉스나 윈도우에서 서로 다른 파티션으로 파일을 이동할 때는 데이터 파일을 먼저 복사하고 복사를 완료하면 원본 파티션의 파일을 삭제하는 형태로 처리한다.
MySQL 서버의 RENAME TABLE에서도 똑같이 작동한다.

RENAME TABLE을 이용해 테이블을 db1에서 db2로 이동할 때 db1과 db2가 서로 다른 운영체제의 파일 시스템을 사용하고 있었다면 이 RENAME TABLE 명령은 데이터 파일의 복사 작업이 필요하기 때문에
데이터 파일의 크기에 비례해서 시간이 소요될 것이다.

때로는 일정 주기로 테이블을 교체(Swap)해야 하는 경우도 있다. 현재 batch라는 테이블이 응용 프로그램에서 사용 중이며, batch_new라는 테이블을 생성하고 새로운 데이터를 저장하고자 한다.
그리고 최종적으로 응용 프로그램에서 사용할 수 있게 batch_new 테이블을 batch라는 이름으로 변경하는 방식으로 다음과 같이 배치 프로그램을 작성했다고 가정해보자.

```
-- // 새로운 테이블 및 데이터 생성
mysql> CREATE TABLE batch_new (...);
mysql> INSERT INTO batch_new SELECT ...;

-- // 기존 테이블과 교체
mysql> RENAME TABLE batch TO batch_old;
mysql> RENAME TABLE batch_new TO batch;
```

위와 같이 배치 프로그램이 작성되면 마지막의 기존 테이블과 신규 테이블을 교체하는 동안 일시적으로 batch 테이블이 없어지는 시점이 발생한다.
2개의 RENAME TABLE 명령이 얼마나 간격을 두고 실행되느냐에 따라 시간은 더 길어질 수도 있다. 이 시점 동안 응용 프로그램은 batch 테이블을 찾지 못해서 에러를 발생시키게 된다.

이 같은 문제점을 막기 위해 MySQL 서버의 RENAME TABLE 명령은 다음과 같이 여러 테이블의 RENAME 명령을 하나의 문장으로 묶어서 실행할 수 있다.

```
mysql> RENAME TABLE batch TO batch_old,
                    batch_new TO batch;
```

이렇게 여러 테이블의 RENAME 명령을 하나의 문장으로 묶으면 MySQL 서버는 RENAME TABLE 명령에 명시된 모든 테이블에 대해 잠금을 걸고 테이블의 이름 변경 작업을 실행하게 된다.
응용 프로그램의 입장에서 보면 batch 테이블을 조회하려고 할 때 이미 잠금이 걸려있기 때문에 대기한다.
그리고 RENAME TABLE 명령이 완료되면 batch 테이블의 잠금이 해제되어 batch 테이블(batch_new 테이블이 batch 테이블로 변경된 후)의 읽기를 실행한다.
즉 쿼리가 시작될 때와 실제 쿼리를 실행할 때의 대상 테이블이 변경됐지만 응용 프로그램은 이를 알아차리지 못하고 투명하게 실행되는 것이다. 잠깐의 잠금 대기가 발생하는 것이지 에러가 발생하지는 않는다.

### [5] 테이블 상태 조회
MySQL의 모든 테이블은 만들어진 시간, 대략의 레코드 건수, 데이터 파일의 크기 등의 정보를 가지고 있다.
또한 데이터 파일의 버전이나 레코드 포맷 등과 같이 자주 사용되지는 않지만 중요한 정보도 가지고 있는데, 이러한 정보를 조회할 수 있는 명령이 "SHOW TABLE STATUS ..."다.
SHOW TABLE STATUS 명령은 "LIKE '패턴'"과 같은 조건을 사용해 특정 테이블의 상태만 조회하는 것도 가능하다.

```
mysql> SHOW TABLE STATUS LIKE 'employees' \G
*************************** 1. row ***************************
           Name: employees
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 300252
 Avg_row_length: 57
    Data_length: 17317888
Max_data_length: 0
   Index_length: 15253504
      Data_free: 5242880
 Auto_increment: NULL
    Create_time: 2020-08-31 18:04:33
    Update_time: NULL
     Check_time: NULL
      Collation: utf8mb4_general_ci
       Checksum: NULL
 Create_options: stats_persistent=0
        Comment:
1 row in set (0.08 sec)
```

위의 SHOW TABLE STATUS 명령 결과를 보면 테이블이 어떤 스토리지 엔진을 사용하는지, 그리고 데이터 파일의 포맷으로 무엇을 사용하고 있는지 등을 조회할 수 있다.
때로는 테이블의 크기가 너무 커서 테이블의 전체 레코드 건수가 궁금한 경우에도 SHOW TABLE STATUS 명령을 유용하게 사용할 수 있다.
위의 결과에서는 대략 30만 건의 레코드를 가지고 있다는 것을 알 수 있다. 그리고 레코드 하나의 평균 크기가 대략 50바이트라는 점도 확인할 수 있다.
여기에 출력되는 레코드 건수나 레코드 평균 크기는 MySQL 서버가 예측하고 있는 값이기 때문에 테이블이 너무 작거나 너무 크면 오차가 더 커질 수도 있다.

> 위의 예제 쿼리에서 마지막에 사용된 "\G"는 레코드의 칼럼을 라인당 하나씩만 표현하게 하는 옵션이다.
> 또한 "\G"는 SQL 문장의 끝을 의미하기도 하기 때문에 "\G"가 있으면 별도로 ";"를 붙이지 않아도 쿼리 입력이 종료된 것으로 간주한다.
> 레코드의 칼럼 개수가 많거나 각 칼럼의 값이 너무 긴 경우에는 쿼리의 마지막에 "\G를 사용해 결과를 좀 더 가독성 있게 출력할 수 있다.

테이블의 상태 정보는 SHOW TABLE STATUS 명령뿐만 아니라 다음과 같이 SELECT 쿼리를 이용해서 조회할 수도 있다.

```
mysql> SELECT * FROM information_schema.TABLES
       WHERE TABLE_SCHEMA='employees' AND TABLE_NAME='employees' \G
*************************** 1. row ***************************
  TABLE_CATALOG: def
   TABLE_SCHEMA: employees
     TABLE_NAME: employees
     TABLE_TYPE: BASE TABLE
         ENGINE: InnoDB
        VERSION: 10
     ROW_FORMAT: Dynamic
     TABLE_ROWS: 300252
 AVG_ROW_LENGTH: 57
    DATA_LENGTH: 17317888
MAX_DATA_LENGTH: 0
   INDEX_LENGTH: 15253504
      DATA_FREE: 5242880
 AUTO_INCREMENT: NULL
    CREATE_TIME: 2020-08-31 18:04:33
    UPDATE_TIME: NULL
     CHECK_TIME: NULL
TABLE_COLLATION: utf8mb4_general_ci
       CHECKSUM: NULL
 CREATE_OPTIONS: stats_persistent=0
  TABLE_COMMENT:
1 row in set (0.00 sec)
```

information_schema 데이터베이스에는 MySQL 서버가 가진 스키마들에 대한 메타 정보를 가진 딕셔너리 테이블이 관리된다.
information_schema 데이터베이스에 존재하는 테이블들은 실제로 존재하는 테이블이 아니라 MySQL 서버가 시작되면서 데이터베이스의 테이블 등에 대한 다양한 메타 정보를 모아서 메모리에 모아두고
사용자가 참조할 수 있는 테이블이다. information_schema 데이터베이스의 TABLES 또는 COLUMNS 뷰를 이용하면 데이터베이스 서버에 대한 많은 정보를 얻을 수 있다.
대표적으로 다음과 같은 쿼리로 MySQL 서버에 존재하는 테이블들이 사용하는 디스크 공간 정보를 조회할 수도 있다.

```
mysql> SELECT TABLE_SCHEMA,
              SUM(DATA_LENGTH)/1024/1024 as data_size_mb,
              SUM(INDEX_LENGTH)/1024/1024 as index_size_mb
       FROM information_schema.TABLES
       GROUP BY TABLE_SCHEMA;

+--------------------+--------------+---------------+
| TABLE_SCHEMA       | data_size_mb | index_size_mb |
+--------------------+--------------+---------------+
| mysql              |   2.17187500 |    0.31250000 |
| sys                |   0.01562500 |    0.00000000 |
| information_schema |   0.00000000 |    0.00000000 |
| performance_schema |   0.00000000 |    0.00000000 |
| employees          | 190.33203125 |   93.01660156 |
+--------------------+--------------+---------------+
```

> information_schema 데이터베이스의 테이블들은 MySQL 서버가 가진 테이블들에 대한 다양한 정보를 제공한다.
> 대표적으로 다음과 같은 정보를 가지고 있으며, 여기서 모두 언급하기 어려울 정도로 많은 정보를 가지고 있다.
>
> - 데이터베이스 객체에 대한 메타 정보
> - 테이블과 칼럼에 대한 간략한 통계 정보
> - 전문 검색 디버깅을 위한 뷰(view)
> - 압축 실행과 실패 횟수에 대한 집계
>
> information_schema 데이터베이스에 어떤 정보가 관리되는지, 각 테이블의 칼럼이 가지는 값의 의미가 무엇인지는
> [매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/information-schema.html)을 참조하자.
>
> 그리고 테이블에 대한 상세한 통계 정보는 mysql 데이터베이스의 innodb_table_stats 테이블과 innodb_index_stats 테이블에도 저장돼 있으므로 함께 참조하자.

### [6] 테이블 구조 복사
테이블의 구조는 같지만 이름만 다른 테이블을 생성할 때는 SHOW CREATE TABLE 명령을 이용해 테이블의 생성 DDL을 조회한 후에 조금 변경해서 만들 수도 있다.
또한 CREATE TABLE ... AS SELECT ... LIMIT 0 명령으로 테이블을 생성할 수도 있다.
하지만 SHOW CREATE TABLE 명령을 이용하면 내용을 조금 변경해야 할 수도 있으며, CREATE TABLE ... AS SELECT ... LIMIT 0은 인덱스가 생성되지 않는다는 단점이 있다.
데이터는 복사하지 않고 테이블의 구조만 동일하게 복사하는 명령으로 "CREATE TABLE ... LIKE"을 사용하면 구조가 같은 테이블을 손쉽게 생성할 수 있다.

```
mysql> CREATE TABLE temp_employees LIKE employees;
```

위의 명령은 employees 테이블에 존재하는 모든 칼럼과 인덱스가 같은 temp_employees라는 테이블을 생성하는 예제다.
CREATE TABLE ... AS SELECT ... 와 마찬가지로 데이터까지 복사하려면 CREATE TABLE ... LIKE 명령을 실행하고 다음과 같은 INSERT ... SELECT 명령을 실행하면 된다.

```
mysql> INSERT INTO temp_employees SELECT * FROM employees;
```

> MySQL 서버는 특정 테이블에 대해 트랜잭션 로그를 활성화하거나 비활성화하는 기능은 제공하지 않는다.
> 그래서 CTAS 구문(CREATE TABLE ... AS SELECT ...)은 리두 로그를 기록하지 않기 때문에 성능이 빠르다는 이야기는 아직 MySQL 서버에는 해당하지 않는다.
> 결국 MySQL 서버에서 CREATE TABLE ... AS SELECT ... 구문은 CREATE TABLE과 INSERT .. SELECT ... 문장으로 나눠서 실행하는 것과 성능적인 면에서 차이는 없다.
>
> CREATE TABLE ... AS SELECT ... 구문을 실행할 때 리두 로그에 기록되는지 여부는 다음과 같이 간단히 확인해볼 수 있다.
>
> ```
> mysql> SHOW ENGINE INNODB STATUS \G
> ...
> ---
> LOG
> ---
> Log sequence number       4850618449
> ...
>
> mysql> CREATE TABLE salaries_temp AS SELECT * FROM salaries;
> Query OK, 2844047 rows affected (18.73 sec)
> mysql> SHOW ENGINE INNODB STATUS \G
> ...
> ---
> LOG
> ---
> Log sequence number       5070032564
> ...
> ```
>
> 이 결과를 보면 salaries 테이블 하나를 복사함으로써 대략 209MB 정도((5070032564 - 4850618449)/1024/1024)의 리두 로그가 증가했다는 것을 알 수 있다.
> ```

### [7] 테이블 삭제
일반적으로 MySQL에서 레코드가 많지 않은 테이블을 삭제하는 작업은 서비스 도중이라고 하더라도 문제가 되지 않는다.
MySQL 8.0 버전에서는 특정 테이블을 삭제하는 작업이 다른 테이블의 DML이나 쿼리를 직접 방해하지는 않는다. MySQL 서버에서 테이블 삭제는 DROP TABLE 명령으로 실행한다.

```
mysql> DROP TABLE [ IF EXISTS ] table1;
```

하지만 용량이 매우 큰 테이블을 삭제하는 작업은 상당히 부하가 큰 작업에 속한다.
테이블이 삭제되면 MySQL 서버는 해당 테이블이 사용하던 데이터 파일을 삭제해야 하는데, 이 파일의 크기가 매우 크고 디스크에서 파일의 조각들이 너무 분산되어 저장돼 있다면 많은 디스크 읽고 쓰기 작업이
필요하다. MySQL 서버의 디스크 읽고 쓰기 부하가 높아지면 다른 커넥션의 쿼리 처리 성능이 떨어질 수도 있다.
테이블 삭제가 직접 다른 커넥션의 쿼리를 방해하지는 않지만 간접적으로는 영향을 미칠 수도 있다. 그래서 테이블이 크다면 서비스 도중에 삭제 작업(DROP TABLE)은 수행하지 않는 것이 좋다.

> MySQL 서버의 데이터 파일이 리눅스의 ext3 파일 시스템을 사용하는 경우 파일의 조각이 디스크이 이곳저곳에 분산되어 저장된다.
> 이로 인해 ext3 파일 시스템의 경우 대용량 테이블 삭제 작업은 디스크 읽고 쓰기 작업을 상당히 많이 발생시킬 수도 있다.
> 하지만 최근에 많이 사용되는 리눅스 운영체제에서는 대부분 ext4 파일 시스템 또는 xls를 사용하는데, 이 경우 파일 삭제 시 디스크 읽고 쓰기 작업이 많이 줄어들었다.
> 아직 ext3 파일 시스템을 사용한다면 테이블을 삭제할 때 주의하자.

테이블 삭제에서 한 가지 더 주의해야 하는 것은 InnoDB 스토리지 엔진의 어댑티브 해시 인덱스(Adaptive hash index)다.
어댑티브 해시 인덱스는 InnoDB 버퍼 풀의 각 페이지가 가진 레코드에 대한 해시 인덱스 기능을 제공하는데, 어댑티브 해시 인덱스가 활성화돼 있는 경우 테이블이 삭제되면 어댑티브 해시 인덱스 정보도 모두
삭제돼야 한다.
어댑티브 해시 인덱스가 삭제될 테이블에 대한 정보를 많이 가지고 있다면 어댑티브 해시 인덱스 삭제 작업으로 인해 MySQL 서버의 부하가 높아지고 간접적으로 다른 쿼리 처리에 영향을 미칠 수도 있다.
어댑티브 해시 인덱스는 자주 사용되는 테이블에 대해서만 해시 인덱스를 빌드하기 때문에 거의 사용되지 않는 테이블이었다면 크게 문제되지 않을 수도 있다.
어댑티브 해시 인덱스는 테이블 삭제뿐만 아니라 테이블의 스키마 변경에도 영향을 미칠 수 있다.
<br/>
<br/>
## (5) 칼럼 변경
테이블 구조 변경 작업은 대부분 칼럼을 추가하거나 칼럼 타입을 변경하는 작업이다. ALTER TABLE 명령을 이용한 칼럼 추가 및 삭제, 칼럼 이름 변경, 칼럼 타입 변경에 대해 간단히 살펴보자.

### [1] 칼럼 추가
MySQL 8.0 버전으로 업그레이드되면서 테이블의 칼럼 추가 작업은 대부분 INPLACE 알고리즘을 사용하는 온라인 DDL로 처리가 가능하다.
그뿐만 아니라 칼럼을 테이블의 제일 마지막 칼럼으로 추가하는 경우에는 INSTANT 알고리즘으로 즉시 추가된다.

```
-- // 테이블의 제일 마지막에 새로운 칼럼을 추가
mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20),
         ALGORITHM=INSTANT;

-- // 테이블의 중간에 새로운 칼럼을 추가
mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20) AFTER emp_no,
         ALGORITHM=INPLACE, LOCK=NONE;
```

위 예제의 첫 번째 DDL 문장은 테이블의 마지막에 새로운 칼럼을 추가하므로 INSTANT 알고리즘으로 즉시 추가가 가능하다.
하지만 두 번째 예제는 테이블의 기존 칼럼 중간에 새로 추가하기 때문에 테이블의 리빌드가 필요하다. 그래서 INSTANT 알고리즘으로 처리가 불가능하며, INPLACE 알고리즘으로 처리돼야 한다.
그래서 테이블이 큰 경우라면 가능하다면 칼럼을 테이블의 마지막 칼럼으로 추가하는 것이 좋다.

> 스키마 변경 작업의 종류별로 어떤 알고리즘이 가능한지를 기억하기는 쉽지 않다.
> 그래서 항상 ALTER TABLE 명령에는 ALGORITHM과 LOCK 절을 추가해서 원하는 성능과 잠금 레벨로 스키마 변경이 가능한지 차례대로 다음과 같이 확인해가면서 스키마 변경을 해보는 것을 권장한다.
>
> ```
> mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20) AFTER emp_no, ALGORITHM=INSTANT;
> mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20) AFTER emp_no, ALGORITHM=INPLACE,
> LOCK=NONE;
> ...
> ```
>
> 참고로 스키마 변경이 INSTANT 알고리즘을 사용하는 경우 아주 일시적인 메타데이터 잠금만 사용되기 때문에 사용자가 잠금을 제어할 수 없다.
> 그래서 ALGORITHM=INSTANT인 경우에는 LOCK 절을 사용할 수 없다. 물론 ALGORITHM=INSTANT 알고리즘을 사용하는 경우에는 잠금 시간이 매우 짧아서 별도로 잠금을 제어할 필요도 없다.

### [2] 칼럼 삭제
칼럼을 삭제하는 작업은 항상 테이블의 리빌드를 필요로 하기 때문에 INSTANT 알고리즘을 사용할 수 없다. 그래서 항상 INPLACE 알고리즘으로만 칼럼 삭제가 가능하다.
다음 예제의 ALTER TABLE 명령에서 COLUMN 키워드는 입력하지 않아도 무방하다.

```
mysql> ALTER TABLE employees DROP COLUMN emp_telno,
       ALGORITHM=INPLACE, LOCK=NONE;
```

### [3] 칼럼 이름 및 칼럼 타입 변경
칼럼의 이름이나 타입을 변경하는 방법은 다음과 같다.
칼럼의 타입 변경은 현재 칼럼의 타입과 변경하고자 하는 데이터 타입에 따라 매우 다양한 형태가 될 수 있는데, 여기서는 간략하게 자주 사용되는 타입 변환만 살펴보자.

```
-- // 칼럼의 이름 변경
mysql> ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL,
             ALGORITHM=INPLACE, LOCK=NONE;

-- // INT 칼럼을 VARCHAR 타입으로 변경
mysql> ALTER TABLE salaries MODIFY salary VARCHAR(20),
             ALGORITHM=COPY, LOCK=SHARED;

-- // VARCHAR 타입의 길이 확장
mysql> ALTER TABLE employees MODIFY last_name VARCHAR(30) NOT NULL,
             ALGORITHM=INPLACE, LOCK=NONE;

-- // VARCHAR 타입의 길이 축소
mysql> ALTER TABLE employees MODIFY last_name VARCHAR(10) NOT NULL,
             ALGORITHM=COPY, LOCK=SHARED;
```

- 첫 번째 DDL은 salaries 테이블의 to_date 칼럼의 이름만 end_date로 변경하는 예제다.
  이렇게 칼럼의 이름을 변경하는 작업은 INPLACE 알고리즘을 사용하지만 실제 데이터 리빌드 작업은 필요치 않다. 그래서 INSTANT 알고리즘과 같이 빠르게 작업이 완료된다.

- 두 번째 DDL은 INT 타입의 칼럼을 VARCHAR 타입으로 변경하는 예제다.
  이렇게 칼럼의 데이터 타입이 변경되는 경우 COPY 알고리즘이 필요하며 온라인 DDL로 실행돼도 스키마 변경 도중에는 테이블의 쓰기 작업은 불가하다.

- 세 번째 DDL은 VARCHAR 타입의 길이를 16에서 30으로 변경하는 예제다.
  VARCHAR 타입의 길이를 확장하는 경우는 현재 길이와 확장하는 길이의 관계에 따라 테이블의 리빌드가 필요할 수도 있고 아닐 수도 있다.

- 네 번째 DDL은 VARCHAR 타입의 길이를 16에서 10으로 변경하는 예제다. VARCHAR 타입의 길이를 축소하는 경우는 완전히 다른 타입으로 변경되는 경우와 같이 COPY 알고리즘을 사용해야 한다.
  그리고 스키마를 변경하는 중 해당 테이블의 데이터 변경은 허용되지 않으므로 LOCK은 SHARED로 사용돼야 한다.

아마도 칼럼의 타입을 변경하는 경우 중 가장 빈번한 경우가 VARCHAR나 VARBINARY 타입의 길이를 확장하는 것일 것이다.
VARCHAR나 VARBINARY 타입의 경우 칼럼의 최대 허용 사이즈는 메타데이터에 저장되지만 실제 칼럼이 가지는 값의 길이는 데이터 레코드의 칼럼 헤더에 저장된다.
그런데 값의 길이를 위해서 사용하는 공간의 크기는 VARCHAR 칼럼이 최대 가질 수 있는 바이트 수만큼 필요하다.
즉 칼럼값의 길이 저장용 공간은 칼럼의 값이 최대 가질 수 있는 바이트 수가 255 이하인 경우 1바이트만 사용하며, 256 바이트 이상인 경우 2바이트를 사용한다.
그래서 동일한 값이라고 하더라도 VARCHAR(10) 칼럼에 저장될 때보다 VARCHAR(1000) 칼럼에 저장될 때는 1바이트를 더 사용한다.

INPLACE 알고리즘으로 VARCHAR(10)에서 VARCHAR(20)으로 변경하는 경우라면 둘 다 255바이트 이하이므로 테이블 리빌드가 필요 없다.
하지만 UTF8MB4 문자 셋을 사용하는 경우 VARCHAR(10)에서 VARCHAR(64)로 변경하는 경우에는 테이블 리빌드가 필요하다.
UTF8MB4 문자 셋은 한 글자가 최대 4바이트를 사용할 수 있기 때문에 VARCHAR(64)는 최대 256바이트를 사용한다.
그래서 이 경우에는 칼럼값의 길이를 1바이트에서 2바이트로 변경해야 하므로 테이블의 레코드 전체를 다시 리빌드해야 한다.
<br/>
<br/>
## (6) 인덱스 변경
MySQL 8.0 버전에서는 대부분의 인덱스 변경 작업이 온라인 DDL로 처리 가능하도록 개선됐다. 여기서는 인덱스의 종류별로 추가 및 변경, 삭제하는 방법을 살펴보겠다.

> MySQL 서버에서 전문 검색 인덱스와 공간 검색 인덱스를 제외하면 나머지 인덱스는 모두 B-Tree 자료 구조를 사용한다.
> MySQL 서버의 매뉴얼에서는 "USING BTREE" 또는 "USING HASH" 절을 이용해 해시 인덱스를 지원하는 것으로 보이지만 "USING HASH" 절은 MySQL Cluster (NDB)를 위한 옵션이지 MySQL
> 서버의 InnoDB나 MyISAM 스토리지 엔진을 위한 옵션이 아니다.

### [1] 인덱스 추가
MySQL 서버에서 사용 가능한 인덱스의 종류나 인덱싱 알고리즘별로 대략 사용 가능한 ALTER TABLE ADD INDEX 문장의 형태를 나열해봤다.
다음 예제에서는 각 인덱스의 종류나 알고리즘별로 온라인 DDL이 가능한지, 어떤 알고리즘과 잠금으로 생성 가능한지도 함께 추가했다.

```
mysql> ALTER TABLE employees ADD PRIMARY KEY (emp_no),
             ALGORITHM=INPLACE, LOCK=NONE;

mysql> ALTER TABLE employees ADD UNIQUE INDEX ux_empno (emp_no),
             ALGORITHM=INPLACE, LOCK=NONE;

mysql> ALTER TABLE employees ADD INDEX ix_lastname (last_name),
             ALGORITHM=INPLACE, LOCK=NONE;

mysql> ALTER TABLE employees ADD FULLTEXT INDEX fx_firstname_lastname (first_name, last_name),
             ALGORITHM=INPLACE, LOCK=SHARED;

mysql> ALTER TABLE employees ADD SPATIAL INDEX fx_loc (last_location),
             ALGORITHM=INPLACE, LOCK=SHARED;
```

전문 검색을 위한 인덱스와 공간 검색을 위한 인덱스는 INPLACE 알고리즘으로 인덱스 생성이 가능하지만 SHARED 잠금이 필요하다는 것을 알 수 있다.
그러나 나머지 B-Tree 자료 구조를 사용하는 인덱스의 추가는 프라이머리 키라고 하더라도 INPLACE 알고리즘에 잠금 없이 온라인으로 인덱스 생성이 가능한 것을 알 수 있다.

### [2] 인덱스 조회
MySQL 서버에서 인덱스의 목록을 조회할 때는 SHOW INDEXES 명령을 사용하거나 SHOW CREATE TABLE 명령으로 표시되는 테이블 생성 명령을 참조하면 된다.

```
mysql> SHOW INDEX FROM employees;
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+
| Table     | Non_unique | Key_name              | Seq_in_index | Column_name | Collation | Cardinality |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+
| employees |          0 | PRIMARY               |            1 | emp_no      | A         |      286659 |
| employees |          1 | ix_firstname          |            1 | first_name  | A         |        1021 |
| employees |          1 | ix_hiredate           |            1 | hire_date   | A         |        3628 |
| employees |          1 | ix_gender_birthdate   |            1 | gender      | A         |           4 |
| employees |          1 | ix_gender_birthdate   |            2 | birth_date  | A         |       10876 |
| employees |          1 | fx_firstname_lastname |            1 | first_name  | NULL      |      300030 |
| employees |          1 | fx_firstname_lastname |            2 | last_name   | NULL      |      300030 |
+-----------+------------+-----------------------+--------------+-------------+-----------+-------------+
```

SHOW INDEXES 명령은 테이블의 인덱스만 표시하는데, 인덱스 칼럼별로 한 줄씩 표시해준다.
표시된 결과에서 Key_name 칼럼은 인덱스의 이름을 나타내고, Seq_in_index 칼럼의 값은 인덱스에서 해당 칼럼의 위치를 보여준다.
단일 칼럼으로 생성된 인덱스는 Seq_in_index 칼럼이 1만 표시되며, 복합 칼럼 인덱스인 경우 Seq_in_index 칼럼의 값이 1부터 2, 3, 4, ... 형태로 증가한다.
Cardinality 칼럼에는 인덱스에서 해당 칼럼까지의 유니크한 값의 개수를 보여준다.
단일 칼럼으로 구성된 인덱스의 경우 해당 칼럼이 가지는 유니크한 값의 개수를 표시하지만 복합 칼럼 인덱스인 경우 인덱스의 첫 번째 칼럼부터 해당 칼럼까지의 조합으로 유니크한 값의 개수를 표시한다.
예를 들어, ix_gender_birthdate 인덱스의 경우 gender 칼럼과 birth_date 칼럼의 조합으로 인덱스가 생성돼 있는데, gender 칼럼까지만 고려하면 유니크한 값의 개수가 4개이며, gender와
birth_date 칼럼의 조합으로 따지면 유니크한 값의 개수가 10876개라는 것을 의미한다.

SHOW CREATE TABLE 명령은 다음과 같이 테이블의 생성 구문을 그대로 보여준다. 인덱스와 함께 테이블의 모든 칼럼까지 표시하기 때문에 장황해 보일 수 있다.
하지만 인덱스별로 한 줄로 표시하기 때문에 어떤 인덱스가 있는지, 그 인덱스에 어떤 칼럼이 어떤 순서로 구성돼 있는지 파악하기 쉽다.

```
mysql> SHOW CREATE TABLE employees;

CREATE TABLE `employees` (
  `emp_no` int NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) COLLATE utf8mb4_general_ci NOT NULL,
  `last_name` varchar(16) COLLATE utf8mb4_general_ci NOT NULL,
  `gender` enum('M','F') COLLATE utf8mb4_general_ci NOT NULL,
  `hire_date` date NOT NULL,
  `emp_telno` varchar(20) COLLATE utf8mb4_general_ci DEFAULT 'abc',
  PRIMARY KEY (`emp_no`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_hiredate` (`hire_date`),
  KEY `ix_gender_birthdate` (`gender`,`birth_date`),
  FULLTEXT KEY `fx_firstname_lastname` (`first_name`,`last_name`)
)
```

### [3] 인덱스 이름 변경
쿼리 문장에서 인덱스의 이름을 힌트로 사용하면 MySQL 서버에서 해당 인덱스를 삭제하거나 다른 인덱스로 대체하는 경우 응용 프로그램의 코드를 변경해야 했다.
물론 인덱스를 삭제하고 동일 이름으로 새로운 인덱스를 생성하면 되지만 이 순서로 작업을 실행하면 새로운 인덱스를 생성하는 동안 필요한 인덱스가 없어지므로 손쉽게 작업하기가 어려웠다.
사실 이는 간단히 인덱스의 이름만 변경할 수 있다면 새로운 인덱스로 기존 인덱스를 대체할 수 있지만 MySQL 5.6 버전까지도 인덱스의 이름을 변경할 수 있는 방법이 없었다.

MySQL 5.7 버전부터는 다음과 같이 인ㄷ게스의 이름을 변경할 수 있게 됐다.

```
mysql> ALTER TABLE salaries RENAME INDEX ix_salary TO ix_salary2,
         ALGORITHM=INPLACE, LOCK=NONE;
```

인덱스의 이름을 변경하는 작업은 INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 필요로 하지는 않는다.
그래서 응용 프로그램에서 힌트로 해당 인덱스의 이름을 사용 중이라고 하더라도 짧은 시간에 인덱스를 교체할 수 있게 됐다.
다음 예제는 employees 테이블이 이미 가지고 있던 "ix_firstname(first_name)"를 대신해서 "ix_firstname (first_name, last_name)" 인덱스를 교체하는 작업 방식을 보여준다.

```
-- // 1. index_new라는 이름으로 새로운 인덱스를 생성
mysql> ALTER TABLE employees
         ADD INDEX index_new (first_name, last_name),
         ALGORITHM=INPLACE, LOCK=NONE;

-- // 2. 기존 인덱스(ix_firstname)를 삭제하고,
-- //    동시에 새로운 인덱스(index_new)의 이름을 ix_firstname으로 변경
mysql> ALTER TABLE employees
         DROP INDEX ix_firstname,
         RENAME INDEX index_new TO ix_firstname,
         ALGORITHM=INPLACE, LOCK=NONE;
```

### [4] 인덱스 가시성 변경
MySQL 서버에서 인덱스를 삭제하는 작업은 ALTER TABLE DROP INDEX 명령으로 즉시 완료된다. 하지만 한 번 삭제된 인덱스를 새로 생성하는 것은 매우 많은 시간이 걸릴 수도 있다.
특정 인덱스를 사용하지 않는다고 판단하고 삭제했는데, 실제 그 인덱스를 사용하는 쿼리가 있었다면 어떻게 될지는 이미 예측할 수 있을 것이다.
최악의 경우에는 응용 프로그램의 서비스를 멈추고, 인덱스를 다시 생성하고 응용 프로그램을 다시 시작해야 한다.
그래서 인덱스나 테이블을 삭제하는 작업은 데이터베이스 관리자에게는 매우 긴장되는 작업이었으며, 이러한 이유로 데이터베이스 서버의 인덱스는 한 번 생성되면 거의 삭제하지 못하는 경우가 많다.

하지만 MySQL 8.0 버전부터는 인덱스의 가시성을 제어할 수 있는 기능이 도입됐다. 인덱스의 가시성이란 MySQL 서버가 쿼리를 실행할 때 해당 인덱스를 사용할 수 있게 할지 말지를 결정하는 것이다.
다음 예제는 응용 프로그램의 쿼리에서 특정 인덱스가 사용되지 못하게 하는 DDL 문장이다.

```
mysql> ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;
```

인덱스가 INVISIBLE 상태로 변경되면 MySQL 옵티마이저는 INVISIBLE 상태의 인덱스는 없는 것으로 간주하고 실행 계획을 수립한다.
다음 예제는 first_name 칼럼의 인덱스가 INVISIBLE 상태로 바뀌기 전과 후의 실행 계획 변화를 보여준다.

```
mysql> EXPLAIN SELECT * FROM employees WHERE first_name='Matt';
+----+-------------+-----------+------+--------------+------+-------+
| id | select_type | table     | type | key          | rows | Extra |
+----+-------------+-----------+------+--------------+------+-------+
|  1 | SIMPLE      | employees | ref  | ix_firstname |  233 | NULL  |
+----+-------------+-----------+------+--------------+------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;
Query OK, 0 rows affected (0.01 sec)

mysql> EXPLAIN SELECT * FROM employees WHERE first_name='Matt';
+----+-------------+-----------+------+------+--------+-------------+
| id | select_type | table     | type | key  | rows   | Extra       |
+----+-------------+-----------+------+------+--------+-------------+
|  1 | SIMPLE      | employees | ALL  | NULL | 300030 | Using where |
+----+-------------+-----------+------+------+--------+-------------+
```

INVISIBLE 상태의 인덱스를 다시 사용할 수 있게 하려면 VISIBLE 옵션을 명시하면 된다.

```
mysql> ALTER TABLE employees ALTER INDEX ix_firstname VISIBLE;
```

ALTER TABLE ... ALTER INDEX ... [VISIBLE | INVISIBLE] 명령은 메타데이터만 변경하기 때문에 온라인 DDL로 실행되는지 여부를 고려하지 않아도 된다.
MySQL 8.0 버전부터는 인덱스를 삭제하기 전에 먼저 해당 인덱스를 보이지 않게 변경해서 하루 이틀 정도 상황을 모니터링한 후 안전하게 인덱스를 삭제할 수 있게 됐다.

그뿐만 아니라 최초 인덱스를 생성할 때도 가시성을 설정할 수 있다.
SHOW CREATE TABLE 명령으로 테이블의 구조를 살펴보면 추가된 ix_firstname_lastname 인덱스의 끝부분에 "INVISIBLE" 키워드가 설정된 것을 확인할 수 있다.

```
mysql> ALTER TABLE employees ADD INDEX ix_firstname_lastname (first_name, last_name) INVISIBLE;
Query OK, 0 rows affected (0.55 sec)

mysql> SHOW CREATE TABLE employees \G
CREATE TABLE `employees` (
  `emp_no` int NOT NULL,
  ...
  PRIMARY KEY (`emp_no`),
  KEY `ix_hiredate` (`hire_date`),
  KEY `ix_gender_birthdate` (`gender`,`birth_date`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_firstname_lastname` (`first_name`,`last_name`) /*!80000 INVISIBLE */
) ENGINE=InnoDB
```

새로운 인덱스를 생성하는 것은 크게 문제되지 않을 거라고 생각할 수도 있다.
하지만 비슷한 칼럼으로 구성된 인덱스가 많아지면 MySQL 옵티마이저는 기존에 사용하던 인덱스와는 다른 인덱스를 사용할 수도 있다. 물론 더 성능이 빨라질 수도 있지만 성능이 더 악화될 수도 있다.
이러한 부분이 우려된다면 인덱스를 처음 생성할 때는 INVISIBLE 인덱스로 생성하고, 적절히 부하가 낮은 시점을 골라서 인덱스를 VISIBLE로 변경하면 된다.
서버의 성능이 떨어진다면 다시 INVISIBLE로 바꾸고 원인을 좀 더 분석해볼 수도 있다. 즉 인덱스를 생성하고 삭제하는 작업을 하지 않고도 쿼리가 인덱스를 사용할지 말지를 변경할 수 있게 됐다.

MySQL 서버의 optimizer_switch 시스템 변수에 use_invisible_indexes 옵션이 ON으로 설정된 경우 MySQL 옵티마이저는 쿼리가 INVISIBLE 상태의 인덱스도 사용할 수 있게 한다.
optimizer_switch의 use_invisible_indexes 옵션의 기본값은 OFF로 설정돼 있다.

### [5] 인덱스 삭제
ALTER TABLE ... DROP INDEX ... 명령으로 삭제할 수 있다. MySQL 서버의 인덱스 삭제는 일반적으로 매우 빨리 처리된다.
세컨더리 인덱스 삭제 작업은 INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 필요로 하지는 않는다.
하지만 프라이머리 키의 삭제 작업은 모든 세컨더리 인덱스의 리프 노드에 저장된 프라이머리 키 값을 삭제해야 하기 때문에 임시 테이블로 레코드를 복사해서 테이블을 재구축해야 한다.

다음 예제는 종류별로 인덱스를 삭제하는 명령이다.
프라이머리 키 삭제는 COPY 알고리즘을 사용해야 하며, 프라이머리 키 삭제 도중 레코드 쓰기는 불가능한 SHARED 모드의 잠금이 필요하다는 것을 확인할 수 있다.

```
mysql> ALTER TABLE employees DROP PRIMARY KEY, ALGORITHM=COPY, LOCK=SHARED;
mysql> ALTER TABLE employees DROP INDEX ux_empno, ALGORITHM=INPLACE, LOCK=NONE;
mysql> ALTER TABLE employees DROP INDEX fx_loc, ALGORITHM=INPLACE, LOCK=NONE;
```
<br/>

## (7) 테이블 변경 묶음 실행
하나의 테이블에 대해 여러 가지 스키마 변경을 해야 하는 경우 개별 ALTER TABLE 명령을 차례대로 실행하는 경우를 본 적이 있다.
온라인 DDL로 빠르게 스키마 변경을 처리할 수 있다면 개별로 실행하는 것이 좋지만 그렇지 않다면 모아서 실행하는 것이 효율적이다. 다음과 같은 인덱스를 생성해야 한다고 가정해보자.

```
mysql> ALTER TABLE employees ADD INDEX ix_lastname (last_name, first_name),
             ALGORITHM=INPLACE, LOCK=NONE;

mysql> ALTER TABLE employees ADD INDEX ix_birthdate (birth_date),
             ALGORITHM=INPLACE, LOCK=NONE;
```

2개의 ALTER TABLE 명령으로 인덱스를 각각 생성하면 인덱스를 생성할 때마다 테이블의 레코드를 풀스캔해서 인덱스를 생성하게 된다.
하지만 다음과 같이 하나의 ALTER TABLE 명령으로 모아서 실행하면 MySQL 서버는 테이블의 레코드를 한 번만 풀 스캔해서 2개의 인덱스를 한꺼번에 생성할 수 있게 된다.

```
mysql> ALTER TABLE employees
         ADD INDEX ix_lastname (last_name, first_name),
         ADD INDEX ix_birthdate (birth_date),
         ALGORITHM=INPLACE, LOCK=NONE;
```

물론 2개의 인덱스를 한 번에 생성하면 인덱스 하나를 생성할 때보다는 더 많은 시간이 걸리겠지만 2개의 인덱스를 각각 ALTER TABLE 명령으로 생성하는 데 걸리는 시간보다는 훨씬 시간을 단축할 수 있다.

2개의 스키마 변경 작업이 하나는 INSTANT 알고리즘을 사용하고 다른 하나는 INPLACE 알고리즘을 사용한다면 굳이 일허게 모아서 실행할 필요는 없다.
가능하면 같은 알고리즘을 사용하는 스키마 변경 작업이라면 모아서 실행하는 것이 효율적일 것이다.
INPLACE 알고리즘을 사용한다고 하더라도 테이블 리빌드가 필요한 작업과 그렇지 않은 작업끼리도 구분하고 모아서 실행할 수 있다면 더 효율적으로 스키마 관리를 할 수 있다.
<br/>
<br/>
## (8) 프로세스 조회 및 강제 종료
MySQL 서버에 접속된 사용자의 목록이나 각 클라이언트 사용자가 현재 어떤 쿼리를 실행하고 있는지는 SHOW PROCESSLIST 명령으로 확인할 수 있다.

```
mysql> SHOW PROCESSLIST;
+------+---------+----------------+------+---------+---------+--------------+------------------+
| Id   | User    | Host           | db   | Command | Time    | State        | Info             |
+------+---------+----------------+------+---------+---------+--------------+------------------+
| 2740 | db_user | 192.168.0.1:14 | db1  | Sleep   |     527 |              | NULL             |
| 4228 | db_user | 192.168.0.1:15 | db1  | Query   |   53216 | Sending data | SELECT ...       |
| 6243 | root    | localhost      | NULL | Query   |       0 | NULL         | SHOW PROCESSLIST |
+------+---------+----------------+------+---------+---------+--------------+------------------+
```

SHOW PROCESSLIST 명령의 결과에는 현재 MySQL 서버에 접속된 클라이언트의 요청을 처리하는 스레드 수만큼의 레코드가 표시된다. 각 칼럼에 포함된 값의 의미는 다음과 같다.

- Id: MySQL 서버의 스레드 아이디이며, 쿼리나 커넥션을 강제 종료할 때는 이 칼럼(id) 값을 식별자로 사용한다.
- User: 클라이언트가 MySQL 서버에 접속할 때 인증에 사용한 사용자 계정을 의미한다.
- Host: 클라이언트의 호스트명이나 IP 주소가 표시된다.
- db: 클라이언트가 기본으로 사용하는 데이터베이스의 이름이 표시된다.
- Command: 해당 스레드가 현재 어떤 작업을 처리하고 있는지 표시한다.
- Time: Command 칼럼에 표시되는 작업이 얼마나 실행되고 있는지 표시한다.
  위의 예제에서 두 번째 라인은 53216초 동안 SELECT 쿼리를 실행하고 있음을 보여주고, 첫 번째 라인은 이 스레드가 대기(Sleep) 상태로 527초 동안 아무것도 하지 않고 있음을 보여준다.
- State: Command 칼럼에 표시되는 내용이 해당 스레드가 처리하고 있는 작업의 큰 분류를 보여준다면 State 칼럼에는 소분류 작업 내용을 보여준다. 이 칼럼에 표시될 수 있는 내용은 상당히 많다.
  자세한 내용은 [MySQL 매뉴얼의 "스레드의 상태 모니터링"](https://dev.mysql.com/doc/refman/8.0/en/thread-information.html)을 참조하자.
- Info: 해당 스레드가 실행 중인 쿼리 문장을 보여준다. 쿼리는 화면의 크기에 맞춰서 표시 가능한 부분까지만 표시된다. 쿼리의 모든 내용을 확인하려면 SHOW FULL PROCESSLIST 명령을 사용하면 된다.

SHOW PROCESSLIST 명령은 MySQL 서버가 어떤 상태인지를 판단하는 데도 많은 도움이 된다.
일반적으로 쾌적한 상태로 서비스되는 MySQL에서는 대부분 프로세스의 Command 칼럼이 Sleep 상태로 표시된다.
그런데 Command 칼럼의 값이 "Query"이면서 Time이 상당히 큰 값을 가지고 있다면 쿼리가 상당히 장시간 실행되고 있음을 의미한다.

SHOW PROCESSLIST의 결과에서 특별히 관심을 둬야 할 부분은 State 칼럼의 내용이다.
State 칼럼에 표시될 수 있는 값은 종류가 다양한데, 대표적으로 "Copying ..." 그리고 "Sorting ..."으로 시작하는 값들이 표시될 때는 주의 깊게 살펴봐야 한다.
각 Command나 State 칼럼에 표시될 수 있는 내용은 ["스레드의 상태 모니터링"](https://dev.mysql.com/doc/refman/8.0/en/thread-information.html)을 참고한다.

SHOW PROCESSLIST 명령의 결과에서 Id 칼럼값은 접속된 커네션의 요청을 처리하는 전용 스레드의 번호를 의미한다.
특정 스레드에서 실행 중인 쿼리나 커넥션 자체를 강제 종료하려면 KILL 명령을 사용하면 된다.

```
mysql> KILL QUERY 4228;
mysql> KILL 4228;
```

위 예제의 첫 번째 명령은 아이디가 4228인 스레드가 실행 중인 쿼리는 강제 종료시키지만 커넥션 자체는 그대로 유지한다.
반면 두 번째 명령은 해당 4228번 스레드가 실행하고 있는 쿼리뿐만 아니라 해당 커넥션까지 강제 종료시키는 명령이다.
커넥션이 강제 종료되면 그 커넥션에서 처리하고 있던 트랜잭션은 자동으로 롤백 처리된다.
<br/>
<br/>
## (9) 활성 트랜잭션 조회
쿼리가 오랜 시간 실행되고 있는 경우도 문제지만 트랜잭션이 오랜 시간 완료되지 않고 활성 상태로 남아있는 것도 MySQL 서버의 성능에 영향을 미칠 수 있다.
MySQL 서버의 트랜잭션 목록은 다음과 같이 information_schema.innodb_trx 테이블을 통해 확인할 수 있다.

```
mysql> SELECT trx_id,
         (SELECT CONCAT(user,'@',host)
          FROM information_schema.processlist
          WHERE id=trx_mysql_thread_id) AS source_info,
         trx_state,
         now(),
         (unix_timestamp(now()) - unix_timestamp(trx_started)) AS lasting_sec,
         trx_requested_lock_id,
         trx_wait_started,
         trx_mysql_thread_id,
         trx_tables_in_use,
         trx_tables_locked
       FROM information_schema.innodb_trx
       WHERE (unix_timestamp(now()) - unix_timestamp(trx_started))>5 \G
************************** 1. row **************************
               trx_id: 23000
          source_info: root@localhost
            trx_state: RUNNING
          trx_started: 2020-09-01 22:50:49
                now(): 2020-09-01 22:53:39
          lasting_sec: 170
trx_requested_lock_id: NULL
     trx_wait_started: NULL
  trx_mysql_thread_id: 14
    trx_tables_in_use: 0
    trx_tables_locked: 1
1 row in set (0.00 sec)
```

위의 쿼리는 트랜잭션이 5초 이상 활성 상태로 남아있는 프로세스만 조사하는 쿼리다.
조회 결과를 보면 14번 프로세스(trx_mysql_thread_id 칼럼의 값)는 "root@localhost" 계정으로 접속했으며, 현재 170초(lasting_sec 칼럼의 값) 동안 활성 트랜잭션(Active
Transaction) 상태를 유지하고 있다는 것을 알 수 있다.
평상시보다 오랜 시간 트랜잭션이 활성 상태를 유지하고 있다면 information_schema.innodb_trx 테이블에서 모든 정보를 조회해서 살펴보면 이 트랜잭션이 얼마나 많은 레코드를 변경했고 얼마나 많은
레코드를 잠그고 있는지 확인할 수 있다.

```
mysql> SELECT * FROM information_schema.innodb_trx WHERE trx_id=23000 \G
************************** 1. row **************************
                    trx_id: 23000
                 trx_state: RUNING
               trx_started: 2020-09-01 22:50:49
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 14
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
       trx_schedule_weight: NULL
```

위 결과의 trx_rows_modified 칼럼과 trx_rows_locked 칼럼의 값을 참조해보면 23000번 트랜잭션은 1개의 레코드를 변경했고 1개의 레코드에 대해서 잠금을 가지고 있는 것을 확인할 수 있다.
이때 어떤 레코드를 잠그고 있는지는 performance_schema.data_locks 테이블을 참조하면 된다.

```
mysql> SELECT * FROM data_locks \G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 5022884824:1361:140521184204800
ENGINE_TRANSACTION_ID: 23000
            THREAD_ID: 63
             EVENT_ID: 177
        OBJECT_SCHEMA: employees
          OBJECT_NAME: employees
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140521184204800
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 5022884824:212:9:2:140521188400672
ENGINE_TRANSACTION_ID: 23000
            THREAD_ID: 63
             EVENT_ID: 188        OBJECT_SCHEMA: employees
          OBJECT_NAME: employees
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140521188400672
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10001
2 rows in set (0.00 sec)
```

위의 결과를 보면 23000번 트랜잭션은 다음과 같이 2개의 잠금을 가지고 있다는 것을 알 수 있다.

- employees 테이블에 대한 IX 잠금(Intention Exclusive Lock)을 가지고 있음
- employees 테이블의 프라이머리 키의 RECORD에 대해서 배타적 잠금을 가지고 있음(LOCK_MODE 칼럼은 잠금이 갭락은 아닌 단순 레코드 락임을 의미함)

performance_schema.data_locks 테이블과 performance_schema.data_lock_waits 테이블, information_schema.innodb_trx 테이블을 이용해 트랜잭션 간 잠금 대기를 확인할 수 있다.

이처럼 장시간에 걸쳐 트랜잭션이 쿼리를 실행 중인 상태에서 그 쿼리만 강제 종료시키면 커넥션이나 트랜잭션은 여전히 활성 상태로 남아있게 된다.
응용 프로그램에서 쿼리의 에러를 감지해서 트랜잭션을 롤백하게 돼 있다면 다음과 같이 쿼리만 종료하면 된다.

```
mysql> KILL QUERY 14;
```

그런데 응용 프로그램에서 쿼리 에러에 대한 핸들링이 확실하지 않다면 쿼리를 종료시키는 것보다 커넥션 자체를 강제 종료시키는 방법이 더 안정적일 수 있다.

```
mysql> KILL 14;
```