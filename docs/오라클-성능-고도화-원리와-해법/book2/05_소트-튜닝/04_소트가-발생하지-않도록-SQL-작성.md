# 04. 소트가 발생하지 않도록 SQL 작성

데이터 모델 측면에선 이상이 없는데, 불필요한 소트가 발생하도록 SQL을 작성하는 경우가 있다.
예를 들어, 아래처럼 union을 사용하면 옵티마이저는 상단과 하단의 두 집합 간 중복을 제거하려고 sort unique 연산을 수행한다.
```
SQL> select empno, job, mgr from emp where deptno = 10
  2  union
  3  select empno, job, mgr from emp where deptno = 20;

--------------------------------------------------------------------------------
| Id  | Operation              | Name | Rows  | Bytes | Cost  (%CPU)| Time     |
--------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |      |    10 |   190 |     8   (63)| 00:00:01 |
|   1 |   SORT UNIQUE          |      |    10 |   190 |     8   (63)| 00:00:01 |
|   2 |     UNION-ALL          |      |       |       |             |          |
|   3 |       TABLE ACCESS FULL| EMP  |     5 |    95 |     3    (0)| 00:00:01 |
|   4 |       TABLE ACCESS FULL| EMP  |     5 |    95 |     3    (0)| 00:00:01 |
--------------------------------------------------------------------------------
```
하지만 PK 컬럼인 empno를 select-list에 포함하므로 두 집합간에는 중복 가능성이 전혀 없다. 따라서 union all을 사용해야 한다.
union all은 중복을 확인하지 않고 두 집합을 단순히 결합하므로 소트 부하가 없기 때문이다.
(select-list에 empno가 없다면 10번과 20번 부서에 job, mgr이 같은 사원이 있을 수 있어 union과 union all의 의미가 달라진다.)

그럼에도 위와 같이 union을 즐겨 사용하는 개발자를 종종 볼 수 있다.
둘 중 하나에 속하는데, union과 union all 처리 방식의 차이를 모르거나 집합 개념이 부족해 혹시 중복이 발생할지도 모른다는 불안감 때문에 그렇게 하는 경우다.

distinct를 사용하는 경우도 매우 흔한데, 대부분 exists 서브커리로 대체함으로써 소트 연산을 없앨 수 있다. 가장 흔한 사례를 하나 들어보기로 하자.

아래는 특정 지역(:reg)에서 특정월(:yyyymm) 이전에 과금이 발생했던 연월을 조회하는 쿼리다. 야간 배치 프로그램에서 발췌한 것으로서, 이 쿼리 결과를 이용해 다른 많은 작업을 수행한다.
```
select distinct 과금연월
from   과금
where  과금연월 <= :yyyymm
and    지역 like :reg || '%'

call      count       cpu      elapsed      disk     query   current      rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1      0.00         0.00         0         0         0         0
Execute       1      0.00         0.00         0         0         0         0
Fetch         4     27.65        98.38     32648   1586208         0        35
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         6     27.65        98.38     32648   1586208         0        35

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     35  HASH UNIQUE (cr=1586208 pr=32648 pw=0 time=98704640 us)
9845517    PARTITION RANGE ITERATOR PARTITION: 1 KEY (cr=1586208 pr=32648 ... )
9845517      TABLE ACCESS FULL 과금 (cr=1586208 pr=32648 pw=0 time=70155864 us)
```
입력한 과금연월(yyyymm) 이전에 발생한 과금 데이터를 모두 스캔하는 동안 1,586,208개 블록을 읽었고, 무려 1,000만 건에 가까운 레코드에서 중복 값을 제거하고 고작 35건을 출력했다.
매우 비효율적인 방식으로 수행되었고, 쿼리 소요시간은 1분 38초다.

각 월별로 과금이 발생한 적이 있는지 여부만 확인하면 되므로 쿼리를 아래처럼 바꿀 수 있다.
소량의 데이터만을 갖는 연월테이블을 먼저 드라이빙해 과금 테이블을 exists 서브쿼리로 필터링하는 방식이다.
```
select 연월
from   연월테이블 a
where  연월 <= :yyyymm
and    exists (
  select 'x'
  from   과금
  where  과금연월 = a.연월
  and    지역 like :reg || '%'
)

call      count       cpu      elapsed      disk     query   current      rows
-------- ------ --------- ------------ --------- --------- --------- ---------
Parse         1      0.00         0.00         0         0         0         0
Execute       1      0.00         0.00         0         0         0         0
Fetch         4      0.00         0.01         0        82         0        35
-------- ------ --------- ------------ --------- --------- --------- ---------
Total         6      0.00         0.01         0        82         0        35

Rows     Row Source Operation
-------  ---------------------------------------------------------------------
     35  NESTED LOOPS SEMI (cr=82 pr=0 pw=0 time=19568 us)
     36    TABLE ACCESS FULL 연월테이블 (cr=6 pr=0 pw=0 time=557 us)
     35    PARTITION RANGE ITERATOR PARTITION: KEY KEY (cr=76 pr=0 pw=0 time=853 us)
     35      INDEX RANGE SCAN 과금_N1 (cr=76 pr=0 pw=0 time=683 us)
```
exists 서브쿼리의 가장 큰 특징은, 메인 쿼리로부터 건건이 입력 받은 값에 대한 조건을 만족하는 첫 번째 레코드를 만나는 순간 true를 반환하고 서브쿼리 수행을 마친다는 점이다.
따라서 과금 테이블에 [과금연월 + 지역] 순으로 인덱스를 구성해 주기만 하면 가장 최적으로 수행될 수 있다.

그 결과, 소트를 발생시키지 않았음은 물론 82개 블록만 읽고 0.01초 만에 수행을 완료했다. 물론 연월(yyyymm)을 관리하는 테이블이 따로 있을 때 적용 가능한 기법이다.
아래와 같이 일자 및 연월 테이블을 미리 생성해 두면 여러모로 활용가치가 높다.
```
create table 일자테이블
as
select to_char(ymd, 'yyyymmdd') ymd
     , to_char(ymd, 'yyyy') year
     , to_char(ymd, 'mm') month
     , to_char(ymd, 'dd') day
     , to_char(ymd, 'dy') weekday
     , to_char(next_day(ymd, '일')-7, 'w') week_monthly
     , to_number(to_char(next_day(ymd, '일')-7, 'ww') ) week_yearly
from (
  select to_date('19691231', 'yyyymmdd') + rownum ymd
  from   dual connect by level <= 365*100
) ;

create table 연월테이블
as
select substr(ymd, 1, 6) yyyymm
     , min(ymd) first_day, max(ymd) last_day
     , min(year) year, min(month) month
from   일자테이블
group by substr(ymd, 1, 6) ;
```
과금연월 콤보(combo) 박스에 과금이 발생했던 연월말 보여지도록 하고 싶을 때, 만약 위와 같은 튜닝 기법을 적용하지 않는다면 어떤 일이 벌어질까?

조회 버튼을 누르기도 전, 화면이 열리는 단계에서 이미 성능 때문에 고생할 것이고, 아주 빈번히 조회되는 화면이라면 이 때문에 시스템은 몸살을 앓게 될 것이다.