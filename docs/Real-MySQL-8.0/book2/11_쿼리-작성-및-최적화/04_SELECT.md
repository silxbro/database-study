# [CH 11-4] SELECT

웹 서비스 같이 일반적인 온라인 트랜잭션 처리 환경의 데이터베이스에서는 INSERT나 UPDATE 같은 작업은 거의 레코드 단위로 발생하므로 성능상 문제가 되는 경우는 별로 없다.
하지만 SELECT는 여러 개의 테이블로부터 데이터를 조합해서 빠르게 가져와야 하기 때문에 여러 개의 테이블을 어떻게 읽을 것인가에 많은 주의를 기울여야 한다.
하나의 애플리케이션에서 사용되는 쿼리 중에서도 SELECT 쿼리의 비율은 높다. 이번 절에서는 SELECT 쿼리의 각 부분에 사용될 수 있는 기능을 성능 위주로 살펴보겠다.

---
<br/>

## (1) SELECT 절의 처리 순서
SELECT 문장이라고 하면 SQL 전체를 의미한다. 그리고 SELECT 키워드와 실제 가져올 칼럼을 명시한 부분만 언급할 때는 SELECT 절이라고 표현한다.
여기서 절이란 우리가 주로 알고 있는 키워드(SELECT, FROM, JOIN, WHERE, GROUP BY, HAVING, ORDER BY, LIMIT)와 그 뒤에 기술된 표현식을 묶어서 말한다.
다음 예제는 여러 가지 절이 포함된 쿼리다.

```
SELECT s.emp_no, COUNT(DISTINCT e.first_name) AS cnt
FROM salaries s
  INNER JOIN employees e ON e.emp_no=s.emp_no
WHERE s.emp_no IN (100001, 100002)
GROUP BY s.emp_no
HAVING AVG(s.salary) > 1000
ORDER BY AVG(s.salary)
LIMIT 10;
```

이 쿼리 예제를 각 절로 나눠보면 다음과 같다. 하지만 더 상세히 SQL의 위치를 언급할 때는 INNER JOIN 키워드나 그 뒤의 ON까지 구분해서 INNER JOIN 절 또는 ON 절이라고 표현하기도 한다.

- SELECT 절: SELECT s.emp_no, COUNT(DISTINCT e.first_name) AS cnt
- FROM 절: FROM salaries s INNER JOIN employees e ON e.emp_no=s.emp_no
- WHERE 절: WHERE s.emp_no IN (100001, 100002)
- GROUP BY 절: GROUP BY s.emp_no
- HAVING 절: HAVING AVG(s.salary) > 1000
- ORDER BY 절: ORDER BY AVG(s.salary)
- LIMIT 절: LIMIT 10

위의 예제 쿼리는 SELECT 문장에 지정할 수 있는 대부분의 절이 포함돼 있다.
가끔 이런 쿼리에서 어느 절이 먼저 실행될지 예측하지 못할 때가 있는데, 어느 절이 먼저 실행되는지를 모르면 처리 내용이나 처리 결과를 예측할 수 없다.

#### [그림 11.3] 각 쿼리 절의 실행 순서
<img src="https://github.com/user-attachments/assets/c92bf914-ba08-4a33-8ecb-a9a574b0c867" width="550"/><br/>

위 그림에서 각 요소가 없는 경우는 가능하지만, 이 순서가 바뀌어서 실행되는 형태의 쿼리는 거의 없다(아래 그림, 그리고 CTE와 윈도우 함수 제외).
또한 SQL에는 ORDER BY나 GROUP BY 절이 있더라도 인덱스를 이용해 처리할 때는 그 단계 자체가 불필요하므로 생략된다.

#### [그림 11.4] 쿼리 각 절의 실행 순서(예외적으로 ORDER BY가 조인보다 먼저 실행되는 경우)
<img src="https://github.com/user-attachments/assets/f74cebbd-3849-4aaa-bd29-d0fa2bbc31f9" width="570"/><br/>

위 그림은 ORDER BY가 사용된 쿼리에서 예외적인 순서로 실행되는 경우를 보여준다.
이 경우는 첫 번째 테이블만 읽어서 정렬을 수행한 뒤에 나머지 테이블을 읽는데, 주로 GROUP BY 절이 없이 ORDER BY만 사용된 쿼리에서 사용될 수 있는 순서다.

위에서 소개한 실행 순서를 벗어나는 쿼리가 필요하다면 서브쿼리로 작성된 인라인 뷰(Inline View)를 사용해야 한다.
예를 들어, 위의 쿼리에서 LIMIT을 먼저 적용하고 ORDER BY를 실행하고자 한다면 다음과 같이 인라인 뷰를 사용해야 한다.
다음 쿼리는 ORDER BY와 LIMIT의 순서가 바뀌었기 때문에 위의 쿼리와 동일한 결과를 반환하지 않을 수도 있다.

```
SELECT emp_no, cnt
FROM (
    SELECT s.emp_no, COUNT(DISTINCT e.first_name) AS cnt, MAX(s.salary) AS max_salary
    FROM salaries s
      INNER JOIN employees e ON e.emp_no=s.emp_no
    WHERE s.emp_no IN (100001, 100002)
    GROUP BY s.emp_no
    HAVING MAX(s.salary) > 1000
    LIMIT 10
) temp_view
ORDER BY max_salary;
```

LIMIT을 GROUP BY 전에 실행하고자 할 때도 마찬가지로 서브쿼리로 인라인 뷰를 만들어서 그 뷰 안에서 LIMIT을 적용하고 바깥 쿼리(아우터 쿼리)에서 GROUP BY와 ORDER BY를 적용해야 한다.
하지만 이렇게 인라인 뷰가 사용되면 임시 테이블이 사용되기 때문에 주의해야 한다.
MySQL의 LIMIT은 오라클의 ROWNUM과 조금 성격이 달라서 WHERE 조건으로 사용하지 않고 항상 모든 처리의 결과에 대해 레코드 건수를 제한하는 형태로 사용한다.

> MySQL 8.0 버전에서는 FROM 절에 위치한 서브쿼리(Derived Table)를 외부 쿼리와 병합하면서 쿼리를 최적화할 수도 있다.
> 이 경우도 결국 FROM 절의 서브쿼리가 아니라 조인으로 실행되는 형태가 되기 때문에 [그림 11.3] 또는 [그림 11.4] 중 하나의 순서로 실행되는 것이다.

여기에 표시되지는 않았지만 MySQL 8.0에 새로 도입된 WITH 절(CTE, Common Table Expression)은 항상 제일 먼저 실행되어 임시 테이블로 저장된다.
그리고 WITH 절로 만들어진 임시 테이블은 단독으로 조회되거나 조인되는 테이블로 활용된다. 또한 MySQL 8.0에 새로 추가된 윈도우 함수에서도 쿼리의 각 절이 실행되는 순서가 중요하다.
<br/>
<br/>
## (2) WHERE 절과 GROUP BY 절, ORDER BY 절의 인덱스 사용
WHERE 절의 조건뿐만 아니라 GROUP BY나 ORDER BY 절도 인덱스를 이용해 빠르게 처리할 수 있다는 점은 이미 언급했다.
이번 절에서는 각 절에서 어떤 요건을 갖췄을 때 인덱스를 이용할 수 있는지 좀 더 자세히 살펴보겠다.

### [1] 인덱스를 사용하기 위한 기본 규칙
WHERE 절이나 ORDER BY 또는 GROUP BY가 인덱스를 사용하려면 기본적으로 인덱스된 칼럼의 값 자체를 변환하지 않고 그대로 사용한다는 조건을 만족해야 한다.
인덱스는 칼럼의 값을 아무런 변환 없이 B-Tree에 정렬해서 저장한다. WHERE 조건이나 GROUP BY 또는 ORDER BY에서도 원본값을 검색하거나 정렬할 때만 B-Tree에 정렬된 인덱스를 이용한다.
즉, 인덱스는 salary 칼럼으로 만들어져 있는데, 다음 예제의 WHERE 절과 같이 salary 칼럼을 가공한 후 다른 상숫값과 비교한다면 이 쿼리는 인덱스를 적절히 이용하지 못하게 된다.

```
mysql> SELECT * FROM salaries WHERE salary*10 > 150000;
```

사실 이 쿼리는 간단히 다음과 같이 변경해서 salary 칼럼의 값을 변경하지 않고 검색하도록 유도할 수 있지만 MySQL 옵티마이저에서는 인덱스를 최적으로 이용할 수 있게 표현식을 변환하지는 못한다.

```
mysql> SELECT * FROM salaries WHERE salary > 150000/10;
```

이러한 형태는 아주 단순한 예제이며, 복잡한 연산을 수행한다거나 MD5() 함수와 같이 해시 값을 만들어서 비교해야 하는 경우라면 미리 계산된 값을 저장하도록 MySQL의 가상 칼럼(Virtual Column)을
추가하고 그 칼럼에 인덱스를 생성하거나 함수 기반의 인덱스를 사용하면 된다.
결론적으로 인덱스의 칼럼을 변형해서 비교하는 경우(그 변형이 아무리 간단한 연산이라고 하더라도)에는 인덱스를 이용할 수 없게 된다는 점에 주의하자.

추가로 WHERE 절에 사용되는 비교 조건에서 연산자 양쪽의 두 비교 대상 값은 데이터 타입이 일치해야 한다. 사실 이 내용은 위에서 언급한 값 자체를 변환하지 않는다는 것과 같은 범주에 속하는 이야기다.
다음과 같은 간단한 쿼리로 이 내용을 한번 살펴보자.

```
mysql> CREATE TABLE tb_test (age VARCHAR(10), INDEX ix_age (age));
mysql> INSERT INTO tb_test VALUES ('1'), ('2'), ('3'), ('4'), ('5'), ('6'), ('7');

mysql> SELECT * FROM tb_test WHERE age=2;
```

위 예제와 같이 age라는 VARCHAR 타입의 칼럼이 있는 테이블을 생성하고, 테스트용 레코드를 INSERT하자. 그리고 SELECT 쿼리의 실행 계획을 한번 확인해 보자.
이 쿼리는 분명히 age라는 칼럼에 인덱스가 준비돼 있어서 실행 계획의 type 칼럼에 "ref"나 "range"가 표시됐어야 할 것으로 기대하지만 사실은 "index"라고 표시된다.
"index"는 인덱스 풀 스캔을 의미한다.

인덱스 레인지 스캔을 사용하지 못하고 인덱스를 풀 스캔한 이유는 age 칼럼의 데이터 타입(VARCHAR 타입)과 비교되는 값 2(INTEGER 타입)의 데이터 타입이 다르기 때문이다.
이렇게 비교되는 두 값의 타입이 문자열 타입(VARCHAR나 CHAR)과 숫자 타입(INTEGER)으로 다를 때 MySQL 옵티마이저가 내부적으로 문자열 타입을 숫자 타입으로 변환한 후 비교 작업을 처리한다.
결국 문자열 타입인 age 칼럼이 숫자 타입으로 변환된 후 비교돼야 하므로 인덱스 레인지 스캔이 불가능한 것이다. 이 쿼리를 다음과 같이 변경하면 인덱스 레인지 스캔을 사용하도록 유도할 수 있다.

```
mysql> SELECT * FROM tb_test WHERE age='2';
```

저장하고자 하는 값의 타입에 맞춰 칼럼의 타입을 선정하고, SQL을 작성할 때는 데이터의 타입에 맞춰서 비교 조건을 사용하길 권장한다.
데이터 타입이 조금이라도 다른 경우 최적화되지 못하는 현상은 MySQL 서버의 버전이 업그레이되된다고 해서 해결될 수 있는 부분이 아니므로 항상 주의하자.

### [2] WHERE 절의 인덱스 사용
WHERE 절의 조건이 인덱스를 사용하는 방법은 크게 작업 범위 결정 조건과 체크 조건의 두 가지 방식으로 구분할 수 있다.
두 방식 중 작업 범위 결정 조건은, WHERE 절에서 동등 비교 조건이나 IN으로 구성된 조건에 사용된 칼럼들이 인덱스의 칼럼 구성과 좌측에서부터 비교했을 때 얼마나 일치하는가에 따라 달라진다.

#### [그림 11.5] WHERE 조건의 인덱스 사용 규칙
<img src="https://github.com/user-attachments/assets/c9fa6ebb-915c-4a28-b0aa-2aeacd9cd517" width="450"/><br/>

위 그림에서 위쪽은 4개의 칼럼이 순서대로 결합 인덱스로 생성돼 있는 것을 의미하며, 아래쪽은 SQL의 WHERE 절에 존재하는 조건을 의미한다.
"WHERE 조건절의 순서"에 나열된 조건들의 순서는 실제 인덱스의 사용 여부와 무관하다.
즉, WHERE 조건절에 나열된 순서가 인덱스와 다르더라도 MySQL 서버 옵티마이저는 인덱스를 사용할 수 있는 조건들을 뽑아서 최적화를 수행할 수 있다.
COL_1과 COL_2는 동등 비교 조건이며 COL_3의 조건이 동등 비교 조건이 아닌 크다 또는 작다와 같은 범위 비교 조건이므로 뒤 칼럼인 COL_4의 조건은 작업 범위 결정 조건으로 사용되지 못하고 체크
조건(점선 표기)으로 사용된다.
이는 WHERE 조건절과 인덱스의 칼럼 순서가 일치하지 않기 때문이 아니라 인덱스 순서상 COL_4의 직전 칼럼인 COL_3가 동등 비교 조건이 아니라 범위 비교 조건으로 사용됐기 때문이다.

MySQL 8.0 이전 버전까지는 하나의 인덱스를 구성하는 각 칼럼의 정렬 순서가 혼합되어 사용할 수 없었다.
하지만 MySQL 8.0 버전부터는 다음 예제와 같이 인덱스를 구성하는 칼럼별로 정순(오름차순)과 역순(내림차순) 정렬을 혼합해서 생성할 수 있게 개선됐다.

```
ALTER TABLE ... ADD INDEX ix_col1234 (col_1 ASC, col_2 DESC, col_3 ASC, col_4 ASC);
```

하지만 이해도를 높이기 위해 여기서는 모든 칼럼이 정순으로만 정렬된 인덱스를 가정하자.

> WHERE 절의 조건이 인덱스에 명시된 칼럼의 순서대로 나열됐어야 하는 것이 아닌지 궁금해하는 사용자가 많았다.
> GROUP BY나 ORDER BY와는 달리 WHERE 절의 조건절은 순서를 변경해도 결과의 차이가 없기 때문에 WHERE 절에서의 각 조건이 명시된 순서는 중요치 않고 인덱스를 구성하는 칼럼에 대한 조건이 있는지
> 없는지가 중요하다.
> 위 그림에서 인덱스에는 두꺼운 화살표로 방향 표시를 해 둔 것은 칼럼의 순서가 중요함을 의미하는 것이며, WHERE 절에서는 나열되는 칼럼의 순서가 중요하지 않기 때문에 화살표가 없는 것이다.

지금까지 보여준 모든 WHERE 조건은 AND 연산자로 연결되는 경우를 가정한 것이며, 다음과 같이 OR 연산자가 있으면 처리 방법이 완전히 바뀐다.

```
mysql> SELECT *
       FROM employees
       WHERE first_name='Kebin' OR last_name='Poly';
```

위의 쿼리에서 first_name='Kebin' 조건은 인덱스를 이용할 수 있지만 last_name='Poly'는 인덱스를 사용할 수 없다.
이 두 조건이 AND 연산자로 연결됐다면 first_name의 인덱스를 이용하겠지만 OR 연산자로 연결됐기 때문에 옵티마이저는 풀 테이블 스캔을 선택할 수 밖에 없다.
(풀 테이블 스캔) + (인덱스 레인지 스캔)의 작업량보다는 (풀 테이블 스캔) 한 번이 더 빠르기 때문이다.
이 경우 first_name과 last_name 칼럼에 각각 인덱스가 있다면 index_merge 접근 방법으로 실행할 수 있다.
물론 이 방법은 풀 테이블 스캔보다는 빠르지만 여전히 제대로 된 인덱스 하나를 레인지 스캔하는 것보다는 느리다.
WHERE 절에서 각 조건이 AND로 연결되면 읽어와야 할 레코드의 건수를 줄이는 역할을 하지만 각 조건이 OR로 연결되면 읽어서 비교해야 할 레코드가 더 늘어나기 때문에 WHERE 조건에 OR 연산자가 있다면
주의해야 한다.

### [3] GROUP BY 절의 인덱스 사용
SQL에 GROUP BY가 사용되면 인덱스의 사용 여부는 어떻게 결정될까? GROUP BY 절의 각 칼럼은 비교 연산자를 가지지 않으므로 작업 범위 결정 조건이나 체크 조건과 같이 구분해서 생각할 필요는 없다.
GROUP BY 절에 명시된 칼럼의 순서가 인덱스를 구성하는 칼럼의 순서와 같으면 GROUP BY 절은 일단 인덱스를 이용할 수 있다. 조금 풀어서 사용 조건을 정리해보면 다음과 같다.
여기서 설명하는 내용은 여러 개의 칼럼으로 구성된 다중 칼럼 인덱스를 기준으로 한다. 하지만 칼럼이 하나인 단일 칼럼 인덱스도 똑같이 적용된다.

- GROUP BY 절에 명시된 칼럼이 인덱스 칼럼의 순서와 위치가 같아야 한다.
- 인덱스를 구성하는 칼럼 중에서 뒤쪽에 있는 칼럼은 GROUP BY 절에 명시되지 않아도 인덱스를 사용할 수 있지만 인덱스의 앞쪽에 있는 칼럼이 GROUP BY 절에 명시되지 않으면 인덱스를 사용할 수 없다.
- WHERE 조건절과는 달리 GROUP BY 절에 명시된 칼럼이 하나라도 인덱스에 없으면 GROUP BY 절은 전혀 인덱스를 이용하지 못한다.

#### [그림 11.6] GROUP BY 절의 인덱스 사용 규칙
<img src="https://github.com/user-attachments/assets/47daa9e8-6d07-4573-a93f-662a51aee4dd" width="450"/><br/>

위 그림은 GROUP BY 절이 인덱스를 사용하기 위한 조건을 간단하게 보여준다.
위쪽은 (COL_1, COL_2, COL_3, COL_4)로 만들어진 인덱스를 의미하며, 아래쪽은 COL_1부터 COL_3을 순서대로 GROUP BY 절에 명시된 칼럼을 의미한다.
이 그림에서 GROUP BY 절과 인덱스를 구성하는 칼럼의 순서가 중요하므로 굵은 화살표로 방향 표시를 넣어둔 것이다. 다음에 예시된 GROUP BY 절은 모두 위 그림의 인덱스를 이용하지 못하는 경우다.

```
... GROUP BY COL_2, COL_1
... GROUP BY COL_1, COL_3, COL_2
... GROUP BY COL_1, COL_3
... GROUP BY COL_1, COL_2, COL_3, COL_4, COL_5
```

위의 예제가 인덱스를 사용하지 못하는 원인을 살펴보자.

- 첫 번째와 두 번째 예제는 GROUP BY 칼럼이 인덱스를 구성하는 칼럼의 순서와 일치하지 않기 때문에 사용하지 못하는 것이다.
- 세 번째 예제는 GROUP BY 절에 COL_3가 명시됐지만 COL_2가 그 앞에 명시되지 않았기 때문이다.
- 네 번째 예제에서는 GROUP BY 절의 마지막에 있는 COL_5가 인덱스에는 없어서 인덱스를 사용하지 못하는 것이다.

다음 예제는 GROUP BY 절이 인덱스를 사용할 수 있는 패턴이다. 다음 예제는 WHERE 조건 없이 단순히 GROUP BY만 사용된 형태의 쿼리다.

```
... GROUP BY COL_1
... GROUP BY COL_1, COL_2
... GROUP BY COL_1, COL_2, COL_3
... GROUP BY COL_1, COL_2, COL_3, COL_4
```

WHERE 조건절에 COL_1이나 COL_2가 동등 비교 조건으로 사용된다면 GROUP BY 절에 COL_1이나 COL_2가 빠져도 인덱스를 이용한 GROUP BY가 가능할 때도 있다.
다음 예제는 인덱스의 앞쪽에 있는 칼럼을 WHERE 절에서 상수로 비교하기 때문에 GROUP BY 절에 해당 칼럼이 명시되지 않아도 인덱스를 이용한 그루핑이 가능한 예제다.

```
... WHERE COL_1='상수' ... GROUP BY COL_2, COL_3
... WHERE COL_1='상수' AND COL_2='상수' ... GROUP BY COL_3, COL_4
... WHERE COL_1='상수' AND COL_2='상수' AND COL_3='상수' ... GROUP BY COL_4
```

위 예제와 같이 WHERE 절과 GROUP BY 절이 혼용된 쿼리가 인덱스를 이용해 WHERE 절과 GROUP BY 절이 모두 처리될 수 있는지는 다음 예제와 같이 WHERE 조건절에서 동등 비교 조건으로 사용된 칼럼을
GROUP BY 절로 옮겨보면 된다.

```
-- // 원본 쿼리
... WHERE COL_1='상수' ... GROUP BY COL_2, COL_3

-- // WHERE 조건절의 COL_1 칼럼을 GROUP BY 절의 앞쪽으로 포함시켜 본 쿼리
... WHERE COL_1='상수' ... GROUP BY COL_1, COL_2, COL_3
```

위의 예제에서 COL_1은 상숫값과 비교되므로 "GROUP BY COL_2, COL_3"는 "GROUP BY COL_1, COL_2, COL_3"과 똑같은 결과를 만들어 낸다.
이처럼 GROUP BY 절을 고쳐도 똑같은 결과가 조회된다면 WHERE 절과 GROUP BY 절이 모두 인덱스를 사용할 수 있는 쿼리로 판단하면 된다.

### [4] ORDER BY 절의 인덱스 사용
MySQL에서 GROUP BY와 ORDER BY는 처리 방법이 상당히 비슷하다. 그래서 ORDER BY 절의 인덱스 사용 여부는 GROUP BY의 요건과 거의 흡사하다.
하지만 ORDER BY는 조건이 하나 더 있는데, 정렬되는 각 칼럼의 오름차순(ASC) 및 내림차순(DESC) 옵션이 인덱스와 같거나 정반대인 경우에만 사용할 수 있다는 것이다.
여기서 MySQL의 인덱스는 모든 칼럼이 오름차순으로만 정렬돼 있기 때문에 ORDER BY 절의 모든 칼럼이 오름차순이거나 내림차순일 때만 인덱스를 사용할 수 있다.
인덱스의 모든 칼럼이 ORDER BY 절에 사용돼야 하는 것은 아니지만, ORDER BY 절의 칼럼들이 인덱스에 정의된 칼럼의 왼쪽부터 일치해야 하는 것에는 변함이 없다.
아래 그림은 ORDER BY 절이 인덱스를 이용하기 위한 요건을 보여준다.

#### [그림 11.7] ORDER BY 절의 인덱스 사용 규칙
<img src="https://github.com/user-attachments/assets/8a348296-37bf-4f25-8e1e-2af89fe3072e" width="380"/><br/>

위 그림과 같은 인덱스에서 다음 예제의 ORDER BY 절은 인덱스를 이용할 수 없다. 참고로 ORDER BY 절에 ASC나 DESC와 같이 정렬 순서가 생략되면 오름차순(ASC)으로 해석한다.

```
... ORDER BY COL_2, COL_3
... ORDER BY COL_1, COL_3, COL_3
... ORDER BY COL_1, COL_2 DESC, COL_3
... ORDER BY COL_1, COL_3
... ORDER BY COL_1, COL_2, COL_3, COL_4, COL_5
```

위의 각 예제가 인덱스를 사용하지 못하는 원인을 살펴보자.

- 첫 번째 예제는 인덱스의 제일 앞쪽 칼럼인 COL_1이 ORDER BY 절에 명시되지 않았기 때문에 인덱스를 사용할 수 없다.
- 두 번째 예제는 인덱스와 ORDER BY 절의 칼럼 순서가 일치하지 않기 때문에 인덱스를 사용할 수 없다.
- 세 번째 예제는 ORDER BY 절의 다른 칼럼은 모두 오름차순인데, 두 번째 칼럼인 COL_2의 정렬 순서가 내림차순이라서 인덱스를 사용할 수 없다.
  인덱스가 "(COL_1 ASC, COL_2 DESC, COL_3 ASC, COL_4 ASC)"와 같이 정의됐다면 이 정렬은 인덱스를 사용할 수 있게 된다.
- 네 번째 예제는 인덱스에는 COL_1과 COL_3 사이에 COL_2 칼럼이 있지만 ORDER BY 절에는 COL_2 칼럼이 명시되지 않았기 때문에 인덱스를 사용할 수 없다.
- 다섯 번째 예제는 인덱스에 존재하지 않는 COL_5가 ORDER BY 절에 명시됐기 때문에 인덱스를 사용하지 못한다.

### [5] WHERE 조건과 ORDER BY(또는 GROUP BY) 절의 인덱스 사용
일반적으로 우리가 사용하는 쿼리는 WHERE 절을 가지고 있으며, 선택적으로 ORDER BY나 GROUP BY 절을 포함할 것이다.
쿼리에 WHERE 절만 또는 GROUP BY나 ORDER BY 절만 포함돼 있다면 사용된 절 하나에만 초점을 맞춰서 인덱스를 사용할 수 있게 튜닝하면 된다.
하지만 애플리케이션에서 사용되는 쿼리는 그렇게 단순하지 않다.
SQL 문장이 WHERE 절과 ORDER BY 절을 가지고 있다고 가정했을 때 WHERE 조건은 A 인덱스를 사용하고 ORDER BY는 B 인덱스를 사용하도록 쿼리가 실행될 수는 없다.
이는 WHERE 절과 GROUP BY 절이 같이 사용된 경우와 GROUP BY와 ORDER BY가 같이 사용된 쿼리에서도 마찬가지다.

WHERE 절과 ORDER BY 절이 같이 사용된 하나의 쿼리 문장은 다음 3가지 중 한 가지 방법으로만 인덱스를 이용한다.

- WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 이용:
  WHERE 절의 비교 조건에서 사용하는 칼럼과 ORDER BY 절의 정렬 대상 칼럼이 모두 하나의 인덱스에 연속해서 포함돼 있을 때 이 방식으로 인덱스를 사용할 수 있다.
  이 방법은 아래 2가지 방식보다 훨씬 빠른 성능을 보이기 때문에 가능하다면 이 방식으로 처리할 수 있게 쿼리를 튜닝하거나 인덱스를 생성하는 것이 좋다.

- WHERE 절만 인덱스를 이용: ORDER BY 절은 인덱스를 이용한 정렬이 불가능하며, 인덱스를 통해 검색된 결과 레코드를 별도의 정렬 처리 과정(Using Filesort)을 거쳐 정렬을 수행한다.
  주로 이 방법은 WHERE 절의 조건에 일치하는 레코드의 건수가 많지 않을 때 효율적인 방식이다.

- ORDER BY 절만 인덱스를 이용: ORDER BY 절은 인덱스를 이용해 처리하지만 WHERE 절은 인덱스를 이용하지 못한다.
  이 방식은 ORDER BY 절의 순서대로 인덱스를 읽으면서 레코드 한 건씩 WHERE 절의 조건에 일치하는지 비교하고, 일치하지 않을 때는 버리는 형태로 처리한다.
  주로 아주 많은 레코드를 조회해서 정렬해야 할 때는 이런 형태로 튜닝하기도 한다.

또한 WHERE 절에서 동등 비교 조건으로 비교된 결과 ORDER BY 절에 명시된 칼럼이 순서대로 빠짐없이 인덱스 칼럼의 왼쪽부터 일치해야 한다.
WHERE 절에 동등 비교 조건으로 사용된 칼럼과 ORDER BY 절의 칼럼이 중첩되는 부분은 인덱스를 사용할 때 문제가 되지 않는다.
하지만 중간에 빠지는 칼럼이 있으면 WHERE 절이나 ORDER BY 절 모두 인덱스를 사용할 수 없다. 이때는 주로 WHERE 절만 인덱스를 이용할 수 있다.
(MySQL 8.0 버전에 새롭게 추가된 인덱스 스킵 스캔 최적화는 인덱스에 나열된 칼럼의 순서상 선행되는 칼럼의 조건이 없다고 하더라돠 인덱스의 후행 칼럼을 이용할 수 있게 해준다.
여기서 설명의 이해도를 높이기 위해 인덱스 스킵 스캔과 같은 예외적인 형태의 최적화는 배제한다.)

#### [그림 11.8] WHERE 절과 ORDER BY 절의 인덱스 사용 규칙
<img src="https://github.com/user-attachments/assets/e001616e-15e6-4aad-a061-7cabd273876b" width="470"/><br/>

위 그림은 WHERE 절과 ORDER BY 절이 결합된 두 가지 패턴의 쿼리를 표현한 것이다.
오른쪽과 같이 ORDER BY 절에 해당 칼럼이 사용되고 있다면 WHERE 절에 동등 비교 이외의 연산자로 비교돼도 WHERE 조건과 ORDER BY 조건이 모두 인덱스를 이용할 수 있다.
일반적으로 WHERE 절에서 동등 비교로 사용된 칼럼과 ORDER BY 절의 칼럼이 인덱스를 구성하는 칼럼과 같은 순서로 연속해서 사용됐는지를 확인해야 한다. 왼쪽 패턴의 쿼리 예제를 한번 살펴보자.

```
mysql> SELECT *
       FROM tb_test
       WHERE COL_1=10
       ORDER BY COL_2, COL_3;
```

이 예제 쿼리는 얼핏 보면 ORDER BY 절의 칼럼 순서가 인덱스의 칼럼 순서와 달라서 정렬할 때 인덱스를 이용하지 못할 것처럼 보인다.
이럴 때는 ORDER BY 절에 인덱스의 첫 번째 칼럼인 COL_1 칼럼을 포함해서 다시 한번 쿼리를 작성하고 결과를 살펴본다.

```
mysql> SELECT *
       FROM tb_test
       WHERE COL_1=10
       ORDER BY COL_1, COL_2, COL_3;
```

이 쿼리에서 WHERE 조건이 상수로 동등 비교를 하고 있기 때문에 ORDER BY 절에 COL_1 칼럼을 추가해도 정렬 순서에 변화가 없다. 즉, 이 쿼리는 변경되기 이전의 쿼리와 같다.
하지만 이렇게 변경된 쿼리에서는 WHERE 절과 ORDER BY 절이 동시에 인덱스를 이용할 수 있는지를 더 쉽게 판단할 수 있다.

GROUP BY나 ORDER BY가 인덱스를 사용할 수 있을지 없을지 모호할 때는 이처럼 조금 변경한 쿼리와 원본 쿼리가 같은 순서나 결과를 보장하는지 확인해 보면 된다.
여기서 쿼리를 잠깐 변경한 것은 이해를 돕기 위한 것일 뿐, 실제로 변경하기 이전과 이후의 쿼리 모두 MySQL 옵티마이저는 인덱스를 적절히 사용할 수 있게 실행 계획을 수립한다.

지금까지는 쉽게 설명하기 위해 동등 조건만 예를 들었는데, WHERE 조건절에서 범위 조건의 비교가 사용되는 쿼리를 한번 살펴보자.

```
mysql> SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_1, COL_2, COL_3;
mysql> SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_2, COL_3;
```

위의 첫 번째 예제 쿼리에서 COL_1>10 조건을 만족하는 COL_1 값은 여러 개일 수 있다.
하지만 ORDER BY 절에 COL_1부터 COL_3까지 순서대로 모두 명시됐기 때문에 인덱스를 사용해 WHERE 조건절과 ORDER BY 절을 처리할 수 있다.
하지만 두 번째 쿼리에서는 WHERE 절에서 COL_1이 동등 조건이 아니라 범위 조건으로 검색됐는데, ORDER BY 절에는 COL_1이 명시되지 않았기 때문에 정렬할 때는 인덱스를 이용할 수 없게 된다.

다음과 같이 WHERE 절과 ORDER BY 절에 명시된 칼럼의 순서가 일치하지 않거나 중간에 빠지는 칼럼이 있으면 인덱스를 이용해 WHERE 절과 ORDER BY 절을 모두 처리하기란 불가능하다.

```
... WHERE COL_1=10 ORDER BY COL_3, COL_4
... WHERE COL_1>10 ORDER BY COL_2, COL_3
... WHERE COL_1 IN (1,2,3,4) ORDER BY COL_2
```

지금까지는 WHERE 절과 ORDER BY 절 위주였지만 WHERE 절과 GROUP BY 절의 조합도 모두 똑같은 기준이 적용된다.
WHERE 절과 ORDER BY나 GROUP BY 절의 조합에서 인덱스의 사용 여부를 판단하는 능력은 상당히 중요하므로 여러 가지 경우에 대해 직접 테스트해보는 것이 좋다.

### [6] GROUP BY 절과 ORDER BY 절의 인덱스 사용
GROUP BY와 ORDER BY 절이 동시에 사용된 쿼리에서 두 절이 모두 하나의 인덱스를 사용해서 처리되려면 GROUP BY 절에 명시된 칼럼과 ORDER BY에 명시된 칼럼의 순서와 내용이 모두 같아야 한다.
GROUP BY와 ORDER BY가 같이 사용된 쿼리에서는 둘 중 하나라도 인덱스를 이용할 수 없을 때는 둘 다 인덱스를 사용하지 못한다.
즉 GROUP BY는 인덱스를 이용할 수 있지만 ORDER BY가 인덱스를 이용할 수 없을 때 이 쿼리의 GROUP BY와 ORDER BY 절은 모두 인덱스를 이용하지 못한다. 물론 그 반대의 경우도 마찬가지다.

```
... GROUP BY COL_1, COL_2 ORDER BY COL_2
... GROUP BY COL_1, COL_2 ORDER BY COL_1, COL_3
```

MySQL 5.7 버전까지는 GROUP BY는 GROUP BY 칼럼에 대한 정렬까지 함께 수행하는 것이 기본 작동 방식이었다.
하지만 MySQL 8.0 버전부터는 GROUP BY 절이 칼럼의 정렬까지는 보장하지 않는 형태로 바뀌었다.
그래서 MySQL 8.0 버전부터는 GROUP BY 칼럼으로 그루핑과 정렬을 모두 수행하기 위해서는 GROUP BY 절과 ORDER BY 절을 모두 명시해야 한다.

### [7] WHERE 조건과 ORDER BY 절, GROUP BY 절의 인덱스 사용
WHERE 절과 GROUP BY 절, ORDER BY 절이 모두 포함된 쿼리가 인덱스를 사용하는지 판단하는 방법을 알아보자. 다음과 같은 3개의 질문을 기본으로 해서 아래 그림의 흐름을 적용해 보면 된다.

- [1] WHERE 절이 인덱스를 사용할 수 있는가?
- [2] GROUP BY 절이 인덱스를 사용할 수 있는가?
- [3] GROUP BY 절과 ORDER BY 절이 동시에 인덱스를 사용할 수 있는가?

#### [그림 11.9] WHERE 조건과 ORDER BY 절, GROUP BY 절의 인덱스 사용 여부 판단
<img src="https://github.com/user-attachments/assets/0299a39e-099d-4333-9ec1-5fb447334e0e" width="500"/><br/>
<br/>

## (3) WHERE 절의 비교 조건 사용 시 주의사항
WHERE 절에 사용되는 비교 조건의 표현식은 상당히 중요하다. 쿼리가 최적으로 실행되려면 적합한 인덱스와 함께 WHERE 절에 사용되는 비교 조건의 표현식을 적절하게 사용해야 한다.

### [1] NULL 비교
다른 DBMS와는 조금 다르게 MySQL에서는 NULL 값이 포함된 레코드도 인덱스로 관리된다. 이는 인덱스에서는 NULL을 하나의 값으로 인정해서 관리한다는 것을 의미한다.
SQL 표준에서 NULL의 정의는 비교할 수 없는 값이다. 그래서 두 값이 모두 NULL을 가진다고 하더라도 이 두 값이 동등한지 비교하는 것은 불가능하다.
연산이나 비교에서 한쪽이라도 NULL이면 그 결과도 NULL이 반환되는 이유가 바로 여기에 있다. 쿼리에서 NULL인지 비교하려면 "IS NULL"(또는 "<=>") 연산자를 사용해야 한다.
그 밖의 방법으로는 칼럼의 값이 NULL인지 알 수 있는 방법이 없다.

```
mysql> SELECT NULL=NULL;
+-----------+
|      NULL |
+-----------+

mysql> SELECT NULL<=>NULL;
+-------------+
|           1 |
+-------------+

mysql> SELECT CASE WHEN NULL=NULL THEN 1 ELSE 0 END;
+----------------------------------------+
|                                      0 |
+----------------------------------------+

mysql> SELECT CASE WHEN NULL IS NULL THEN 1 ELSE 0 END;
+-----------------------------------------+
|                                       1 |
+-----------------------------------------+
```

다음 예제 쿼리의 NULL 비교가 인덱스를 사용하는 방법을 한번 살펴보자.

```
mysql> SELECT * FROM titles WHERE to_date IS NULL;
```

위의 쿼리는 to_date 칼럼이 NULL인 레코드를 조회하는 쿼리지만 to_date 칼럼에 생성된 ix_todate 인덱스를 ref 방식으로 적절히 이용하고 있음을 알 수 있다.

```
+----+--------+------+-----------+--------------------------+
| id | table  | type | key       | Extra                    |
+----+--------+------+-----------+--------------------------+
|  1 | titles | ref  | ix_todate | Using where; Using index |
+----+--------+------+-----------+--------------------------+
```

사실 칼럼의 값이 NULL인지 확인할 때는 ISNULL()이라는 함수를 사용해도 된다. 하지만 ISNULL() 함수를 WHERE 조건에서 사용할 때는 주의할 점이 있다. 다음 예제 쿼리를 한번 살펴보자.

```
mysql> SELECT * FROM titles WHERE to_date IS NULL;
mysql> SELECT * FROM titles WHERE ISNULL(to_date);
mysql> SELECT * FROM titles WHERE ISNULL(to_date)=1;
mysql> SELECT * FROM titles WHERE ISNULL(to_date)=true;
```

위에 나열된 4개의 쿼리는 전부 정상적으로 to_date 칼럼이 NULL인지 판별해 낼 수 있는 쿼리다.
예제에서 첫 번째와 두 번째 쿼리는 titles 테이블의 ix_todate 인덱스를 레인지 스캔으로 사용할 수 있다. 하지만 세 번째와 네 번째 쿼리는 인덱스나 테이블을 풀 스캔하는 형태로 처리된다.
NULL 비교를 할 때는 가급적 IS NULL 연산자를 사용하길 권장한다. 예상외로 세 번째나 네 번째 쿼리가 자주 사용되는데, 이러한 비교는 인덱스를 사용하지 못한다는 점을 확실히 알아두기 바란다.

### [2] 문자열이나 숫자 비교
문자열 칼럼이나 숫자 칼럼을 비교할 때는 반드시 그 타입에 맞는 상숫값을 사용할 것을 권장한다.
즉 비교 대상 칼럼이 문자열 칼럼이라면 문자열 리터럴을 사용하고, 숫자 타입이라면 숫자 리터럴을 이용하는 규칙만 지켜주면 된다.

```
mysql> SELECT * FROM employees WHERE emp_no=10001;
mysql> SELECT * FROM employees WHERE first_name='Smith'l
mysql> SELECT * FROM employees WHERE emp_no='10001';
mysql> SELECT * FROM employees WHERE first_name=10001;
```

첫 번째와 두 번째 쿼리는 적절히 타입을 맞춰서 비교를 수행했지만, 세 번째와 네 번째 쿼리는 칼럼의 타입과 비교 상수의 타입이 일치하지 않는 WHERE 조건이 포함돼 있다.
위 예제의 쿼리가 어떻게 실행되는지 쿼리별로 한번 살펴보자.

- 첫 번째와 두 번째 쿼리는 칼럼의 타입과 비교하는 상숫값이 동일한 타입으로 사용됐기 때문에 인덱스를 적절히 이용할 수 있다.
- 세 번째의 쿼리는 emp_no 칼럼이 숫자 타입이기 때문에 문자열 상숫값을 숫자로 타입 변환해서 비교를 수행하므로 특별히 성능 저하는 발생하지 않는다.
- 네 번째 쿼리는 first_name이 문자열 칼럼이지만 비교되는 상숫값이 숫자 타입이므로 옵티마이저는 우선순위를 가지는 숫자 타입으로 비교를 수행하려고 실행 계획을 수립한다.
  그래서 first_name 칼럼의 문자열을 숫자로 변환해서 비교를 수행한다. 하지만 first_name 칼럼의 타입 변환이 필요하기 때문에 ix_firstname 인덱스를 사용하지 못한다.

  ```
  mysql> EXPLAIN
         SELECT * FROM employees WHERE first_name=10001;

  +----+-----------+------+------+--------+-------------+
  | id | table     | type | key  | rows   | Extra       |
  +----+-----------+------+------+--------+-------------+
  |  1 | employees | ALL  | NULL | 299920 | Using where |
  +----+-----------+------+------+--------+-------------+
  ```

옵티마이저가 어떤 경우에 어떻게 타입 변환을 유도하는지 정확히 아는 것도 필요하지만 칼럼의 타입에 맞게 상수 리터럴을 비교 조건에 사용하는 것이 중요하다.

### [3] 날짜 비교
SQL을 처음 접하거나 익숙하지 않은 사용자가 가장 많이 실수하는 부분이 아마 날짜 타입의 비교가 아닐까 싶다.
특히나 MySQL에서는 날짜만 저장하는 DATE 타입과 날짜와 시간을 함께 저장하는 DATETIME과 TIMESTAMP 타입이 있으며, 시간만 저장하는 TIME이라는 타입도 있기 때문에 상당히 복잡하게 느껴질 수 있다.
이번 절에서는 자주 사용되는 DATE와 DATETIME 비교 방식과 함께 TIMESTAMP와 DATETIME 비교에 대해 살펴보겠다.

#### ◼︎ DATE 또는 DATETIME과 문자열 비교
DATE 또는 DATETIME 타입의 값과 문자열을 비교할 때는 문자열 값을 자동으로 DATETIME 타입의 값으로 변환해서 비교를 수행한다.
다음 예제에서 첫 번째 쿼리는 DATE 타입의 hire_date 칼럼과 비교하기 위해 STR_TO_DATE() 함수를 이용해 문자열 "2011-07-23"을 DATE 타입으로 변환했다.
하지만 이렇게 칼럼의 타입이 DATE나 DATETIME 타입이면 별도로 문자열을 DATE나 DATETIME 타입으로 명시적으로 변환하지 않아도 MySQL이 내부적으로 변환을 수행한다.
결과적으로 두 번째 예제도 첫 번째 예제와 동일하게 처리된다. 물론 첫 번째 쿼리와 두 번째 쿼리 모두 인덱스를 효율적으로 이용하기 때문에 성능과 관련된 문제는 고민하지 않아도 된다.
DATETIME 타입도 DATE 타입과 마찬가지로 동작한다.

```
mysql> SELECT COUNT(*)
       FROM employees
       WHERE hire_date>STR_TO_DATE('2011-07-23','%Y-%m-%d');

mysql> SELECT COUNT(*)
       FROM employees
       WHERE hire_date>'2011-07-23';
```

그런데 가끔 위의 쿼리를 다음과 같이 작성하는 경우가 있다. 이미 눈치챘겠지만 다음 쿼리는 hire_date 타입을 강제적으로 문자열로 변경하기 때문에 인덱스를 효율적으로 이용하지 못한다.
가능하면 DATE나 DATETIME 타입의 칼럼을 변경하지 말고 상수를 변경하는 형태로 조건을 사용하는 것이 좋다.

```
mysql> SELECT COUNT(*)
       FROM employees
       WHERE DATE_FORMAT(hire_date,'%Y-%m-%d') > '2011-07-23';
```

위의 예제와 같이 날짜 타입의 포맷을 변환하는 형태를 포함해서 날짜 타입 칼럼의 값을 더하거나 빼는 함수로 변형한 후 비교해도 마찬가지로 인덱스를 이용할 수 없다.

```
mysql> SELECT COUNT(*)
       FROM employees
       WHERE DATE_ADD(hire_date, INTERVAL 1 YEAR) > '2011-07-23';

mysql> SELECT COUNT(*)
       FROM employees
       WHERE hire_date > DATE_SUM('2011-07-23', INTERVAL 1 YEAR);
```

위의 첫 번째 쿼리는 hire_date 칼럼을 DATE_ADD() 함수로 변형하기 때문에 인덱스를 사용할 수 없다. 따라서 두 번째 쿼리와 같이 칼럼이 아니라 상수를 변형하는 형태로 쿼리를 작성해야 한다.

#### ◼︎ DATE와 DATETIME의 비교
DATETIME 값에서 시간 부분만 떼어 버리고 비교하려면 다음 예제와 같이 쿼리를 작성하면 된다. DATE() 함수는 DATETIME 타입의 값에서 시간 부분은 버리고 날짜 부분만 반환하는 함수다.

```
mysql> SELECT COUNT(*)
       FROM employees
       WHERE hire_date>DATE(NOW());
```

DATETIME 타입의 값을 DATE 타입으로 만들지 않고 그냥 비교하면 MySQL 서버가 DATE 타입의 값을 DATETIME으로 변환해서 같은 타입을 만든 다음 비교를 수행한다.
즉, 다음 예제에서 DATE 타입의 값 "2011-06-30"과 DATETIME 타입의 값 "2011-06-30 00:00:01"을 비교하는 과정에서는 "2011-06-30"을 "2011-06-30 00:00:00"으로 변환해서 비교를
수행한다.

```
mysql> SELECT
STR_TO_DATE('2011-06-30','%Y-%m-%d') < STR_TO_DATE('2011-06-30 00:00:01','%Y-%m-%d %H:%i:%s');
+-------+
|     1 |
+-------+

mysql> SELECT
STR_TO_DATE(''2011-06-30','%Y-%m-%d') >= STR_TO_DATE('2011-06-30 00:00:01','%Y-%m-%d %H:%i:%s');
+-------+
|     0 |
+-------+
```

DATETIME과 DATE 타입의 비교에서 타입 변환은 인덱스의 사용 여부에 영향을 미치지 않기 때문에 성능보다는 쿼리의 결과에 주의해서 사용하면 된다.

#### ◼︎ DATETIME과 TIMESTAMP의 비교
DATE나 DATETIME 타입의 값과 TIMESTAMP의 값을 별도의 타입 변환 없이 비교하면 문제없이 작동하고 실제 실행 계획도 인덱스 레인지 스캔을 사용해서 동작하는 것처럼 보이지만 사실은 그렇지 않다.

```
mysql> SELECT COUNT(*) FROM employees WHERE hire_date < '2011-07-23 11:10:12';
+----------+
| COUNT(*) |
+----------+
|   300024 |
+----------+

mysql> SELECT COUNT(*) FROM employees WHERE hire_date > UNIX_TIMESTAMP('1986-01-01 00:00:00');
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
1 row in set, 2 warnings (0.04 sec)

mysql> SHOW WARNINGS;
+---------+------+-------------------------------------------------------------------+
| Level   | Code | Message                                                           |
+---------+------+-------------------------------------------------------------------+
| Warning | 1292 | Incorrect date value: '504889200' for column 'hire_date' at row 1 |
| Warning | 1292 | Incorrect date value: '504889200' for column 'hire_date' at row 1 |
+---------+------+-------------------------------------------------------------------+
```

UNIX_TIMESTAMP() 함수의 결괏값은 MySQL 내부적으로는 단순 숫자 값에 불과할 뿐이므로 두 번째 쿼리와 같이 비교해서는 원하는 결과를 얻을 수 없다.
이때는 반드시 비교 값으로 사용되는 상수 리터럴을 비교 대상 칼럼의 타입에 맞게 변환해서 사용하는 것이 좋다.
칼럼이 DATETIME 타입이라면 FROM_UNIXTIME() 함수를 이용해 TIMESTAMP 값을 DATETIME 타입으로 만들어서 비교해야 한다.
그리고 반대로 칼럼의 타입이 TIMESTAMP라면 UNIX_TIMESTAMP() 함수를 이용해 DATETIME을 TIMESTAMP로 변환해서 비교해야 한다. 또는 간단하게 NOW() 함수를 이용해도 된다.

```
mysql> SELECT COUNT(*) FROM employees WHERE hire_date < FROM_UNIXTIME(UNIX_TIMESTAMP());
mysql> SELECT COUNT(*) FROM employees WHERE hire_date < NOW();
```

### [4] Short-Circuit Evaluation
프로그램 개발 경험이 있다면 "Short-circuit Evaluation"이라는 말을 들어본 적이 있을 것이다. 간단히 다음 의사 코드를 이용해 "Short-circuit Evaluation"의 작동 방식을 살펴보자.

```
boolean in_transaction;

if ( in_transaction && has_modified() ) {
  commit();
}
```

위의 예제 코드에서는 in_transaction 불리언 변수의 값이 TRUE이면 has_modified() 함수를 호출하고, 그 결괏값이 TRUE라면 commit() 함수가 실행될 것이다.
그런데 in_transaction 불리언 변수의 값이 FALSE라면 has_modified() 함수의 결괏값에 관계없이 commit() 함수는 호출되지 않는다.
그래서 많은 프로그래밍 언어에서는 빠른 성능을 위해 in_transaction 불리언 변숫값이 FALSE라면 has_modified() 함수를 호출도 하지 않고 다음 코드를 실행한다.
이처럼 여러 개의 표현식이 AND 또는 OR 논리 연산자로 연결된 경우 선행 표현식의 결과에 따라 후행 표현식을 평가할지 말지 결정하는 최적화를 "Short-circuit Evaluation"이라고 한다.

그럼 이제 MySQL 서버에서 "Short-circuit Evaluation"이 어떻게 쿼리의 성능에 영향을 미치는지 한번 살펴보자.
다음 쿼리는 salaries 테이블에서 2개의 조건을 모두 만족하는 레코드를 조회하는 쿼리다. 이해를 돕기 위해서 2개의 조건이 각각 별도로 사용됐을 때 일치하는 레코드의 건수도 함께 확인했다.

```
-- // salaries 테이블의 전체 레코드 건수
mysql> SELECT COUNT(*) FROM salaries;
+----------+
| COUNT(*) |
+----------+
|  2844047 |
+----------+

-- // 1번 조건을 만족하는 레코드 건수
mysql> SELECT COUNT(*) FROM salaries
       WHERE CONVERT_TZ(from_date,'+00:00','+09:00')>'1991-01-01';
+----------+
| COUNT(*) |
+----------+
|  2442943 |
+----------+

-- // 2번 조건을 만족하는 레코드 건수
mysql> SELECT COUNT(*) FROM salaries
       WHERE to_date<'1985-01-01';
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+

-- // 1번과 2번 조건 결합
mysql> SELECT * FROM  salaries
       WHERE CONVERT_TZ(from_date,'+00:00','+09:00')>'1991-01-01' /* 1번 조건 */
         AND to_date<'1985-01-01'                                 /* 2번 조건 */
```

위의 예제 쿼리에서 사용된 두 개의 조거은 모두 인덱스를 사용하지 못하기 때문에 이 쿼리는 풀 테이블 스캔을 하게 된다. 그리고 1번과 2번 조건을 AND로 연결한 3번째 쿼리의 결과는 0건이 될 것이다.
이런 형태의 쿼리에서 WHERE 절에 나열되는 조건의 순서가 쿼리의 성능에 영향을 미칠 것이라는 생각을 해본 사용자는 많지 않을 것이다.
이제 WHERE 절에서 1번 조건과 2번 조건의 순서만 바꿔가면서 쿼리 성능을 한번 확인해보자.

```
mysql> SELECT * FROM salaries
       WHERE CONVERT_TZ(from_date,'+00:00','+09:00')>'1991-01-01' /* 1번 조건 */
         AND to_date<'1985-01-01'                                 /* 2번 조건 */
==> (0.73 sec)

mysql> SELECT * FROM salaries
       WHERE to_date<'1985-01-01'                                 /* 2번 조건 */
         AND CONVERT_TZ(from_date,'+00:00','+09:00')>'1991-01-01' /* 1번 조건 */
==> (0.52 sec)
```

WHERE 절에 1번 조건을 먼저 사용한 쿼리의 응답 시간은 0.73초인 반면, 2번 조건을 먼저 사용한 쿼리의 응답 시간은 0.52초다.
사실 별 차이가 아니라고 생각할 수도 있지만, 1번 조건이 더 많은 CPU를 사용하거나 더 많은 자원을 소모하는 조건이었다면 두 쿼리의 시간 차이는 훨씬 더 커졌을 것이다.

WHERE 절에 1번 조건이 먼저 사용된 쿼리의 경우 MySQL 서버는 salaries 테이블 전체 레코드에 대해 CONVERT_TZ(from_date, ...) 함수를 실행하고 그 결과에 대해 to_date 칼럼의 비교 작업을
한다. 즉, CONVERT_TZ() 함수가 2844047번 실행되고 to_date 칼럼 비교 작업이 2442943번 실행돼야 한다.
하지만 WHERE 절에 2번 조건이 먼저 사용된 쿼리의 경우 salaries 테이블에서 "to_date<'1985-01-01'" 조건을 만족하는 레코드가 한 건도 없기 때문에 to_date 칼럼의 비교 작업만 2844047번
실행하면 되고 CONVERT_TZ() 함수는 한 번도 호출되지 않는다. 이로 인해 두 번째 쿼리의 성능이 30% 정도 빨라진 것이다.

MySQL 서버는 쿼리의 WHERE 절에 나열된 조건을 순서대로 "Short-circuit Evaluation" 방식으로 평가해서 해당 레코드를 반환해야 할지 말지를 결정한다.
그런데 WHERE 절의 조건 중에서 인덱스를 사용할 수 있는 조건이 있다면 "Short-circuit Evaluation"과는 무관하게 MySQL 서버는 그 조건을 가장 최우선으로 사용한다.
그래서 WHERE 조건절에 나열된 조건의 순서가 인덱스의 사용 여부를 결정하지는 않는다. 예를 들어, 다음 예제를 한번 살펴보자.

```
mysql> SELECT * FROM employees
       WHERE last_name='Aamodt'
         AND first_name='Matt';
```

위의 쿼리에서 last_name 칼럼 조건은 인덱스를 사용할 수 없지만 first_name 칼럼 조건은 인덱스를 효율적으로 사용할 수 있다.
이러한 경우 WHERE 절에 나열된 조건의 순서와 무관하게 MySQL 서버는 인덱스를 사용할 수 있는 조건을 먼저 평가한다.
그래야만 MySQL 서버는 employees 테이블의 ix_firstname (first_name) 인덱스를 이용해서 꼭 필요한 레코드만 빠르게 가져올 수 있다.
그러고 나서 WHERE 절에 first_name 조건을 제외한 나머지 조건을 순서대로 평가한다.
WHERE 조건절에 다음과 같이 인덱스를 사용하지 못하는 또 다른 조건이 있었다면 MySQL 서버는 first_name 조건을 평가하고 그다음 last_name 조건, 그리고 마지막으로 birth_date 조건을 평가하는
순서를 사용한다.

```
mysql> SELECT * FROM employees
       WHERE last_name='Aamodt'
         AND first_name='Matt'
         AND MONTH(birth_date)=1
```

이제 조금 더 복잡한 예제를 한번 살펴보자.
참고로 아래 예제 쿼리의 EXISTS (subquery) 부분은 GROUP BY ... HAVING ... 절을 가지고 있기 때문에 MySQL의 세미 조인 최적화를 활용할 수가 없는 형태다.
다음 패턴의 쿼리가 세미 조인 최적화를 활용해 서브쿼리가 조인으로 변경되어 실행된다면 다음 2개의 예제 쿼리는 내부 처리 방식에 차이가 없을 수도 있다.

```
mysql> SELECT *
       FROM employees e
       WHERE e.first_name='Matt'
         AND EXISTS (SELECT 1 FROM salaries s
                     WHERE s.emp_no=e.emp_no AND s.to_date>'1995-01-01'
                     GROUP BY s.salary HAVING COUNT(*)>1)
         AND e.last_name='Aamodt';
```

위의 예제 쿼리에서 first_name 칼럼의 조건은 인덱스를 사용할 수 있으므로 MySQL 서버 옵티마이저는 최우선으로 ix_firstname (first_name) 인덱스를 사용해 필요한 데이터의 범위를 최소화할
것이다. 그리고 그 결과에서 last_name='Aamodt' 조건과 서브쿼리 조건을 만족하는 레코드만 걸러서 결과를 반환한다.
이때 WHERE 절에 서브쿼리 조건이 먼저 나열됐기 때문에 MySQL 서버는 last_name 칼럼의 조건보다 EXISTS (subquery) 조건을 먼저 평가한다.
last_name 조건은 이미 가져온 레코드에서 last_name 칼럼의 값이 'Aamodt'인지 단순 비교만 하면 된다.
하지만 MySQL 서버 옵티마이저는 WHERE 절에 EXISTS (subquery) 조건이 먼저 나열됐기 때문에 salaries 테이블을 검색하는 서브쿼리를 먼저 실행해서 결과를 판단한다.

다음은 EXISTS (subquery) 조건과 last_name='Aamodt' 조건의 순서만 바꿔서 쿼리를 실행해본 결과다.

```
mysql> FLUSH STATUS; /* 현재 커넥션의 상태 값 초기화 */
mysql> SELECT *
       FROM employees e
       WHERE e.first_name='Matt'
         AND e.last_name='Aamodt'
         AND EXISTS (SELECT 1 FROM salaries s
                     WHERE s.emp_no=e.emp_no AND s.to_date>'1995-01-01'
                     GROUP BY s.salary HAVING COUNT(*)>1);

mysql> SHOW STATUS LIKT 'Handler%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Handler_read_key           | 9     |
| Handler_read_next          | 247   |
| Handler_read_rnd_next      | 8     |
| Handler_write              | 7     |
+----------------------------+-------+

mysql> FLUSH STATUS; /* 현재 커넥션의 상태 값 초기화 */
mysql> SELECT *
       FROM employees e
       WHERE e.first_name='Matt'
         AND EXISTS (SELECT 1 FROM salaries s
                     WHERE s.emp_no=e.emp_no AND s.to_date>'1995-01-01'
                     GROUP BY s.salary HAVING COUNT(*)>1)
         AND e.last_name='Aamodt';

mysql> SHOW STATUS LIKE 'Handler%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Handler_read_key           | 1807  |
| Handler_read_next          | 2454  |
| Handler_read_rnd_next      | 1806  |
| Handler_write              | 1573  |
+----------------------------+-------+
```

첫 번째 쿼리에서는 last_name='Aamodt' 조건이 EXISTS (subquery) 조건보다 먼저 나열됐기 때문에 MySQL 서버는 ix_firstname 인덱스를 통해 233건의 레코드를 가져온 다음
last_name='Aamodt' 조건을 만족하는지를 먼저 평가했다. 그리고 first_name='Matt'이면서 last_name='Aamodt'인 레코드 1건에 대해 EXISTS (subquery) 조건의 만족 여부를 평가한 것이다.
결과적으로 MySQL 서버는 첫 번째 쿼리를 위해 260번 정도의 레코드 처리로 쿼리를 완료했다. 이는 Handler_xxx 상태 값들을 통해서도 확인할 수 있다.

하지만 EXISTS (subquery) 조건이 last_name 조건보다 먼저 나열된 경우의 쿼리에서는 ix_firstname 인덱스를 통해 가져온 233건에 대해 복잡한 EXISTS (subquery) 조건을 평가하면서 상당히
많은 레코드를 읽고 쓰는 작업을 한 것을 확인할 수 있다. 단순히 레코드를 많이 읽기만 한 것이 아니라 임시 테이블에 레코드를 쓰기도 상당히 많이 실행했다는 것을 확인할 수 있다.

MySQL 서버에서 쿼리를 작성할 때 가능하면 복잡한 연산 또는 다른 테이블의 레코드를 읽어야 하는 서브쿼리 조건 등은 WHERE 절의 뒤쪽으로 배치하는 것이 성능상 도움이 될 것이다.
물론 WHERE 조건 중에서 인덱스를 사용할 수 있는 조건은 WHERE 절의 어느 위치에 나열되든지 그 순서에 관계없이 가장 먼저 평가되기 때문에 고려하지 않아도 된다.
<br/>
<br/>
## (4) DISTINCT
특정 칼럼의 유니크한 값을 조회하려면 SELECT 쿼리에 DISTINCT를 사용한다. 많은 사용자가 조인을 하는 경우 레코드가 중복해서 출력되는 것을 막기 위해 DISTINCT를 남용하는 경향이 있다.
DISTINCT를 남용하는 것은 성능적인 문제도 있지만 쿼리의 결과도 의도한 바와 달라질 수 있다. 이 부분은 매우 중요하다.

> 여러 테이블을 조인하는 쿼리에서는 조인 조건에 따라서 레코드가 몇 배씩 불어나기도 하는데, 각 테이블 간의 업무적인 연결 조건을 이해하지 못하고 쿼리를 작성하는 경우 주로 이렇게 DISTINCT를 남용하는
> 경우가 발생한다. 테이블 간 조인 쿼리를 작성하는 경우 각 테이블 간의 조인이 1:1 조인인지, 1:M 조인인지 업무적인 특성을 잘 이해하는 것이 중요하다.

## (5) LIMIT n
LIMIT은 쿼리 결과에서 지정된 순서에 위치한 레코드만 가져오고자 할 때 사용한다. 우선 LIMIT이 사용된 예제 쿼리를 한번 살펴보자.

```
mysql> SELECT * FROM employees
       WHERE emp_no BETWEEN 10001 AND 10010
       ORDER BY first_name
       LIMIT 0, 5;
```

위의 쿼리는 다음과 같은 순서로 실행된다.

- [1] employees 테이블에서 WHERE 절의 검색 조건에 일치하는 레코드를 전부 읽어 온다.
- [2] [1]번에서 읽어온 레코드를 first_name 칼럼값에 따라 정렬한다.
- [3] 정렬된 결과에서 상위 5건만 사용자에게 반환한다.

오라클의 ROWNUM에 익숙한 사용자에게는 조금 이상하겠지만 MySQL의 LIMIT은 WHERE 조건이 아니기 때문에 항상 쿼리의 가장 마지막에 실행된다.
LIMIT의 중요한 특성은 LIMIT에서 필요한 레코드 건수만 준비되면 즉시 쿼리를 종료한다는 것이다.
즉, 위의 쿼리에서 모든 레코드의 정렬이 완료되지 않았다고 하더라도 상위 5건까지만 정렬되면 작업을 멈춘다.(하지만 정렬 후 5건만 가져온다고 하더라도 정렬의 특성상 상당히 많은 작업이 완료돼야만 상위
5건을 가려낼 수 있기 때문에 정렬이 필요한 쿼리에서 LIMIT 절이 추가된다고 해서 성능 향상 효과가 크지는 않은 것이 일반적이다.)

다른 GROUP BY 절이나 DISTINCT 등과 같이 LIMIT이 사용됐을 때 어떻게 동작하는지 다음의 쿼리로 조금 더 살펴보자.

```
mysql> SELECT * FROM employees LIMIT 0, 10;
mysql> SELECT * FROM employees GROUP BY first_name LIMIT 0, 10;
mysql> SELECT DISTINCT first_name FROM employees LIMIT 0, 10;

mysql> SELECT * FROM employees
       WHERE emp_no BETWEEN 10001 AND 11000
       ORDER BY first_name
       LIMIT 0, 10;
```

- 첫 번째 쿼리에서 LIMIT이 없을 때는 employees 테이블을 처음부터 끝까지 읽는 풀 테이블 스캔을 실행할 것이다.
  하지만 LIMIT 조건이 있기 때문에 풀 테이블 스캔을 실행하면서 MySQL이 스토리지 엔진으로부터 10개의 레코드를 읽어 들이는 순간 스토리지 엔진으로부터 읽기 작업을 멈춘다.
  이렇게 정렬이나 그루핑 또는 DISTINCT가 없는 쿼리에서 LIMIT 조건을 사용하면 쿼리가 상당히 빨리 끝날 수 있다.

- 두 번째 쿼리는 GROUP BY가 있기 때문에 GROUP BY 처리가 완료되고 나서야 LIMIT 처리를 수행할 수 있다.
  인덱스를 사용하지 못하는 GROUP BY는 그루핑과 정렬의 특성을 모두 가지고 있기 때문에 일단 GROUP BY 작업이 모두 완료돼야만 LIMIT을 수행할 수 있다.
  결국 LIMIT이 GROUP BY와 함께 사용되는 경우에는 LIMIT 절이 있더라도 실질적인 서버의 작업 내용을 크게 줄여주지는 못한다.

- 세 번째 쿼리에서 사용한 DISTINCT는 정렬에 대한 요건 없이 유니크한 그룹만 만들어 내면 된다.
  MySQL은 스토리지 엔진을 통해 풀 테이블 스캔 방식을 이용해 employees 테이블 레코드를 읽어 들임과 동시에 DISTINCT를 위한 중복 제거 작업(임시 테이블을 사용)을 진행한다.
  이 작업을 반복적으로 처리하다가 유니크한 레코드가 LIMIT 건수만큼 채워지면 그 순간 쿼리를 멈춘다.
  예를 들어, employees 테이블의 레코드를 10건 읽었는데, first_name 값이 모두 달랐다면 유니크한 first_name 값 10개를 가져온 것이므로 employees 테이블을 더는 읽지 않고 쿼리를
  완료한다는 것이다. DISTINCT와 함께 사용된 LIMIT은 실질적인 중복 제거 작업의 범위를 줄이는 역할을 한다.
  이 쿼리에서는 10만 건의 레코드를 읽어야 할 작업을 10건만 읽어서 완료할 수 있게 했으므로 LIMIT 절이 작업량을 상당히 줄여준 것이다.

- 네 번째 쿼리는 employees 테이블로부터 WHERE 조건절에 일치하는 레코드를 읽은 후 first_name 칼럼의 값으로 정렬을 수행한다.
  정렬을 수행하면서 필요한 10건이 완성되는 순간, 나머지 작업을 멈추고 결과를 사용자에게 반환한다.
  정렬을 수행하기 전에 WHERE 조건에 일치하는 모든 레코드를 읽어 와야 하지만 읽어온 결과가 전부 정렬돼야 쿼리가 완료되는 것이 아니라 필요한 만큼만 정렬되면 된다.
  하지만 이 쿼리도 두 번째 쿼리와 같이 크게 작업량을 줄여주지는 못한다.

이 예제에서도 알 수 있듯이 쿼리 문장에 GROUP BY나 ORDER BY 같은 전체 범위 작업이 선행되더라도 LIMIT 절이 있다면 크진 않지만 나름의 성능 향상은 있다고 볼 수 있다.
이 예제는 모두 ORDER BY나 DISTINCT, GROUP BY가 인덱스를 적절히 이용하지 못하는 경우를 설명한 것이다.
ORDER BY나 GROUP BY 또는 DISTINCT가 인덱스를 이용해 처리될 수 있다면 LIMIT 절은 꼭 필요한 만큼의 레코드만 읽게 만들어주기 때문에 쿼리의 작업량을 상당히 줄여준다.

LIMIT 절은 1개 또는 2개의 인자를 사용할 수 있는데, 인자가 1개인 경우에는 상위 n개의 레코드를 가져오며, 2개의 인자를 지정하는 경우에는 첫 번째 인자에 지정된 위치부터 두 번째 인자에 명시된
개수만큼의 레코드를 가져온다. LIMIT 절에서 2개의 인자를 사용할 경우 첫 번째 인자(시작 위치, 오프셋)는 0부터 시작한다는 것에 주의하자.
LIMIT 10과 같이 인자가 1개인 경우는 사실 LIMIT 0, 10과 동일하다. 다음 예제의 첫 번째 쿼리는 상위 10개의 레코드만, 두 번째 쿼리는 상위 11번째부터 10개의 레코드를 가져온다.

```
mysql> SELECT * FROM employees LIMIT 10;
mysql> SELECT * FROM employees LIMIT 10, 10;
```

자주 부딪히는 LIMIT 제한 사항으로는 LIMIT의 인자로 표현식이나 별도의 서브쿼리를 사용할 수 없다는 점이 있다. 다음 예제 쿼리를 보면 쉽게 이해할 수 있을 것이다.

```
mysql> SELECT * FROM employees LIMIT (100-10);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to
your MySQL server version for the right syntax to use near '(100-10)' at line 1
```

LIMIT을 사용할 때 한 가지 더 주의해야 할 것이 있다. 실제 쿼리의 성능은 사용자의 화면에 레코드가 몇 건이 출력되느냐보다 MySQL 서버가 그 결과를 만들어 내기 위해 어떠한 작업들을 했는지가 중요하다.
많은 응용 프로그램에서 테이블의 데이터를 SELECT할 때 조금씩 잘라서(페이징) 가져가게 되는데, 이때 다음과 같이 LIMIT을 사용하는 경우가 많다.

```
SELECT * FROM salaries ORDER BY salary LIMIT n, m;
```

일반적으로 처음 몇 개 페이지만 자주 조회되기 때문에 이런 쿼리들은 문제가 되지 않는다.
하지만 특별한 경우 LIMIT의 "n"과 "m"에 주어지는 수치가 매우 커질 수가 있는데, 이런 경우에는 쿼리 실행에 상당히 오랜 시간이 걸린다.

```
mysql> SELECT * FROM salaries ORDER BY salary LIMIT 0,10;
10 rows in set (0.00 sec)

mysql> SELECT * FROM salaries ORDER BY salary LIMIT 2000000,10;
10 rows in set (1.57 sec)
```

"LIMIT 2000000, 10"은 먼저 salaries 테이블을 처음부터 읽으면서 2000010건의 레코드를 읽은 후, 2000000건은 버리고 마지막 10건만 사용자에게 반환한다.
실제 사용자의 화면에는 10건만 표시되지만, MySQL 서버는 2000010건의 레코드를 읽어야 하기 때문에 쿼리가 느려지는 것이다.

LIMIT 조건의 페이징이 처음 몇 개 페이지 조회로 끝나지 않을 가능성이 높다면 다음과 같이 WHERE 조건절로 읽어야 할 위치를 찾고 그 위치에서 10개만 읽는 형태의 쿼리를 사용하는 것이 좋다.

```
-- // 첫 페이지 조회용 쿼리
mysql> SELECT * FROM salaries ORDER BY salary LIMIT 0, 10;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
| 253406 |  38623 | 2002-02-20 | 9999-01-01 |
...
| 274049 |  38864 | 1996-09-01 | 1997-09-01 |
+--------+--------+------------+------------+
10 rows in set (0.01 sec)

-- // 두 번째 페이지 조회용 쿼리(첫 페이지의 마지막 salary 값과 emp_no 값을 이용)
mysql> SELECT * FROM salaries
       WHERE salary>=38864 AND NOT (salary=38864 AND emp_no<=274049)
       ORDER BY salary LIMIT 0, 10;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
| 473390 |  38872 | 1995-03-20 | 1995-09-22 |
...
| 401786 |  38942 | 2001-11-02 | 9999-01-01 |
+--------+--------+------------+------------+
10 rows in set (0.01 sec)

...

-- // 마지막 페이지의 데이터 조회 쿼리
mysql> SELECT * FROM salaries
       WHERE salary>=154888 AND NOT (salary=154888 AND emp_no<=109334)
       ORDER BY salary LIMIT 0, 10;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
| 109334 | 155190 | 2002-02-11 | 9999-01-01 |
...
| 401786 | 158220 | 2002-03-22 | 9999-01-01 |
+--------+--------+------------+------------+
7 rows in set (0.01 sec)
```

위의 예제에서 "NOT (salary=38864 AND emp_no<=274049)" 조건은 이전 페이지에서 이미 조회됐던 건을 제외하기 위해 추가한 조건이다.
salaries 테이블의 salary 칼럼에 인덱스가 있는데, 이 인덱스는 중복이 허용되는 인덱스이기 때문에 단순히 이전 페이지의 마지막 salary 값(이전 페이지에서 가장 큰 salary 값)보다 큰 것을
조회하거나 크거나 같은 경우를 조회하면 중복이나 누락이 발생할 수 있다.
중복이나 누락을 제외하기 위한 방법은 사용하는 인덱스가 몇 개의 칼럼으로 구성돼 있는지, 유니크한지에 따라 달라질 수 있으니 쿼리 작성 시 주의하자.
<br/>
<br/>
## (6) COUNT()
COUNT() 함수는 결과 레코드의 건수를 반환하는 함수다. COUNT() 함수는 칼럼이나 표현식을 인자로 받으며, 특별한 형태로 "*"를 사용할 수도 있다.
여기서 "\*"는 SELECT 절에 사용될 때처럼 모든 칼럼을 가져오라는 의미가 아니라 그냥 레코드 자체를 의미하는 것이다. 실제로 COUNT(\*)라고 해서 레코드의 모든 칼럼을 읽는 형태로 처리하지는 않는다.
그래서 굳이 COUNT(프라이머리 키 칼럼) 또는 COUNT(1)과 같이 사용하지 않고 COUNT(\*)라고 표현해도 동일한 처리 성능을 보인다.

MyISAM 스토리지 엔진을 사용하는 테이블은 항상 테이블의 메타 정보에 전체 레코드 건수를 관리한다.
그래서 "SELECT COUNT(*) FROM tb_table"과 같이 WHERE 조건이 없는 COUNT(\*) 쿼리는 MySQL 서버가 실제 레코드 건수를 세어 보지 않아도 바로 결과를 반환할 수 있기 때문에 빠르게 처리된다.
하지만 WHERE 조건이 있는 COUNT(\*) 쿼리는 그 조건에 일치하는 레코드를 읽어 보지 않는 이상 알 수 없으므로 일반적인 DBMS와 같이 처리된다.
InnoDB 스토리지 엔진을 사용하는 테이블에서는 WHERE 조건이 없는 COUNT(\*) 쿼리라고 하더라도 직접 데이터나 인덱스를 읽어야만 레코드 건수를 가져올 수 있기 때문에 큰 테이블에서 COUNT() 함수를
사용하는 작업은 주의해야 한다.

> 테이블이 가진 대략적인 레코드 건수로 충분하다면 SELECT COUNT(*)보다는 SHOW TABLE STATUS 명령으로 통계 정보를 참조하는 것도 좋은 방법이다.
> 때로는 통계 정보의 레코드 건수는 실제 테이블의 레코드 건수와 많은 차이가 있을 수 있는데, 이런 경우에는 ANALYZE TABLE 명령을 실행해 통계 정보를 갱신하면 된다.
>
> ```
> mysql> SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_ROWS,
>               (DATA_LENGTH+INDEX_LENGTH)/1024/1024/1024 AS TABLE_SIZE_GB
>        FROM information_schema.TABLES
>        WHERE TABLE_SCHEMA='employees' AND TABLE_NAME='employees';
> +--------------+------------+------------+----------------+
> | TABLE_SCHEMA | TABLE_NAME | TABLE_ROWS | TABLE_SIZE_GB  |
> +--------------+------------+------------+----------------+
> | employees    | employees  |     299960 | 0.030334472656 |
> +--------------+------------+------------+----------------+
> ```

COUNT(*) 쿼리에서 가장 많이 하는 실수는 ORDER BY 구문이나 LEFT JOIN과 같은 레코드 건수를 가져오는 것과는 무관한 작업을 포함하는 것이다.
대부분 COUNT(\*) 쿼리는 페이징 처리를 위해 사용할 때가 많은데, 많은 개발자가 SELECT 쿼리를 그대로 복사해서 칼럼이 명시된 부분만 삭제하고 그 부분을 COUNT(\*) 함수로 대체해서 사용하곤 한다.
그래서 단순히 COUNT(\*)만 실행하는 쿼리임에도 ORDER BY가 포함돼 있다거나 별도의 체크 조건을 가지지도 않는 LEFT JOIN이 사용된 채로 실행될 때가 많다.
COUNT(\*) 쿼리에서 ORDER BY 절은 어떤 경우에도 필요치 않다.
그리고 LEFT JOIN 또한 레코드 건수의 변화가 없거나 아우터 테이블에서 별도의 체크를 하지 않아도 되는 경우에는 모두 제거하는 것이 성능상 좋다.

> MySQL 8.0 버전부터는 SELECT COUNT(*) 쿼리에 사용된 ORDER BY 절은 옵티마이저가 무시하도록 개선됐다.
> 하지만 여전히 꼭 필요한 부분만 간결하게 사용해 쿼리를 작성하는 것은 쿼리의 복잡도를 낮추고 가독성을 높인다는 장점이 있다.

많은 사용자가 일반적으로 칼럼의 값을 SELECT하는 쿼리보다 COUNT(*) 쿼리가 훨씬 빠르게 실행될 것으로 생각한다.
하지만 인덱스를 제대로 사용하도록 튜닝되지 못한 COUNT(\*) 쿼리는 페이징해서 데이터를 가져오는 쿼리보다 몇 배 또는 몇십 배 더 느리게 실행될 수도 있다.
COUNT(\*) 쿼리도 많은 부하를 일으키기 때문에 주의 깊게 작성해야 한다.

COUNT() 함수에 칼럼명이나 표현식이 인자로 사용되면 그 칼럼이나 표현식의 결과가 NULL이 아닌 레코드 건수만 반환한다.
예를 들어, "COUNT(column1)"이라고 SELECT 쿼리에 사용하면 column1이 NULL이 아닌 레코드의 건수를 가져온다.
그래서 NULL이 될 수 있는 칼럼을 COUNT() 함수에 사용할 때는 의도대로 쿼리가 작동하는지 확인하는 것이 좋다.

> 게시물 목록을 보여줄 때 일반적으로 다음과 같은 쿼리를 사용한다. 물론 이 형태는 아주 간단한 형태이며, 실제 서비스 요건은 이것보다는 훨씬 복잡한 조건들이 더 추가될 것이다.
>
> ```
> -- // 게시물 건수 확인
> mysql> SELECT COUNT(*) FROM articles WHERE board_id=1
>
> -- // 특정 페이지의 게시물 조회
> mysql> SELECT * FROM articles WHERE board_id=? ORDER BY article_id DESC LIMIT 0, 10
> ```
>
> 많은 사람이 게시물 건수를 확인하는 첫 번째 쿼리가 어느 정도로 MySQL 서버에 부하를 유발하는지 잘 모르고 개발하는 것처럼 보인다.
> board_id가 1인 레코드 건수가 백만 건이라면 첫 번째 쿼리는 articles 테이블에서 100만 건을 읽어야 한다. 물론 인덱스만 읽어서 처리(커버링 인덱스)가 가능하다면 쿼리 성능은 빨라질 것이다.
> 하지만 실제 서비스에서는 보여줄지 말지, 그리고 삭제됐는지 여부 등을 식별해야 하기 때문에 테이블의 데이터를 읽어야만 하는 경우가 대부분이다.
> 그렇다면 건수를 확인하는 첫 번째 쿼리의 부하는 게시물 10건을 가져오는 두 번째 쿼리보다 10만 배 느리게 처리될 것이다.
> 물론 게시물을 가져오는 두 번째 쿼리도 다음 페이지로 넘어가면 갈수록 성능이 더 느려질 가능성이 높다.
> 게시물의 전체 건수를 조회하는 작업은 피하는 것이 좋은데, 게시물의 페이지 번호를 보여주는 방식보다는 "이전"과 "다음" 버튼만 표시하는 방식을 검토해볼 것을 권장한다.
<br/>

## (7) JOIN
OUTER JOIN이나 INNER JOIN 등과 같은 JOIN의 여러 가지 유형에 대해서는 이미 익숙할 것이다. 여기서는 JOIN이 어떻게 인덱스를 사용하는지에 대해 쿼리 패턴별로 자세히 살펴보자.
또한 JOIN의 유형별로 주의할 사항도 함께 살펴보겠다.

### [1] JOIN의 순서와 인덱스
인덱스 레인지 스캔은 인덱스를 탐색(Index Seek)하는 단계와 인덱스를 스캔(Index Scan)하는 과정으로 구분해 볼 수 있다.
일반적으로 인덱스를 이용해서 쿼리하는 작업에서는 가져오는 레코드의 건수가 소량이기 때문에 인덱스 스캔 작업은 부하가 작지만 특정 인덱스 키를 찾는 인덱스 탐색 작업은 상대적으로 부하가 높은 편이다.

조인 작업에서 드라이빙 테이블을 읽을 때는 인덱스 탐색 작업을 단 한 번만 수행하고, 그 이후부터는 스캔만 실행하면 된다.
하지만 드리븐 테이블에서는 인덱스 탐색 작업과 스캔 작업을 드라이빙 테이블에서 읽은 레코드 건수만큼 반복한다.
드라이빙 테이블과 드리븐 테이블이 1:1로 조인되더라도 드리븐 테이블을 읽는 것이 훨씬 더 큰 부하를 차지한다.
그래서 옵티마이저는 항상 드라이빙 테이블이 아니라 드리븐 테이블을 최적으로 읽을 수 있게 실행 계획을 수립한다.
다음과 같이 employees 테이블과 dept_emp 테이블을 조인하는 쿼리로 이 내용을 한번 살펴보자.

```
SELECT *
FROM employees e, dept_emp de
WHERE e.emp_no=de.emp_no;
```

이 두 테이블의 조인 쿼리에서 employees 테이블의 emp_no 칼럼과 dept_emp 테이블의 emp_no 칼럼에 각각 인덱스가 있을 때와 없을 때 조인 순서가 어떻게 달라지는지 한번 살펴보자.

- 두 칼럼 모두 각각 인덱스가 있는 경우: employees 테이블의 emp_no 칼럼과 dept_emp 테이블의 emp_no 칼럼에 모두 인덱스가 준비돼 있을 때는 어느 테이블을 드라이빙으로 선택하든 인덱스를
  이용해 드리븐 테이블의 검색 작업을 빠르게 처리할 수 있다. 이럴 때 옵티마이저가 통계 정보를 이용해 적절히 드라이빙 테이블을 선택하게 된다.
  각 테이블의 통계 정보에 있는 레코드 건수에 따라 employees가 드라이빙 테이블이 될 수도 있고, dept_emp 테이블이 드라이빙 테이블로 선택될 수도 있다.
  보통의 경우 어느 쪽 테이블이 드라이빙 테이블이 되든 옵티마이저가 선택하는 방법이 최적일 때가 많다.

- employees.emp_no에만 인덱스가 있는 경우: employees.emp_no에만 인덱스가 있을 때 dept_emp 테이블이 드리븐 테이블로 선택된다면 employees 테이블의 레코드 건수만큼 dept_emp
  테이블을 플 스캔해야만 "e.emp_no=de.emp_no" 조건에 일치하는 레코드를 찾을 수 있다.
  그래서 옵티마이저는 항상 dept_emp 테이블을 드라이빙 테이블로 선택하고, employees 테이블을 드리븐 테이블로 선택한다.
  이때는 "e.emp_no=100001"과 같이 employees 테이블을 아주 효율적으로 접근할 수 있는 조건이 있더라도 옵티마이저는 employees 테이블을 드라이빙 테이블로 선택하지 않을 가능성이 높다.

- dept_emp.emp_no에만 인덱스가 있는 경우: 위의 "employees.emp_no에만 인덱스가 있는 경우"와는 반대로 처리된다
  이때는 employees 테이블의 반복된 풀 스캔을 피하기 위해 employees 테이블을 드라이빙 테이블로 선택하고 dept_emp 테이블을 드리븐 테이블로 조인을 수행하게 실행 계획을 수립한다.

- 두 칼럼 모두 인덱스가 없는 경우: "두 칼럼 모두 각각 인덱스가 있는 경우"와 마찬가지로 어느 테이블을 드라이빙으로 선택하더라도 드리븐 테이블의 풀 스캔(각 테이블의 데이터 조회 범위를 좁힐 수 있는
  조건이 있다면 풀 스캔이 아닌 인덱스 레인지 스캔으로 범위를 좁힐 수 있다. 즉, 조인 조건 자체는 효율적이지 못하지만 조인 대상을 찾을 범위는 줄일 수 있다.)은 발생하기 때문에 옵티마이저가 적절히
  드라이빙 테이블을 선택한다. 단 레코드 건수가 적은 테이블을 드라이빙 테이블로 선택하는 것이 훨씬 효율적이다.
  이렇게 조인 조건을 빠르게 처리할 적절한 인덱스가 없는 경우 MySQL 8.0.18 이전 버전까지는 블록 네스티드 루프 조인을 사용했다.
  하지만 MySQL 8.0.18 버전부터는 블록 네스티드 루프 조인이 없어지고 해시 조인이 도입되면서 해시 조인으로 처리된다.

### [2] JOIN 칼럼의 데이터 타입
WHERE 절에 사용되는 조건에서 비교 대상 칼럼과 표현식의 데이터 타입을 반드시 동일하게 사용해야 하는 이유는 이미 자세히 살펴봤다. 이것은 테이블의 조인 조건에서도 동일하다.
조인 칼럼 간의 비교에서 각 칼럼의 데이터 타입이 일치하지 않으면 인덱스를 효율적으로 이용할 수 없다. 다음 예제를 살펴보자.

```
mysql> CREATE TABLE tb_test1 (user_id INT, user_type INT, PRIMARY KEY(user_id));
mysql> CREATE TABLE tb_test2 (user_type CHAR(1), type_desc VARCHAR(10), PRIMARY KEY(user_type));

mysql> SELECT *
       FROM tb_test1 tb1, tb_test2 tb2
       WHERE tb1.user_type=tb2.user_type;
```

tb_test2 테이블의 user_type 칼럼은 프라이머리 키다. 이 쿼리는 최소한 드리븐 테이블은 프라이머리 키를 이용한 인덱스 레인지 스캔을 사용해 조인이 처리될 것으로 예상할 수 있다.
하지만 이 쿼리의 실행 계획은 두 테이블을 모두 풀 테이블 스캔으로 접근한다.
게다가 드리븐 테이블(실행 계획의 두 번째 줄)의 Extra 칼럼에 "Using join buffer (hash join)"가 표시된 것으로 봐서 조인 버퍼를 이용해 해시 조인이 실행된 것을 알 수 있다.

```
+----+-------+------+------+--------------------------------------------+
| id | table | type | key  | Extra                                      |
+----+-------+------+------+--------------------------------------------+
|  1 | tb1   | ALL  | NULL | NULL                                       |
|  1 | tb2   | ALL  | NULL | Using where; Using join buffer (hash join) |
+----+-------+------+------+--------------------------------------------+
```

사실 이 문제는 '문자열이나 숫자 비교'에서 자세히 살펴본 것과 같은 것이다. 이 쿼리에서는 비교 조건의 양쪽 항이 모두 테이블의 칼럼이라는 점만 다르다.
즉, 비교 조건에서 양쪽 항이 상수이든 테이블의 칼럼이든 관계없이 인덱스를 사용하려면 양쪽 항의 데이터 타입을 일치시켜야 한다는 것은 마찬가지다.
이 쿼리에서는 tb_test2 테이블의 user_type 칼럼을 CHAR(1)에서 INT로 변환해서 비교를 수행한다.
그로 인해 인덱스의 변형이 필요하기 때문에 tb_test2 테이블의 인덱스를 제대로 사용할 수 없게 된 것이다.

옵티마이저는 드리븐 테이블이 인덱스 레인지 스캔을 사용하지 못하고, 드리븐 테이블의 풀 테이블 스캔이 필요한 것을 알고 조금이라도 빨리 실행되도록 조인 버퍼를 활용한 해시 조인을 사용한다.

인덱스 사용에 영향을 미치는 데이터 타입 불일치는 CHAR 타입과 VARCHAR 타입, 또는 INT 타입과 BIGINT 타입, 그리고 DATE 타입과 DATETIME 타입 사이에서는 발생하지 않는다.
즉, CHAR 타입과 VARCHAR 타입의 비교는 특별히 문제가 되지 않으며, INT 타입과 BIGINT 타입 또는 SMALLINT 타입과의 비교 등도 문제가 되지 않는다.
하지만 대표적으로 다음의 비교 패턴은 문제가 될 가능성이 높다.

- CHAR 타입과 INT 타입의 비교와 같이 데이터 타입의 종류가 완전히 다른 경우
- 같은 CHAR 타입이더라도 문자 집합이나 콜레이션이 다른 경우
- 같은 INT 타입이더라도 부호(Sign)의 존재 여부가 다른 경우

두 개의 테이블에서 같은 값을 저장하는 각 칼럼이 서로 다른 문자 집합과 콜레이션으로 생성됐을 때 조인에 어떤 영향을 미치게 되는지 다음 예제 쿼리로 살펴보자.

```
mysql> CREATE TABLE tb_test1(
         user_id INT,
         user_type CHAR(1) COLLATE utf8mb4_general_ci,
         PRIMARY KEY(user_id)
       );

mysql> CREATE TABLE tb_test2(
         user_type CHAR(1) COLLATE latin1_general_ci,
         type_desc VARCHAR(10),
         INDEX ix_usertype (user_type)
       );

mysql> SELECT *
       FROM tb_test1 tb1, tb_test2 tb2
       WHERE tb1.user_type=tb2.user_type;
```

위와 같이 tb_test1과 tb_test2 테이블을 생성하고, 두 테이블을 조인하는 쿼리의 실행 계획을 한번 살펴보자.

```
+----+-------+------+------+--------------------------------------------+
| id | table | type | key  | Extra                                      |
+----+-------+------+------+--------------------------------------------+
|  1 | tb1   | ALL  | NULL | NULL                                       |
|  1 | tb2   | ALL  | NULL | Using where; Using join buffer (hash join) |
+----+-------+------+------+--------------------------------------------+
```

드리븐 테이블을 풀 테이블 스캔하는 실행 계획으로 조인이 실행됐기 때문에 옵티마이저가 조인 버퍼를 사용했다.
기본적인 표준화 규칙을 가지고 데이터 모델링된 경우에는 이러한 케이스가 잘 발생하지 않지만 규칙 없이 조금씩 데이터 모델을 변경하다 보면 이런 현상이 자주 발생한다.
이럴 때는 칼럼의 문자 집합과 콜레이션을 통일하는 것만이 유일한 해결책이다. 데이터베이스 모델에 대한 표준화 규칙을 수립하고, 규칙을 기반으로 설계를 진행한다면 이런 문제를 최소화할 수 있을 것이다.
표준 규칙을 수립하기가 어렵다면 각 칼럼에 저장되는 데이터 타입에 맞게 칼럼의 타입을 선정하는 것이 중요하다. 그리고 조인이 수행되는 칼럼은 데이터 타이블 일치시키기 위해 최종 점검을 하는 것이 좋다.

### [3] OUTER JOIN의 성능과 주의사항
이너 조인(INNER JOIN)은 조인 대상 테이블에 모두 존재하는 레코드만 결과 집합으로 반환한다. 이너 조인의 이 같은 특성 때문에 아우터 조인(OUTER JOIN)으로만 조인을 실행하는 쿼리들도 자주 보인다.
다음 예제 쿼리는 3개 테이블을 조인하면서 LEFT JOIN만 사용하고 있다.

```
mysql> SELECT *
       FROM employees e
         LEFT JOIN dept_emp de ON de.emp_no=e.emp_no
         LEFT JOIN departments d ON d.dept_no=de.dept_no AND d.dept_name='Development';
```

이 쿼리의 실행 계획을 보면 다음과 같이 제일 먼저 employees 테이블을 풀 스캔하면서 dept_emp 테이블과 departments 테이블을 드리븐 테이블로 사용한다는 것을 알 수 있다.

```
+----+-------+--------+-------------------+--------+-------------+
| id | table | type   | key               | rows   | Extra       |
+----+-------+--------+-------------------+--------+-------------+
|  1 | e     | ALL    | NULL              | 299920 | NULL        |
|  1 | de    | ref    | ix_empno_fromdate |      1 | NULL        |
|  1 | d     | eq_ref | PRIMARY           |      1 | Using where |
+----+-------+--------+-------------------+--------+-------------+
```

employees 테이블에 존재하는 사원 중에서 dept_emp 테이블에 레코드를 갖지 않는 경우가 있다면 아우터 조인이 필요하지만, 대부분 그런 경우는 없으므로 굳이 아우터 조인을 사용할 필요가 없다.
즉 테이블의 데이터가 일관되지 않은 경우에만 아우터 조인이 필요한 경우인 것이다.
MySQL 옵티마이저는 절대 아우터로 조인되는 테이블을 드라이빙 테이블로 선택하지 못하기 때문에 풀 스캔이 필요한 employees 테이블을 드라이빙 테이블로 선택한다.
그 결과 쿼리의 성능이 떨어지는 실행 계획을 수립한 것이다.

이 쿼리에 이너 조인을 이용했다면 다음과 같이 departments 테이블에서 부서명이 "Development"인 레코드 1건만 찾아서 조인을 실행하는 실행 계획을 선택했을 것이다.

```
+----+-------+--------+-------------------+-------+-------------+
| id | table | type   | key               | rows  | Extra       |
+----+-------+--------+-------------------+-------+-------------+
|  1 | d     | ref    | ux_deptname       |     1 | Using index |
|  1 | de    | ref    | PRIMARY           | 41392 | NULL        |
|  1 | e     | eq_ref | PRIMARY           |     1 | NULL        |
+----+-------+--------+-------------------+-------+-------------+
```

이너 조인으로 사용해도 되는 쿼리를 아우터 조인으로 작성하면 MySQL 옵티마이저가 조인 순서를 변경하면서 수행할 수 있는 최적화의 기회를 빼앗아버리는 결과가 된다.
필요한 데이터와 조인되는 테이블 간의 관계를 정확히 파악해서 꼭 필요한 경우가 아니라면 이너 조인을 사용하는 것이 업무 요건을 정확히 구현함과 동시에 쿼리의 성능도 향상시킬 수 있다.

아우터 조인(OUTER JOIN) 쿼리를 작성하면서 많이 하는 또 다른 실수는 다음 에제와 같이 아우터(OUTER)로 조인되는 테이블에 대한 조건을 WHERE 절에 함께 명시하는 것이다.

```
mysql> SELECT *
       FROM employees e
         LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no
       WHERE mgr.dept_no='d001';
```

ON 절에 조인 조건은 명시했지만 아우터로 조인되는 테이블인 dept_manager의 dept_no='d001' 조건을 WHERE 절에 명시한 것은 잘못된 조인 방법이다.
위의 LEFT JOIN이 사용된 쿼리는 WHERE 절의 조건 때문에 MySQL 옵티마이저가 LEFT JOIN을 다음 쿼리와 같이 INNER JOIN으로 변환해서 실행해버린다.

```
mysql> SELECT *
       FROM employees e
         INNER JOIN dept_manager mgr ON mgr.emp_no=e.emp_no
       WHERE mgr.dept_no='d001';
```

정상적인 아우터 조인이 되게 만들려면 다음 쿼리와 같이 WHERE 절의 "mgr.dept_no='d001'" 조건을 LEFT JOIN의 ON 절로 옮겨야 한다.

```
mysql> SELECT *
       FROM employees e
         LEFT JOIN dept_manager mgr ON mgr.emp_no=e.emp_no AND mgr.dept_no='d001';
```

예외적으로 OUTER JOIN으로 연결되는 테이블의 칼럼에 대한 조건을 WHERE 절에 사용해야 하는 경우가 있는데, 다음과 같이 안티 조인(ANTI-JOIN) 효과를 기대하는 경우가 그렇다.

```
mysql> SELECT *
       FROM employees e
         LEFT JOIN dept_manager dm ON dm.emp_no=e.emp_no
       WHERE dm.emp_no IS NULL
       LIMIT 10;
```

위 쿼리는 사원 중에서 매니저가 아닌 사용자들만 조회하는 쿼리인데, WHERE 절에 아우터로 조인된 dept_manager 테이블의 emp_no 칼럼이 NULL인 레코드들만 조회한다.
이런 형태의 요건이 아우터 테이블의 칼럼이 WHERE 절에 사용될 수 있는 유일한 경우다. 그 외의 경우 MySQL 서버는 LEFT JOIN을 INNER JOIN으로 자동 변환한다는 것을 꼭 기억하자.

### [4] JOIN과 외래키(FOREIGH KEY)
데이터베이스에 외래키(FOREIGN KEY)가 생성돼 있어야만 조인할 수 있는지 궁금해하는 게시물을 본 적이 있다. 외래키는 조인과 아무런 연관이 없다.
외래키를 생성하는 주목적은 데이터의 무결성을 보장하기 위해서다. 외래키와 연관된 무결성을 참조 무결성이라고 표현한다.
예를 들어, 부서 테이블과 사원 테이블이 있고, 사원 테이블에 이 사원이 소속된 부서 정보를 저장하는 칼럼이 있다.
이때 사원 테이블의 부서 코드는 반드시 부서 테이블에 존재하는 부서 정보만 사용해야 하는데, 이것이 바로 참조 무결성이다.
그런데 애플리케이션의 버그 등의 이유로 부서 테이블에는 존재하지 않는 부서 코드가 사원 테이블에 있을 수 있다. 이렇게 참조 무결성이 깨지는 문제를 DBMS 차원에서 막기 위해 외래키를 생성한다.

하지만 SQL로 테이블 간의 조인을 수행하는 것은 전혀 무관한 칼럼을 조인 조건으로 사용해도 문법적으로는 문제가 되지 않는다. 데이터 모델링을 할 때는 각 테이블 간의 관계를 필수적으로 그려 넣어야 한다.
하지만 그 데이터 모델을 데이터베이스에 생성할 때는 그 테이블 간의 관계는 외래키로 생성하지 않을 때가 더 많다. 하지만 테이블 간의 조인을 사용하기 위해 외래키가 필요한 것은 아니다.

### [5] 지연된 조인(Delayed Join)
조인을 사용해서 데이터를 조회하는 쿼리에 GROUP BY 또는 ORDER BY를 사용할 때 각 처리 방법에서 인덱스를 사용한다면 이미 최적으로 처리되고 있을 가능성이 높다.
하지만 그렇지 못하다면 MySQL 서버는 우선 모든 조인을 실행하고 난 다음 GROUP BY나 ORDER BY를 처리할 것이다. 조인은 대체로 실행되면 될수록 결과 레코드 건수가 늘어난다.
그래서 조인의 결과를 GROUP BY하거나 ORDER BY하면 조인을 실행하기 전의 레코드에 GROUP BY나 ORDER BY를 수행하는 것보다 많은 레코드를 처리해야 한다.
지연된 조인이란 조인이 실행되기 이전에 GROUP BY나 ORDER BY를 처리하는 방식을 의미한다. 지연된 조인은 주로 LIMIT이 함께 사용된 쿼리에서 더 큰 효과를 얻을 수 있다.

인덱스를 사용하지 못하는 GROUP BY와 ORDER BY 쿼리를 지연된 조인으로 처리하는 방법을 한번 살펴보자.

```
mysql> SELECT e.*
       FROM salaries s, employees e
       WHERE e.emp_no=s.emp_no
         AND s.emp_no BETWEEN 10001 AND 13000
       GROUP BY s.emp_no
       ORDER BY SUM(s.salary) DESC
       LIMIT 10;
```

위 쿼리의 실행 계획은 다음과 같다.
실행 계획상으로는 employees 테이블을 드라이빙 테이블로 선택해서 "emp_no BETWEEN 10001 AND 13000" 조건을 만족하는 레코드 3000건을 읽고, salaries 테이블을 조인했다.
이때 조인을 수행한 횟수는 12,000번(3000 * 4) 정도라는 것을 알 수 있다. 그리고 조인의 결과 12,000건의 레코드를 임시 테이블에 저장하고 GROUP BY 처리를 통해 3000건으로 줄였다.
그리고 ORDER BY를 처리해서 상위 10건만 최종적으로 반환한다.

```
+----+-------+-------+---------+------+----------------------------------------------+
| id | table | type  | key     | rows | Extra                                        |
+----+-------+-------+---------+------+----------------------------------------------+
|  1 | e     | range | PRIMARY | 3000 | Using where; Using temporary; Using filesort |
|  1 | s     | ref   | PRIMARY |   10 | NULL                                         |
+----+-------+-------+---------+------+----------------------------------------------+
```

이제 지연된 조인으로 변경한 쿼리를 한번 살펴보자.
다음 쿼리에서는 salaries 테이블에서 가능한 모든 처리(WHERE 조건 및 GROUP BY와 ORDER BY, LIMIT까지)를 수행한 다음, 그 결과를 임시 테이블에 저장했다.
그리고 임시 테이블의 결과를 employees 테이블과 조인하도록 고친 것이다. 즉, 모든 처리를 salaries 테이블에서 수행하고, 최종 10건만 employees 테이블과 조인했다.

```
mysql> SELECT e.*
       FROM
         (SELECT s.emp_no
          FROM salaries s
          WHERE s.emp_no BETWEEN 10001 AND 13000
          GROUP BY s.emp_no
          ORDER BY SUM(s.emp_no) DESC
          LIMIT 10) x,
         employees e
       WHERE e.emp_no=x.emp_no;
```

지연된 조인으로 변경한 쿼리의 실행 계획은 다음과 같다.

```
+----+------------+--------+---------+-------+----------------------------------------------+
| id | table      | type   | key     | rows  | Extra                                        |
+----+------------+--------+---------+-------+----------------------------------------------+
|  1 | <derived2> | ALL    | NULL    |    10 | NULL                                         |
|  1 | e          | eq_ref | PRIMARY |     1 | NULL                                         |
|  2 | s          | range  | PRIMARY | 56844 | Using where; Using temporary; Using filesort |
+----+------------+--------+---------+-------+----------------------------------------------+
```

예상했던 대로 FROM 절에 서브쿼리가 사용됐기 때문에 이 서브쿼리의 결과는 파생 테이블(세 번째 줄의 DERIVED)로 처리됐다.
다음 실행 계획에서 FROM 절의 서브쿼리를 위해 전체 56,844건의 레코드를 읽어야 한다고 나왔지만 사실은 28,606건의 레코드만 읽으면 되는 쿼리다.
지연된 조인으로 변경된 이 쿼리는 salaries 테이블에서 28,606건의 레코드를 읽어 임시 테이블에 저장하고, GROUP BY 처리를 통해 3,000건으로 줄였다.
그리고 ORDER BY를 처리해 상위 10건만 임시 테이블(<derived2>)에 저장한다. 최종적으로 임시 테이블의 10건을 읽어서 employees 테이블과 조인을 10번만 수행해서 결과를 반환한다.

지연된 조인으로 개선되기 전과 후의 쿼리가 어떻게 처리되는지 간략하게 살펴봤다. 물론 지연된 조인으로 개선된 쿼리는 임시 테이블(<derived2>)을 한 번 더 사용하기 때문에 느리다고 예상할 수도 있다.
하지만 임시 테이블에 저장할 레코드가 10건밖에 되지 않으므로 메모리를 이용해 빠르게 처리된다. 조인의 횟수를 비교해보면 지연된 조인으로 변경된 쿼리의 조인 횟수가 훨씬 적다는 사실을 알 수 있다.
실행 계획상으로 보면 지연된 조인으로 변경된 쿼리가 오히려 더 느릴 것 같지만, 실제 테스트를 해보면 지연된 조인으로 개선된 쿼리가 3\~4배 정도는 더 빠르게 실행된다는 것을 확인할 수 있다.
지연된 쿼리의 원리를 정확히 이해하지 못한 상태로 지연된 쿼리를 작성하면 오히려 역효과가 날 수도 있다. 하지만 잘 튜닝된 지연된 쿼리는 원래의 쿼리보다 몇십 배, 몇백 배 더 나은 성능을 보일 수도 있다.

지연된 조인은 경우에 따라 상당한 성능 향상을 가져올 수 있지만 모든 쿼리를 지연된 조인 형태로 개선할 수 있는 것은 아니다.
OUTER JOIN과 INNER JOIN에 대해 다음과 같은 조건이 갖춰져야만 지연된 쿼리로 변경해서 사용할 수 있다.

- LEFT (OUTER) JOIN인 경우 드라이빙 테이블과 드리븐 테이블은 1:1 또는 M:1 관계여야 한다.
- INNER JOIN인 경우 드라이빙 테이블과 드리븐 테이블은 1:1 또는 M:1의 관계임과 동시에 (당연한 조건이겠지만)드라이빙 테이블에 있는 레코드는 드리븐 테이블에 모두 존재해야 한다.
  두 번째와 세 번째 조건은 드라이빙 테이블을 서브쿼리로 만들고 이 서브쿼리에 LIMIT을 추가해도 최종 결과의 건수가 변하지 않는다는 보증을 해주는 조건이기 때문에 반드시 정확히 확인한 후 적용해야
  한다.

앞의 예제나 지금 개발하고 있는 페이징 쿼리를 지연된 조인 쿼리로 한번 변경해보고, 성능 차이를 비교해 보자. 이런 작업을 몇 번 해보면 위의 두 조건이 어떤 의미인지 더 쉽게 이해할 수 있을 것이다.

> 지연된 조인은 여기서 언급한 조인의 개수를 줄이는 것뿐만 아니라 GROUP BY나 ORDER BY 처리가 필요한 레코드의 전체 크기를 줄이는 역할도 한다.
> 첫 번째 GROUP BY와 ORDER BY가 포함된 예제 쿼리를 보면 지연된 조인으로 개선되기 전 쿼리는 salaries 테이블과 employees 테이블의 모든 칼럼을 임시 테이블에 저장하고 GROUP BY를 해야
> 한다.
> 하지만 지연된 조인으로 개선된 쿼리는 salaries 테이블의 칼럼만 임시 테이블에 저장하고 GROUP BY를 수행하면 되기 때문에 원래의 쿼리보다는 GROUP BY나 ORDER BY용 버퍼를 더 적게 필요로 한다.

### [6] 래터럴 조인(Lateral Join)
MySQL 8.0 이전 버전까지는 그룹별로 몇 건씩만 가져오는 쿼리를 작성할 수가 없었다.
하지만 MySQL 8.0 버전부터는 래터럴 조인이라는 기능을 이용해 특정 그룹별로 서브쿼리를 실행해서 그 결과와 조인하는 것이 가능해졌다. 예를 들어, 다음 쿼리를 한번 살펴보자.

```
mysql> SELECT *
       FROM employees e
         LEFT JOIN LATERAL (SELECT *
                            FROM salaries s
                            WHERE s.emp_no=e.emp_no
                            ORDER BY s.from_date DESC LIMIT 2) s2 ON s2.emp_no=e.emp_no
       WHERE e.first_name='Matt';
```

위의 쿼리는 employees 테이블에서 이름이 'Matt'인 사원에 대해 사원별로 가장 최근 급여 변경 내역을 최대 2건씩만 반환한다.
래터럴 조인에서 가장 중요한 부분은 FROM 절에 사용된 서브쿼리(Derived Table)에서 외부 쿼리의 FROM 절에 정의된 테이블의 칼럼을 참조할 수 있다는 것이다.
이 예제에서는 salaries 테이블을 읽는 서브쿼리에서 employees 테이블의 emp_no를 참조한다.
이렇게 FROM 절에 사용된 서브쿼리가 외부 쿼리의 칼럼을 참조하기 위해서는 "LATERAL" 키워드가 명시돼야 한다. LATERAL 키워드 없이 외부 쿼리의 칼럼을 참조하면 다음과 같은 에러가 발생한다.

```
mysql> SELECT *
       FROM employees e
       LEFT JOIN (SELECT *
                  FROM salaries s
                  WHERE s.emp_no=e.emp_no
                  ORDER BY s.from_date DESC LIMIT 2) s2 ON s2.emp_no=e.emp_no
       WHERE e.first_name='Matt';

ERROR 1054 (42S22): Unknown column 'e.emp_no' in 'where clause'
```

LATERAL 키워드를 가진 서브쿼리는 조인 순서상 후순위로 밀리고, 외부 쿼리의 결과 레코드 단위로 임시 테이블이 생성(외부 쿼리 결과의 레코드 단위로 임시 테이블이 생성되기 때문에 래터럴 조인의 실행
계획에는 Extra 칼럼에 "Rematerialize"라는 메시지가 표시된다.)되기 때문에 꼭 필요한 경우에만 사용해야 한다.

### [7] 실행 계획으로 인한 정렬 흐트러짐
MySQL 8.0 이전 버전까지는 네스티드-루프 방식의 조인만 가능했지만 MySQL 8.0 버전부터는 해시 조인 방식이 도입됐다.
네스티드-루프 조인은 알고리즘의 특성상 드라이빙 테이블에서 읽은 레코드의 순서가 다른 테이블이 모두 조인돼도 그대로 유지된다.
그래서 MySQL에서 조인을 사용하는 쿼리의 결과는 드라이빙 테이블을 읽은 순서로 정렬된다고 생각할 때가 많다.
실제로도 주어진 조건에 의해 드라이빙 테이블을 인덱스 스캔이나 풀 테이블 스캔을 하고, 그때 드라이빙 테이블을 읽은 순서가 그대로 최종 결과에 반영된다.

하지만 쿼리의 실행 계획에서 네스티드 루프 조인 대신 해시 조인이 사용되면 쿼리 결과의 레코드 정렬 순서가 달라진다.
해시 조인뿐만 아니라 MySQL 8.0 이전 버전에서 사용되던 블록 네스티드 루프 조인이 사용되는 경우도 동일하게 쿼리 결과의 정렬 순서가 드라이빙 테이블을 읽는 순서와 다르게 출력됐다.

해시 조인이 사용되는 쿼리에서 결과가 어떤 순서로 출력되는지 한번 살펴보자. 예제 쿼리의 실행 계획을 보면 네스티드 루프 조인이 아니라 해시 조인이 사용된 것을 확인할 수 있다.

```
mysql> SELECT e.emp_no, e.first_name, e.last_name, de.from_date
       FROM dept_emp de, employees e
       WHERE de.from_date>'2001-10-01' AND e.emp_no<10005;

+----+-------+-------+-------------+---------------------------------------------------------+
| id | table | type  | key         | Extra                                                   |
+----+-------+-------+-------------+---------------------------------------------------------+
|  1 | e     | range | PRIMARY     | Using where                                             |
|  1 | de    | range | ix_fromdate | Using where; Using index; Using join buffer (hash join) |
+----+-------+-------+-------------+---------------------------------------------------------+
```

네스티드 루프 방식으로 조인이 처리되면 드라이빙 테이블을 읽은 순서대로 결과가 조회되는 것이 일반적이다. 즉, 이 예제에서는 employees 테이블의 프라이머리 키인 emp_no 값의 순서대로 조회돼야 한다.
하지만 이 쿼리의 결과는 emp_no 칼럼으로 정렬돼 있지 않고, emp_no가 반복적으로 순환되는 결과가 만들어진 것을 확인할 수 있다.

```
+--------+------------+-----------+------------+
| emp_no | first_name | last_name | from_date  |
+--------+------------+-----------+------------+
|  10001 | Georgi     | Facello   | 2001-10-02 |
|  10002 | Bezalel    | Simmel    | 2001-10-02 |
|  10003 | Parto      | Bamford   | 2001-10-02 |
|  10004 | Christian  | Koblick   | 2001-10-02 |
|  10001 | Georgi     | Facello   | 2001-10-02 |
|  10002 | Bezalel    | Simmel    | 2001-10-02 |
|  10003 | Parto      | Bamford   | 2001-10-02 |
|  10004 | Christian  | Koblick   | 2001-10-02 |
|  10001 | Georgi     | Facello   | 2001-10-02 |
|  10002 | Bezalel    | Simmel    | 2001-10-02 |
...
```

실행 계획은 MySQL 옵티마이저에 의해 그때그때 상황에 따라 달라질 수 있다.
그러므로 정렬된 결과가 필요한 경우라면 드라이빙 테이블의 순서에 의존하지 말고 ORDER BY 절을 명시적으로 사용하는 것이 좋다.
<br/>
<br/>
## (8) GROUP BY
GROUP BY는 특정 칼럼의 값으로 레코드를 그루핑하고, 그룹별로 집계된 결과를 하나의 레코드로 조회할 때 사용한다.
이번에는 MySQL에서 GROUP BY를 사용할 때 함께 사용할 수 있는 유용한 기능과 주의사항 위주로 살펴보겠다.

### [1] WITH ROLLUP
GROUP BY가 사용된 쿼리에서는 그루핑된 그룹별로 소계를 가져올 수 있는 롤입(ROLLUP) 기능을 사용할 수 있다.
ROLLUP으로 출력되는 소계는 단순히 최종 합만 가져오는 것이 아니라 GROUP BY에 사용된 칼럼의 개수에 따라 소계의 레벨이 달라진다.
MySQL의 GROUP BY ... ROLLUP 쿼리는 엑셀의 피벗 테이블과 거의 동일한 기능으로 생각하면 된다. ROLLUP 쿼리의 결과를 보면서 살펴보자.

```
mysql> SELECT dept_no, COUNT(*)
       FROM dept_emp
       GROUP BY dept_no WITH ROLLUP;
```

WITH ROLLUP과 함께 사용된 GROUP BY 쿼리의 결과는 그룹별로 소계를 출력하는 레코드가 추가되어 표시된다. 소계 레코드의 칼럼값은 항상 NULL로 표시된다는 점에 주의해야 한다.
이 예제에서는 GROUP BY 절에 dept_no 칼럼 1개만 있기 때문에 소계가 1개만 존재하고 dept_no 칼럼값은 NULL로 표기됐다.

<img src="https://github.com/user-attachments/assets/e966af9c-aeaf-40b7-a5f7-2b933649f1a6" width="250"/><br/>

GROUP BY 절에 칼럼이 2개인 다음 쿼리를 한번 살펴보자. 다음 쿼리는 사원의 first_name과 last_name으로 그루핑하는 예제다.

```
mysql> SELECT first_name, last_name, COUNT(*)
       FROM employees
       GROUP BY first_name, last_name WITH ROLLUP;
```

이 쿼리의 결과는 다음과 같은데, GROUP BY 절에 칼럼이 2개로 늘어나면서 소계가 2단계로 표시됐다.
이 쿼리에서 ROLLUP 결과는 first_name 그룹별로 소계 레코드가 출력되고, 맨 마지막에 전체 총계가 출력된다.
first_name 그룹별 소계 레코드의 first_name 칼럼은 NULL이 아니지만 last_name 칼럼의 값은 NULL로 채워져 있다.
마지막의 총계는 first_name과 last_name 칼럼이 모두 NULL로 채워져 있다. 소계나 총계 레코드는 항상 해당 그룹의 마지막에 나타난다.

<img src="https://github.com/user-attachments/assets/432777b0-7753-4b55-b4f6-3a35ccccb02a" width="320"/><br/>

MySQL 8.0 버전부터는 그룹 레코드에 표시되는 NULL을 사용자가 변경할 수 있게 GROUPING() 함수를 지원한다.

```
mysql> SELECT
         IF(GROUPING(first_name), 'All first_name', first_name) AS first_name,
         IF(GROUPING(last_name), 'All last_name', last_name) AS last_name,
         COUNT(*)
       FROM employees
       GROUP BY first_name, last_name WITH ROLLUP;
```

GROUPING() 함수의 사용 결과에서는 더이상 NULL로 표시되지 않고, 주어진 문자열('All last_name'과 'All first_name')이 표시되는 것을 확인할 수 있다.

<img src="https://github.com/user-attachments/assets/d593aff1-e0b6-4e51-b785-32ba83c52380" width="350"/><br/>

### [2] 레코드를 칼럼으로 변환해서 조회
GROUP BY나 집합 함수를 통해 레코드를 그루핑할 수 있지만 하나의 레코드를 여러 개의 칼럼으로 나누거나 변환하는 SQL 문법은 없다.
하지만 SUM()이나 COUNT() 같은 집합 함수와 CASE WHEN ... END 구문을 이용해 레코드를 칼럼으로 변환하거나 하나의 칼럼을 조건으로 구분해서 2개 이상의 칼럼으로 변환하는 것은 가능하다.
레코드를 칼럼으로 변환한다는 것은 엑셀의 피봇(Pivot) 테이블을 만드는 것과 동일한 개념이다.

#### ◼︎ 레코드를 칼럼으로 변환
우선 다음과 같이 dept_emp 테이블을 이용해 부서별로 사원의 수를 확인하는 쿼리를 생각해 보자.

```
mysql> SELECT dept_no, COUNT(*) AS emp_count
       FROM dept_emp
       GROUP BY dept_no;

+---------+-----------+
| dept_no | emp_count |
+---------+-----------+
|    d001 |     20211 |
|    d002 |     17346 |
|    d003 |     17786 |
|    d004 |     73485 |
        ...
+---------+-----------+
```

위 쿼리로 부서 번호와 부서별 사원 수를 그루핑한 결과가 만들어진다. 하지만 레포팅 도구나 OLAP 같은 도구에서는 자주 이러한 결과를 반대로 만들어야 할 수도 있다.
즉, 레코드를 칼럼으로 변환해야 하는 것이다. 이때는 위의 GROUP BY 쿼리 결과를 SUM(CASE WHEN ...) 구문을 사용해 한 번 더 변환하면 된다. 다음 예제를 한번 살펴보자.

```
SELECT
  SUM(CASE WHEN dept_no='d001' THEN emp_count ELSE 0 END) AS count_d001
  SUM(CASE WHEN dept_SUM(CASE WHEN dept_no='d002' THEN emp_count ELSE 0 END) AS count_d002,
  SUM(CASE WHEN dept_no='d003' THEN emp_count ELSE 0 END) AS count_d003,
  SUM(CASE WHEN dept_no='d004' THEN emp_count ELSE 0 END) AS count_d004,
  SUM(CASE WHEN dept_no='d005' THEN emp_count ELSE 0 END) AS count_d005,
  SUM(CASE WHEN dept_no='d006' THEN emp_count ELSE 0 END) AS count_d006,
  SUM(CASE WHEN dept_no='d007' THEN emp_count ELSE 0 END) AS count_d007,
  SUM(CASE WHEN dept_no='d008' THEN emp_count ELSE 0 END) AS count_d008,
  SUM(CASE WHEN dept_no='d009' THEN emp_count ELSE 0 END) AS count_d009,
  SUM(emp_count) AS count_total
FROM (
  SELECT dept_no, COUNT(*) AS emp_count FROM dept_emp GROUP BY dept_no
) tb_derived;
```

위 쿼리의 결과로 다음고 같이 부서 정보와 부서별 사원의 수가 가로(레코드)가 아니라 세로(칼럼)로 변환된 것을 확인할 수 있다.

<img src="https://github.com/user-attachments/assets/9a0bfe72-2c79-4c00-8fff-9d4fc293b7ce" width="550"/><br/>

변환의 원리는 간단하다. 우선 부서별로 9개의 레코드를 한 건의 레코드로 만들어야 하기 때문에 GROUP BY된 결과를 서브쿼리로 만든 후 SUM() 함수를 적용했다.
즉, 9개의 레코드를 1건의 레코드로 변환했다. 그리고 부서 번호의 순서대로 CASE WHEN 구문을 이용해 각 칼럼에서 필요한 값만 선별해서 SUM()을 했다.

이처럼 레코드를 칼럼으로 변환하는 작업을 할 때는 목적이나 용도에 맞게 COUNT, MIN, MAX, AVG, SUM 등의 집합 함수를 사용하면 된다.
이 예제의 한 가지 단점은 부서 번호가 쿼리의 일부로 사용되기 때문에 부서 번호가 변경되거나 추가되면 쿼리까지도 변경돼야 한다는 것이다. 이런 부분은 동적으로 쿼리를 생성하는 방법 등으로 보완하면 된다.

#### ◼︎ 하나의 칼럼을 여러 칼럼으로 분리
다시 앞의 쿼리로 돌아가보자. 다음 결과는 단순히 부서별로 전체 사원의 수만 조회하는 쿼리였다.

```
mysql> SELECT dept_no, COUNT(*) AS emp_count
       FROM dept_emp
       GROUP BY dept_no;
```

SUM(CASE WHEN ...) 문장은 소그룹을 특정 조건으로 나눠서 사원의 수를 구하는 용도로도 사용할 수 있다. 다음 쿼리는 전체 사원 수와 함께 입사 연도별 사원 수를 구하는 쿼리다.

```
SELECT de.dept_no,
  SUM(CASE WHEN e.hire_date BETWEEN '1980-01-01' AND '1989-12-31' THEN 1 ELSE 0 END) AS cnt_1980,
  SUM(CASE WHEN e.hire_date BETWEEN '1990-01-01' AND '1999-12-31' THEN 1 ELSE 0 END) AS cnt_1990,
  SUM(CASE WHEN e.hire_date BETWEEN '2000-01-01' AND '2009-12-31' THEN 1 ELSE 0 END) AS cnt_2000,
  COUNT(*) AS cnt_total
FROM dept_emp de, employees e
WHERE e.emp_no=de.emp_no
GROUP BY de.dept_no;
```

위 쿼리의 결과는 다음과 같이 1980년도, 1990년도, 2000년도의 부서별 입사 사원의 수를 보여준다.

<img src="https://github.com/user-attachments/assets/cec0ea04-fcb8-490b-820f-fb54b35cef06" width="550"/><br/>

dept_emp 테이블만으로는 사원의 입사 일자를 알 수 없으므로 employees 테이블을 조인했으며, 조인된 결과를 dept_emp 테이블의 dept_no별로 GROUP BY를 실행했다.
그루핑된 부서별 사원의 정보에서 CASE WHEN으로 사원의 입사 연도를 구분해서 연도별로 합계(SUM 함수)를 실행하면 원하는 결과를 얻을 수 있다.
이처럼 간단한 SQL 문장으로 상당히 많은 프로그램 코드를 줄일 수 있다. 그리고 이러한 형태의 쿼리에 WITH ROLLUP 기능을 함께 사용한다면 더 유용한 결과를 만들어 낼 수 있을 것이다.
<br/>
<br/>
## (9) ORDER BY
ORDER BY는 검색된 레코드를 어떤 순서로 정렬할지 결정한다. ORDER BY 절이 사용되지 않으면 SELECT 쿼리의 결과는 어떤 순서로 정렬될까?

- 인덱스를 사용한 SELECT의 경우에는 인덱스에 정렬된 순서대로 레코드를 가져온다.
- 인덱스를 사용하지 못하고 풀 테이블 스캔을 실행하는 SELECT를 가정해보자. MyISAM 테이블은 테이블에 저장된 순서대로 가져오는데, 이 순서가 정확히 INSERT된 순서는 아닐 수도 있다.
  일반적으로 테이블의 레코드가 삭제되면서 빈 공간이 생기고, INSERT되는 레코드는 항상 테이블의 마지막이 아니라 빈 공간이 있으면 그 빈 공간에 저장되기 때문이다.
  InnoDB의 경우에는 항상 프라이머리 키로 클러스터링돼 있기 때문에 풀 테이블 스캔의 경우에는 기본적으로 프라이머리 키 순서대로 레코드를 가져온다.
- SELECT 쿼리가 임시 테이블을 거쳐 처리되면 조회되는 레코드의 순서를 예측하기는 어렵다.

ORDER BY 절이 없는 SELECT 쿼리 결과의 순서는 처리 절차에 따라 달라질 수 있다. 어떤 DBMS도 ORDER BY 절이 명시되지 않은 쿼리에 대해서는 어떠한 정렬도 보장하지 않는다.
예를 들어, 인덱스를 사용한 SELECT 쿼리이기 때문에 ORDER BY 절을 사용하지 않아도 된다는 것은 잘못된 생각이다. 항상 정렬이 필요한 곳에서는 ORDER BY 절을 사용해야 한다.

ORDER BY에서 인덱스를 사용하지 못할 때는 추가 정렬 작업이 수행되며, 쿼리 실행 계획에 있는 Extra 칼럼에 "Using filesort"라는 코멘트가 표시된다.
"filesort"라는 단어에 포함된 "file"은 디스크의 파일을 이용해 정렬을 수행한다는 의미가 아니라 쿼리를 수행하는 도중에 MySQL 서버가 명시적으로 정렬 알고리즘을 수행했다는 의미 정도로 이해하면
된다. 정렬 대상이 많은 경우에는 여러 부분으로 나눠서 처리하는데, 정렬된 결과를 임시로 디스크나 메모리에 저장해 둔다.
실제로 메모리만 이용해 정렬이 수행됐는지 디스크의 파일을 이용했는지는 실행 계획을 통해서는 알 수 없지만 MySQL 서버의 상태 값을 확인해보면 알 수 있다.

```
mysql> SHOW STATUS LIKE 'Sort%';
+-------------------+----------+
| Variable_name     | Value    |
+-------------------+----------+
| Sort_merge_passes | 316      |
| Sort_range        | 0        |
| Sort_rows         | 14257137 |
| Sort_scan         | 27       |
+-------------------+----------+
```

Sort_merge_passes 상태 값은 메모리의 버퍼(sort_buffer_size 시스템 변수로 설정되는 메모리 공간)와 디스크에 저장된 레코드를 몇 번이나 병합했는지를 보여준다.
이 상태 값이 0보다 크다면 이는 정렬해야 할 데이터가 정렬용 버퍼보다 커서 디스크를 이용했다는 것을 의미한다.
Sort_range와 Sort_scan은 인덱스 레인지 스캔을 통해서 읽은 레코드를 정렬한 횟수와 풀 테이블 스캔을 통해서 읽은 레코드를 정렬한 횟수를 누적한 값이다.
Sort_rows는 정렬을 수행했던 전체 레코드 건수의 누적된 값을 나타낸다.

### [1] ORDER BY 사용법 및 주의사항
ORDER BY 절은 1개 또는 그 이상 여러 개의 칼럼으로 정렬을 수행할 수 있으며, 정렬 순서(오름차순, 내림차순)는 칼럼별로 다르게 명시할 수 있다.
일반적으로 정렬할 대상은 칼럼명이나 표현식으로 명시하지만 SELECT되는 칼럼의 순번을 명시할 수도 있다.
즉, "ORDER BY 2"라고 명시하면 SELECT되는 칼럼 중에서 2번째 칼럼으로 정렬하라는 의미가 된다. 다음 2개의 쿼리는 동일한 정렬을 수행한다.
이 예제에서 2번째 칼럼은 last_name이므로 ORDER BY 2는 ORDER BY last_name과 같은 의미가 된다.

```
mysql> SELECT first_name, last_name FROM employees
       ORDER BY last_name;

mysql> SELECT first_name, last_name FROM employees
       ORDER BY 2;
```

하지만 다음과 같이 ORDER BY 뒤에 숫자 값이 아닌 문자열 상수를 사용하는 경우에는 옵티마이저가 ORDER BY 절 자체를 무시한다.
칼럼명이라 하더라도 다음 쿼리와 같이 따옴표를 이용해 문자 리터럴로 표시하면 상숫값으로 정렬하라는 의미가 된다.
상숫값으로 정렬을 수행하는 것은 아무런 의미가 없으므로 옵티마이저는 이렇게 문자 리터럴이 ORDER BY 절에 사용되면 모두 무시한다.

```
mysql> SELECT first_name, last_name FROM employees
       ORDER BY "last_name";
```

다른 DBMS에서 쌍따옴표는 식별자를 표현하기 위해 사용하지만 MySQL에서 쌍따옴표는 문자열 리터럴을 표현한느 데 사용된다.
다른 DBMS에 익숙한 사용자라면 위와 같이 ORDER BY 절을 작성하면 last_name 칼럼으로 정렬될 것이라고 생각할 수도 있다.
하지만 MySQL의 기본 모드(sql_mode 시스템 변수의 기본 설정)에서 쌍따옴표는 문자열 리터럴로 인식된다. 결국 위의 쿼리는 문자열 리터럴 값으로 정렬하라는 의미가 된다.

### [2] 여러 방향으로 동시 정렬
MySQL 8.0 이전 버전까지는 여러 개의 칼럼을 조합해서 정렬할 때 각 칼럼의 정렬 순서가 오름차순과 내림차순이 혼용되면 인덱스를 이용할 수 없다.
하지만 MySQL 8.0 버전부터는 다음 예제와 같이 오름차순과 내림차순을 혼용해서 인덱스를 생성할 수 있게 개선됐다.

```
mysql> ALTER TABLE salaries ADD INDEX ix_salary_fromdate (salary DESC, from_date ASC);
```

응용 프로그램에서 오름차순과 내림차순을 혼용해서 정렬하고자 하는 경우에는 위 예제와 같이 ASC와 DESC 옵션을 섞어서 하나의 인덱스를 생성하면 된다.
그런데 응용 프로그램에서 다음 에제 쿼리와 같이 내림차순으로만 조회하는 경우를 한번 가정해보자.

```
mysql> SELECT * FROM salaries
       ORDER BY salary DESC LIMIT 10;
```

다음 2개의 인덱스 중 하나만 있어도 옵티마이저는 위 쿼리가 적절히 인덱스를 이용해서 정렬할 수 있게 최적화할 수 있다.

```
mysql> ALTER TABLE salaries ADD INDEX ix_salary_asc  (salary ASC );
mysql> ALTER TABLE salaries ADD INDEX ix_salary_desc (salary DESC);
```

하지만 쿼리가 내림차순으로만 레코드를 정렬해서 가져간다면 인덱스는 당연히 ix_salary_desc를 생성하는 것이 좋다.

### [3] 함수나 표현식을 이용한 정렬
하나 또는 여러 칼럼의 연산 결과를 이용해 정렬하는 것도 가능하다.
MySQL 8.0 이전까지는 연산의 결과를 기준으로 정렬하기 위해서는 가상 칼럼(Virtual Column)을 추가하고 인덱스를 생성하는 방법을 사용해야 했다.
하지만 MySQL 8.0 버전부터는 함수 기반의 인덱스를 지원하기 시작했다. 그래서 다음과 같이 연산의 결괏값을 기준으로 정렬하는 작업이 인덱스를 사용하도록 튜닝하는 것이 가능해졌다.

```
mysql> SELECT *
       FROM salaries
       ORDER BY COS(salary);
```
<br/>

## (10) 서브쿼리
쿼리를 작성할 때 서브쿼리를 사용하면 단위 처리별로 쿼리를 독립적으로 작성할 수 있다. 조인처럼 여러 테이블을 섞어 두는 형태가 아니어서 쿼리의 가독성도 높아지며, 복잡한 쿼리도 손쉽게 작성할 수 있다.
MySQL 5.6 버전까지는 서브쿼리를 최적으로 실행하지 못할 때가 많았지만 MySQL 8.0 버전부터는 서브쿼리 처리가 많이 개선됐다.

서브쿼리는 쿼리의 여러 위치에서 사용될 수 있는데, 대표적으로 SELECT 절과 FROM 절, WHERE 절에 사용될 수 있다.
하지만 사용되는 위치에 따라 쿼리의 성능 영향도와 MySQL 서버의 최적화 방법은 완전히 달라진다.
서브쿼리가 사용되는 위치별로 어떻게 최적화되는지, 그리고 어떻게 쿼리를 작성해야 성능에 도움이 될지를 살펴보자.

### [1] SELECT 절에 사용된 서브쿼리
SELECT 절에 사용된 서브쿼리는 내부적으로 임시 테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들지는 않기 때문에 서브쿼리가 적절히 인덱스를 사용할 수 있다면 크게 주의할 사항은 없다.

일반적으로 SELECT 절에 서브쿼리를 사용하면 그 서브쿼리는 항상 칼럼과 레코드가 하나인 결과를 반환해야 한다.
그 값이 NULL이든 아니든 관계없이 레코드가 1건이 존재해야 한다는 것인데, MySQL에서는 이 체크 조건이 조금은 느슨하다. 다음 예제로 한번 살펴보자.

```
mysql> SELECT emp_no, (SELECT dept_name FROM departments WHERE dept_name='Sales1')
       FROM dept_emp LIMIT 10;
+--------+-----------+
| 100001 | NULL      |
| 100003 | NULL      |
| 100004 | NULL      |
| 100005 | NULL      |
| 100006 | NULL      |
| 100007 | NULL      |
| 100008 | NULL      |
| 100009 | NULL      |
| 100010 | NULL      |
+--------+-----------+

mysql> SELECT emp_no, (SELECT dept_name FROM departments)
       FROM dept_emp LIMIT 10;
ERROR 1242 (21000): Subquery returns more than 1 row

mysql> SELECT emp_no, (SELECT dept_no, dept_name FROM departments WHERE dept_name='Sales1')
       FROM dept_emp LIMIT 10;
ERROR 1241 (21000): Operand should contain 1 column(s)
```

위 예제의 각 쿼리에서 주의할 점을 살펴보자.

- 첫 번째 쿼리에서 사용된 서브쿼리는 항상 결과가 0건이다. 하지만 첫 번째 쿼리는 에러를 발생하지 않고, 서브쿼리의 결과는 NULL로 채워져서 반환된다.
- 두 번째 쿼리에서 서브쿼리가 2건 이상의 레코드를 반환하는 경우에는 에러가 나면서 쿼리가 종료된다.
- 세 번째 쿼리와 같이 SELECT 절에 사용된 서브쿼리가 2개 이상의 칼럼을 가져오려고 할 때도 에러가 발생한다.

즉, SELECT 절의 서브쿼리에는 로우 서브쿼리를 사용할 수 없고, 오로지 스칼라 서브쿼리만 사용할 수 있다.

> 서브쿼리는 만들어 내는 결과에 따라 스칼라 서브쿼리(Scalar subquery)와 로우 서브쿼리(Row 또는 Record, 매뉴얼에서는 "Row subquery"로 소개하고 있음)로 구분할 수 있다.
> 스칼라 서브쿼리는 레코드의 칼럼이 각각 하나인 결과를 만들어내는 서브쿼리며, 스칼라 서브쿼리보다 레코드 건수가 많거나 칼럼 수가 많은 결과를 만들어 내는 서브쿼리를 로우 서브쿼리 또는 레코드
> 서브쿼리라고 한다.

가끔 조인으로 처리해도 되는 쿼리를 SELECT 절의 서브쿼리를 사용해서 작성할 때도 있다.
하지만 서브쿼리로 실행될 때보다 조인으로 처리할 때가 조금 더 빠르기 때문에 가능하다면 조인으로 쿼리를 작성하는 것이 좋다. 다음 예제를 한번 살펴보자.

```
mysql> SELECT
         COUNT(CONCAT(e1.first_name,
                     (SELECT e2.first_name FROM employees e2 WHERE e2.emp_no=e1.emp_no))
              ) FROM employees e1;

mysql> SELECT COUNT(CONCAT(e1.first_name, e2.first_name))
       FROM employees e1, employees e2
       WHERE e1.emp_no=e2.emp_no;
```

위의 두 예제 쿼리 모두 employees 테이블을 두 번씩 프라이머리 키를 이용해 참조하는 쿼리다. 물론 위 emp_no는 프라이머리 키라서 조인이나 서브쿼리 중 어떤 방식을 사용해도 같은 결과를 가져온다.
서브쿼리를 사용한 첫 번째 쿼리는 평균 0.78초가 걸렸지만, 조인을 사용한 두 번째 쿼리는 평균 0.65초가 걸렸다.
처리해야 하는 레코드 건수가 많아지면 많아질수록 성능 차이가 커질 수도 있으므로 가능하면 조인으로 쿼리를 작성하는 방법을 권장한다.

그리고 가끔 다음 예제와 같이 SELECT 절에 서브쿼리가 사용되는 경우에 동일한 서브쿼리가 여러 번 사용되기도 한다.

```
mysql> SELECT e.emp_no, e.first_name,
         (SELECT s.salary FROM salaries s
           WHERE s.emp_no=e.emp_no
           ORDER BY s.from_date DESC LIMIT 1) AS salary,
         (SELECT s.from_date FROM salaries s
           WHERE s.emp_no=e.emp_no
           ORDER BY s.from_date DESC LIMIT 1) AS salary_from_date,
         (SELECT s.to_date FROM salaries s
           WHERE s.emp_no=e.emp_no
           ORDER BY s.from_date DESC LIMIT 1) AS salary_to_date
       FROM employees e
       WHERE e.emp_no=499999;
```

이 쿼리의 경우, 서브쿼리에 "LIMIT 1" 조건 때문에 salaries 테이블을 조인으로 사용할 수가 없었다.
하지만 MySQL 8.0 버전부터 도입된 래터럴 조인을 이용하면 이렇게 동일한 레코드의 각 칼럼을 가져오기 위해서 서브쿼리를 3번씩이나 남용하지 않아도 된다.

```
mysql> SELECT e.emp_no, e.first_name,
              s2.salary, s2.from_date, s2.to_date
       FROM employees e
         INNER JOIN LATERAL (
           SELECT * FROM salaries s
           WHERE s.emp_no=e.emp_no
           ORDER BY s.from_date DESC
           LIMIT 1) s2 ON s2.emp_no=e.emp_no
       WHERE e.emp_no=499999;
```

3번의 서브쿼리를 하나의 래터럴 조인으로 변경했기 때문에 이제 salaries 테이블을 한 번만 읽어서 쿼리를 처리할 수 있다.

> 안타깝게도 MySQL 8.0의 래터럴 조인은 한 가지 문제점이 있다. 다음 표는 서브쿼리를 사용한 쿼리와 래터럴 조인을 사용한 쿼리를 실행했을 때 MySQL 서버의 상태 값 변화를 확인해본 결과다.
> 상태 값 수집은 다음과 같이 실행했다.
>
> ```
> mysql> FLUSH STATUS;
>
> mysql> -- // 각 쿼리 실행
> mysql> SHOW STATUS LIKE 'Handler_%';
> ```
>
> <img src="https://github.com/user-attachments/assets/68b13430-3f76-4f85-b4cb-1dd0f4cf99eb" width="450"/><br/>
>
> 래터럴 조인은 내부적으로 임시 테이블을 생성하기 때문에 Handler_write 값과 Handler_read_next 값이 증가할 수 있다.
> 그리고 서브쿼리를 사용한 경우는 salaries 테이블을 여러 번 읽기 때문에 Handler_read_key 값이 증가할 수 있는 실행 계획이 맞다.
> 하지만 래터럴 조인을 사용한 쿼리에서 Handler_read_next 값이 6이 된 것은 잘못된 것이다. 서브쿼리를 사용한 경우와 래터럴 조인을 사용한 경우의 쿼리 실행 계획을 한번 비교해보자.
>
> #### [서브커리를 사용한 쿼리의 실행 계획]
> ```
> +----+--------------------+-------+-------+---------+------+----------------------------------+
> | id | select_type        | table | type  | key     | rows | Extra                            |
> +----+--------------------+-------+-------+---------+------+----------------------------------+
> |  1 | PRIMARY            | e     | const | PRIMARY |    1 | NULL                             |
> |  4 | DEPENDENT SUBQUERY | s     | ref   | PRIMARY |    5 | Backward index scan              |
> |  3 | DEPENDENT SUBQUERY | s     | ref   | PRIMARY |    5 | Backward index scan; Using index |
> |  2 | DEPENDENT SUBQUERY | s     | ref   | PRIMARY |    5 | Backward index scan              |
> +----+--------------------+-------+-------+---------+------+----------------------------------+
> ```
>
> #### [래터럴 조인을 사용한 쿼리의 실행 계획]
> ```
> +----+-------------------+------------+-------+-------------+------+----------------------------+
> | id | select_type       | table      | type  | key         | rows | Extra                      |
> +----+-------------------+------------+-------+-------------+------+----------------------------+
> |  1 | PRIMARY           | e          | const | PRIMARY     |    1 | Rematerialize (<derived2>) |
> |  1 | PRIMARY           | <derived2> | ref   | <auto_key0> |    1 | NULL                       |
> |  2 | DEPENDENT DERIVED | s          | ref   | PRIMARY     |   10 | Using filesort             |
> +----+-------------------+------------+-------+-------------+------+----------------------------+
> ```
>
> 래터럴 조인을 사용한 경우 "Using filesort"가 표시된 것을 확인할 수 있다.
> 인덱스를 이용해 충분히 정렬된 결과를 가져올 수 있음에도 불구하고 정렬을 실행했으며, 이로 인해 Handler_read_next 값이 6으로 증가했다.
> 이는 MySQL 8.0 버전의 버그로 식별됐는데, 아직 이 버그는 해결되지 않은 상태다.
> 버그가 해결됐는지 알고 싶다면 [MySQL 버그 페이지](https://bugs.mysql.com/bug.php?id=94903)를 참조하자.

### [2] FROM 절에 사용된 서브쿼리
이전 버전의 MySQL 서버에서는 FROM 절에 서브쿼리가 사용되면 항상 서브쿼리의 결과를 임시 테이블로 저장하고 필요할 때 다시 임시 테이블을 읽는 방식으로 처리했다.
그래서 가능하면 FROM 절의 서브 외부 쿼리로 병합하는 형태로 쿼리 튜닝을 했다. 하지만 MySQL 5.7 버전부터는 옵티마이저가 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화를 수행하도록 개선됐다.

다음 예제는 MySQL 옵티마이저가 FROM 절에 사용된 서브쿼리를 어떻게 병합했는지, 그리고 병합된 쿼리의 실행 계획이 어떻게 표시되는지를 보여준다.
EXPLAIN 명령을 실행한 후 SHOW WARNINGS 명령을 실행하면 MySQL 서버가 서브쿼리를 병합해서 재작성한 쿼리의 내용을 확인할 수 있다.

```
mysql> EXPLAIN SELECT * FROM (SELECT * FROM employees) y;
+----+-------------+-----------+------+------+--------+-------+
| id | select_type | table     | type | key  | rows   | Extra |
+----+-------------+-----------+------+------+--------+-------+
|  1 | SIMPLE      | employees | ALL  | NULL | 299920 | NULL  |
+----+-------------+-----------+------+------+--------+-------+

mysql> SHOW WARNINGS \G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select
  `employees`.`employees`.`emp_no` AS `emp_no`,
  `employees`.`employees`.`birth_date` AS `birth_date`,
  `employees`.`employees`.`first_name` AS `first_name`,
  `employees`.`employees`.`last_name` AS `last_name`,
  `employees`.`employees`.`gender` AS `gender`,
  `employees`.`employees`.`hire_date` AS `hire_date`
from `employees`.`employees`
```

서브쿼리의 외부 쿼리 병합은 꼭 FROM 절의 서브쿼리에 대해서만 적용되는 최적화는 아니다.
FROM 절에 사용된 뷰(View)의 경우에도 MySQL 옵티마이저는 뷰 쿼리와 외부 쿼리를 병합해서 최적화된 실행 계획을 사용한다.

FROM 절의 모든 서브쿼리를 외부 쿼리로 병합할 수 있는 것은 아니다. 대표적으로 다음과 같은 기능이 서브쿼리에 사용되면 FROM 절의 서브쿼리는 외부 쿼리로 병합되지 못한다.

- 집합 함수 사용(SUM(), MIN(), MAX(), COUNT() 등)
- DISTINCT
- GROUP BY 또는 HAVING
- LIMIT
- UNION(UNION DISTINCT) 또는 UNION ALL
- SELECT 절에 서브쿼리가 사용된 경우
- 사용자 변수 사용(사용자 변수에 값이 할당되는 경우)

외부 쿼리와 병합되는 FROM 절의 서브쿼리가 ORDER BY 절을 가진 경우에는 외부 커리가 GROUP BY나 DISTINCT 같은 기능을 사용하지 않는다면 서브쿼리의 정렬 조건을 외부 쿼리로 같이 병합한다.
외부 쿼리에서 GROUP BY나 DISTINCT와 같은 기능이 사용되고 있다면, 서브쿼리의 정렬 작업은 무의미하기 때문에 서브쿼리의 ORDER BY 절은 무시된다.

MySQL 서버에서 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화는 optimizer_switch 시스템 변수로 제어할 수 있다.

### [3] WHERE 절에 사용된 서브쿼리
WHERE 절의 서브쿼리는 SELECT 절이나 FROM 절보다는 다양한 형태(연산자)로 사용될 수 있는데, 크게 다음 3가지로 구분해서 살펴보겠다.
이는 MySQL 옵티마이저가 최적화하는 형태를 기준으로 구분해본 것이다.

- 동등 또는 크다 작다 비교(= (subquery))
- IN 비교(IN (subquery))
- NOT IN 비교(NOT IN (subquery))

#### ◼︎ 동등 또는 크다 작다 비교
MySQL 5.5 이전 버전까지는 서브쿼리 외부의 조건으로 쿼리를 실행하고, 최종적으로 서브쿼리를 체크 조건으로 사용했다.
하지만 이러한 처리 방식의 경우 풀 테이블 스캔이 필요한 경우가 많아서 성능 저하가 심각했다. 다음 예제 쿼리를 한번 살펴보자.

```
mysql> SELECT * FROM dept_emp de
       WHERE de.emp_no=(SELECT e.emp_no
                        FROM employees e
                        WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
```

MySQL 5.5 이전 버전까지는 위 쿼리의 경우 dept_emp 테이블을 풀 스캔하면서 서브쿼리의 조건에 일치하는지 여부를 체크했다. 하지만 이는 많은 사용자가 기대하는 실행 순서는 아니었을 것이다.

MySQL 5.5 버전부터는 이 쿼리의 실행 계획은 그 이전 버전과는 정반대로 실행되도록 개선됐다. 서브쿼리를 먼저 실행한 후 상수로 변환한다.
그리고 상숫값으로 서브쿼리를 대체해서 나머지 쿼리 부분을 처리한다. 그래서 위 쿼리의 경우 다음과 같은 실행 계획을 사용한다.

```
+----+-------------+-------+------+-------------------+------+-------------+
| id | select_type | table | type | key               | rows | Extra       |
+----+-------------+-------+------+-------------------+------+-------------+
|  1 | PRIMARY     | de    | ref  | ix_empno_fromdate |    1 | Using where |
|  2 | SUBQUERY    | e     | ref  | ix_firstname      |  253 | Using where |
+----+-------------+-------+------+-------------------+------+-------------+
```

위의 실행 계획에 의하면 dept_emp 테이블을 풀 스캔하지 않고 (emp_no, from_date) 조합의 인덱스를 사용했다는 것을 알 수 있다.
이를 위해서는 서브쿼리 부분이 먼저 처리돼야 한다는 것도 예측할 수 있다. 조금 더 명확히 순서를 확인해보기 위해서는 "FORMAT=TREE" 옵션을 추가해서 쿼리의 실행 계획을 확인해보면 된다.

```
mysql> EXPLAIN FORMAT=TREE
       SELECT * FROM dept_emp de
       WHERE de.emp_no=(SELECT emp_no
                        FROM employees e
                        WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);

-> Filter: (de.emp_no = (select #2))  (cost=1.10 rows=1)
  -> Index lookup on de using ix_empno_fromdate (emp_no=(select #2))  (cost=1.10 rows=1)
  -> Select #2 (subquery in condition: run only once)
    -> Limit: 1 row(s)
      -> Filter: (e.last_name = 'Facello')  (cost=70.49 rows=25)
        -> Index lookup on e using ix_firstname (first_name='Georgi')  (cost=70.49 rows=253)
```

위의 TREE 포맷의 실행 계획을 보면 제일 하단의 제일 안쪽 "Index lookup on e using ix_firstname" 라인을 확인할 수 있다
즉, employees 테이블의 ix_firstname 인덱스로 서브쿼리를 처리한 후, 그 결과를 이용해 dept_emp 테이블의 ix_empno_fromdate 인덱스를 검색해 쿼리가 완료된다는 것을 의미한다.

여기서는 동등 비교만 예시로 살펴봤지만 동등 비교 대신 크다 또는 작다 비교가 사용돼도 동일한 실행 계획을 사용한다.

다음과 같이 단일 값 비교가 아닌 튜플 비교 방식이 사용되면 서브쿼리가 먼저 처리되어 상수화되긴 하지만 외부 쿼리는 인덱스를 사용하지 못하고 풀 테이블 스캔을 실행하는 것을 확인할 수 있다.
MySQL 8.0 버전이라고 하더라도 아직 튜플 형태의 비교는 주의해서 사용해야 한다.

```
mysql> EXPLAIN
       SELECT *
       FROM dept_emp de WHERE (emp_no, from_date) = (
             SELECT emp_no, from_date
             FROM salaries
             WHERE emp_no=100001 limit 1);

+----+-------------+----------+------+---------+--------+-------------+
| id | select_type | table    | type | key     | rows   | Extra       |
+----+-------------+----------+------+---------+--------+-------------+
|  1 | PRIMARY     | de       | ALL  | NULL    | 331143 | Using where |
|  2 | SUBQUERY    | salaries | ref  | PRIMARY |      4 | Using index |
+----+-------------+----------+------+---------+--------+-------------+
```

#### ◼︎ IN 비교(IN (subquery))
실제 조인은 아니지만 다음 예제와 같이 테이블의 레코드가 다른 테이블의 레코드를 이용한 표현식(또는 칼럼 그 자체)과 일치하는지를 체크하는 형태를 세미 조인(Semi-Join)이라고 한다.
즉 WHERE 절에 사용된 IN (subquery) 형태의 조건을 조인의 한 방식인 세미 조인이라고 보는 것이다.

```
mysql> SELECT *
       FROM employees e
       WHERE e.emp_no IN
         (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```

MySQL 5.5 버전까지는 세미 조인의 최적화가 매우 부족해서 대부분 풀 테이블 스캔을 했다. 그래서 이런 세미 조인 형태는 MySQL 서버에서 사용하면 안 되는 패턴으로 기억하는 사용자가 많을 것이다.
하지만 MySQL 5.6 버전부터 8.0 버전까지 세미 조인의 최적화가 많이 개선되면서 이제 더 이상은 IN (subquery) 형태를 2개의 쿼리로 쪼개어 실행하거나 다른 우회 방법을 찾을 필요가 없어졌다.

MySQL 서버의 세미 조인 최적화는 쿼리 특성이나 조인 관계에 맞게 다음과 같이 5개의 최적화 전략을 선택적으로 사용한다.

- 테이블 풀-아웃(Table Pull-out)
- 퍼스트 매치(Firstmatch)
- 루스 스캔(Loosescan)
- 구체화(Materialization)
- 중복 제거(Duplicated Weed-out)

MySQL 8.0을 사용한다면 세미 조인 최적화에 익숙해져야 한다.
예전처럼 불필요하게 쿼리를 여러 조각으로 분리해서 실행하는 습관은 버리고, MySQL 8.0의 기능을 적극 활용해 개발 생산성을 높이는 방향을 추천한다.

#### ◼︎ NOT IN 비교(NOT IN (subquery))
IN (subquery)와 비슷한 형태지만 이 경우를 안티 세미 조인(Anti Semi-Join)이라고 명명한다.
일반적인 RDBMS에서 Not-Equal 비교(<> 연산자)는 인덱스를 제대로 활용할 수 없듯이 안티 세미 조인 또한 최적화할 수 있는 방법이 많지 않다.
MySQL 옵티마이저는 안티 세미 조인 쿼리가 사용되면 다음 두 가지 방법으로 최적화를 수행한다.

- NOT EXISTS
- 구체화(Materialization)

두 가지 최적화 모두 그다지 성능 향상에 도움이 되지 않는 방법이므로 쿼리가 최대한 다른 조건을 활용해서 데이터 검색 범위를 좁힐 수 있게 하는 것이 좋다.
WHERE 절에 단독으로 안티 세미 조인 조건만 있다면 풀 테이블 스캔을 피할 수 없으니 주의하자.
<br/>
<br/>
## (11) CTE(Common Table Expression)
CTE(Common Table Expression)는 이름을 가지는 임시 테이블로서, SQL 문장 내에서 한 번 이상 사용될 수 있으며 SQL 문장이 종료되면 자동으로 CTE 임시 테이블은 삭제된다.
CTE는 재귀적 반복 실행 여부를 기준으로 Non-recursive와 Recursive CTE로 구분된다. MySQL 서버의 CTE는 재귀 여부에 관계없이 다음과 같이 다양한 SQL 문장에서 사용할 수 있다.

- SELECT, UPDATE, DELETE 문장의 제일 앞쪽
  - WITH cte1 AS (SELECT ...) SELECT ...
  - WITH cte1 AS (SELECT ...) UPDATE ...
  - WITH cte1 AS (SELECT ...) DELETE ...

- 서브쿼리의 제일 앞쪽
  - SELECT ... FROM ... WHERE id IN (WITH cte1 AS (SELECT ...) SELECT ...) ...
  - SELECT ... FROM (WITH cte1 AS (SELECT ...) SELECT ...)

- SELECT 절의 바로 앞쪽
  - INSERT ... WITH cte1 AS (SELECT ...) SELECT ...
  - REPLACE ... WITH cte1 AS (SELECT ...) SELECT ...
  - CREATE TABLE ... WITH cte1 AS (SELECT ...) SELECT ...
  - CREATE VIEW ... WITH cte1 AS (SELECT ...) SELECT ...
  - DECLARE CURSOR ... WITH cte1 AS (SELECT ...) SELECT ...
  - EXPLAIN ... WITH cte1 AS (SELECT ...) SELECT ...

여기서는 재귀 실행도지 않는 CTE를 간단히 살펴보고, 많은 MySQL 사용자가 기다려왔던 재귀적 CTE를 살펴보겠다.

### [1] 비 재귀적 CTE(Non-Recursive CTE)
MySQL 서버에서는 ANSI 표준을 그대로 이용해서 WITH 절을 이용해 CTE를 정의한다. 다음 쿼리는 CTE를 사용하는 예제 쿼리다.

```
mysql> WITH cte1 AS (SELECT * FROM departments)
       SELECT * FROM cte1;
```

CTE 쿼리는 WITH 절로 정의하고, CTE 쿼리로 생성되는 임시 테이블의 이름은 WITH 바로 뒤에 "cte1"로 정의했다.
이 쿼리에서 cte1 임시 테이블은 한 번만 사용되기 때문에 다음 쿼리처럼 FROM 절의 서브쿼리로 바꿔 사용할 수 있다. 실제 두 쿼리는 실행 계획까지 동일하게 사용된다.

```
mysql> SELECT *
       FROM (SELECT * FROM departments) cte1;
```

CTE는 다음 쿼리와 같이 여러 개의 임시 테이블을 하나의 쿼리에서 사용할 수도 있다.

```
mysql> WITH cte1 AS (SELECT * FROM departments),
            cte2 AS (SELECT * FROM dept_emp)
       SELECT *
       FROM cte1
         INNER JOIN cte2 ON cte2.dept_no=cte1.dept_no;
```

물론 이렇게 여러 개의 CTE 임시 테이블을 사용하는 쿼리도 FROM 절의 서브쿼리(Derived Table)로 대체해서 사용할 수 있다.
하지만 다음과 같이 임시 테이블이 여러 번 사용되는 쿼리는 둘의 실행 계획이 조금 달라진다.

#### [CTE를 이용한 쿼리의 실행 계획]
```
mysql> EXPLAIN
       WITH cte1 AS (SELECT emp_no, MIN(from_date) FROM salaries GROUP BY emp_no)
       SELECT * FROM employees e
         INNER JOIN cte1 t1 ON t1.emp_no=e.emp_no
         INNER JOIN cte1 t2 ON t2.emp_no=e.emp_no;

+----+-------------+------------+--------+-------------+--------+--------------------------+
| id | select_type | table      | type   | key         | rows   | Extra                    |
+----+-------------+------------+--------+-------------+--------+--------------------------+
|  1 | PRIMARY     | <derived2> | ALL    | NULL        | 273035 | NULL                     |
|  1 | PRIMARY     | e          | eq_ref | PRIMARY     |      1 | NULL                     |
|  1 | PRIMARY     | <derived2> | ref    | <auto_key0> |     10 | NULL                     |
|  2 | DERIVED     | salaries   | range  | PRIMARY     | 273035 | Using index for group-by |
+----+-------------+------------+--------+-------------+--------+--------------------------+
```

#### [FROM 절의 서브쿼리를 이용한 경우]
```
mysql> EXPLAIN
       SELECT * FROM employees e
         INNER JOIN (SELECT emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t1
               ON t1.emp_no=e.emp_no
         INNER JOIN (SELECT emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t2
               ON t2.emp_no=e.emp_no;

+----+-------------+------------+--------+-------------+--------+--------------------------+
| id | select_type | table      | type   | key         | rows   | Extra                    |
+----+-------------+------------+--------+-------------+--------+--------------------------+
|  1 | PRIMARY     | <derived2> | ALL    | NULL        | 273035 | NULL                     |
|  1 | PRIMARY     | e          | eq_ref | PRIMARY     |      1 | NULL                     |
|  1 | PRIMARY     | <derived3> | ref    | <auto_key0> |     10 | NULL                     |
|  3 | DERIVED     | salaries   | range  | PRIMARY     | 273035 | Using index for group-by |
|  2 | DERIVED     | salaries   | range  | PRIMARY     | 273035 | Using index for group-by |
+----+-------------+------------+--------+-------------+--------+--------------------------+
```

CTE를 이용한 쿼리에서는 salaries 테이블을 이용한 cte1 임시 테이블(<derived2>)을 한 번만 생성하지만 FROM 절의 서브쿼리를 이용한 쿼리에서는 2개의 임시 테이블(<derived2>와
<derived3>)을 생성하기 위해서 각 서브쿼리에서 salaries 테이블을 읽었다는 것을 알 수 있다.

그리고 CTE로 생성된 임시 테이블은 다른 CTE 쿼리에서 참조할 수 있다는 장점도 있다.

```
WITH
  cte1 AS (SELECT emp_no, MIN(from_date) as salary_from_date
           FROM salaries
           WHERE salary BETWEEN 50000 AND 51000
           GROUP BY emp_no),
  cte2 AS (SELECT de.emp_no, MIN(from_date) as dept_from_date
           FROM cte1
             INNER JOIN dept_emp de ON de.emp_no=cte1.emp_no
           GROUP BY emp_no)
SELECT * FROM employees e
  INNER JOIN cte1 t1 ON t1.emp_no=e.emp_on
  INNER JOIN cte2 t2 ON t2.emp_no=e.emp_no;
```

위의 예제 쿼리에서는 2개의 CTE 임시 테이블이 정의됐는데, cte2 테이블의 정의는 직전에 정의된 cte1 테이블을 이용해서 조인하도록 정의돼 있다.
이렇게 WITH 절에 정의된 순서대로 CTE 임시 테이블은 재사용될 수 있는데, WITH 절에서 뒤에 정의된 cte2 테이블을 먼저 선언된 cte1의 CTE 쿼리에서는 참조할 수 없다.

CTE를 재귀적으로 사용하지 않더라도 기존 FROM 절에 사용되던 서브쿼리에 비해 다음의 3가지 장점이 있다.

- CTE 임시 테이블은 재사용 가능하므로 FROM 절의 서브쿼리보다 효율적이다.
- CTE로 선언된 임시 테이블을 다른 CTE 쿼리에서 참조할 수 있다.
- CTE는 임시 테이블의 생성 부분과 사용 부분의 코드를 분리할 수 있으므로 가독성이 높다.

### [2] 재귀적 CTE(Recursive CTE)
재귀 쿼리는 아마도 많은 MySQL 사용자가 기다리던 기능일 것이다. 데이터베이스 업계에서는 윈백(Win Back) 프로젝트라는 말을 자주 듣는다.(예를 들어, 지금까지는 오라클 RDBMS나 PostgreSQL을
사용하다가 MySQL 서버가 조금씩 좋아지고 비용도 저렴해서 기존 DBMS를 MySQL로 마이그레이션하는 작업을 윈백 프로젝트라고 한다.
정확히 윈백(Win Back)이라는 단어는 처음 프로젝트를 시작할 때는 MySQL 서버를 검토했다가 MySQL 서버의 기능이나 처리 성능이 부족해서 다른 DBMS를 선택했지만 시간이 지나서 다시 MySQL 서버로
되돌아가는 것을 의미한다. 아무튼 특정 DBMS에서 다른 DBMS로 일괄 마이그레이션하는 작업을 주로 윈백 프로젝트라고 한다.)
윈백 프로젝트에서 많은 사용자가 특정 기능이 없어서 MySQL 서버를 사용하기 어렵다고 이야기했는데, 그중에서 가장 대표적인 기능이 재귀 쿼리였다.
MySQL 8.0 버전에서야 비로소 CTE를 이용한 재귀 쿼리가 가능해졌다. 우선 재귀적으로 사용되는 CTE를 사용한 예제 쿼리를 먼저 살펴보자.

```
mysql> WITH RECURSIVE cte (no) AS (
         SELECT 1
         UNION ALL
         SELECT (no + 1) FROM cte WHERE no < 5
       )
       SELECT * FROM cte;

+------+
| no   |
+------+
|    1 |
|    2 |
|    3 |
|    4 |
|    5 |
+------+
5 rows in set (0.00 sec)
```

비 재귀적 CTE는 단순히 쿼리를 한 번만 실행해 그 결과를 임시 테이블로 저장한다.
재귀적 CTE 쿼리는 비 재귀적 쿼리 파트와 재귀적 파트로 구분되며, 이 둘을 UNION(UNION DISTINCT) 또는 UNION ALL로 연결하는 형태로 반드시 쿼리를 작성해야 한다.
위의 예제에서 UNION ALL 위쪽의 "SELECT 1"은 비 재귀적 파트이며, UNION ALL 아래의 "SELECT (no + 1) FROM cte WHERE no < 5"는 재귀적 파트다.
비 재귀적 파트는 처음 한 번만 실행되지만 재귀적 파트는 쿼리 결과가 없을 때까지 반복 실행된다.

위의 예제 쿼리가 작동하는 방법은 다음과 같다.

- [1] CTE 쿼리의 비 재귀적 파트의 쿼리를 실행
- [2] [1]번의 결과를 이용해 cte라는 이름의 임시 테이블 생성
- [3] [1]번의 결과를 cte라는 임시 테이블에 저장
- [4] [1]번 결과를 입력으로 사용해 CTE 쿼리의 재귀적 파트의 쿼리를 수행
- [5] [4]번의 결과를 cte라는 임시 테이블에 저장(이때 UNION 또는 UNION DISTINCT의 경우 중복 제거를 실행)
- [6] 전 단계의 결과를 입력으로 사용해 CTE 쿼리의 재귀적 파트 쿼리를 실행
- [7] [6]번 단계에서 쿼리 결과가 없으면 CTE 쿼리를 종료
- [8] [6]번의 결과를 cte라는 임시 테이블에 저장
- [9] [6]번으로 돌아가서 반복 실행

[1]번 과정에서 매우 중요한 부분이 결정되는데, 바로 CTE 임시 테이블의 구조가 그것이다.
CTE 임시 테이블의 구조(테이블의 칼럼명과 칼럼의 데이터 타입)는 CTE 쿼리의 비 재귀적 쿼리 파트의 결과로 결정된다.
예를 들어, 비 재귀적 파트의 결과와 재귀적 파트의 결과에서 칼럼 개수나 칼럼의 타입, 칼럼 이름이 서로 다른 경우 MySQL 서버는 비 재귀적 파트에 정의된 결과를 사용한다.
CTE의 비 재귀적 쿼리 파트는 초기 데이터와 임시 테이블의 구조를 준비하고, 재귀적 쿼리 파트에서는 이후 데이터를 생성해내는 역할을 한다.

재귀적 쿼리 파트를 실행할 때는 지금까지의 모든 단계에서 만들어진 결과 셋이 아니라 직전 단계의 결과만 재귀 쿼리의 입력으로 사용된다.
즉, 예제 쿼리에서 재귀적 쿼리 파트는 "SELECT (no + 1) FROM cte WHERE no < 5"로 작성돼 있는데, 이 문장이 cte 테이블의 모든 레코드를 조회하는 것이 아니라 직전 단계에서 만들어진 결과만을
참조해서 쿼리가 실행된다. 앞의 예제 쿼리에서 재귀적 쿼리 파트가 2번 실행됐다면 cte 임시 테이블은 3건의 레코드(no 칼럼의 값이 1, 2, 3)를 갖게 된다.
이 상태에서 재귀적 쿼리 파트가 3번째 실행되는 시점에는 cte 임시 테이블의 레코드 1건(no 칼럼의 값이 3)만 참조하게 되는 것이다.

각 재귀 실행 차수별로 CTE의 각 쿼리 파트가 실행되기 전 "cte 임시 테이블이 실제 가진 레코드"와 CTE 쿼리 파트의 실행을 위한 입력과 출력은 다음과 같다.

<img src="https://github.com/user-attachments/assets/362df6e4-6b7e-4f74-8295-af3ea8deec8f" width="550"/><br/>

재귀적으로 실행되는 CTE에서 하나 더 주의할 것은 반복 실행의 종료 조건이다. 이 예제에서는 재귀적 파트 쿼리의 WHERE 조건절에 "no < 5"라는 조건이 반복 종료 조건으로 사용됐다.
이 예제를 보면 왠지 모든 재귀 쿼리에는 이런 조건이 필요할 것처럼 보인다. 하지만 실제 재귀 쿼리가 반복을 멈추는 조건은 재귀 파트 쿼리의 결과가 0건일 때까지다.
실제 응용 프로그램의 쿼리에서 사용하는 데이터는 몇 단계까지 재귀적으로 실행돼야 할지 알 수 없는 경우가 더 많다. 부서의 조직도가 대표적이라고 할 수 있다.
조직 관리 프로그램을 개발하는데, 모든 회사의 조직이 항상 5단계로만 구성된다고 보장할 수 없기 때문이다.

> 기본적으로 CTE 쿼리로 만들어지는 임시 테이블의 칼럼 이름과 각 칼럼의 데이터 타입은 비 재귀적 쿼리 파트의 결과를 그대로 차용한다.
> 하지만 CTE 임시 테이블의 칼럼명을 변경하고자 한다면 CTE 별명(Alias) 뒤에 "(...)"를 이용해 새로운 이름을 부여할 수 있다.
> 하지만 칼럼의 이름을 별도로 명시하는 경우에도 각 칼럼의 데이터 타입은 비 재귀적 쿼리 파트의 결과에 의해 결정된다. 이는 재귀적 CTE와 비 재귀적 CTE 모두 적용되는 규칙이다.
>
> 다음의 두 예제에서 첫 번째 예제 쿼리는 departments 테이블의 결과의 칼럼명과 칼럼 타입을 그대로 이용하는 경우이며, 두 번째 예제 쿼리는 칼럼의 이름을 fd1, fd2, fd3로 변경한 경우다.
>
> ```
> mysql> WITH cte1 AS (SELECT * FROM departments)
>        SELECT * FROM cte1;
> +---------+------------------+-----------+
> | dept_no | dept_name        | emp_count |
> +---------+------------------+-----------+
> | d001    | Marketing        |      NULL |
> | d002    | Finance          |      NULL |
> ...
>
> mysql> WITH cte1 (fd1, fd2, fd3) AS (SELECT * FROM departments)
>        SELECT * FROM cte1;
> +------+------------------+------+
> | fd1  | fd2              | fd3  |
> +------+------------------+------+
> | d001 | Marketing        | NULL |
> | d002 | Finance          | NULL |
> ...
> ```

데이터의 오류나 쿼리 작성자의 실수로 재귀적 CTE가 종료 조건을 만족하지 못해서 무한 반복하는 경우도 발생할 수 있다.
이 같은 오류를 막기 위해서 MySQL 서버는 `cte_max_recursion_depth` 시스템 변수를 이용해 최대 반복 실행 횟수를 제한할 수 있다.
cte_max_recursion_depth 시스템 변수의 기본값은 1000인데, 단순히 1부터 시작하는 시리얼 번호를 가지는 임시 테이블을 생성하는 경우를 제외한다면 이 기본값은 너무 큰 편이다.
따라서 가능하면 cte_max_recursion_depth 시스템 변수의 값을 적절히 낮은 값으로 변경하고, 꼭 필요한 쿼리에서만 SET_VAR 힌트를 이용해 해당 쿼리에서만 반복 호출 횟수를 늘리는 방법을 권장한다.

```
-- // 최대 재귀 호출 횟수를 10번으로 제한
mysql> SET cte_max_recursion_depth=10;

-- // 10번 이상 재귀 호출을 실행하는 쿼리 실행 시 에러 발생
mysql> WITH RECURSIVE cte (no) AS (
         SELECT 1 AS no
         UNION ALL
         SELECT (no + 1) AS no FROM cte WHERE no < 1000
       )
       SELECT * FROM cte;

ERROR 3636 (HY000): Recursive query aborted after 11 iterations.
                    Try increasing @@cte_max_recursion_depth to a larger value.
```

10번 이상의 재귀 호출이 필요한 경우에는 SET_VAR 옵티마이저 힌트를 이용해 해당 쿼리에 대해서만 일시적으로 제한을 완화할 수 있다.

```
WITH RECURSIVE cte (no) AS (
  SELECT 1 as no
  UNION ALL
  SELECT (no + 1) as no FROM cte WHERE no < 1000
)
SELECT /*+ SET_VAR(cte_max_recursion_depth=10000) */ * FROM cte;
+------+
| no   |
+------+
|    1 |
|    2 |
|    3 |
...
|  997 |
|  998 |
|  999 |
| 1000 |
+------+
1000 rows in set (0.01 sec)
```

### [3] 재귀적 CTE(Recursive CTE) 활용
지금까지 재귀적 CTE 쿼리가 어떤 원리로 작동하는지를 살펴봤다. 하지만 지금까지 살펴본 예제는 실제 테이블을 사용하는 것이 아니기 때문에 재귀적 쿼리 파트의 내용이 매우 단순한 형태였다.
이제 실제 응용 프로그램에서 사용될 쿼리 예제를 한번 살펴보자.

우선 재귀적 CTE를 활용하기 위해 다음과 같이 테스트용 테이블과 데이터를 준비했다. employees 데이터베이스의 employees 테이블과의 충돌을 막기 위해 test 데이터베이스에 테이블을 생성했다.

```
mysql> CREATE DATABASE test;
mysql> USE test;

mysql> CREATE TABLE test.employees (
         id         INT PRIMARY KEY NOT NULL,
         name       VARCHAR(100) NOT NULL,
         manager_id INT NULL,
         INDEX (manager_id),
         FOREIGN KEY (manager_id) REFERENCES employees (id)
       );

mysql> INSERT INTO test.employees VALUES
         (333,  "Yasmina", NULL),  # CEO (manager_id is NULL)
         (198,  "John",     333),  # John의 idsms 198이며, John의 매니저는 Yasmina
         (692,  "Tarek",    333),
         (29,   "Pedro",    198),
         (4610, "Sarah",     29),
         (72,   "Pierre",    29),
         (123,  "Adil",     692);
```

이제 employees 테이블에서 직원 'Adil'(id=123)의 상위 조직장을 찾는 쿼리를 한번 살펴보자.

```
mysql> WITH RECURSIVE
         managers AS (
           SELECT *, 1 AS lv FROM employees WHERE id=123
           UNION ALL
           SELECT e.*, lv+1 FROM managers m
                      INNER JOIN employees e ON e.id=m.manager_id AND m.manager_id IS NOT NULL
         )
       SELECT * FROM managers
       ORDER BY lv DESC;

+------+---------+------------+------+
| id   | name    | manager_id | lv   |
+------+---------+------------+------+
|  333 | Yasmina |       NULL |    3 |
|  692 | Tarek   |        333 |    2 |
|  123 | Adil    |        123 |    1 |
+------+---------+------------+------+
```

위의 예제 쿼리를 보면 지금까지 사용하던 재귀적 쿼리 파트와는 조금 다르게 CTE 임시 테이블인 managers 테이블과 employees 테이블을 조인한다.
지금까지 단순한 숫자만 가져오던 예제에는 별도로 데이터를 읽는 테이블이 없었기 때문에 재귀적 쿼리 파트의 FROM 절에 CTE 임시 테이블만 사용했다.
그리고 위의 예제에서는 조직장의 레벨을 가져오기 위해 employees 테이블이 가진 칼럼 이외에 "lv"라는 칼럼도 추가했으며, 상위 조직장 순서대로 결과를 정렬하기 위해 lv 칼럼을 역순으로 정렬했다.

다음 쿼리는 재귀적 쿼리를 최상위 조직장들부터 시작하게 해서 상위 조직장들이 순서대로 나열된 칼럼을 만드는 예제다.

```
mysql> WITH RECURSIVE
         managers AS (
           SELECT *,
                  CAST(id AS CHAR(100)) AS manager_path,
                  1 as lv
             FROM employees WHERE manager_id IS NULL
           UNION ALL
           SELECT e.*,
                  CONCAT(e.id, '->', m.manager_path) AS manager_path,
                  lv+1
           FROM managers m
           INNER JOIN employees e ON e.manager_id=m.id
         )
       SELECT * FROM managers
       ORDER BY lv ASC;

+------+---------+------------+--------------------------+------+
| id   | name    | manager_id | manager_path             | lv   |
+------+---------+------------+--------------------------+------+
|  333 | Yasmina |       NULL | 333                      |    1 |
|  198 | John    |        333 | 198 -> 333               |    2 |
|  692 | Tarek   |        333 | 692 -> 333               |    2 |
|   29 | Pedro   |        198 | 29 -> 198 -> 333         |    3 |
|  123 | Adil    |        692 | 123 -> 692 -> 333        |    3 |
|   72 | Pierre  |         29 | 72 -> 29 -> 198 -> 333   |    4 |
| 4610 | Sarah   |         29 | 4610 -> 29 -> 198 -> 333 |    4 |
+------+---------+------------+--------------------------+------+
```

위의 예제는 최상위 조직장부터 시작하기 위해 비 재귀적 쿼리 파트의 WHERE 조건절에 "manager_id IS NULL" 조건을 사용했다.
그리고 재귀적 쿼리 파트는 자신의 조직 멤버들을 한 명씩 찾아가면서 "하위조직장 -> 상위조직장" 형태의 칼럼이 추가되게 했다.

위 예제의 비 재귀적 쿼리 파트에서는 CAST() 함수를 이용해 id 칼럼에 대한 타입 변환을 했다. employees 테이블의 id 칼럼이 INT 타입이기 때문에 문자열로 변경한 것이다.
그렇다면 employees 테이블의 id 칼럼이 문자열을 저장하는 VARCHAR(5) 타입이었다면 결과는 어떻게 됐을까?
id 칼럼의 데이터 타입이 VARCHAR라 하더라도 가질 수 있는 문자열의 길이 때문에 다음과 같은 에러가 발생하므로 타입 변환은 여전히 필요하다.
위 예제 쿼리의 결과에서 manager_path 칼럼의 최대 길이는 24글자이기 때문에 VARCHAR(5)로는 길이가 부족하기 때문이다.

```
ERROR 1406 (22001): Data too long for column 'manager_path' at row 1
```

재귀적 쿼리를 활용할 수 있는 업무 요건은 상당히 많다.
대표적으로 단순히 1부터 단조 증가하는 값을 가지는 임시 테이블이나 일 단위로 증가하는 날짜를 가진 임시 테이블을 생성하는 것부터, 조직도 조회 그리고 BOM(Bil Of Material) 쿼리 등에 활용할 수
있다. 재귀적 쿼리 파트는 조금은 혼란스러울 수 있는데, 작동 원리만 이해하면 어렵지 않게 자유자재로 쿼리를 작성할 수 있을 것이다.
재귀적으로 사용되든 아니든 CTE를 사용하는 방법은 자세히 공부해 두면 개발에 많은 도움이 될 것이다.
<br/>
<br/>
## (12) 윈도우 함수(Window Function)
윈도우 함수는 조회하는 현재 레코드를 기준으로 연관된 레코드 집합의 연산을 수행한다.
집계 함수는 주어진 그룹(GROUP BY 절에 나열된 칼럼의 값에 따른 그룹 또는 GROUP BY 절 없이 전체 그룹)별로 하나의 레코드로 묶어서 출력하지만 윈도우 함수는 조건에 일치하는 레코드 건수는 변하지
않고 그대로 유지한다. 이것이 윈도우 함수와 집계 함수의 가장 큰 차이점이라고 할 수 있다.

이는 윈도우 함수의 특징을 이해하기 위한 중요한 내용이므로 조금 더 자세히 살펴보자.
일반적인 SQL 문장에서 하나의 레코드를 연산할 때 다른 레코드의 값을 참조할 수 없는데, 예외적으로 GROUP BY 또는 집계 함수를 이용하면 다른 레코드의 칼럼값을 참조할 수 있다.
하지만 GROUP BY 또는 집계 함수를 사용하면 결과 집합의 모양이 바뀐다. 그에 반해, 윈도우 함수는 결과 집합을 그대로 유지하면서 하나의 레코드 연산에 다른 레코드의 칼럼값을 참조할 수 있다.

### [1] 쿼리 각 절의 실행 순서
윈도우 함수를 사용하는 쿼리의 결과에 보여지는 레코드는 FROM 절과 WHERE 절, GROUP BY와 HAVING 절에 의해 결정되고, 그 이후 윈도우 함수가 실행된다.
그리고 마지막으로 SELECT 절과 ORDER BY 절, LIMIT 절이 실행되어 최종 결과가 반환된다. 아래 그림은 윈도우 함수가 사용된 쿼리의 각 절이 처리되는 순서를 보여준다.

#### [그림 11.10] 쿼리 각 절의 실행 순서
<img src="https://github.com/user-attachments/assets/d436ce89-7e4a-477f-8382-1accc015c164" width="400"/><br/>

쿼리에서 각 절의 실행 순서를 숙지하고 있어야 정확한 쿼리를 작성할 수 있다. 예를 들어, 윈도우 함수를 GROUP BY 칼럼으로 사용하거나 WHERE 절에 사용할 수 없다는 것을 위 그림을 보면 알 수 있다.
그리고 LIMIT을 먼저 실행한 다음 윈도우 함수를 적용할 수 없다는 것을 알 수 있다. 이 순서를 벗어나는 쿼리를 작성하고자 한다면 FROM 절의 서브쿼리(Derived Table)를 사용해야 한다.
다음 예제는 FROM 절의 서브쿼리에서 "LIMIT 5"를 사용한 경우와 FROM 절의 서브쿼리 없이 "LIMIT 5"를 사용한 경우의 쿼리 결과를 보여준다.

```
mysql> SELECT emp_no, from_date, salary,
              AVG(salary) OVER() AS avg_salary
       FROM salaries
       WHERE emp_no=10001
       LIMIT 5;
+--------+------------+--------+------------+
| emp_no | from_date  | salary | avg_salary |
+--------+------------+--------+------------+
|  10001 | 1986-06-26 |  60117 | 75388.9412 |
|  10001 | 1986-06-26 |  62102 | 75388.9412 |
|  10001 | 1986-06-26 |  66074 | 75388.9412 |
|  10001 | 1986-06-26 |  66596 | 75388.9412 |
|  10001 | 1986-06-26 |  66961 | 75388.9412 |
+--------+------------+--------+------------+

mysql> SELECT emp_no, from_date, salary,
              AVG(salary) OVER() AS avg_salary
       FROM (SELECT * FROM salaries WHERE emp_no=10001 LIMIT 5) s2;
+--------+------------+--------+------------+
| emp_no | from_date  | salary | avg_salary |
+--------+------------+--------+------------+
|  10001 | 1986-06-26 |  60117 | 64370.0000 |
|  10001 | 1986-06-26 |  62102 | 64370.0000 |
|  10001 | 1986-06-26 |  66074 | 64370.0000 |
|  10001 | 1986-06-26 |  66596 | 64370.0000 |
|  10001 | 1986-06-26 |  66961 | 64370.0000 |
+--------+------------+--------+------------+
```

위의 두 예제에서 첫 번째 쿼리는 FROM 절의 서브쿼리 없이 "LIMIT 5"를 사용했다.
이는 우선 emp_no=10001 조건에 일치하는 17건의 레코드를 모두 가져온 다음 윈도우 함수(AVG(salary) OVER())를 실행하고, 그 결과에서 5건만 반환한 것이다.
즉 avg_salary 칼럼의 값 75388.9412는 최종 5건의 평균이 아니라 emp_no=10001 조건에 일치하는 17건의 평균인 것이다. 그리고 두 번째 예제의 64370.0000은 최종 5건의 평균이다.

### [2] 윈도우 함수 기본 사용법
윈도우 함수의 기본 사용법은 다음과 같다.

```
AGGREGATE_FUNC() OVER (<partition> <order>) AS window_func_column
```

윈도우 함수는 용도별로 다양한 함수들을 사용할 수 있는데, 집계 함수와는 달리 함수 뒤에 OVER 절을 이용해 연산 대상을 파티션하기 위한 옵션을 명시할 수 있다.
이렇게 OVER 절에 의해 만들어진 그룹을 파티션(Partition) 또는 윈도우(Window)라고 한다.

우선 간단히 직원들의 입사 순서를 조회하는 쿼리를 윈도우 함수를 사용해 한번 작성해보자.

```
mysql> SELECT e.*,
              RANK() OVER (ORDER BY e.hire_date) AS hire_date_rank
       FROM employees e;
+--------+------------+-----------------+-----------------+--------+------------+----------------+
| emp_no | birth_date | first_name      | last_name       | gender | hire_date  | hire_date_rank |
+--------+------------+-----------------+-----------------+--------+------------+----------------+
| 110022 | 1956-09-12 | Magareta        | Markovitch      | M      | 1985-01-01 |              1 |
| 110085 | 1959-10-28 | Ebru            | Alpin           | M      | 1985-01-01 |              1 |
...
| 111692 | 1954-10-05 | Tony            | Butterworth     | F      | 1985-01-01 |              1 |
| 110114 | 1957-03-28 | Isamu           | Legleitner      | F      | 1985-01-14 |             10 |
| 200241 | 1956-06-04 | Jaques          | Kalefeld        | M      | 1985-02-01 |             11 |
...
```

예제 쿼리에서 "RANK() OVER(ORDER BY e.hire_date)"는 소그룹을 별도로 구분하지 않고 전체 결과 집합에서 e.hire_date 칼럼으로 정렬한 후 순위(RANK() 함수)를 매기게 했다.
부서별로 입사 순위를 매기고자 한다면 다음과 같이 부서 코드로 파티션을 하면 된다.

```
mysql> SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date,
              RANK() OVER (PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
       FROM employees e
         INNER JOIN dept_emp de ON de.emp_no=e.emp_no
       ORDER BY de.dept_no, e.hire_date;

+---------+--------+---------------+------------+----------------+
| dept_no | emp_no | first_name    | hire_date  | hire_date_rank |
+---------+--------+---------------+------------+----------------+
| d001    | 110022 | Margareta     | 1985-01-01 |              1 |
| d001    |  51773 | Eric          | 1985-02-02 |              2 |
...
| d001    | 481016 | Toney         | 1985-02-02 |              2 |
| d001    |  70562 | Morris        | 1985-02-03 |             12 |
| d001    | 226633 | Xuejun        | 2000-01-04 |          20211 |
| d002    | 110085 | Ebru          | 1985-01-01 |              1 |
| d002    | 110114 | Isamu         | 1985-01-14 |              2 |
...
```

소그룹 파티션이나 정렬이 필요치 않은 경우 PARTITION이나 ORDER BY 없이 비어 있는 OVER() 절을 사용하면 된다.
다음 예제는 salaries 테이블에서 emp_no가 10001인 사원의 급여 이력과 평균 급여를 조회하는 쿼리다.

```
mysql> SELECT emp_no, from_date, salary,
              AVG(salary) OVER() as avg_salary
       FROM salaries
       WHERE emp_no=10001;
+--------+------------+--------+------------+
| emp_no | from_date  | salary | avg_salary |
+--------+------------+--------+------------+
|  10001 | 1986-06-26 |  60117 | 75388.9412 |
|  10001 | 1987-06-26 |  62102 | 75388.9412 |
|  10001 | 1988-06-25 |  66074 | 75388.9412 |
|  10001 | 1989-06-25 |  66596 | 75388.9412 |
...
```

예제에서는 사원 번호가 10001인 사원의 모든 급여 변경 이력의 평균을 조회하기 위해 OVER() 절의 내용에 PARTITION 키워드를 생략한 것이다.
그리고 평균을 계산하는 것이므로 별도의 정렬도 필요치 않으므로 ORDER BY 키워드도 생략했다. 그래서 결과의 avg_salary 칼럼의 값은 모든 레코드가 동일한 값을 표시한다.

윈도우 함수의 각 파티션 안에서도 연산 대상 레코드별로 연산을 수행할 소그룹이 사용되는데, 이를 프레임이라고 한다.
윈도우 함수에서 프레임을 명시적으로 지정하지 않아도 MySQL 서버는 상황에 맞게 프레임을 묵시적으로 선택한다. MySQL 서버가 묵시적으로 선택하는 프레임에 대해서는 마지막에 다시 살펴보겠다.
프레임은 레코드의 순서대로 현재 레코드 기준 앞뒤 몇 건을 연산 범위로 제한하는 역할을 한다. 프레임은 다음과 같이 정의할 수 있다.

```
AGGREGATE_FUNC() OVER (<partition> <order> <frame>) AS window_func_column

frame:
  {ROWS | RANGE} {frame_start | frame_between}

frame_between:
  BETWEEN frame_start AND frame_end

frame_start, frame_end: {
    CURRENT ROW
  | UNBOUNDED PRECEDING
  | UNBOUNDED FOLLOWING
  | expr PRECEDING
  | expr FOLLOWING
}
```

프레임을 만드는 기준으로 ROWS와 RANGE 중 하나를 선택할 수 있다.

- ROWS: 레코드의 위치를 기준으로 프레임을 생성
- RANGE: ORDER BY 절에 명시된 칼럼을 기준으로 값의 범위로 프레임 생성

프레임의 시작과 끝을 의미하는 키워드들의 의미는 다음과 같다.

- CURRENT ROW: 현재 레코드
- UNBOUNDED PRECEDING: 파티션의 첫 번째 레코드
- UNBOUNDED FOLLOWING: 파티션의 마지막 레코드
- expr PRECEDING: 현재 레코드로부터 n번째 이전 레코드
- expr FOLLOWING: 현재 레코드로부터 n번째 이후 레코드

프레임이 ROWS로 구분되면 expr에는 레코드의 위치를 명시하고, RANGE로 구분되면 expr에는 칼럼과 비교할 값이 설정돼야 한다.
그래서 프레임의 시작과 끝이 expr을 가지는 경우는 다음 예시와 같이 사용될 수 있다.

- 10 PRECEDING: 현재 레코드로부터 10건 이전부터
- INTERVAL 5 DAY PRECEDING: 현재 레코드의 ORDER BY 칼럼값보다 5일 이전 레코드부터
- 5 FOLLOWING: 현재 레코드로부터 5건 이후까지
- INTERVAL '2:30' MINUTE_SECOND FOLLOWING: 현제 레코드의 ORDER BY 칼럼값보다 2분 30초 이후까지

프레임의 사용법이 조금 복잡할 수 있는데, 이해를 돕기 위해 "<frame>" 절만 간단히 예제 몇 가지를 통해서 한번 살펴보자.

- ROWS UNBOUNDED PRECEDING: 파티션의 첫 번째 레코드로부터 현재 레코드까지
- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW: 파티션의 첫 번째 레코드로부터 현재 레코드까지("ROWS UNBOUNDED PRECEDING"와 동일함)
- ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING: 파티션에서 현재 레코드를 기준으로 앞 레코드부터 뒤 레코드까지
- RANGE INTERVAL 5 DAY PRECEDING: ORDER BY에 명시된 칼럼의 값이 5일 전인 레코드부터 현재 레코드까지
- RANGE BETWEEN 1 DAY PRECEDING AND 1 DAY FOLLOWING: ORDER BY에 명시된 칼럼의 값이 1일 전인 레코드부터 1일 이후인 레코드까지

지금까지 프레임 절을 사용하는 방법을 살펴봤는데, 이를 응용해서 다음의 예제 쿼리에서 사용한 프레임 절의 의미와 실제 조회된 값을 한번 비교하면서 살펴보면 조금 더 명확히 이해될 것이다.
아직 프레임 절의 내용이 잘 이해되지 않더라도 윈도우 함수별로 조회되는 값에 대한 코멘트를 추가해 뒀으니 어렵지 않게 이해할 수 있을 것이다.

```
SELECT emp_no, from_date, salary,

  -- // 현재 레코드의 from_date를 기준으로 1년 전부터 지금까지 급여 중 최소 급여
  MIN(salary) OVER (ORDER BY from_date RANGE INTERVAL 1 YEAR PRECEDING) AS min_1,

  -- // 현재 레코드의 from_date를 기준으로 1년 전부터 2년 후까지의 급여 중 최대 급여
  MAX(salary) OVER (ORDER BY from_date
                    RANGE BETWEEN INTERVAL 1 YEAR PRECEDING AND INTERVAL 2 YEAR FOLLOWING) AS max_1,

  -- // from_date 칼럼으로 정렬 후, 첫 번째 레코드부터 현재 레코드까지의 평균
  AVG(salary) OVER (ORDER BY from_date ROWS UNBOUNDED PRECEDING) AS avg_1,

  -- // from_date 칼럼으로 정렬 후, 현재 레코드를 기준으로 이전 건부터 이후 레코드까지의 급여 평균
  AVG(salary) OVER (ORDER BY from_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS avg_2

FROM salaries
WHERE emp_no=10001;

+--------+------------+--------+-------+-------+------------+------------+
| emp_no | from_date  | salary | min_1 | max_1 | avg_1      | avg_2      |
+--------+------------+--------+-------+-------+------------+------------+
|  10001 | 1986-06-26 |  60117 | 60117 | 66074 | 60117.0000 | 61109.5000 |
|  10001 | 1987-06-26 |  62102 | 60117 | 66596 | 61109.5000 | 62764.3333 |
|  10001 | 1988-06-25 |  66074 | 62012 | 66961 | 62764.3333 | 64924.0000 |
|  10001 | 1989-06-25 |  66596 | 66074 | 71046 | 63722.2500 | 66543.6667 |
|  10001 | 1990-06-25 |  66961 | 66596 | 74333 | 64370.0000 | 68201.0000 |
|  10001 | 1991-06-25 |  71046 | 66961 | 75286 | 65482.6667 | 70780.0000 |
|  10001 | 1992-06-24 |  74333 | 71046 | 75994 | 66747.0000 | 73555.0000 |
|  10001 | 1993-06-24 |  75286 | 74333 | 76884 | 67814.3750 | 75204.3333 |
|  10001 | 1994-06-24 |  75994 | 75286 | 80013 | 68723.2222 | 76054.6667 |
|  10001 | 1995-06-24 |  76884 | 75994 | 81025 | 69539.3000 | 77630.3333 |
|  10001 | 1996-06-23 |  80013 | 76884 | 81097 | 70491.4545 | 79307.3333 |
|  10001 | 1997-06-23 |  81025 | 80013 | 84917 | 71369.2500 | 80711.6667 |
|  10001 | 1998-06-23 |  81097 | 81025 | 85112 | 72117.5385 | 82346.3333 |
|  10001 | 1999-06-23 |  84917 | 81097 | 85112 | 73031.7857 | 83708.6667 |
|  10001 | 2000-06-22 |  85112 | 84917 | 88958 | 73837.1333 | 85042.0000 |
|  10001 | 2001-06-22 |  85097 | 85097 | 88958 | 74540.8750 | 86389.0000 |
|  10001 | 2002-06-22 |  88958 | 85097 | 88958 | 75388.9412 | 87027.5000 |
+--------+------------+--------+-------+-------+------------+------------+
```

> 윈도우 함수에서 프레임이 별도로 명시되지 않으면 무조건 파티션의 모든 레코드가 연산의 대상이 되는 것은 아니다. OVER() 절이 ORDER BY를 가지는지 여부에 따라 묵시적인 프레임의 범위가 달라진다.
> OVER()가 ORDER BY를 가지는 경우 파티션의 첫 번째 레코드부터 현재 레코드까지가 프레임이 된다.
> 그리고 OVER() 절이 ORDER BY를 가지지 않으면 파티션의 모든 레코드가 묵시적인 프레임으로 선택된다. 즉 ORDER BY 여부에 따라서 프레임이 다음과 같이 묵시적으로 선택된다.
>
> - ORDER BY 사용 시: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
> - ORDER BY 미 사용 시: RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

일부 윈도우 함수들은 프레임이 미리 고정돼 있다. SQL 문장에서 프레임을 별도로 명시하더라도 이러한 윈도우 함수에서는 사용자가 정의한 프레임은 모두 무시된다.
이 경우 에러는 발생하지 않기 때문에 결과가 혼란스러울 수 있으므로 주의하자. 다음 윈도우 함수들은 자동으로 프레임이 파티션의 전체 레코드로 설정된다.

- CUME_DIST()
- DENSE_RANK()
- LAG()
- LEAD()
- NTILE()
- PERCENT_RANK()
- RANK()
- ROW_NUMBER()

### [3] 윈도우 함수
MySQL 서버의 윈도우 함수에는 집계 함수와 비 집계 함수를 모두 사용할 수 있다.
집계 함수는 GROUP BY 절과 함께 사용할 수 있는 함수들을 의미하는데, 집계 함수는 OVER() 절 없이 단독으로도 사용될 수 있고 OVER() 절을 가진 윈도우 함수로도 사용될 수 있다.
반면 비 집계 함수는 반드시 OVER() 절을 가지고 있어야 하며 윈도우 함수로만 사용될 수 있다.

#### [집계 함수(Aggregate Function)]
|함수|기능|
|:---|:---|
|**AVG()**|평균 값 반환|
|**BIT_AND()**|AND 비트 연산 결과 반환|
|**BIT_OR()**|OR 비트 연산 결과 반환|
|**BIT_XOR()**|XOR 비트 연산 결과 반환|
|**COUNT()**|건수 반환|
|**JSON_ARRAYAGG()**|결과를 JSON 배열로 반환|
|**JSON_OBJECTAGG()**|결과를 JSON OBJECT 배열로 반환|
|**MAX()**|최댓값 반환|
|**MIN()**|최솟값 반환|
|**STDDEV_POP(),<br/>STDDEV(), STD()**|표준 편차 값 반환|
|**STDDEV_SAMP()**|표본 표준 편차 값 반환|
|**SUM()**|합계 값 반환|
|**VAR_POP(), VARIANCE()**|표준 분산 값 반환|
|**VAR_SAMP()**|표본 분산 값 반환|

#### [비 집계 함수(Non-Aggregate Function)]
|함수|기능|
|:---|:---|
|**CUME_DIST()**|누적 분포 값 반환<br/>(파티션별 현재 레코드보다 작거나 같은 레코드의 누적 백분율)|
|**DENSE_RANK()**|랭킹 값 반환(Gap 없음)<br/>(동일한 값에 대해서는 동일 순서를 부여하며, 동일한 순위가 여러 건이어도 한 건으로 취급)|
|**FIRST_VALUE()**|파티션의 첫 번째 레코드 값 반환|
|**LAG()**|파티션 내에서 파라미터(N)를 이용해 N번째 이전 레코드 값 반환|
|**LAST_VALUE()**|파티션의 마지막 레코드 값 반환|
|**LEAD()**|파티션 내에서 파라미터(N)를 이용해 N번째 이후 레코드 값 반환|
|**NTH_VALUE()**|파티션의 n번째 값 반환|
|**NTILE()**|파티션별 전체 건수를 파라미터(N)로 N-등분한 값 반환|
|**PERCENT_RANK()**|퍼센트 랭킹 값 반환|
|**RANK()**|랭킹 값 반환(Gap 있음)|
|**ROW_NUMBER()**|파티션의 레코드 순번 반환|

집계 함수는 이미 GROUP BY와 함께 많이 사용되므로 예제나 설명을 생략하겠다. 그리고 비 집계 함수는 윈도우 함수에서 자주 사용되는 일부 함수들만 살펴보고 나머지는 생략하겠다.
각 윈도우 함수의 자세한 사용법에 대해서는 MySQL 매뉴얼을 참조하자.

#### ◼︎ DENSE_RANK()와 RANK(), ROW_NUMBER()
DENSE_RANK() 함수와 RANK() 함수는 모두 ORDER BY 기준으로 매겨진 순위를 반환한다.
RANK() 함수는 동점인 레코드가 두 건 이상인 경우 그다음 레코드를 동점인 레코드 수만큼 증가시킨 순위를 반환하지만, DENSE_RANK() 함수는 동점인 레코드를 1건으로 가정하고 순위를 매기기 때문에
연속된 순위를 가진다. ROW_NUMBER() 함수는 똑같이 순위를 매기지만 ROW_NUMBER() 함수는 이름 그대로 각 레코드의 고유한 순번을 반환한다.
그래서 ROW_NUMBER() 함수는 동점에 대한 고려 없이 정렬된 순서대로 레코드 번호를 부여한다.

- #### RANK()

  ```
  mysql> SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date,
                RANK() OVER (PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
         FROM employees e
           INNER JOIN dept_emp de ON de.emp_no=e.emp_no
         WHERE de.dept_no='d001'
         ORDER BY de.dept_no, e.hire_date
         LIMIT 20;

  +---------+--------+------------+------------+----------------+
  | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
  +---------+--------+------------+------------+----------------+
  | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
  | d001    |  98351 | Florina    | 1985-02-02 |              2 |
  | d001    | 456487 | Jouko      | 1985-02-02 |              2 |
  ...
  | d001    | 430759 | Fumiko     | 1985-02-02 |              2 |
  | d001    | 447306 | Pranav     | 1985-02-03 |             12 |
  ...
  ```

- #### DENSE_RANK()

  ```
  mysql> SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date
                DENSE_RANK() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
         FROM employees e
           INNER JOIN dept_emp de ON de.emp_no=e.emp_no
         WHERE de.dept_no='d001'
         ORDER BY de.dept_no, e.hire_date
         LIMIT 20;

  +---------+--------+------------+------------+----------------+
  | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
  +---------+--------+------------+------------+----------------+
  | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
  | d001    |  98351 | Florina    | 1985-02-02 |              2 |
  | d001    | 456487 | Jouko      | 1985-02-02 |              2 |
  ...
  | d001    | 430759 | Fumiko     | 1985-02-02 |              2 |
  | d001    | 447306 | Pranav     | 1985-02-03 |              3 |
  ...
  ```

- #### ROW_NUMBER()

  ```
  mysql> SELECT de.dept_no, e.emp_no, e.first_name, e.hire_date
                ROW_NUMBER() OVER(PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
         FROM employees e
           INNER JOIN dept_emp de ON de.emp_no=e.emp_no
         WHERE de.dept_no='d001'
         ORDER BY de.dept_no, e.hire_date
         LIMIT 20;

  +---------+--------+------------+------------+----------------+
  | dept_no | emp_no | first_name | hire_date  | hire_date_rank |
  +---------+--------+------------+------------+----------------+
  | d001    | 110022 | Margareta  | 1985-01-01 |              1 |
  | d001    |  98351 | Florina    | 1985-02-02 |              2 |
  | d001    | 456487 | Jouko      | 1985-02-02 |              3 |
  | d001    | 481016 | Toney      | 1985-02-02 |              4 |
  | d001    | 491200 | Ortrun     | 1985-02-02 |              5 |
  | d001    |  51773 | Eric       | 1985-02-02 |              6 |
  | d001    |  65515 | Phillip    | 1985-02-02 |              7 |
  | d001    |  95867 | Shakhar    | 1985-02-02 |              8 |
  | d001    | 288310 | Mohammed   | 1985-02-02 |              9 |
  | d001    | 288790 | Cristinel  | 1985-02-02 |             10 |
  | d001    | 430759 | Fumiko     | 1985-02-02 |             11 |
  | d001    | 447306 | Pranav     | 1985-02-03 |             12 |
  ...
  ```

#### ◼︎ LAG()와 LEAD()
LAG() 함수는 파티션 내에서 현재 레코드를 기준으로 n번째 이전 레코드를 반환하며, LEAD() 함수는 반대로 n번째 이후 레코드를 반환한다.
LEAD() 함수와 LAG() 함수는 3개의 파라미터를 필요로 하는데, 첫 번째와 두 번째 파라미터는 필수이며, 세 번째 파라미터는 선택 사항이다.

```
mysql> SELECT from_date, salary,
              LAG(salary, 5) OVER (ORDER BY from_date) AS prior_5th_value,
              LEAD(salary, 5) OVER (ORDER BY from_date) AS next_5th_value,
              LAG(salary, 5, -1) OVER (ORDER BY from_date) AS prior_5th_with_default,
              LEAD(salary, 5, -1) OVER (ORDER BY from_date) AS next_5th_with_default
       FROM salaries
       WHERE emp_no=10001;

+------------+--------+-----------------+----------------+------------------------+-----------------------+
| from_date  | salary | prior_5th_value | next_5th_value | prior_5th_with_default | next_5th_with_default |
+------------+--------+-----------------+----------------+------------------------+-----------------------+
| 1986-06-26 |  60117 |            NULL |          71046 |                     -1 |                 71046 |
| 1986-06-26 |  62102 |            NULL |          74333 |                     -1 |                 74333 |
| 1986-06-25 |  66704 |            NULL |          75286 |                     -1 |                 75286 |
| 1986-06-25 |  66596 |            NULL |          75994 |                     -1 |                 75994 |
| 1986-06-25 |  66961 |            NULL |          76884 |                     -1 |                 76884 |
| 1986-06-25 |  71046 |           60117 |          80013 |                  60117 |                 80013 |
| 1986-06-24 |  74333 |           62102 |          81025 |                  62102 |                 81025 |
| 1986-06-24 |  75286 |           66704 |          81097 |                  66704 |                 81097 |
| 1986-06-24 |  75994 |           66596 |          84917 |                  66596 |                 84917 |
| 1986-06-24 |  76884 |           66961 |          85112 |                  66961 |                 85112 |
| 1986-06-23 |  80013 |           71046 |          85097 |                  71046 |                 85097 |
| 1986-06-23 |  81025 |           74333 |          88958 |                  74333 |                 88958 |
| 1986-06-23 |  81097 |           75286 |           NULL |                  75286 |                    -1 |
| 1986-06-23 |  84917 |           75994 |           NULL |                  75994 |                    -1 |
| 1986-06-22 |  85112 |           76884 |           NULL |                  76884 |                    -1 |
| 1986-06-22 |  85097 |           80013 |           NULL |                  80013 |                    -1 |
| 1986-06-22 |  88958 |           81025 |           NULL |                  81025 |                    -1 |
+------------+--------+-----------------+----------------+------------------------+-----------------------+
```

### [4] 윈도우 함수와 성능
MySQL 서버의 윈도우 함수는 8.0 버전에 처음 도입됐으며, 아직 인덱스를 이용한 최적화가 부족한 부분도 있다. 예를 들어, 다음 쿼리를 한번 살펴보자.
다음 두 쿼리는 결과는 차이가 있지만 사용자별로 MAX(from_date) 값을 구하는 쿼리다.

```
mysql> SELECT MAX(from_date) OVER (PARTITION BY emp_no) AS max_from_date
       FROM salaries;

mysql> SELECT MAX(from_date) FROM salaries GROUP BY emp_no;
```

윈도우 함수와 GROUP BY 쿼리는 근본적으로 차이가 있어서 동등한 비교는 어렵지만 그래도 위의 두 쿼리에 대해 실행 계획과 실제 연산에 사용했던 레코드 건수를 한번 비교해보자.

```
-- // 윈도우 함수를 사용하는 쿼리의 실행 계획
+----+-------------+----------+-------+-----------+---------+-----------------------------+
| id | select_type | table    | type  | key       | rows    | Extra                       |
+----+-------------+----------+-------+-----------+---------+-----------------------------+
|  1 | SIMPLE      | salaries | index | ix_salary | 2838663 | Using index; Using filesort |
+----+-------------+----------+-------+-----------+---------+-----------------------------+

-- // GROUP BY 절을 사용하는 쿼리의 실행 계획
+----+-------------+----------+-------+---------+--------+--------------------------+
| id | select_type | table    | type  | key     | rows   | Extra                    |
+----+-------------+----------+-------+---------+--------+--------------------------+
|  1 | SIMPLE      | salaries | range | PRIMARY | 273035 | Using index for group-by |
+----+-------------+----------+-------+---------+--------+--------------------------+
```

윈도우 함수를 사용하는 쿼리는 인덱스를 풀 스캔했으며, "Using filesort"를 보면 레코드 정렬 작업까지 실행했다는 것을 알 수 있다.
반면 GROUP BY 절을 사용하는 쿼리는 별도의 정렬 작업 없이 루스 인덱스 스캔으로 사원별 MAX(from_date) 값을 찾아냈다는 것을 알 수 있다.
이로 인해 실제 두 쿼리가 연산에 사용한 레코드 건수도 상당히 차이가 크게 난다는 것을 다음 결과로 알 수 있다.

<img src="https://github.com/user-attachments/assets/d09f76a2-1243-4520-aab6-b8d5337d1482" width="400"/><br/>

윈도우 함수는 salaries 테이블의 모든 레코드 건수만큼의 결과를 만들어야 하지만 GROUP BY 절을 사용하는 쿼리는 유니크한 emp_no별로 레코드 1건씩만 결과를 만들면 된다.
그래서 기본적으로 레코드 건수에서 차이가 날 수 있다.
하지만 앞의 결과를 보면 윈도우 함수를 사용한 쿼리는 예상보다 훨씬 많은 레코드를 가공했고, 그로 인해 MySQL 서버 내부적으로 레코드의 읽고 쓰기가 상당히 많이 발생했다는 것을 알 수 있다.
실제 쿼리의 실행 시간도 GROUP BY 쿼리가 1.2초 걸린 반면 윈도우 함수를 사용한 쿼리는 3.2초 정도 소요됐다.

특히 윈도우 함수를 사용한 쿼리는 프라이머리 키(emp_no, from_date)를 충분히 활용할 법한 쿼리였지만 윈도우 함수 부분은 이 인덱스를 전혀 활용하지 못했다.
물론 인덱스 풀 스캔을 하긴 했지만, 이는 윈도우 함수 처리를 위한 것이 아니라 ix_salary 인덱스에 쿼리 처리에 필요한 칼럼(from_date와 emp_no 칼럼)이 모두 포함(ix_salary 인덱스는 salary
칼럼으로만 구성된 인덱스지만, 실제 프라이머리 키인 (emp_no, from_date) 칼럼이 ix_salary 인덱스의 리프 노드에 저장돼 있다.)돼 있으면서 프라이머리 키보다 크기가 작기 때문에 활용했을 뿐이다.

쿼리 요건에 따라 GROUP BY나 다른 기존 기능으로는 윈도우 함수를 대체할 수 없겠지만, 가능하다면 윈도우 함수에 너무 의존하지 않는 것이 좋다.
또한 배치 프로그램이라면 윈도우 함수를 사용해도 무방하겠지만 온라인 트랜잭션 처리에서는 많은 레코드에 대해 윈도우 함수를 적용하는 것은 가능하면 피하자.
소량의 레코드에 대해서라면 윈도우 함수를 사용해도 메모리에서 빠르게 처리될 것이므로 특별히 성능에 대해 고민하지 않아도 된다.
<br/>
<br/>
## (13) 잠금을 사용하는 SELECT
InnoDB 테이블에 대해서는 레코드를 SELECT할 때 레코드에 아무런 잠금도 걸지 않는데, 이를 잠금 없는 읽기(Non Locking Consistent Read)라고 한다.
하지만 SELECT 쿼리를 이용해 읽은 레코드의 칼럼 값을 애플리케이션에서 가공해서 다시 업데이트하고자 할 때는 SELECT가 실행된 후 다른 트랜잭션이 그 칼럼의 값을 변경하지 못하게 해야 한다.
이럴 때는 레코드를 읽으면서 강제로 잠금을 걸어 둘 필요가 있는데, 이때 사용하는 옵션이 FOR SHARE와 FOR UPDATE 절이다.
FOR SHARE는 SELECT 쿼리로 읽은 레코드에 대해서 읽기 잠금을 걸고, FOR UPDATE는 SELECT 쿼리가 읽은 레코드에 대해서 쓰기 잠금을 건다.
다음 쿼리는 잠금을 사용하는 SELECT 쿼리의 간단한 예제다.

```
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR SHARE;
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR UPDATE;
```

> MySQL 8.0 이전 버전에서는 SELECT로 읽은 레코드에 대해서 읽기 잠금을 위해 LOCK IN SHARE MODE 절을 사용했지만, MySQL 8.0 버전부터는 FOR SHARE로 변경됐다.
> 물론 MySQL 8.0 버전에서 여전히 LOCK IN SHARE MODE 문법을 지원하지만, 이는 이전 버전과의 호환성 차원에서 지원되는 것이므로 가능하다면 FOR SHARE 절을 사용하는 것을 권장한다.
> 또한 MySQL 8.0 버전부터는 SELECT 쿼리의 잠금을 위해 여러 가지 새로운 기능을 제공하는데, 이 기능을 제대로 활용하려면 FOR SHARE와 FOR UPDATE 절을 사용해야 한다.

이 두 가지 잠금 옵션은 모두 자동 커밋(AUTO-COMMIT)이 비활성화(OFF)된 상태 또는 BEGIN 명령이나 START TRANSACTION 명령으로 트랜잭션이 시작된 상태에서만 잠금이 유지된다.

- FOR SHARE 절은 SELECT된 레코드에 대해 읽기 잠금(공유 잠금, Shared lock)을 설정하고 다른 세션에서 해당 레코드를 변경하지 못하게 한다.
  물론 다른 세션에서 잠금이 걸린 레코드를 읽는 것은 가능하다.

- FOR UPDATE 절은 쓰기 잠금(배타 잠금, Exclusive lock)을 설정하고, 다른 트랜잭션에서는 그 레코드를 변경하는 것뿐만 아니라 읽기(FOR SHARE 절을 사용하는 SELECT 쿼리)도 수행할 수 없다.

한 가지 주의할 사항은 FOR UPDATE나 FOR SHARE 절을 가지지 않는 SELECT 쿼리의 작동 방식이다.
InnoDB 스토리지 엔진을 사용하는 테이블에서는 잠금 없는 읽기가 지원되기 때문에 특정 레코드가 "SELECT ... FOR UPDATE" 쿼리에 의해서 잠겨진 상태라 하더라도 FOR SHARE나 FOR UPDATE 절을
가지지 않는 단순 SELECT 쿼리는 아무런 대기 없이 실행된다. 다음 표는 간단히 예제로 잠금 대기 여부를 시간 순서대로 나열해본 것이다.

#### [잠금 대기하지 않는 경우]
<img src="https://github.com/user-attachments/assets/68111943-dffa-4c90-8411-9777c6371dff" width="550"/><br/>

#### [잠금 대기하는 경우]
<img src="https://github.com/user-attachments/assets/6e5bd241-ff6b-4878-9e77-7d6535feec30" width="550"/><br/>

### [1] 잠금 테이블 선택
다음 쿼리는 사원의 정보를 조회하는 SELECT 쿼리에 FOR UPDATE를 함께 사용하는 예제다.

```
mysql> SELECT *
       FROM employees e
         INNER JOIN dept_emp de ON de.emp_no=e.emp_no
         INNER JOIN departments d ON d.dept_no=de.dept_no
       FOR UPDATE;
```

이 쿼리는 employees 테이블과 dept_emp 테이블, departments 테이블을 조인해서 읽으면서 FOR UPDATE 절을 사용했다.
그래서 InnoDB 스토리지 엔진은 3개 테이블에서 읽은 레코드에 대해 모두 쓰기 잠금(Exclusive Lock)을 걸게 된다.
그런데 dept_emp 테이블과 departments 테이블은 그냥 참고용으로만 읽고, 실제 쓰기 잠금은 employees 테이블에만 걸고 싶다면 어떻게 해야 할까?
MySQL 8.0 이전 버전에서는 선택적으로 잠금을 걸 수 있는 옵션이 없었지만 MySQL 8.0 버전부터는 다음과 같이 잠금을 걸 테이블을 선택할 수 있도록 기능이 개선됐다.
다음 예제와 같이 FOR UPDATE 뒤에 "OF 테이블" 절을 추가하면 해당 테이블에 대해서만 잠금을 걸게 된다. 테이블에 대해 별칭(Alias)이 사용된 경우에는 별명을 명시해야 한다.
SELECT 쿼리에 사용된 테이블 중에서 특정 테이블만 잠금을 획득하는 옵션은 FOR UPDATE와 FOR SHARE 절 모두 적용할 수 있다.

```
mysql> SELECT *
       FROM employees e
       INNER JOIN dept_emp de ON de.emp_no=e.emp_no
         INNER JOIN departments d ON d.dept_no=de.dept_no
       WHERE e.emp_no=10001
       FOR UPDATE OF e;
```

### [2] NOWAIT & SKIP LOCKED
MySQL 8.0 버전부터는 NOWAIT과 SKIP LOCKED 옵션을 사용할 수 있게 기능이 추가됐다.
지금까지의 MySQL 잠금은 누군가가 레코드를 잠그고 있다면 다른 트랜잭션은 그 잠금이 해제될 때까지 기다려야 했다. 때로는 일정 시간이 지나면 잠금 획득 실패 에러 메시지를 받을 수도 있었다.
하지만 이런 작동 방식은 휴대폰의 화면을 보면서 응답을 기다리고 있을 사용자를 생각하면 때로는 적절한 작동 방식이 아닐 수도 있다. 우선 간단히 다음 예제를 이용해 두 옵션의 필요성을 살펴보자.

```
mysql> BEGIN;
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR UPDATE;

... 응용 프로그램에서 필요한 연산을 수행 ...

mysql> UPDATE employees SET ... WHERE emp_no=10001;
mysql> COMMIT;
```

애플리케이션에서는 트랜잭션을 시작하고 emp_no가 10001인 사원을 먼저 읽어서 그 값을 이용해 연산을 수행한 후 다시 employees 테이블에 업데이트한다.
그런데 응용 프로그램에서 연산을 수행하는 동안 다른 트랜잭션이 employees 테이블의 emp_no가 10001인 사원의 정보를 변경하지 못하게 막기 위해 FOR UPDATE 절을 사용했다.
먼저 실행된 트랜잭션이 employees 테이블의 emp_no가 10001인 레코드에 대해서 변경 작업을 장시간 수행하고 있다면 SELECT ... FOR UPDATE 구문은 선행 트랜잭션이 완료될 때가지 기다려야 할
것이다. 또는 innodb_lock_wait_timeout 시스템 변수에 설정된 시간(기본적으로 50초) 동안 기다렸다가 에러 메시지를 받게 될 것이다.

그런데 애플리케이션의 어떤 기능에서는 emp_no=10001인 레코드가 이미 잠겨진 상태라면 그냥 무시하고 즉시 에러를 반환하면 응용 프로그램에서 다른 처리를 수행하거나 다시 트랜잭션을 시작하도록 구현해야
할 때도 있다. 이럴 때 SELECT 쿼리의 마지막에 NOWAIT 옵션을 사용하면 된다.
FOR UPDATE나 FOR SHARE 절이 없는 SELECT 쿼리는 잠금 대기 자체가 없기 때문에 NOWAIT 옵션을 사용하는 것은 의미가 없다.

```
mysql> SELECT * FROM employees
       WHERE emp_no=10001
       FOR UPDATE NOWAIT;
```

NOWAIT 옵션을 사용하면 SELECT 쿼리가 해당 레코드에 대해 즉시 잠금을 획득했다면 NOWAIT 옵션이 없을 때와 동일하게 실행된다.
하지만 해당 레코드가 다른 트랜잭션에 의해서 잠겨진 상태라면 다음과 같이 에러를 반환하면서 쿼리는 즉시 종료된다.

```
mysql> SELECT * FROM employees WHERE emp_no=10001 FOR UPDATE NOWAIT;
ERROR 3572 (HY000): Statement aborted
  because lock(s) could not be acquired immediately and NOWAIT is set.
```

SKIP LOCKED 옵션은 SELECT하려는 레코드가 다른 트랜잭션에 의해 이미 잠겨진 상태라면 에러를 반환하지 않고 잠긴 레코드는 무시하고 잠금이 걸리지 않은 레코드만 가져온다.

```
mysql> BEGIN;
mysql> SELECT * FROM salaries WHERE emp_no=10001 FOR UPDATE SKIP LOCKED;

... 응용 프로그램에서 필요한 연산을 수행 ...

mysql> UPDATE salaries SET ... WHERE emp_no=10001 AND from_date='1986-06-26';
mysql> COMMIT;
```

다음은 FOR UPDATE SKIP LOCKED 절을 사용했을 때와 그렇지 않을 때의 결과를 비교해보기 위한 예제다.

<img src="https://github.com/user-attachments/assets/36722d99-a0b0-433a-a947-ec4e4316197b" width="570"/>
<img src="https://github.com/user-attachments/assets/13b4a45c-b853-42a5-9201-9c4037b7886c" width="570"/><br/>

세션 1번에서 salaries 테이블의 특정 레코드를 SELECT ... FOR UPDATE 구문을 사용해 잠근 상태에서 세션 2번에서 동일 조건으로 FOR UPDATE 절 없이 그냥 실행했을 때는 세션 1번에서 조회한
레코드와 동일한 레코드를 반환한 것을 확인할 수 있다.
하지만 동일한 SELECT 쿼리에 FOR UPDATE SKIP LOCKED 절을 추가하면 MySQL 서버는 세션 1번에서 잠그고 있는 레코드는 무시(SKIP LOCKED)하고 그다음 레코드를 반환했다.
이런 이유로 SKIP LOCKED 절을 가진 SELECT 구문은 확정적이지 않은(NOT-DETERMINISTIC) 쿼리가 된다.

> "확정적(DETERMINISTIC)"이란 말의 의미는 입력이 동일하면 시점에 관계없이 동일한 결과를 반환하는 것을 의미한다.
> 하지만 SKIP LOCKED 절을 가진 SELECT 쿼리는 실행하는 시점에 따라(아무런 데이터 변경이 없는 상태에서도) 각 트랜잭션의 간섭에 의해 다른 결과를 반환할 수도 있는데, 이를
> 비확정적(NOT-DETERMINISTIC)이라고 한다. 이렇게 비확정적인 쿼리는 문장(STATEMENT) 기반의 복제에서 소스 서버와 레플리카 서버의 데이터를 다르게 만들 수도 있다.
> 그래서 가능하면 복제의 바이너리 로그 포맷으로 STATEMENT보다는 ROW 또는 MIXED를 사용하자.

NOWAIT이나 SKIP LOCKED 기능은 큐(Queue)와 같은 기능을 MySQL 서버에서 구현하고자 할 때 매우 유용하다. 예를 들어, 다음과 같은 간단한 요건을 가지는 쿠폰 발급 기능을 한번 생각해보자.

- 하나의 쿠폰은 한 사용자만 사용 가능하다.
- 쿠폰의 개수는 1000개 제한이며, 선착순으로 요청한 사용자에게 발급한다.

일반적으로 이 같은 요건을 처리하기 위해 다음과 같은 테이블을 생성했다.

```
CREATE TABLE coupon (
  coupon_id     BIGINT NOT NULL,
  owned_user_id BIGINT NULL DEFAULT 0, /*+ 쿠폰이 발급되면 소유한 사용자의 id를 저장 */
  coupon_code   VARCHAR(15) NOT NULL,
  ...
  PRIMARY KEY (coupon_id),
  INDEX ix_owneduserid (owned_user_id)
);
```

그리고 응용 프로그램에서는 다음과 같은 절차를 거쳐 쿠폰을 발급하게 될 것이다. 물론 실제 프로그램 코드는 훨씬 복잡하겠지만 사용할 쿼리만 간략하게 정리해 보면 대략 다음과 같은 절차를 거칠 것이다.
응용 프로그램 코드에서는 우선 아직 주인이 없는(owned_user_id=0) 쿠폰을 검색해서 하나를 가져온다. 이때 다른 트랜잭션에서 해당 쿠폰을 가져가지 못하게 FOR UPDATE 절을 사용했다.

```
mysql> BEGIN;
mysql> SELECT * FROM coupon
       WHERE owned_user_id=0 ORDER BY coupon_id ASC LIMIT 1 FOR UPDATE;

... 응용 프로그램 연산 수행 ...

mysql> UPDATE coupon SET owned_user_id=? WHERE coupon_id=?;
mysql> COMMIT;
```

많은 사용자가 이미 경험했겠지만 많은 사용자에게 인기 있는 쿠폰이라면 애플리케이션 서버는 단번에 응답 불능 상태가 될 것이다.
동시에 1000명의 사용자가 쿠폰을 요청하면 애플리케이션 서버는 그 요청만큼 프로세스를 생성해서 위의 트랜잭션을 동시에 실행할 것이다.
하지만 각 트랜잭션에서 실행하는 SELECT ... FOR UPDATE 쿼리는 coupon 테이블에서 하나의 레코드로 집중해서 잠금 획득을 하려고 할 것이다.
물론 그중에서 처음으로 잠금을 획득하는 트랜잭션은 있겠지만 나머지 999개의 트랜잭션은 첫 번째 트랜잭션이 작업을 끝내고 COMMIT할 때까지 대기해야 한다.
그리고 두 번째와 세 번째, 네 번째 순으로 트랜잭션이 하나씩 실행돼야 한다.
트랜잭션의 처리 속도에 따라 일정 시점 이후 트랜잭션은 대기 시간(innodb_lock_wait_timeout 시스템 변수에 설정된 시간) 동안 잠금을 획득하지 못해서 결국 에러를 반환한다.

> SELECT 쿼리의 "ORDER BY coupon_id" 때문에 이렇게 모든 트랜잭션이 하나의 레코드로 집중된다고 생각할 수도 있다.
> 하지만 DBMS 서버는 쿼리의 실행이 항상 실행 계획을 기반으로 수행되기 때문에 ORDER BY 절과 무관하게 (ORDER BY 절이 있든 없든)모든 트랜잭션은 항상 순서대로 레코드를 읽을 것이다.
> 이 예제에서 ORDER BY 절은 별도의 정렬을 필요로 하지도 않고, 쿼리 실행 시에는 그냥 ix_owneduserid 인덱스를 순서대로 읽기 때문에 이로 인해서 쿼리의 성능이 떨어지지는 않는다.

MySQL 8.0 이전 버전에서는 이런 문제를 해결하기 위해 레디스(Redis)나 멤캐시(Memcached) 같은 캐시 솔루션을 별도로 구축해서 쿠폰 발급 기능을 구현했다.
하지만 MySQL 8.0 버전부터는 FOR UPDATE SKIP LOCKED 절을 사용하면 트랜잭션이 수행되는 데 걸리는 시간과 관계없이 다른 트랜잭션에 의해서 이미 사용 중인(잠겨진) 레코드를 스킵하는 시간만
지나면 각자의 트랜잭션을 실행할 수 있다.
이 예제에서는 1000개의 쿠폰을 가정했는데, MySQL 서버에서 1000건의(가장 마지막 트랜잭션이 잠금을 획득하기 위해 스킵해야 할 레코드 건수) 레코드를 스캔하는 데 걸리는 시간은 매우 짧다.
그래서 FOR UPDATE SKIP LOCKED 절을 사용한다면 실제 MySQL 서버에서는 1000개의 트랜잭션을 동시에 처리하게 되는 효과를 얻을 수도 있다.
아래 그림은 SKIP LOCKED를 사용한 경우와 그렇지 않은 경우의 동시 처리 및 스루풋 비교를 보여준다.

#### [그림 11.11] SKIP LOCKED 미 사용 시 동적 처리 성능
<img src="https://github.com/user-attachments/assets/69434107-0fae-456f-9597-6fe4d7587c98" width="470"/><br/>

#### [그림 11.12] SKIP LOCKED 사용 시 동시 처리
<img src="https://github.com/user-attachments/assets/db258f81-d15e-42c5-b435-188fe6bf526e" width="470"/><br/>

[그림 11.11]에서 "A"로 표시된 선의 길이는 하나의 트랜잭션이 처리되는 데 걸리는 시간을 의미하고, [그림 11.12]에서 "B"로 표시된 선의 길이는 잠겨진 레코드 1건을 스킵(SKIP LOCKED)하는 데
걸린 시간을 의미한다. 이해를 돕기 위해 B의 길이(잠겨진 레코드 1건을 스킵하는 데 걸린 시간)를 길게 표시했지만 MySQL 서버에서 레코드 1건을 읽는 데 걸리는 시간은 매우 짧은 시간일 것이다.

[그림 11.12]에서 보다시피 FOR UPDATE SKIP LOCKED는 MySQL 서버로 동시에 유입된 트랜잭션들이 대기 시간 없이 잠긴 레코드를 스킵하고 사용 가능한 레코드를 찾기만 하면 즉시 트랜잭션 처리를
시작할 수 있다.
하지만 [그림 11.11]에서는 SKIP LOCKED 절 없이 FOR UPDATE만 사용한 경우에는 동시에 유입된 트랜잭션이 모두 잠금 대기를 하고 있다가 첫 번째 레코드를 잠근 트랜잭션이 완료돼야 비로소 두 번째
트랜잭션이 시작될 수 있고, 세 번째 트랜잭션은 두 번째 트랜잭션이 완료돼야 시작될 수 있다. 그래서 겨우 3개 트랜잭션만 완료되고 나머지 트랜잭션들은 아직도 대기 중인 상태로 남아 있는 것이다.
아무리 MySQL 서버가 많은 CPU와 메모리를 가지고 있다고 하더라도 이렇게 처리가 순차적(Serialization)으로 처리되면 서버의 남는 자원을 제대로 활용하지 못한다.

> 많은 프로젝트에서 자신의 애플리케이션이 겪는 문제의 원인을 제대로 분석하지 못하고, 애플리케이션 코드는 그대로 놔두고 MySQL 서버가 느려서 트랜잭션이 느려진다고들 한다.
> 그러고는 MySQL 서버의 메모리와 CPU, 디스크의 성능만 계속 높이는 경우도 있다. 서비스 처리 지연이 발생하면 항상 병목 지점을 정확히 찾고 원인을 분석해서 그에 맞는 해결책을 사용해야 한다.

NOWAIT와 SKIP LOCKED 절은 SELECT ... FOR UPDATE 구문에서만 사용할 수 있으며, 당연히 UPDATE나 DELETE 쿼리에서는 사용할 수 없다.
그런데 왜 UPDATE나 DELETE에는 NOWAIT과 SKIP LOCKED를 사용하지 못하게 막아뒀을까?
NOWAIT과 SKIP LOCKED 절은 쿼리 자체를 비확정적으로 만들기 때문에 NOWAIT이나 SKIP LOCKED가 UPDATE나 DELETE 문장에서 사용된다면 실행될 때마다 데이터베이스의 상태를 다른 결과로 만들게
된다. 즉 UPDATE나 DELETE 문장이 정상적으로 실행됐지만 어떤 레코드가 업데이트되거나 삭제됐는지 알 수 없게 되는 것이다. 이는 사용자들을 혼란에 빠뜨리게 될 것이다.
또한 MySQL 서버의 복제에서는 더 큰 문제를 일으킬 수도 있다.