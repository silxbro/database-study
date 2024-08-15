# 01. Nested Loops 조인

<br/>

## (1) 기본 메커니즘
프로그래밍을 해 본 독자라면 누구나 중첩 루프문(Nested Loop)의 수행 구조를 이해할 것이고, 그렇다면 Nested Loops 조인(이하 NL 조인)도 어렵지 않게 이해할 수 있다.

중첩 루프문과 같은 수행 구조를 사용하는 NL 조인이 실제 어떤 순서로 데이터를 액세스하는지 아래 PL/SQL문이 잘 설명해 준다.
```
begin
  for outer in (select deptno, empno, rpad(ename, 10) ename from emp)
  loop    -- outer 루프
    for inner in (select dname from dept where deptno = outer.deptno)
    loop  -- inner 루프
      dbms_output.put_line(outer.empno||' : '||outer.ename||' : '||inner.dname);
    end loop;
  end loop;
end;
```
위 PL/SQL문은 아래 쿼리와 100% 같은 순서로 데이터를 액세스하고, 데이터 출력순서도 같다. 내부적으로(=Recursive하게) 쿼리를 반복 수행하지 않는다는 점만 다르다.
```
select /*+ ordered use_nl(d) */ e.empno, e.ename, d.dname
from   emp e, dept d
where  d.deptno = e.deptno
```
사실 뒤에서 설명하는 소트 머지 조인과 해시 조인도 각각 Sort Area와 Hash Area에 가공해 둔 데이터를 이용한다는 점만 다를 뿐 기본적인 조인 프로세싱은 다르지 않다.

<br/>

## (2) 힌트를 이용해 NL 조인을 제어하는 방법
```
select /*+ ordered use_nl(e) */
from   dept d, emp e
where  e.deptno = d.deptno
```
ordered 힌트는 from 절에 기술된 순서대로 조인하라고 옵티마이저에게 지시할 때 사용하고, use_nl 힌트는 NL 방식으로 조인하라고 지시할 때 사용한다.
위에서는 ordered와 use_nl(e) 힌트를 같이 사용했으므로 dept 테이블(→ Driving 또는 Outer Table)을 기준으로 emp 테이블(→ Inner 테이블)과 조인할 때 NL 방식으로 조인하라는 뜻이 된다.
- ### [Outer 테이블, Inner 테이블]

  조인을 설명하려면 우선 Outer와 Inner에 대한 용어 정의를 명확히 할 필요가 있다.
  아래는 scott 계정에 있는 dept와 emp 테이블을 조인하는 간단한 SQL문을 NL, 소트 머지, 해시 조인으로 각각 유도했을 때의 실행계획이다.

  ```
  select *
  from   dept d, emp e
  where  d.deptno = e.deptno

  Execution Plan
  ------------------------------------------------------------------------------------
  0        SELECT STATEMENT Optimmizer=ALL_ROWS
  1    0     NESTED LOOPS
  2    1       TABLE ACCESS (FULL) OF 'DEPT' (TABLE)      → Outer/Driving
  3    1       TABLE ACCESS (FULL) OF 'EMP' (TABLE)       → Inner/Driven

  Execution Plan
  ------------------------------------------------------------------------------------
  0        SELECT STATEMENT Optimmizer=ALL_ROWS
  1    0     MERGE JOIN
  2    1       SORT (JOIN)
  3    2         TABLE ACCESS (FULL) OF 'DEPT' (TABLE)    → Outer/First
  4    1       SORT (JOIN)
  5    4         TABLE ACCESS (FULL) OF 'EMP' (TABLE)     → Inner/Second

  Execution Plan
  ------------------------------------------------------------------------------------
  0        SELECT STATEMENT Optimmizer=ALL_ROWS
  1    0     HASH JOIN
  2    1       TABLE ACCESS (FULL) OF 'DEPT' (TABLE)      → Outer/Build Input
  3    1       TABLE ACCESS (FULL) OF 'EMP' (TABLE)       → Inner/Probe Input
  ```

  우선, Outer와 Inner라는 용어가 Outer 조인에 사용되는 두 테이블을 지칭할 때만 사용하는 것이 아님을 이해하기 바란다. 지금 보고 있는 쿼리는 분명 Outer 조인이 아니다.

  10053 트레이스 파일을 열어보면 조인방식에 상관없이 실행계획상 위쪽에 있는 dept 테이블을 Outer, 아래쪽에 있는 emp 테이블을 Inner로 표시하고 있다.

  NL과 소트 머지 조인은 위쪽에 있는 dept 테이블을 스캔하면서 아래쪽 emp 테이블을 탐색하는 메커니즘이므로 각각 outer와 inner로 칭하는 것이 부자연스럽지 않다.
  하지만 해시 조인의 처리 메커니즘은 아래쪽에 있는 emp 테이블을 스캔하면서 dept 테이블을 (해시 테이블을 통해) 탐색하는 방식이다.
  따라서 거꾸로 정의하는 게 맞지 않는가라는 생각을 하게 되며, 이 때문에 늘 헷갈리곤 한다.

  그런데 트레이스 파일에 사용된 용어가 표준 용어는 아니므로 크게 혼란스러워 할 필요는 없다.
  이벤트 트레이스는 오라클 개발자들이 디버깅 용도로 개발해 사용돼 온 것이고, 가장 오래된 NL 조인의 틀에 맞추다 보니 그렇게 된 것이라고 이해하면 된다.

  본서에서는 아래와 같이 정의하고 설명을 진행하기로 하겠다.

  ||NL 조인|소트 머지 조인|해시 조인|
    |:---|:---|:---|:---|
  |실행계획상 위쪽<br/>실행계획상 아래쪽|Outer(=Driving) 테이블<br/>Inner(=Driven) 테이블|Outer(=First) 테이블<br/>Inner(=Second) 테이블|Build Input<br/>Probe Input|

  참고로, 해시 조인에서도 Build Input을 드라이빙(Driving) 테이블이라고 표현하기도 하지만 소트 머지 조인에서는 그런 표현을 쓰지 않는다.

위에서는 두 개 테이블을 조인하고 있지만 세 개 이상을 조인할 때는 힌트를 아래처럼 사용하는 것이 올바른 사용법이다.
```
select /*+ ordered use_nl(B) use_nl(C) use_hash(D) */ *
from   A, B, C, D
where  .....
```
해석해 보면, A → B → C → D 순으로 조인하되, B와 조인할 때 그리고 이어서 C와 조인할 때는 NL 방식으로 조인하고, D와 조인할 때는 해시 방식으로 조인하라는 뜻이다.

ordered 대신 leading 힌트를 사용해 조인 순서를 제어할 수도 있다. 오라클 9i까지는 leading 힌트에 인자를 하나만 입력할 수 있었다.
조인할 때 가장 처음에 읽을 기준 집합(=Driving Table) 하나만 명시하는 것이다.
그러다 보니 leading 힌트만으로는 조인 순서를 세밀하게 제어할 수 없어 9i 이전 버전에서는 ordered 힌트를 주로 사용했다. from절의 테이블 순서를 일일이 바꿔주면서 말이다.

10g부터는 leading 힌트에 2개 이상 테이블을 기술할 수 있도록 기능이 개선돼, from 절을 바꾸지 않고도 마음껏 순서를 제어할 수 있게 되었다.
이 때문에 ordered 힌트의 인기가 예전만 못하다.
```
select /*+ leading(C, A, D, B) use_nl(A) use_nl(D) use_hash(B) */
from   A, B, C, D
where  .....
```
아래는 ordered나 leading 힌트를 기술하지 않았으므로 4개 테이블을 NL 방식으로 조인하되 순서는 옵티마이저가 스스로 정하도록 맡기는 것이다.
```
select /*+ use_nl(A, B, C, D) */ *
from   A, B, C, D
where  .....
```

<br/>

## (3) NL 조인 수행 방식
이제 NL 조인의 기본 알고리즘과 이를 제어하는 힌트 사용법까지 이해했다. 그럼 아래 조인문에서 조건절 비교 순서는 어떻게 될까?
(scott 스키마의 dept 테이블에는 gb 컬럼이 없지만 NL 조인 과정을 효과적으로 설명하기 위해 추가되었다.)
```
select /*+ ordered use_nl(e) */
       e.empno, e.ename, d.dname, e.job, e.sal
from   dept d, emp e
where  e.deptno = d.deptno    --- [1]
and    d.loc = 'SEOUL'        --- [2]
and    d.gb = '2'             --- [3]
and    e.sal >= 1500          --- [4]
order by sal desc
```
인덱스 상황은 다음과 같다.
```
* pk_dept          : dept.deptno
* dept_loc_idx     : dept.loc
* pk_emp           : emp.empno
* emp_deptno_idx   : emp.deptno
* emp_sal_idx      : emp.sal
```
조건절 비교 순서와 함께 위 5개 인덱스 중 어떤 것이 사용될지도 고민해 보기 바란다.
```
Execution Plan
------------------------------------------------------------------------------------
0        SELECT STATEMENT Optimmizer=ALL_ROWS
1    0     SORT ORDER BY
2    1       NESTED LOOPS
3    2         TABLE ACCESS BY INDEX ROWID DEPT
4    3           INDEX RANGE SCAN DEPT_LOC_IDX
5    2         TABLE ACCESS BY INDEX ROWID EMP
6    5           INDEX RANGE SCAN EMP_DEPTNO_IDX
```
사용되는 인덱스는 dept_loc_idx와 emp_deptno_idx인 것을 위 실행계획을 보고 알 수 있다. 그럼 조건비교 순서는? SQL 조건절에 표시한 [2] → [3] → [1] → [4] 순이다.

실행계획을 해석할 때, 형제(Sibling) 노드 간에는 위에서 아래로 읽는다. 부모-자식(Parent-Child) 노드 간에는 안쪽에서 바깥쪽으로, 즉 자식 노드부터 읽는다.
- 이 규칙을 벗어날 때도 있다.

위 실행계획의 실행 순서를 나열하면 다음과 같다.
- [1] dept_loc_idx 인덱스 범위 스캔(ID=4)
  - dept.loc = 'SEOUL' 조건을 만족하는 레코드를 찾으려고 dept_loc_idx 인덱스를 범위 스캔한다.
- [2] 인덱스 rowid로 dept 테이블 액세스(ID=3)
  - dept_loc_idx 인덱스에서 읽은 rowid를 가지고 dept 테이블을 액세스해 dept.gb = '2' 필터 조건을 만족하는 레코드를 찾는다.
- [3] emp_deptno_idx 인덱스 범위 스캔(ID=6)
  - dept 테이블에서 읽은 deptno 값을 가지고 조인 조건을 만족하는 emp 쪽 레코드를 찾으려고 emp_deptno_idx 인덱스를 범위 스캔한다.
- [4] 인덱스 rowid로 emp 테이블 액세스(ID=5)
  - emp_deptno_idx 인덱스에서 읽은 rowid를 가지고 emp 테이블을 액세스해 sal >= 1500 필터 조건을 만족하는 레코드를 찾는다.
- [5] sal 기준 내림차순(desc) 정렬(ID=1)
  - 1 ~ 4 과정을 통과한 레코드들을 sal 컬럼 기준 내림차순(desc)으로 정렬한 후 결과를 리턴한다.

여기서 기억할 것은, 각 단계를 완료하고 나서 다음 단계로 넘어가는 게 아니라 한 레코드씩 순차적으로 진행한다는 사실이다.
단, order by는 전체 집합을 대상으로 정렬해야 하므로 작업을 모두 완료한 후에 다음 오퍼레이션을 진행한다.

여기서, dept_loc_idx 인덱스를 스캔하는 양에 따라 전체 일량이 좌우됨을 이해하기 바란다.
여기서는 단일 컬럼 인덱스를 '=' 조건으로 스캔했으므로 비효율 없이 6(=5+1)건을 읽었고, 그만큼 테이블 Random 액세스가 발생했다. 우선 이 부분이 NL 조인의 첫 번째 부하지점이다.

만약 dept 테이블로 많은 양의 Random 액세스가 있었는데 gb = '2' 조건에 의해 필터링되는 비율이 높다면 어떻게 해야 할까?
이미 1장에서 배웠듯이 dept_loc_idx에 gb 컬럼을 추가하는 방안을 고려해야 한다.

두 번째 부하지점은 emp_deptno_idx 인덱스를 탐색하는 부분이며, Outer 테이블인 dept를 읽고 나서 조인 액세스가 얼만큼 발생하느냐에 의해 결정된다.
이것 역시 Random 액세스에 해당하며, gb = '2' 조건을 만족하는 건수만큼의 조인시도가 있었다.
만약 emp_deptno_idx의 높이(height)가 3이면 매 건마다 그만큼의 블록 I/O가 발생하고(버퍼 Pinning 효과를 논외로 한다면), 리프 블록을 스캔하면서 추가적인 블록 I/O가 더해진다.

세 번째 부하지점은 emp_deptno_idx를 읽고 나서 emp 테이블을 액세스하는 부분이다.
여기서도 sal >= 1500 조건에 의해 필터링되는 비율이 높다면 emp_deptno_idx 인덱스에 sal 컬럼을 추가하는 방안을 고려해야 한다.

OLTP 시스템에서 조인을 튜닝할 때는 일차적으로 NL 조인부터 고려하는 것이 올바른 순서다. 우선, NL 조인 메커니즘을 따라 각 단계의 수행 일량을 분석해 과도한 Random 액세스가 발생하는 지점을 파악한다.
조인 순서를 변경해 Random 액세스 발생량을 줄일 수 있는 경우가 있고, 그렇지 못할 때는 인덱스 컬럼 구성을 변경하거나 다른 인덱스의 사용을 고려해야 한다.

여러 가지 방안을 검토한 결과 NL 조인이 효과적이지 못하다고 판단될 때 해시 조인이나 소트 머지 조인을 검토한다.

<br/>

## (4) NL 조인의 특징
오라클은 블록 단위로 I/O를 수행하며, 하나의 레코드를 읽으려고 블록을 통째로 읽는 Random 액세스 방식은 설령 메모리 버퍼에서 빠르게 읽더라도 비효율이 존재한다.
그런데 NL 조인의 첫 번째 특징이 **Random 액세스 위주의 조인 방식**이라는 점이다. 따라서 인덱스 구성이 아무리 완벽하더라도 대량의 데이터를 조인할 때 매우 비효율적이다.

두 번째 특징은, 조인을 **한 레코드씩 순차적으로 진행**한다는 점이다.
첫 번째 특징 때문에 대용량 데이터 처리 시 매우 치명적인 한계를 드러내지만, 반대로 이 두 번째 특징 때문에 아무리 대용량 집합이더라도 매우 극적인 응답 속도를 낼 수 있다.
부분범위처리가 가능한 상황에서 그렇다. 그리고 순차적으로 진행하는 특징 때문에 먼저 액세스되는 테이블의 처리 범위에 의해 전체 일량이 결정된다.

다른 조인 방식과 비교했을 때 **인덱스 구성 전략이 특히 중요**하다는 것도 NL 조인의 중요한 특징이다.
조인 컬럼에 대한 인덱스가 있느냐 없느냐, 있다면 컬럼이 어떻게 구성됐느냐에 따라 조인 효율이 크게 달라진다.

이런 여러 가지 특징을 종합할 때, NL 조인은 **소량의 데이터를 주로 처리하거나 부분범위처리가 가능한 온라인 트랜잭션 환경에 적합한 조인 방식**이라고 할 수 있다.

<br/>

## (5) NL 조인 튜닝 실습
지금부터 SQL 트레이스 분석을 통해 실제 튜닝하는 과정을 실습해 보자. jobs, employees 두 테이블이 있고, 인덱스 상황은 다음과 같다.
```
* pk_jobs            : jobs.job_id
* jobs_max_sal_idx   : jobs.max_salary
* pk_employees       : employees.employee_id
* emp_job_idx        : employees.job_id
* emp_hiredate_idx   : employees.hire_date
```
튜닝하고자 하는 쿼리는 다음과 같다.
```
select /*+ ordered use_nl(e) index(j) index(e) */
       j.job_title, e.first_name, e.last_name
     , e.hire_date, e.salary, e.email, e.phone_number
from   jobs j, employees e
where  e.job_id = j.job_id                             --- [1]
and    j.max_salary >= 1500                            --- [2]
and    j.job_type = 'A'                                --- [3]
and    e.hire_date >= to_date('19960101', 'yyyymmdd')  --- [4]
```
우선 조건절 비교 순서와 사용되는 인덱스를 분석해 보자.
앞에서 NL 조인 메커니즘을 설명하면서 보았던 예제와 흡사한데, 사용되는 인덱스는 jobs_max_sal_idx와 emp_job_idx이고, 조건비교 순서는 [2] → [3] → [1] → [4] 순이다.

참고로, index 힌트에 어떤 인덱스를 사용하라고 명시하지 않았으므로 emp_hiredate_idx 인덱스를 이용해 [2] → [3] → [4] → [1] 순으로 처리될 수도 있다.
hire_date 조건으로 카티션 곱(Cartesian Product)이 만들어지고 나서 조인 조건 job_id를 필터링하는 방식이다.
이런 식으로 처리될 수도 있다는 것일 뿐, 일반적으로 [2] → [3] → [1] → [4] 순으로 처리되므로 이것을 기준으로 설명하겠다.

아래는 위 쿼리에 대한 SQL 트레이스 결과인데, 블록 I/O가 9개밖에 안 되므로 튜닝할 필요가 없어 보인다.
(지금부터 설명하는 내용은 HR 스키마에 있는 jobs, employees 테이블을 실제 쿼리한 결과가 아니라 NL 조인 튜닝 과정을 설명하기 위해 가공한 것임을 밝힌다.)

```
call      count       cpu      elapsed    disk     query    current   rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1      0.00         0.00       0         0          0      0
Execute       1      0.00         0.00       0         0          0      0
Fetch         2      0.00         0.01       1         9          0      5
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4      0.00         0.01       1         9          0      5
  
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    5  NESTED LOOPS
    3    TABLE ACCESS BY INDEX ROWID JOBS
    5      INDEX RANGE SCAN JOBS_MAX_SAL_IDX
    5    TABLE ACCESS BY INDEX ROWID EMPLOYEES
    8      INDEX RANGE SCAN EMP_JOB_IDX
```
만약 트레이스 결과가 아래와 같았다고 하자. 어디에 문제가 있어 보이는가?
```
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    5  NESTED LOOPS
    3    TABLE ACCESS BY INDEX ROWID JOBS
  278      INDEX RANGE SCAN JOBS_MAX_SAL_IDX
    5    TABLE ACCESS BY INDEX ROWID EMPLOYEES
    8      INDEX RANGE SCAN EMP_JOB_IDX
```
jobs_max_sal_idx 인덱스를 스캔하고서 jobs 테이블을 액세스한 횟수가 278인데, 테이블에서 job_type = 'A' 조건을 필터링한 결과는 3건에 그친다.
불필요한 테이블 액세스를 많이 한 셈이고, 이처럼 테이블을 액세스한 후에 필터링되는 비율이 높다면 인덱스에 테이블 필터 조건 컬럼을 추가하는 것을 고려해 볼 필요가 있다.

jobs_max_sal_idx 인덱스에 job_type 컬럼을 추가하고서 트레이스를 다시 확인해 보니 아래와 같이 불필요한 테이블 액세스가 없어졌다.
```
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    5  NESTED LOOPS
    3    TABLE ACCESS BY INDEX ROWID JOBS
    3      INDEX RANGE SCAN JOBS_MAX_SAL_IDX
    5    TABLE ACCESS BY INDEX ROWID EMPLOYEES
    8      INDEX RANGE SCAN EMP_JOB_IDX
```
Rows에 표시된 숫자만 보면 비효율적인 액세스가 없어 보이지만 테이블을 액세스하기 전 인덱스 스캔 단계에서의 일량을 확인하지 못했으므로 튜닝이 끝났다고 볼 수 없다.

인덱스가 [max_salary + job_type]이고, 조건절을 보면 인덱스 선두 컬럼이 부등호 조건이다.
max_salary >= 1500 조건에 해당하는 레코드가 엄청 많다면 많은 양의 인덱스 블록을 스캔하면서 job_type = 'A' 조건을 필터링했을 것이다.

오라클 7 버전에서는 Rows 부분에 각 단계의 처리한 건수(processing count)를 보여 주었으므로 실제 스캔량을 쉽게 확인할 수 있었다.
그러다가 8i부터 조금씩 바뀌기 시작해 9i에서 완전히 출력 건수를 보여주는 방식으로 바뀌다 보니 각 단계의 처리 일량을 따로 분석해야 하는 불편함이 생겼다.
(반대로, 7 버전에서는 스캔 후 출력 건수를 따로 분석해야 했다.)

그래서 9iR2부터는 아래와 같이 각 처리 단계별 논리적인 블록 요청 횟수(cr)와 디스크에서 읽은 블록 수(pr) 그리고 디스크에 쓴 블록 수(pw) 등을 표시하기 시작했다.
- 오라클 9iR2에서 OS와 버전에 따라 오퍼레이션 단계별 수행 일량이 표시되지 않는 경우가 있다. 그럴 때는 statistics_level을 all로 변경해 보기 바란다.
  그래도 같은 현상이 지속된다면 RBO 모드로 작동하는 것이 원인이므로 통계정보를 수집해 주거나 all_rows, first_rows, index 같은 힌트를 명시하면 된다.

```
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    5  NESTED LOOPS (cr=1015 pr=255 pw=0 time=...)
    3    TABLE ACCESS BY INDEX ROWID JOBS (cr=1003 pr=254 pw=0 time=...)
    3      INDEX RANGE SCAN JOBS_MAX_SAL_IDX (cr=1000 pr=254 pw=0 time=...)
    5    TABLE ACCESS BY INDEX ROWID EMPLOYEES (cr=12 pr=1 pw=0 time=...)
    8      INDEX RANGE SCAN EMP_JOB_IDX (cr=8 pr=0 pw=0 time=...)
``` 
여기서 보면 jobs_max_sal_idx 인덱스로부터 3건을 리턴하기 위해 인덱스 블록을 1,000개 읽은 것을 알 수 있다.

튜닝 방법은? jobs_max_sal_idx 인덱스 컬럼 순서를 조정해 [job_type + max_salary] 순으로 구성해주면 된다. (물론 다른 쿼리에 미치는 '영향도 분석'이 선행되어야 한다.)

이번에는 트레이스 결과가 아래와 같았다고 하자. 부하지점이 어디인가?
```
Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    5  NESTED LOOPS (cr=2732 pr=386 pw=0 time=...)
 1278    TABLE ACCESS BY INDEX ROWID JOBS (cr=166 pr=2 pw=0 time=...)
 1278      INDEX RANGE SCAN JOBS_MAX_SAL_IDX (cr=4 pr=0 pw=0 time=...)
    5    TABLE ACCESS BY INDEX ROWID EMPLOYEES (cr=2566 pr=384 pw=0 time=...)
    8      INDEX RANGE SCAN EMP_JOB_IDX (cr=2558 pr=383 pw=0 time=...)
``` 
jobs 테이블을 읽는 부분에서 비효율이 없어 보인다. 인덱스에서 스캔한 블록이 4개뿐이고 테이블을 액세스하고서 필터링되는 레코드도 전혀 없다. 일량은 많지만 비효율은 없다고 볼 수 있다.
문제는 jobs 테이블을 읽고 나서 employees 테이블과의 조인 시도 횟수다. 1,278번 조인 시도를 했지만 최종적으로 조인에 성공한 결과집합은 5건뿐이다.

이럴 때는 조인순서를 바꾸는 것을 고려해 볼 수 있다. 만약 hire_date 조건절에 부합하는 레코드가 별로 없다면 튜닝에 성공할 가능성이 높다.

하지만 그 반대의 결과가 나타날 수도 있다.
위에서 employees와 조인 후에 5건으로 줄어든 것은 jobs로부터 넘겨받는 job_id와 hire_date 두 컬럼을 조합했을 때 그런 것이지 hire_date 단독으로 조회했을 때는 데이터량이 생각보다 많을 수 있기 때문이다.

조인 순서를 바꾸어도 별 소득이 없다면 소트 머지 조인과 해시 조인을 검토해 봐야 한다.

<br/>

## (6) 테이블 Prefetch
지금부터 NL 조인과 관련된 몇 가지 확장 메커니즘을 살펴보자.

우선, 오라클 9i부터 NL 조인 실행계획에 변화가 생겼다.
아래와 같이 인덱스 rowid에 의한 Inner 테이블 액세스가 Nested Loops 위쪽에 표시되곤 하는데, 이는 해당 테이블 액세스 단계에 Prefetch 기능이 적용되었음을 표현하기 위함이다.

```
select * from dept d, emp e where e.deptno = d.deptno

Execution Plan
------------------------------------------------------------------------------------
0        SELECT STATEMENT Optimmizer=ALL_ROWS (Cost=4 Card=14 Bytes=868)
1    0     TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=4 Bytes=128)
2    1       NESTED LOOPS (Cost=4 Card=14 Bytes=868)
3    2         TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=4 Bytes=120)
4    2         INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (INDEX) (Cost=0 Card=5)
```
테이블 Prefetch를 제어하는 파라미터 중 하나인 _table_lookup_prefetch_size를 0으로 설정하면, 똑같은 SQL인데 아래와 같이 전통적인 방식의 NL 조인 실행계획으로 되돌아간다.
```
Execution Plan
------------------------------------------------------------------------------------
0        SELECT STATEMENT Optimmizer=ALL_ROWS (Cost=4 Card=14 Bytes=868)
1    0     NESTED LOOPS (Cost=4 Card=14 Bytes=868)
2    1       TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=3 Card=4 Bytes=120)
3    1       TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=4 Bytes=128)
4    3         INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (INDEX) (Cost=0 Card=5)
```
새로운 포맷의 실행계획이 나타난다고 항상 테이블 Prefetch가 작동하는 것은 아니다. 단지 그 기능이 활성화되었음을 의미할 뿐이다.
Prefetch 방식으로 디스크 블록을 읽었는데 실제 버퍼 블록 액세스로 연결되지 못한 채 메모리에서 밀려나는 비율이 높다면, 실행계획은 그대로인 채 내부적으로 기능이 비활성화되기 때문이다.

참고로, Prefetch 기능이 실제 작동할 때면 db file sequential read 대기 이벤트 대신 db file parallel reads 대기 이벤트가 나타난다.

Prefetch는 디스크 I/O와 관련 있다. 디스크 I/O를 수행하려면 비용이 많이 들기 때문에 한 번 I/O Call이 필요한 시점에, 곧이어 읽을 가능성이 큰 블록들을 캐시에 미리 적재해 두는 기능이다.
한 번의 I/O Call로써 여러 Single Block I/O를 동시에 수행한다. 이 기능에 대한 자세한 설명은 1권을 참조하기 바란다.

NL 조인에서 항상 새 포맷의 실행계획이 나타나는 것은 아니다.
기본적으로 Outer 쪽 인덱스를 Unique Scan 할 때는 작동하지 않는다. 이 경우를 제외하면 언제든 새 포맷의 실행계획이 나타날 수 있는데, 정리하면 다음과 같다.

- Inner 쪽 Non-Unique 인덱스를 Range Scan할 때는 테이블 Prefetch 실행계획이 항상 나타난다.
- Inner 쪽 Unique 인덱스를 Non-Unique 조건(모든 인덱스 구성컬럼이 '=' 조건이 아닐 때)으로 Range Scan할 때도 테이블 Prefetch 실행계획이 항상 나타난다.
- Inner 쪽 Unique 인덱스를 Unique 조건(모든 인덱스 구성컬럼이 '=' 조건)으로 액세스할 때도 테이블 Prefetch 실행계획이 나타날 수 있다. 이때 인덱스는 Range Scan으로 액세스한다.
  테이블 Prefetch 실행계획이 안 나타날 때는 Unique Scan으로 액세스한다.

세 번째 경우로써 테이블 Prefetch가 작동하는 경우는 흔치 않은데, 아래 쿼리를 통해 확인해보자.
'지분보고_PK' 인덱스는 [회사코드 + 보고서구분코드 + 최초보고일자 + 보고서id + 보고일련번호] 순으로 구성돼 있고, '지분보고내역'으로부터 '지분보고'로 조인할 때 PK 인덱스 컬럼이 모두 '=' 조건으로
제공되므로 PK 인덱스를 Unique Scan 하고 있다.

```
select /*+ ordered use_nl(b) cardinality(a 110) */
       max(a.최종보고일자), avg(b.주권주식수), count(*)
from   지분보고내역 a, 지분보고 b
where  b.회사코드     = a.회사코드
and    b.보고서구분코드 = a.보고서구분코드
and    b.최초보고일자  = a.최초보고일자
and    b.보고서id    = a.보고서id
and    b.보고일련번호  = a.보고일련번호
and    a.회사코드 between '000010' and '000011'

Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    1  SORT AGGREGATE (cr=1282 pr=0 pw=0 time=4779 us)
  332    NESTED LOOPS (cr=1282 pr=0 pw=0 time=4703 us)
  332      TABLE ACCESS BY INDEX ROWID 지분보고내역 (cr=284 pr=0 pw=0 time=1042 us)
  332        INDEX RANGE SCAN 지분보고내역_PK (cr=5 pr=0 pw=0 time=368 us)
  332      TABLE ACCESS BY INDEX ROWID 지분보고 (cr=998 pr=0 pw=0 time=2987 us)
  332        INDEX UNIQUE SCAN 지분보고_PK (cr=666 pr=0 pw=0 time=1897 us)
``` 
위 SQL에 cardinality 힌트를 사용한 것을 볼 수 있는데, 드라이빙 집합의 카디널리티를 조금씩 증가시키며 테스트했을 때 110까지는 Inner 테이블 액세스가 전통적인 방식대로 NL 조인 아래쪽에 위치한다.

그러다가 드라이빙 집합의 카디널리티를 111로 지정하는 순간 아래와 같이 실행계획이 바뀌는 것을 관찰할 수 있다. PK 인덱스를 Range Scan 하면서 테이블을 NL 조인 위쪽에서 액세스하는 것을 확인하기 바란다.

```
select /*+ ordered use_nl(b) cardinality(a 111) */ ...

Rows   Row Source Operation
-----  ---------------------------------------------------------------------
    1  SORT AGGREGATE (cr=1231 pr=0 pw=0 time=5369 us)
  332    TABLE ACCESS BY INDEX ROWID 지분보고 (cr=1231 pr=0 pw=0 time=2987 us)
  665      NESTED LOOPS (cr=952 pr=0 pw=0 time=5985 us)
  332        TABLE ACCESS BY INDEX ROWID 지분보고내역 (cr=284 pr=0 pw=0 time=1036 us)
  332          INDEX RANGE SCAN 지분보고내역_PK (cr=5 pr=0 pw=0 time=365 us)
  332        INDEX UNIQUE SCAN 지분보고_PK (cr=668 pr=0 pw=0 time=2316 us)
``` 
이 외에도 몇 가지 다른 테스트를 통해서도, Inner 쪽 테이블을 Unique 조건으로 액세스할 때 테이블 Prefetch 실행계획이 나타나는 것을 확인할 수 있었다. 안타깝게도 정확한 규칙을 찾는 데는 실패했다.

<br/>

## (7) 배치 I/O
- 공식적인 명칭은 아니며, 관련 힌트(nlj_batching, no_nlj_batching)에 착안해 붙인 이름이다.

오라클 11g에서 시작된 배치 I/O 메커니즘에 대해서는 아직 공식적으로 알려진 바가 없지만, 아래 실행계획이 의미하는 바와 같이 Inner쪽 인덱스만으로 조인(id=2)을 하고 나서 테이블과의 조인(id=1)은
나중에 일괄(batch) 처리하는 메커니즘인 것으로 추정된다.

테이블 액세스를 나중에 하지만 부분범위처리는 정상적으로 작동한다.
따라서 인덱스와의 조인을 모두 완료하고 나서 테이블을 액세스하는 것이 아니라 일정량씩(아마도 Fetch Call 단위) 나누어 처리하는 것을 알 수 있다.
```
Execution Plan
------------------------------------------------------------------------------------
0        SELECT STATEMENT Optimmizer=ALL_ROWS (Cost=16 Card=14 Bytes=2K)
1    0     NESTED LOOPS
2    1       NESTED LOOPS (Cost=16 Card=14 Bytes=2K)
3    2         TALBE ACCESS (FULL) OF 'EMP' (TABLE) (Cost=2 Card=14 Bytes=1K)
4    2         INDEX (UNIQUE SCAN) OF 'PK_DEPT' (INDEX (UNIQUE)) (Cost=0 Card=1)
5    1       TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (TABLE) (Cost=1 Card=1 Bytes=30)
```
배치 I/O 방식을 표현한 위 실행계획을 풀어서 설명하면 아래와 같다. (오라클에서 아직 공식적으로 밝힌 바 없으며, 경험적 사실을 설명하려고 세운 가설임을 밝힌다.)

- [1] 드라이빙 테이블에서 일정량의 레코드를 읽어 Inner쪽 인덱스와 조인하면서 중간 결과집합(sub-resultset)을 만든다.
- [2] 중간 결과집합이 일정량 쌓이면 inner쪽 테이블 레코드를 액세스한다. 이때 테이블 블록을 버퍼 캐시에서 찾으면 바로 최종 결과집합에 담고, 못 찾으면 중간 집합에 남겨 둔다.
- [3] [2]번 과정에서 남겨진 중간 집합에 대한 Inner쪽 테이블 블록을 디스크로부터 읽는다. 이때 Multiple Single Block I/O 방식을 사용한다.
- [4] 버퍼 캐시에 올라오면 테이블 레코드를 읽어 최종 결과집합에 담는다.
- [5] 모든 레코드를 처리하거나 사용자가 Fetch Call을 중단할 때까지 1 ~ 4번 과정을 반복한다.

이것은 Outer 테이블로부터 액세스되는 Inner쪽 테이블 블록에 대한 디스크 I/O Call 횟수를 줄이기 위해, 테이블 Prefetch에 이어 추가로 도입된 메커니즘이다.
이 메커니즘이 작동하도록 유도하려면 nlj_batching 힌트를 사용하면 된다. 만약 이 방식을 원하지 않을 때는 no_nlj_batching 또는 nlj_prefetch 힌트를 사용하면 된다.
그러면 테이블 Prefetch 방식으로 전환된다.

주목할 것은, 위 방식을 사용할 때 Inner 쪽 테이블 블록이 모두 버퍼 캐시에서 찾아지지 않으면(버퍼 캐시 히트율(100%) 즉, 실제 배치 I/O가 작동한다면 데이터 정렬 순서가 달라질 수 있다는 사실이다.
모두 버퍼 캐시에서 찾을 때는 (버퍼 캐시 히트율 = 100%) 이전 메커니즘과 똑같은 정렬 순서를 보인다.
테이블 Prefetch 방식이나 전통적인 방식으로 NL 조인할 때는 디스크 I/O가 발생하든 안 하든 데이터 정렬 순서가 항상 일정하다.

<br/>

## (8) 버퍼 Pinning 효과
앞서 설명한 테이블 Prefetch와 배치 I/O 기능이 도입된 시점과 맞물려 버퍼 Pinning 기능에 변화가 생기다 보니 9i와 11g에서 나타난 NL 조인 실행계획 변화를 버퍼 Pinning 효과로 설명하는 자료들을
종종 볼 수 있는데, 이 둘 간에 직접적인 연관성은 없다.

### [8i에서 나타난 버퍼 Pinning 효과]
테이블 블록에 대한 버퍼 Pinning 기능이 작동하기 시작했다. 단, 하나의 버퍼 블록만 Pinning한다. 그리고 하나의 Fetch Call을 완료하는 순간 Pin을 해제한다.

이 기능은 NL 조인에서 Non-Unique 조건으로 Inner쪽 테이블을 액세스할 때도 똑같이 작용한다.
따라서 Inner쪽 인덱스를 통해 액세스되는 테이블 블록이 계속 같은 블록을 가리키면 논리 I/O(=CR Gets)가 추가로 발생하지 않는다.

중요한 사실은, 하나의 Outer 레코드에 대한 Inner 쪽과의 조인을 마치고 다른 레코드를 읽기 위해 Outer 쪽으로 돌아오는 순간 Pin을 해제한다는 점이다.
따라서 8i에서 Pinning 기능을 정확히 테스트하려면 Inner 쪽을 한 번 액세스할 때마다 여러 개의 테이블 레코드를 읽도록 데이터를 구성해야 한다.
클러스터링 팩터가 좋아야 그 효과도 확실하며, 9i 이상 버전에서도 마찬가지다. NL 조인에서도 하나의 Fetch Call을 완료하면 Pin을 해제한다.

### [9i에서 버퍼 Pinning 효과]
9i부터 Inner 쪽 인덱스 루트 블록에 대한 버퍼 Pinning 효과가 나타나기 시작했다. 단, 두 번째 액세스되는 순간 Pinning 한다.

테이블 블록 버퍼에 대한 Pinning도 8i와 똑같이 작동한다. 앞에서도 얘기했지만 Inner쪽을 한 번 액세스할 때마다 Non-Unique 인덱스로 여러 개 테이블 레코드르 읽을 때라야 이 기능이 효과를 발휘한다.
9i부터 Inner 쪽이 Non-Unique 인덱스일 때는 테이블 액세스가 항상 NL 조인 위쪽에 올라가므로(→ 테이블 Prefetch 포맷) 이때는 항상 버퍼 Pinning 효과가 나타나는 셈이다.

반면, 테이블 Prefetch에서 설명했듯이 Inner 쪽 Unique 인덱스를 Unique 조건으로 액세스할 때는 테이블 액세스가 NL 조인 위쪽으로 잘 올라가지 않는다.
그리고 이제는 Unique 액세스이므로 Inner쪽에서 한 건만 읽고 바로 Outer 테이블 쪽으로 돌아간다. 따라서 버퍼 Pinning 효과가 나타날 수 없다.

그러다 보니 테이블 액세스가 NL 조인 위쪽으로 올라가는 9i에서의 실행계획 변화를 버퍼 Pinning 효과와 연관시켜 해석하는 오류를 범하기 쉽다.

Inner 쪽 Unique 인덱스를 Unique 조건으로 액세스할 때도 테이블 액세스가 NL 조인 위쪽으로 올라가는 경우가 있다고 했고, 그냥 지나쳐 왔지만 그때의 버퍼 Pinning 효과를 이미 앞에서 보았다.
테이블 Prefetch를 설명하면서 예시했던 트레이스 결과에서 '지분보고' 테이블을 NL 조인 아래쪽에서 액세스할 때보다 위쪽에서 액세스할 때 53개 블록을 덜 읽은 것을 확인하기 바란다/
총 332번 액세스하는 동안 NL 조인 아래쪽에 있을 때는 332(=998-666)개 블록을 읽었고, 위쪽에 있을 때는 279(=1,231-952)개 블록을 읽었다.

### [10g에서 버퍼 Pinning 효과]
Inner 쪽 인덱스 루트 블록과 테이블 블록을 Pinning하는 기능이 여전히 작동하면서도 한 가지 새로운 기능이 추가되었다.
Inner 쪽 테이블을 Index Range Scan을 거쳐 NL 조인 위쪽에서 액세스할 때는, 하나의 Outer 레코드에 대한 Inner쪽과의 조인을 마치고 Outer 쪽으로 돌아오더라도 테이블 블록에 대한 Pinning
상태를 유지한다.

만약 Inner 쪽 테이블이 한 블록뿐일 때 10g에서의 새로운 버퍼 Pinning 기능이 작동한다면 NL 조인이 진행되는 동안 논리적인 블록 I/O는 단 1회만 발생할 것이다.
실제 그런지 테스트해보자.
```
create table dept as select * from scott.dept;

alter table dept add constraint dept_pk primary key(deptno);

create table t_emp
as
select * from scott.emp, (select rownum no from dual connnect by level <= 100000)
order by dbms_random.value;
```
dept 테이블을 scott에 있는 것과 똑같이 만들었으므로 4건뿐이고, 테이블 블록도 단 한 블록이다.
이 테이블을 여러 번 반복 액세스하려고 emp 테이블을 10만 번 복제해 149만 개 레코드를 가진 t_emp 테이블을 만들었다. 이제 SQL 트레이스를 통해 10g에서의 새로운 버퍼 Pinning 효과를 확인해 보자.
```
select /*+ ordered use_nl(d) */ count(e.ename), count(d.dname)
from   t_emp e, dept d
where  d.deptno = e.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2    47.203       47.378       0   1409213          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4    47.203       47.378       0   1409213          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=1409213 pr=0 pw=0 time=47377681 us)
1400000    NESTED LOOPS (cr=1409213 pr=0 pw=0 time=58800039 us)
1400000      TABLE ACCESS FULL T_EMP (cr=9211 pr=0 pw=0 time=4200032 us)
1400000      TABLE ACCESS BY INDEX ROWID DEPT (cr=1400002 pr=0 pw=0 time=39143015 us)
1400000        INDEX UNIQUE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=15575397 us)
```
우선 인덱스 루트 블록을 버퍼 Pining한 사실이 쉽게 확인된다. Inner 쪽 인덱스(dept_pk)로 140만 번 액세스가 일어났는데, 논리적인 블록 I/O는 고작 2회뿐이다.
두 번째 액세스되는 순간부터 Pinning 되므로 최소한 두 블록 I/O는 일어난다.
10g까지는 루트 블록만 Pinning 대상이지만, 여기선 dept 테이블이 4건짜리 작은 테이블이므로 인덱스 루트 블록이 곧 리프 블록이다. 그래서 두 블록 외에 추가적인 I/O는 발생하지 않은 것이다.

테이블 블록은 어떤가? Inner 쪽을 액세스할 때마다 한 건씩만 읽고 Outer 쪽으로 돌아가므로 테이블 블록에는 Pinning 효과가 나타나지 않았다.
따라서 140만 번 액세스하는 동안 고스란히 140만(=1,400.002-2)번 블록 I/O를 일으켰다. 참고로, 9i에서 테스트해 봐도 위와 100% 똑같은 결과가 나타난다.

이번에는 위 SQL과 결과는 똑같으면서 Unique 인덱스를 Range Scan하도록 조건절을 아래와 같이 between으로 바꿔보자.

```
select /*+ ordered use_nl(d) */ count(e.ename), count(d.dname)
from   t_emp e, dept d
where  d.deptno between e.deptno and e.deptno + 1

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2    25.609       25.716       0      9214          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4    25.609       25.716       0      9214          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=1409213 pr=0 pw=0 time=25715571 us)
1400000    TABLE ACCESS BY INDEX ROWID DEPT (cr=9214 pr=0 pw=0 time=58800062 us)
2800001      NESTED LOOPS (cr=9213 pr=0 pw=0 time=16995984 us)
1400000        TABLE ACCESS FULL T_EMP (cr=9211 pr=0 pw=0 time=4200027 us)
1400000        INDEX RANGE SCAN DEPT_PK (cr=2 pr=0 pw=0 time=15934860 us)
```
Unique 인덱스이지만 Range Scan 방식으로 액세스하였고, 테이블 액세스가 NL 조인 위쪽에 위치했다. 그리고 dept 테이블 블록은 단 하나뿐이다.
따라서 140만개 레코드를 모두 처리하는 동안 버퍼 Pinning 상태가 유지됨으로써 블록 I/O는 단 1(=9,124-9,213)회 발생하였다.
그렇지만 버퍼를 Pinning한 상태에서 내부적으로 140만 번이나 액세스했기 때문에 시간은 25초에 이른다. 9,214개 블록 I/O에 비하면 과도한 시간이다.

인덱스 루트 블록도 여전히 Pinning 한 상태로 액세스한 것을 볼 수 있다.

여기서 count 함수를 사용한 이유는, 버퍼 Pinning 효과는 하나의 데이터베이스 Call 내에서만 유효하기 때문이다.
만약 count 함수를 쓰지 않아 여러 번 Fetch Call이 발생한다면 Pin을 해제했다가 다시 Pin을 걸어야 하기 때문에 약간의 블록 I/O가 추가로 발생한다.

파라미터를 바꿔 테이블 Prefetch가 작동하지 못하도록 하면 위 쿼리의 Inner쪽 테이블 액세스도 NL 조인 아래쪽에서 액세스한다.
따라서 '=' 조건으로 조인하던 원래 쿼리처럼, Inner 쪽을 액세스할 때마다 한 건씩만 읽고 Outer 쪽으로 돌아가므로 테이블 블록은 Pinning 효과가 나타날 수 없다.

참고로, 아래는 9i에서 테스트한 결과다. 한 블록을 반복 액세스했지마나 한 건씩만 읽고 Outer 테이블로 돌아갔기 때문에 NL 조인 위쪽에 있음에도 Pinning 효과가 나타나지 않았다.
(정확히 얘기하면, 9i에서도 10g와 같은 효과가 나타날 때가 있다.
v$sesstat을 통해 모니터링을 해 보면, buffer is pinned count에 전혀 변화가 없는 상태에서 session logical reads와 consistent gets 항목만 계속 변한다.
하지만 현상이 일관성 없게 나타나기 때문에 그것이 버퍼 Pinning 효과인지, 버그인지, 아니면 알려지지 않은 또 다른 메커니즘인지 알 수가 없다.)

```
Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2    56.109       60.066    8601   1409214          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4    56.109       60.067    8601   1409214          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=1409214 r=8601 w=0 time=60066461 us)
1400000    TABLE ACCESS BY INDEX ROWID DEPT (cr=1409214 r=8601 w=0 time=52257524 us)
2800001      NESTED LOOPS (cr=9213 r=8601 w=0 time=34061892 us)
1400000        TABLE ACCESS FULL T_EMP (cr=9211 r=8601 w=0 time=6566852 us)
1400000        INDEX RANGE SCAN DEPT_PK (cr=2 r=0 w=0 time=8535225 us)
```
조인 순서를 바꿔 dept를 드라이비아면서 t_emp 쪽 deptno에 대한 Non-Unique 인덱스로 조인한다면, 테이블 액세스가 NL 위쪽에 있든 아래쪽에 있든 8i 이후 버전에선 모두 버퍼 Pinning 효과가 나타난다.
물론 10g에서 I/O가 좀 더 줄 가능성이 있다.

### [11g에서 나타난 버퍼 Pinning 효과]
11g에서는 User Rowid로 테이블 액세스할 때도 버퍼 Pinning 효과가 나타난다.

또한 NL 조인에서 Inner쪽 루트 아래 인덱스 블록들도 Pinning하기 시작했다.
배치 I/O 기능이 나타남과 동시에 이 기능이 추가되다 보니 11g에서의 NL 조인 실행계획 변화가 인덱스 블록 버퍼 Pinning과 관련 있다고 오해하기 쉽다.

no_nlj_batching 또는 nlg_prefetch 힌트를 사용해 테이블 Prefetch 포맷으로 실행계획을 유도해 보면 이때도 인덱스 블록 I/O가 적게 나타나는 것을 관찰할 수 있다.
즉, 인덱스 블록 버퍼 Pinning 효과는 배치 I/O 실행계획과 상관없이 나타난다.

테스트를 위해 t_emp와 t_dept 테이블을 아래와 같이 만들어 보자.
```
create table t_emp
as
select * from scott.emp. (select rownum no from dual connect by level <= 1000)
order by no, deptno ;

create table t_dept
as
select * from scott.dept, (select rownum no from dual connect by level <= 1000);

alter table t_dept add constraint t_dept_pk primary key (no, deptno);
```
아래와 같이 인덱스 높이(height)를 조회해 보면 브랜치가 1단계이므로 리프 블록까지 총 2단계임을 알 수 있다.
```
SQL> select blevel from user_indexes where index_name = 'T_DEPT_PK';

      BLEVEL
------------
           1
```
이제 t_emp 기준으로 t_dept 테이블과 배치 I/O 방식으로 NL 조인해 보자.
```
select /*+ ordered use_nl_with_index(d) nlj_batching(d) */
       count(e.ename), count(d.dname)
from   t_emp e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.328        0.336       0     14167          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.328        0.336       0     14167          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=14167 pr=0 pw=0 time=0 us)
  14000    NESTED LOOPS (cr=14167 pr=0 pw=0 time=3573 us)
  14000      NESTED LOOPS (cr=167 pr=0 pw=0 time=1898 us cost=2085 size=816 card=12)
  14000        TABLE ACCESS FULL T_EMP (cr=95 pr=0 pw=0 time=222 us cost=26 ...)
  14000        INDEX UNIQUE SCAN T_DEPT_PK (cr=72 pr=0 pw=0 time=0 us cost=0 ...)
  14000      TABLE ACCESS BY INDEX ROWID T_DEPT (cr=14000 pr=0 pw=0 time=0 us ...)
```
Inner 쪽을 14,000번 액세스하는 동안 t_dept_pk 인덱스에 대한 블록 I/O는 72번만 발생하였다. 버퍼 Pinning 기능이 작동한 것이다.

이번에는 테이블 Prefetch 방식으로 액세스해 보자.
```
select /*+ ordered use_nl_with_index(d) nlj_prefetch(d) */
       count(e.ename), count(d.dname)
from   t_emp e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=14167 pr=0 pw=0 time=0 us)
  14000    TABLE ACCESS BY INDEX ROWID T_DEPT (cr=14167 pr=0 pw=0 time=3266 us ...)
  28001      NESTED LOOPS (cr=167 pr=0 pw=0 time=874 us cost=2085 size=816 card=12)
  14000        TABLE ACCESS FULL T_EMP (cr=95 pr=0 pw=0 time=248 us cost=26 ...)
  14000        INDEX UNIQUE SCAN T_DEPT_PK (cr=72 pr=0 pw=0 time=0 us cost=0 ...)
```
테이블 Prefetch 방식으로 액세스하더라도 인덱스 블록에 대한 버퍼 Pinning 효과는 똑같이 나타났다.
11g에서의 실행계획 변화(→ 배치 I/O 포맷)가 인덱스 블록에 대한 버퍼 Pinning과 관련이 없음을 알 수 있다.

이번에는 드라이빙 테이블이 아래와 같이 무순위로 정렬되도록 다시 생성해 보자. 앞에서는 조인 컬럼 no, deptno 순으로 정렬했던 것을 확인하기 바란다.
```
drop table t_emp purge;

create table t_emp
as
select * from scott.emp. (select rownum no from dual connect by level <= 1000)
order by dbms_random.value ;
```
이제 같은 SQL을 차례로 실행시켜 보자. 먼저 배치 I/O 포맷을 테스트해 보면 아래와 같이 인덱스 블록에 대한 버퍼 Pinning 효과가 사라진 것을 볼 수 있다.
```
select /*+ ordered use_nl_with_index(d) nlj_batching(d) */
       count(e.ename), count(d.dname)
from   t_emp e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno

Call      count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.344        0.344       0     28098          0      1
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.344        0.344       0     28098          0      1
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=28098 pr=0 pw=0 time=0 us)
  14000    NESTED LOOPS (cr=28098 pr=0 pw=0 time=3723 us)
  14000      NESTED LOOPS (cr=14098 pr=0 pw=0 time=1987 us cost=2359 size=884 ...)
  14000        TABLE ACCESS FULL T_EMP (cr=94 pr=0 pw=0 time=221 us cost=26 ...)
  14000        INDEX UNIQUE SCAN T_DEPT_PK (cr=14004 pr=0 pw=0 time=0 us cost=0 ...)
  14000      TABLE ACCESS BY INDEX ROWID T_DEPT (cr=14000 pr=0 pw=0 time=0 us ...)
```
이유가 무엇일까? 드라이빙 테이블을 무순위로 정렬한 것이 원인이다. 테이블과 마찬가지로 인덱스 블록도 버퍼 Pinning 효과가 나타나려면 같은 값으로 반복 액세스해야 한다.
아래와 같이 테이블 Prefetch 포맷을 사용하더라도 마찬가지다.
```
select /*+ ordered use_nl_with_index(d) nlj_prefetch(d) */
       count(e.ename), count(d.dname)
from   t_emp e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno
  
Rows     Row Source Operation
-------  ---------------------------------------------------------------------
      1  SORT AGGREGATE (cr=28098 pr=0 pw=0 time=0 us)
  14000    TABLE ACCESS BY INDEX ROWID T_DEPT (cr=28098 pr=0 pw=0 time=3341 us ...)
  28001      NESTED LOOPS (cr=14098 pr=0 pw=0 time=870 us cost=2359 size=884 ..)
  14000        TABLE ACCESS FULL T_EMP (cr=94 pr=0 pw=0 time=248 us cost=26 ...)
  14000        INDEX UNIQUE SCAN T_DEPT_PK (cr=14004 pr=0 pw=0 time=0 us cost=0 ...)
```
count 집계 함수를 쓰지 않고 결과집합을 출력하는 쿼리라면 아래와 같이 드라이빙 집합을 Inner쪽 인덱스 컬럼 순으로 정렬하고 나서 NL 조인함으로써 블록 I/O를 줄일 수 있다.
```
select /*+ ordered use_nl_with_index(d) */ *
from   (select /*+ no_merge */ * from t_emp order by no, deptno) e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno
```
단 이 경우 부분범위처리가 안 되므로, 필요하다면 아래와 같이 미리 정렬된 인덱스를 이용할 수 있다.
```
create index t_emp_idx on t_emp(no, deptno);

select /*+ ordered use_nl_with_index(d) index(e t_emp_idx) */ *
from   t_emp e, t_dept d
where  d.no = e.no
and    d.deptno = e.deptno
```