# [CH 11-6] UPDATE와 DELETE

일반적인 온라인 트랜잭션 프로그램에서 UPDATE와 DELETE 문장은 주로 하나의 테이블에 대해 한 건 또는 여러 건의 레코드를 변경 또는 삭제하기 위해 사용된다.
하지만 MySQL 서버에서는 여러 테이블을 조인해서 한 개 이상 테이블의 레코드를 변경하거나 삭제하는 기능도 제공한다.
특히 잘못된 데이터를 보정하거나 일괄로 많은 레코드를 변경 및 삭제하는 경우에 JOIN UPDATE와 JOIN DELETE 구문은 매우 유용하다.

UPDATE 문장과 DELETE 문장은 작성 방법이나 WHERE 절의 인덱스 사용법 모두 동일하므로 함께 살펴보겠다.

---
<br/>

## (1) UPDATE ... ORDER BY ... LIMIT n
UPDATE와 DELETE는 WHERE 조건절에 일치하는 모든 레코드를 업데이트하는 것이 일반적인 처리 방식이다.
하지만 MySQL에서는 UPDATE나 DELETE 문장에 ORDER BY 절과 LIMIT 절을 동시에 사용해 특정 칼럼으로 정렬해서 상위 몇 건만 변경 및 삭제하는 것도 가능하다.
한 번에 너무 많은 레코드를 변경 및 삭제하는 작업은 MySQL 서버에 과부하를 유발하거나 다른 커넥션의 쿼리 처리를 방해할 수도 있다.
이때 LIMIT을 이용해 조금씩 잘라서 변경하거나 삭제하는 방식을 손쉽게 구현할 수 있다.

하지만 복제 소스 서버에서 ORDER BY ... LIMIT이 포함된 UPDATE나 DELETE 문장을 실행하면 다음과 같은 경고 메시지가 발생할 수도 있다.
물론 바이너리 로그의 포맷이 로우(ROW)일 때는 문제가 되지 않지만 문장(STATEMENT) 기반의 복제에서는 주의가 필요하다.

```
mysql> SET binlog_format=STATEMENT;

mysql> DELETE FROM employees ORDER BY last_name LIMIT 10;
Query OK, 10 rows affected, 1 warning (0.36 sec)

mysql> SHOW WARNINGS \G
************************** 1. row **************************
  Level: Note
   Code: 1592
Message: Unsafe statement written to the binary log using statement format since BINLOG_FORMAT
= STATEMENT. The statement is unsafe because it uses a LIMIT clause. This is unsafe because the
set of rows included cannot be predicted.
```

이 경고 메시지가 발생하는 것은 ORDER BY에 의해 정렬되더라도 중복된 값의 순서가 복제 소스 서버와 레플리카 서버에서 달라질 수도 있기 때문인데, 프라이머리 키로 정렬하면 문제는 없지만 여전히 경고
메시지는 기록된다. 복제가 구축된 MySQL 서버에서 ORDER BY가 포함된 UPDATE나 DELETE 문장을 사용할 때는 주의하자.
<br/>
<br/>
## (2) JOIN UPDATE
두 개 이상의 테이블을 조인해 조인된 결과 레코드를 변경 및 삭제하는 쿼리를 JOIN UPDATE라고 한다.
조인된 테이블 중에서 특정 테이블의 칼럼값을 다른 테이블의 칼럼에 업데이트해야 할 때 주로 조인 업데이트를 사용한다.
또는 꼭 다른 테이블의 칼럼값을 참조하지 않더라도 조인되는 양쪽 테이블에 공통으로 존재하는 레코드만 찾아서 업데이트하는 용도로도 사용할 수 있다.

일반적으로 JOIN UPDATE는 조인되는 모든 테이블에 대해 읽기 참조만 되는 테이블은 읽기 잠금이 걸리고, 칼럼이 변경되는 테이블은 쓰기 잠금이 걸린다.
그래서 JOIN UPDATE 문장이 웹 서비스 같은 OLTP 환경에서는 데드락을 유발할 가능성이 높으므로 너무 빈번하게 사용하는 것은 피하는 것이 좋다.
하지만 배치 프로그램이나 통계용 UPDATE 문장에서는 유용하게 사용할 수 있다.

우선 간단한 JOIN UPDATE 예제를 위해 다음과 같이 테이블을 생성하고, JOIN UPDATE 예제를 한번 실행해 보자.

```
mysql> CREATE TABLE tb_test1 (
         emp_no INT,
         first_name VARCHAR(14),
         PRIMARY KEY (emp_no)
       );

mysql> INSERT INTO tb_test1 VALUES (10001, NULL), (10002, NULL), (10003, NULL), (10004, NULL);

mysql> UPDATE tb_test1 t1, employees e
          SET t1.first_name=e.first_name
        WHERE e.emp_no=t1.emp_no;
```

위의 예제 쿼리는 임시로 생성한 tb_test1 테이블과 employees 테이블을 사원 번호(emp_no)로 조인한 다음, employees 테이블의 first_name 칼럼의 값을 tb_test1 테이블의 first_name
칼럼으로 복사하는 JOIN UPDATE 문장이다. 그런데 JOIN UPDATE 쿼리도 2개 이상의 테이블을 먼저 조인해야 하므로 테이블의 조인 순서에 따라 UPDATE 문장의 성능이 달라질 수 있다.
그래서 JOIN UPDATE 문장도 사용하기 전에 실행 계획을 확인하는 것이 좋다.

이제 GROUP BY가 포함된 JOIN UPDATE에 대해 조금 살펴보자. 다음 예제의 첫 번째 쿼리는 테스트를 목적으로 departments 테이블에 emp_count 칼럼을 추가한 것이다.
departments 테이블에 추가된 emp_count는 해당 부서에 소속된 사원의 수를 저장하기 위한 칼럼이다.

```
mysql> ALTER TABLE departments ADD emp_count INT;

mysql> UPDATE departments d, dept_emp de
          SET d.emp_count=COUNT(*)
        WHERE de.dept_no=d.dept_no
        GROUP BY de.dept_no;
```

위의 GROUP BY를 포함한 JOIN UPDATE는 dept_emp 테이블에서 부서별로 사원의 수를 departments 테이블의 emp_count 칼럼에 업데이트하기 위해 만든 쿼리다.
dept_emp 테이블에서 부서별로 사원의 수를 가져오기 위해 GROUP BY가 사용된 것을 알 수 있다. 하지만 이 쿼리는 작동하지 않고 에러를 발생시킬 것이다.
JOIN UPDATE 문장에서는 GROUP BY나 ORDER BY 절을 사용할 수 없기 때문이다. 그러면 이 작업을 처리하려면 어떻게 해야 할까?
바로 이렇게 문법적으로 지원하지 않는 SQL에 대해 서브쿼리를 이용한 파생 테이블을 사용하는 것이다. 이 JOIN UPDATE 문장을 서브쿼리를 이용해 다시 작성해 보자.

```
mysql> UPDATE departments d,
              (SELECT de.dept_no, COUNT(*) AS emp_count
               FROM dept_emp de
               GROUP BY de.dept_no) dc
          SET d.emp_count=dc.emp_count
        WHERE dc.dept_no=d.dept_no;
```

위의 예제 쿼리에서는 우선 서브쿼리로 dept_emp 테이블을 dept_no로 그루핑하고, 그 결과를 파생 테이블로 저장했다.
그리고 이 결과와 departments 테이블을 조인해 departments 테이블의 emp_count 칼럼에 업데이트한 것이다.
이미 조인에서 배웠지만 이 예제 쿼리와 같이 일반 테이블이 조인될 때는 임시 테이블이 드라이빙 테이블이 되는 것이 일반적이므로 빠른 성능을 보여준다.
MySQL의 옵티마이저가 최적의 조인 방향을 잘 알아서 선택하겠지만, 혹시라도 원하는 조인의 방향을 옵티마이저에게 알려주고 싶다면 다음과 같이 JOIN UPDATE 문장에 STRAIGHT_JOIN이라는 키워드를
사용하면 된다. 또는 MySQL 8.0에 새롭게 추가된 JOIN_ORDER 옵티마이저 힌트를 사용해도 된다.

```
mysql> UPDATE (SELECT de.dept_no, COUNT(*) AS emp_count
               FROM dept_emp de
               GROUP BY de.dept_no) dc
         STRAIGHT_JOIN departments d ON dc.dept_no=d.dept_no
           SET d.emp_count=dc.emp_count;

mysql> UPDATE /*+ JOIN_ORDER(dc, d) */
         (SELECT de.dept_no, COUNT(*) AS emp_count
               FROM dept_emp de
               GROUP BY de.dept_no) dc
         INNER JOIN departments d ON dc.dept_no=d.dept_no
           SET d.emp_count=dc.emp_count;
```

여기에 사용된 STRAIGHT_JOIN 키워드는 조인의 순서를 지정하는 MySQL 힌트이기도 하지만 INNER JOIN 또는 LEFT JOIN과 같이 조인 키워드로 사용되기도 한다.
INNER JOIN이나 LEFT JOIN 키워드는 사실 테이블의 조인 순서를 결정하는 키워드는 아니다. 하지만 STRAIGHT_JOIN 키워드는 조인의 순서까지 결정하는 키워드다.
STRAIGHT_JOIN 키워드 왼쪽에 명시된 테이블이 드라이빙 테이블이며, 오른쪽의 테이블은 드리븐 테이블이 된다.
INNER JOIN 또는 LEFT JOIN을 사용하는 경우 옵티마이저가 원하는 순서대로 조인을 실행하지 않는다면 JOIN_ORDER 힌트를 사용하면 된다.

> 예제에서는 GROUP BY 절을 가진 쿼리의 결과를 임시 테이블에 저장했지만 필요에 따라 다음과 같이 래터럴 조인(LATERAL JOIN)을 이용해 JOIN UPDATE를 구현할 수도 있다.
>
> ```
> mysql> UPDATE departments d
>          INNER JOIN LATERAL (
>            SELECT de.dept_no, COUNT(*) AS emp_count
>            FROM dept_emp de
>            WHERE de.dept_no=d.dept_no
>          ) dc ON dc.dept_no=d.dept_no
>        SET d.emp_count=dc.emp_count;
> ``` 
<br/>

## (3) 여러 레코드 UPDATE
하나의 UPDATE 문장으로 여러 레코드를 업데이트하는 것이 가능하다는 것은 당연히 모두가 알고 있을 것이다.
하지만 하나의 UPDATE 문장으로 여러 개의 레코드를 업데이트하는 경우 다음 예제와 같이 모든 레코드를 동일한 값으로만 업데이트할 수 있었다.

```
mysql> UPDATE departments SET emp_count=10;
mysql> UPDATE departments SET emp_count=emp_count + 10;
```

하지만 MySQL 8.0 버전부터는 다음과 같이 레코드 생성(Row Constructor) 문법을 이용해 레코드별로 서로 다른 값을 업데이트할 수 있게 됐다.

```
mysql> CREATE TABLE user_level (
         user_id BIGINT NOT NULL,
         user_lv INT NOT NULL,
         created_at DATETIME NOT NULL,
         PRIMARY KEY (user_id)
       );

mysql> UPDATE user_level ul
         INNER JOIN (VALUES ROW(1, 1),
                            ROW(2, 4)) new_user_level (user_id, user_lv)
                                       ON new_user_level.user_id=ul.user_id
         SET ul.user_lv=ul.user_lv + new_user_level.user_lv;
```

"VALUES ROWS(...), ROWS(...), ..." 문법을 사용하면 SQL 문장 내에서 임시 테이블을 생성하는 효과를 낼 수 있다.
위의 예제에서는 2건의 레코드((1,1)과 (2,4))를 가지는 임시 테이블 "new_user_level"을 생성하고, new_user_level 임시 테이블과 user_level 테이블을 조인해서 업데이트를 수행하는 "JOIN
UPDATE" 문장의 효과를 낼 수 있게 됐다.
<br/>
<br/>
## (4) JOIN DELETE
JOIN DELETE 문장을 사용하려면 단일 테이블의 DELETE 문장과는 조금 다른 문법으로 쿼리를 작성해야 한다. 우선 3개의 테이블을 조인해서 그중 하나의 테이블에서만 레코드를 삭제하는 예제를 살펴보자.

```
mysql> DELETE e
       FROM employees e, dept_emp de, departments d
       WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
```

위 예제는 employees와 dept_emp, departments라는 3개의 테이블을 조인한 다음, 조인이 성공한 레코드에 대해 employees 테이블의 레코드만 삭제하는 쿼리다.
일반적으로 하나의 테이블에서 레코드를 삭제할 때는 "DELETE FROM table ..."과 같은 문법으로 사용하지만 JOIN DELETE 문장에서는 DELETE와 FROM 절 사이에 삭제할 테이블을 명시해야 한다.
이 예제에서는 조인을 위해 FROM 절에는 employees와 dept_emp, departments 테이블을 명시했고, DELETE와 FROM 절 사이에는 실제로 삭제할 employees 테이블의 별명(e)만 명시했다.

JOIN DELETE 문장으로 하나의 테이블에서만 레코드를 삭제할 수 있는 것은 아니다. 다음 예제를 살펴보자.

```
mysql> DELETE e, de
       FROM employees e, dept_emp de, departments d
       WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';

mysql> DELETE d, de, d
       FROM employees e, dept_emp de, departments d
       WHERE e.emp_no=de.emp_no AND de.dept_no=d.dept_no AND d.dept_no='d001';
```

위 예제에서 첫 번째 쿼리는 employees와 dept_emp, departments 테이블을 조인해서 employees와 dept_emp 테이블에서 동시에 레코드를 삭제하는 쿼리이며, 두 번째 쿼리는 3개의 테이블 모두에서
dept_no='d001'인 레코드를 삭제하는 예제다. 물론 JOIN DELETE 또한 JOIN UPDATE와 마찬가지로 SELECT 쿼리로 변환해서 실행 계획을 확인해 볼 수 있다.
옵티마이저가 적절한 조인 순서를 결정하지 못한다면 다음 예제와 같이 STRAIGHT_JOIN 키워드나 JOIN_ORDER 옵티마이저 힌트를 이용해 조인의 순서를 옵티마이저에게 지시할 수 있다.

```
mysql> DELETE e, de, d
       FROM departments d
         STRAIGHT_JOIN dept_emp de ON de.dept_no=d.dept_no
         STRAIGHT_JOIN employees e ON e.emp_no=de.emp_no
       WHERE d.dept_no='d001';

mysql> DELETE /*+ JOIN_ORDER(d, de, e) */ e, de, d
       FROM departments d
         INNER JOIN dept_emp de ON de.dept_no=d.dept_no
         INNER JOIN employees e ON e.emp_no=de.emp_no
       WHERE d.dept_no='d001';
```