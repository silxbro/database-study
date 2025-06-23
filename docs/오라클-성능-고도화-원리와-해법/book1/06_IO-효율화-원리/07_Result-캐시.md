# 07. Result 캐시

Result 캐시는 버퍼 캐시에 위치하지 않고 Shared Pool에 위치하지만 시스템 I/O 발생량을 최소화하는 데 도움이 되는 기능이다.

DB 버퍼 캐시는 쿼리에서 자주 사용되는 블록들을 캐싱해 두는 메모리 공간이다. DB 버퍼 캐시에 캐싱된 블록을 읽는 것도 때에 따라서는 고비용 구조이다.
따라서 작은 테이블을 메모리 버퍼 캐시에서 읽더라도 반복 액세스가 많이 일어나면 좋은 성능을 기대하기 어렵다.
버퍼 캐시 히트율이 낮을 수밖에 없는 대용량 데이터 쿼리라면 더더욱 I/O 효율화를 위한 튜닝에 곤란을 겪게 되는데, 집계 테이블을 따로 설계하거나 Materialized View를 생성하는 것 외에 별다른 I/O
효율화 방안이 없는 경우가 종종 있다.

이에 오라클은, 한번 수행한 쿼리 또는 PL/SQL 함수의 결과값을 Result 캐시에 저장해 두는 기능을 11g 버전부터 제공하기 시작했다.
**[1] DML이 거의 발생하지 않는 테이블을 참조**하면서, **[2] 반복 수행 요청이 많은** 커리에 이 기능을 사용하면 I/O 발생량을 현격히 감소시킬 수 있다.

Result 캐시 메모리는 다음 두 가지 캐시 영역으로 구성된다.

- SQL Query Result 캐시 : SQL 쿼리 결과를 저장
- PL/SQL 함수 Result 캐시 : PL/SQL 함수 결과값을 저장
  <br/><br/>

자세한 사용법을 알아보기에 앞서 Result 캐시를 위해 추가된 파라미터들을 살펴보자.

|구분|기본값|설명|
|:---|:---:|:---|
|result_cache_mode|manual|Result 캐시 등록 방식을 결정<br/>- manual : result_cache 힌트를 명시한 SQL만 등록<br/>- force : no_result_cache 힌트를 명시하지 않은 모든 SQL을 등록|
|result_cache_max_size|N/A|SGA 내에서 result_cache가 사용할 메모리 총량을 바이트로 지정.<br/>0으로 설정하면 이 기능이 작동하지 않음|
|result_cache_max_result|5|하나의 SQL 결과집합이 전체 캐시 영역에서 차지할 수 있는 최대 크기를 %로 지정|
|result_cache_remote_expiration|0|remote 객체의 결과를 얼마 동안 보관할지를 분 단위로 지정.<br/>remote 객체는 result 캐시에 저장하지 않도록 하려면 0으로 설정.|

result_cache_max_size를 DB 관리자가 직접 지정하지 않으면, 아래 규칙에 따라 오라클이 자동으로 값을 할당한다.

- SGA와 PGA를 통합 관리하는 11g 방식으로 SGA 메모리를 관리하면, memory_target으로 설정된 값의 0.25%를 Result 캐시를 위해 사용한다.
- sga_target 파라미터를 사용하는 10g 방식으로 SGA 메모리를 관리하면, 그 값의 0.5%를 Result 캐시를 위해 사용한다.
- 과거처럼 shared_pool_size를 수동으로 설정하면 그 값의 1%를 Result 캐시를 위해 사용한다.
- 어떤 방식을 사용하든 Result 캐시가 사용할 수 있는 최대 크기는 Shared Pool의 75%를 넘지 않도록 오라클이 관리한다.

Result 캐시는 SGA의 Shared Pool에 저장된다. SGA 영역이므로 모든 세션에서 공유할 수 있고, 인스턴스를 재기동하면 당연히 초기화 된다.
공유영역에 위치하므로 래치가 필요한데, 11g에서 아래 두 가지 래치가 추가된 것을 발견할 수 있다.

- Result Cache: Latch
- Result Cache: SO Latch

지금부터 이 유용한 기능을 실제 어떻게 사용하는지 알아보자.
Force 모드일 때는 no_result_cache 힌트를 사용하지 않은 모든 SQL을 대상으로 캐싱을 시도하므로 따로 설명하지 않으며, Manual 모드 중심으로 설명하고자 한다.

Manual 모드에서 쿼리 결과를 캐싱하려면 다음과 같이 result_cache 힌트를 사용하면 된다.

```
SELECT /*+ RESULT_CACHE */ COL, COUNT(*)
FROM   R_CACHE_TEST
WHERE  GUBUN = 7
GROUP BY COL
```

이 힌트가 지정된 쿼리를 수행할 때 오라클 서버 프로세스(내부적으로는 서버 프로세스에 내장된 ResultCache Operator가 수행)는 Result 캐시 메모리를 먼저 찾아보고, 캐싱돼 있다면 그것을 가져다가
결과 집합을 리턴한다. 캐시에서 찾지 못할 때만 쿼리를 수행해 결과를 리턴하고, Result 캐시에도 저장해 둔다.

Result 캐시에서 결과 집합을 찾았을 때는 실제 쿼리를 수행하지 않기 때문에 블록 I/O가 전혀 발생하지 않는다. 직접 수행한 결과를 통해 확인해 보자.

```
SELECT /*+ RESULT_CACHE */ COL, COUNT(*)
FROM   R_CACHE_TEST
WHERE  GUBUN = 7
GROUP BY COL

Call     Count CPU Time Elapsed Time       Disk      Query    Current       Rows
------- ------ -------- ------------ ---------- ---------- ---------- ----------
Parse        1    0.000        0.002          0          0          0          0
Execute      1    0.000        0.000          0          0          0          0
Parse        4    1.000        4.989       9446     104332          0         24
------- ------ -------- ------------ ---------- ---------- ---------- ----------
total        6    1.000        4.990       9446     104332          0         24

Rows    Row Source Operation
------  ----------------------------------------------------
     0  STATEMENT
    24   RESULT CACHE? c4spdfm6xq0r90fufzs49m2rlw (cr=104332 pr=9446 pw=9446 ... )
    24    HASH GROUP BY (cr=104332 pr=9446 pw=9446 time=0 us cost=3 size=315 ... )
100000     TABLE ACCESS BY INDEX ROWID SRC_TEST (cr=104332 pr=9446 pw=9446 ... )
100000      INDEX RANGE SCAN SRC_TEST_X01 (cr=4351 pr=4351 pw=4351 time=14635 ... )
```

처음 쿼리를 실행하자 104,332개의 블록 I/O가 발생했다. Result 캐시에 등록된 쿼리 목록과 사용 현황은 v$result_cache_objects를 통해서 확인해 볼 수 있다.
아래는 이 뷰를 통해 방금 수행한 쿼리의 캐싱 여부를 조회한 것이다.

```
                                                                             SCAN_
TYPE       STATUS     NAME                                         NAMESPACE COUNT
---------- ---------- ------------------------------------------- ---------- -----
Dependency Published  ORAKING.R_CACHE_TEST                                       0
Result     Published  SELECT /*+ RESULT_CACHE */ COL, COUNT(*) ... SQL           0
```

위 결과를 보고 Result 캐시에 등록됐음을 알 수 있다. 이제 쿼리를 다시 수행해 보자.

```
SELECT /*+ RESULT_CACHE */ COL, COUNT(*)
FROM   R_CACHE_TEST
WHERE  GUBUN = 7
GROUP BY COL

Call     Count CPU Time Elapsed Time       Disk      Query    Current       Rows
------- ------ -------- ------------ ---------- ---------- ---------- ----------
Parse        1    0.000        0.000          0          0          0          0
Execute      1    0.000        0.000          0          0          0          0
Parse        4    0.000        0.000          0          0          0         24
------- ------ -------- ------------ ---------- ---------- ---------- ----------
total        6    0.000        0.000          0          0          0         24

Rows    Row Source Operation
------  ----------------------------------------------------
     0  STATEMENT
    24   RESULT CACHE? c4spdfm6xq0r90fufzs49m2rlw (cr=0 pr=0 pw=0 time=0 us)
     0    HASH GROUP BY (cr=0 pr=0 pw=0 time=0 us cost=3 size=316 card=1)
     0     TABLE ACCESS BY INDEX ROWID SRC_TEST (cr=0 pr=0 pw=0 time=0 us ... )
     0      INDEX RANGE SCAN SRC_TEST_X01 (cr=0 pr=0 pw=0 time=0 us cost=1 ... )
```

캐시에 있는 결과를 사용하니까 위와 같이 블록 I/O가 0으로 나타나는 것을 볼 수 있다. 이처럼 블록 I/O 발생 횟수를 줄여주기 때문에 같은 SQL을 반복 수행할 때 성능을 향상시켜 준다.

단, 아래 경우에는 쿼리 결과집합을 캐싱하지 못한다.

- Dictionary 오브젝트를 참조할 때
- Temporary 테이블을 참조할 때
- 시퀀스로부터 CURRVAL, NEXTVAL Pseudo 컬럼을 호출할 때
- 쿼리에서 아래 SQL 함수를 사용할 때
    - CURRENT_DATE
    - CURRENT_TIMESTAMP
    - LOCAL_TIMESTAMP
    - SYS_CONTEXT (with non-constant variables)
    - SYS_GUID
    - SYSDATE
    - SYSTIMESTAMP
    - USERENV (with non-constant variables)

애플리케이션에서는 대부분 바인드 변수를 사용하는데, 이때는 어떻게 결과집합을 캐싱할까? 결론부터 말하면, 각 바인드 변수 값에 따라 개별적으로 캐싱이 이루어진다.
아래는 같은 바인드 변수에 다른 값을 입력하면서 SQL을 두 번 실행한 후 v$result_cache_objects 뷰를 조회한 결과다.

```
NAME                                                           SCAN_COUNT
-------------------------------------------------------------- ----------
select /*+ result_cache */ * from r_cache_test where key = :x           1
select /*+ result_cache */ * from r_cache_test where key = :x           1
```

만약 바인드 변수 값의 종류가 매우 다양하고 그 값들이 골고루 입력된다면 Result 캐시 영역이 특정 SQL로 가득 채워지는 일이 발생할지도 모른다.
하나의 쿼리 결과 집합을 캐싱하려면 다른 캐시 엔트리를 밀어내야 하므로 캐시에 등록되었다가 밀려나는 일이 빈번하게 발생할수록 캐시의 효율성은 떨어진다.
따라서 변수 값의 종류가 매우 다양하고 수행 빈도가 높은 쿼리(→ OLTP성 쿼리들의 특징이기도 함)를 Result 캐시에 등록하는 것은 삼가야 한다. 함수 Result 캐시 기능을 사용할 때도 마찬가지다.

그나마 위와 같은 상황에서 사용 빈도가 높은 캐시 엔트리들을 보호하고 Result 캐시의 효율성을 높일 목적으로 LRU 알고리즘을 사용해 관리한다.
<br/><br/>

오라클은 캐싱된 쿼리가 참조하는 테이블에 변경이 발생하면 해당 캐시 엔트리를 무효화시킴으로써 쿼리 결과에 대한 정합성을 보장한다.
만약 쿼리에서 두 테이블을 참조한다면, 둘 중 하나에 DML이 발생하는 순간 캐싱된 결과 집합이 무효화된다.
현재 발생한 DML이 캐싱된 결과 집합에 영향을 미치지 않더라도 예외 없이 캐시를 무효화시켜 버린다는 사실을 기억할 필요가 있다. 예를 들어, 아래 쿼리 결과가 캐싱되었다고 하자.

```
select /*+ result_cache */ * from emp where ename = 'SCOTT'
```

이때, 아래처럼 위 결과 집합과 무관한 레코드를 삽입하더라도 위에서 캐싱된 결과 집합은 무효화된다.

```
insert into emp (empno, ename, ......) values ( 7935, 'EDWARD', ...... );
commit;  (→ 이때 무효화된다.)
```

파티션 테이블에 DML이 발생할 때도, 변경이 발생한 파티션과 무관한 파티션을 참조하는 쿼리 결과집합까지 무효화시킨다. 즉, 2월 파티션에 DML이 발생하면 1월 파티션을 참조하는 쿼리의 캐시까지 무효화된다.

함수 Result 캐시도 함수에서 참조하는 테이블에 변경이 발생하면 무효화된다. 아래는 함수 결괄르 캐싱하도록 하는 예제다.

```
create or replace function get_team_name (p_team_cd number)
return varchar2
RESULT_CACHE RELIES_ON (r_cache_function)
is
  l_team_name r_cache_function.team_name%type;
begin
  select team_name into l_team_name
  from   r_cache_function
  where  team_cd = p_team_cd;
  return l_team_name;
end;
```

result_cache 옵션을 사용하면 되고, relies_on 절에 지정된 테이블에 변경이 발생할 때마다 캐싱된 함수 결과 값이 무효화된다.
relies_on절에 지정하지 않은 테이블에 DML이 발생하면 함수가 무효화되지 않아 잘못된 결과를 리턴할 수 있으므로 주의가 필요하다.(의도적으로 그렇게 하고자 하는 경우가 아니라면 반드시 지정해야 한다.)

따라서 DML이 자주 발생하는 테이블을 참조하는 쿼리나 함수를 캐싱하도록 하는 것은 시스템 부하를 오히려 가중시킬 수도 있다.
DML이 발생할 때마다 캐시를 관리하는 비용이 추가되고 그 과정에서 래치 경합도 많이 발생하기 때문이다.
그리고 result_cache 힌트를 사용한 쿼리를 수행할 때마다 Result 캐시를 탐색하는 비용이 추가로 발생하는데, 히트율(Hit Ratio)이 높다면 상관없지만 DML이 자주 발생해 히트율이 낮아진다면 쿼리
수행 비용을 더 높게 만드는 요인으로 작용한다.
<br/><br/>

여러 개 쿼리 블록을 서로 연결해 최종 결과 집합을 완성하는 복잡한 형태의 쿼리에서, 특정 쿼리 블록만 캐싱할 수 있다면 Result 캐시의 활용성은 매우 높아질 수 있다.
다행히 오라클이 그런 기능까지 제공한다. 몇 가지 패턴으로 나눠 살펴보자.

먼저, 아래처럼 인라인 뷰에만 result_cache 힌트를 사용하면 어떻게 될까?

```
select *
from   r_cache_test t1,
      ( SELECT /*+ RESULT_CACHE */ ID FROM R_CACHE_TEST2
        WHERE ID = 1) t2
where  t1.id = t2.id
```

인라인 뷰 쿼리만 독립적으로 캐싱이 된다. WITH 구문에서도 그럴까?

```
with wv_test
as (  SELECT /*+ RESULT_CACHE MATERIALIZE */ SUM(ID) RNUM,ID
      FROM   R_CACHE_TEST
      GROUP BY ID )
select *
from   r_cache_test t1,
       wv_test t2
where  t1.id = t2.rnum
```

마찬가지로 독립적으로 캐싱이 가능하다. union all을 이용해 2개 쿼리의 합집합을 구하는 경우를 살펴보자.

```
select sum(val)
from ( select sum(c) val
       from ext_stat_test
       union all
       SELECT /*+ RESULT_CACHE */ SUM(ID+SUM_DATA)
       FROM   R_CACHE_TEST
     )
```

위 쿼리에서 대문자로 표시한 쿼리 부분만 별도로 캐싱된다.
DML이 자주 발생하는 테이블을 참조하는 쿼리는 Result 캐시 대상으로 부적합하다고 했는데, 쿼리가 union all 형태라면 DML 발생 여부에 따라 각 집합별로 캐싱 여부를 선택해 줄 수 있다.
만약 전체 결과를 캐싱하고 싶다면 바깥쪽 select 절에 힌트를 사용하면 된다.

아래처럼 where절에 사용된 서브쿼리만 캐싱하는 기능은 제공되지 않는다.

```
select *
from   r_cache_test
where  id = ( select /*+ result_cache */ id
              from   r_cache_test2
              where  id = 1 )
```

Result 캐시는 DW 뿐 아니라 OLTP 환경에서도 잘 활용하면 반복적인 I/O 요청횟수를 줄이는 데에 기여할 것이다.
이 기능이 효과가 있으려면 기본적으로 쿼리 사용빈도가 높아야 하며, 더불어 아래와 같은 상황에서 효과가 배가될 것이다.

- 작은 결과 집합을 얻으려고 대용량 데이터를 읽어야 할 때
- 읽기 전용의 작은 테이블을 반복적으로 읽어야 할 때
- 읽기 전용 코드 테이블을 읽어 코드명칭을 반환하는 함수

아래 경우에는 Result 캐시 기능 사용을 자제해야 한다.

- 쿼리가 참조하는 테이블에 DML이 자주 발생할 때
- 함수 또는 바인드 변수를 가진 쿼리에서 입력되는 값의 종류가 많고, 그 값들이 골고루 입력될 때

지금까지 설명한 기능은 SGA에 결과 집합을 저장하는 '서버 측 Result 캐시' 기능이고, 클라이언트 메모리에 결과 집합을 저장하는 '클라이언트 측 Result 캐시' 기능도 11g에서 함께 제공된다.
본서는 앞으로 활용 가능성이 높을 것으로 예상되는 '서버 측 Result 캐시' 기능만 소개하였으므로 '클라이언트 측 Result 캐시' 기능에 관심 있다면 오라클 매뉴얼을 통해 따로 기능을 숙지하기 바란다.