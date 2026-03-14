# 10. V$SQL

모든 업무가 그렇듯 정해진 시간 내에 가장 효율적인 방법으로 효과성을 높이려면 전략적 접근 방법이 필요하다.
튜닝 프로젝트에 나갈 때마다 시스템에서 사용 중인 SQL 개수를 세어 보면, 최소한 수천 개가 넘고, 가장 많을 때 20,000여 개에 이르는 것을 본 적도 있다.
아무리 뛰어난 튜닝팀이라 하더라도 그 많은 SQL을 모두 튜닝할 수는 없다. 이런 상황에서 최소 인력으로 튜닝의 효과성을 극대화하려면 어떻게 해야 할까?

\[그림 3-10\](p.225)은 모 통신회사에서 자체 조사한 자료인데, 다른 시스템도 이와 크게 다르지 않을 것이다.
조사 결과에 의하면, 전체 애플리케이션 물량 중 상위 2%에 해당하는 프로그램이 전체 시스템 부하의 70%를 차지한다. 그리고 상위 14%가 전체 부하량의 90%를 차지하는 것을 볼 수 있다.

잘 알려진 파레토 최적의 법칙(Pareto`s Law) 또는 리처드 코치의 80/20 법칙은 튜닝 대상을 선정하는 데 있어서도 똑같이 적용할 수 있다.
수천, 수만 개의 SQL이 있더라도 그 중 대부분은 사용되지 않거나 가끔 사용되는 것들이다.
주기적으로 사용되는 상위 10% 이내의 프로그램만 집중적으로 튜닝하더라도 시스템 안정화 및 고도화를 이룰 수 있는 이유가 여기에 있다.

본 절에서 설명하려는 v$sql은 개별 SQL 커서의 수행 통계를 분석할 목적으로도 많이 활용되지만, 집중 튜닝이 필요한 대상 SQL을 선정하는 데 활용할 수 있는 매우 유용한 도구다.
그뿐만 아니라 튜닝 전후 성능 향상도를 비교할 목적으로 통계를 내는 데도 활용할 수 있다.

v$sql은 라이브러리 캐시에 캐싱돼 있는 각 Child 커서에 대한 수행통계를 보여준다. 그리고 v$sqlarea는 Parent 커서에 대한 수행통계를 나타내며, 많은 컬럼이 v$sql을 group by 해 구한 값이다.
Parent 커서와 Child 커서에 대한 개념은 추후(4장) 자세히 설명한다. v$sql은 쿼리가 수행을 마칠 때마다 갱신되며, 오랫동안 수행되는 쿼리는 5초마다 갱신이 이루어진다.

```
select sql_id, child_number, sql_text, sql_fulltext, parsing_schema_name      -- [1]
     , sharable_mem, persistent_mem, runtime_mem                              -- [2]
     , loads, invalidations, parse_calls, executions, fetches, rows_processed -- [3]
     , cpu_time, elapsed_time                                                 -- [4]
     , buffer_gets, disk_reads, sorts                                         -- [5]
     , application_wait_time, concurrency_wait_time                           -- [6]
     , cluster_wait_time, user_io_wait_time                                   -- [6]
     , first_load_time, last_active_time                                      -- [7]
from   v$sql
```

- [1] 라이브러리 캐시에 적재된 SQL 커서 자체에 대한 정보
- [2] SQL 커서에 의해 사용되는 메모리 사용량
- [3] 하드파싱 및 무효화 발생횟수, Parse, Execute, Fetch Call 발생 횟수, Execute 또는 Fetch Call 시점에 처리한 로우 건수 등
- [4] SQL을 수행하면서 사용된 CPU time과 소요시간(microsecond)
- [5] SQL을 수행하면서 발생한 논리적 블록 읽기와 디스크 읽기, 그리고 소트 발생 횟수
- [6] SQL 수행 도중 대기 이벤트 때문에 지연이 발생한 시간(microsecond)
- [7] 커서가 라이브러리 캐시에 처음 적재된 시점, 가장 마지막에 수행된 시점

v$sql에 보이는 통계치들도 다른 동적 성능 뷰처럼 누적값이다. 따라서 보이는 수치를 그대로 놓고 판단하는 것은 별 의미가 없다.
SQL 수행횟수로 나눈 평균값, 즉 SQL 한번 수행당 얼만큼의 일량과 시간을 소비하는지를 계산해야 의미 있는 분석이 가능하다. 아래 쿼리를 참조하기 바란다.

```
select parsing_schema_name
     , count(*) sql_cnt
     , count(distinct substr(sql_text, 1, 100)) sql_cnt2
     , sum(executions) executions
     , round(avg(buffer_gets/executions)) buffer_gets
     , round(avg(disk_reads/executions)) disk_reads
     , round(avg(rows_processed/executions)) rows_processed
     , round(avg(elapsed_time/executions/1000000),2) "ELAPSED_TIME(AVG)"
     , count(case when elapsed_time/executions/1000000 >= 10 then 1 end) "BAD SQL"
     , round(max(elapsed_time/executions/1000000,2) "ELAPSED_TIME(MAX)"
from   v$sql
where  parsing_schema_name in ( '원무', '공통', '진료', '사업/행정', '진료지원')
and    last_active_time >= to_date('20090315', 'yyyymmdd')
and    executions > 0
group by parsing_schema_name
```

\[그림 3-11\](p.228)은 어떤 병원 관련 기관에서 위 쿼리 수행 결과를 그대로 표로 옮겨서 분석한 내용이다.

세 번째 컬럼 'SQL개수(Unique)'는 SQL 문자열 중 선행 100개 문자가 같으면 동일 SQL인 것으로 간주하고 집계한 것이다.
같은 SQL인데도 바인드 변수를 사용하지 않으면 Literal 상수값 별로 오라클이 다른 sql_id를 부여해 SQL 개수가 무수히 많은 것으로 집계되는 오류를 보완하려는 것이다.
프로젝트마다 대개 SQL을 식별할 목적으로 select, insert, update, delete 키워드 바로 뒤에 주석으로 고유한 SQL 식별자를 적어 놓기 때문에 선행 100개 문자가 같으면 동일 SQL로 간주하는 데에
큰 무리가 없다.

위 통계 수치에 한 번 수행되었다가 금방 캐시에서 밀려난 쿼리들은 제외되므로 100% 정확하다고 할 수는 없지만 튜닝을 위한 의사결정 시 매우 유용하고 활용가치가 높은 정보를 제공한다.
예를 들어, 위 표에 집계된 결과를 보면 '진료지원' 업무에서 라이브러리 캐시에 로드된 총 SQL 개수가 8,680개인데, Unique하게는 1,822개이므로 바인드 변수를 사용하지 않아 각각 하드파싱을 발생시키며
캐시에 로드된 SQL 비중이 매우 높은 것을 알 수 있다. SQL이 공유될 수 있도록 바인드 변수를 사용하는 방식으로 프로그램을 수정할 필요가 있다.

그리고 '공통' 업무를 보면, 한번 수행할 때의 평균 논리적 I/O가 11,688개로 다른 업무에 대해 매우 높게 나타나고 있다.
논리적 I/O가 많다 보니 디스크 I/O도 많고 당연히 쿼리 평균 소요시간도 가장 높게 나타나고 있다. 게다가 SQL 개수는 가장 적지만 수행 횟수는 가장 많다.
따라서 가장 먼저 시급하게 튜닝해야 할 대상 업무로 판단할 수 있다.

v$sql의 또 다른 활용 예로서, \[그림 3-12\](p.229)는 1년 6개월 간의 개발기간을 거쳐 오픈을 맞은 어떤 시스템에서 4개 서브 업무 시스템별로 SQL 수행통계를 구해 본 것이다.

3초 이내에 수행되는 SQL 비중을 보면, 업무1은 80%, 업무2는 99%, 업무3은 91%인데 반해, 홈페이지 시스템은 66%로 가장 낮은 것으로 나타났다.
특히, 홈페이지는 인터넷으로 오픈된 시스템이므로 가장 시급하게 튜닝이 필요하다고 판단해 오픈 후 2주간 집중 튜닝을 실시하였다.

\[그림 3-13\](p.230)은 2주 후에 다시 SQL 수행 통계를 집계한 결과다. 홈페이지에서 3초 이내 수행된 SQL 비중이, 위 표에서 66%이던 것이 99.71%로 크게 늘어난 것을 확인할 수 있다.
<br/><br/>
v$sql과 조인해서 추가 정보를 얻을 수 있는 유용한 뷰들이 제공되는데, 그 중 v$sql_plan을 통해 실행계획을 확인하고 v$sql_plan_statistics를 통해 각 Row Source별 수행 통계를 확인할 수
있음을 이전에서 확인하였다.

v$sql_bind_capture 뷰도 유용한데, 이를 조회하면 전체는 아니더라도 정해진 기간에 한번씩 샘플링한 바인드 변수 값을 확인할 수 있다.
더 자주 샘플링하도록 하려면 _cursor_bind_capture_interval 파라미터 값을 줄이면 되고, 기본 설정 값은 900초다.

v$sql을 포함해 오라클은 SQL 커서와 관련된 각종 수행 통계를 주기적으로 AWR에 저장하며, 아래와 같은 뷰를 통해 조회 가능하다.

```
SQL> select * from dict
  2  where  table_name like 'DBA_HIST_SQL%' ;

TABLE_NAME                     COMMENTS
------------------------------ --------------------------------------------
DBA_HIST_SQLSTAT               SQL Historical Statistics Information
DBA_HIST_SQLTEXT               SQL Text
DBA_HIST_SQL_SUMMARY           Summary of SQL Statistics
DBA_HIST_SQL_PLAN              SQL Plan Information
DBA_HIST_SQL_BIND_METADATA     SQL Bind Metadata Information
DBA_HIST_SQLBIND               SQL Bind Information
DBA_HIST_SQL_WORKAREA_HSTGRM   SQL Workarea Histogram History
```

물론 스냅샷 시점에 캐시에 남아있던 커서의 수행 통계만 저장된다.
또한, 캐시에 남이 있더라도 그 방대한 양의 SQL 수행 통계를 스냅샷 시점별로 모두 저장할 수는 없으므로 아래와 같은 기준에 따라 Top SQL만 수집한다.

- Parse Calls
- Executions
- Buffer Gets
- Disk Reads
- Elapsed Time
- CPU Time
- Wait Time
- Version Count
- Sharable Memory

이와 관련해 11g에 추가된 유용한 기능이 있다.
'Colored SQL'이라고 명명된 기능으로서, 위 기준에 의해 Top SQL에 포함되지 않더라도 사용자가 명시적으로 지정한 커서의 수행통계가 AWR에 주기적으로 수집되도록 마크하는 기능이다.
Colored SQL로 지정하는 방법은 다음과 같다.

```
SQL> begin
  2    dbms_workload_repository.add_colored_sql(sql_id => '803b7z0t84sq7')
  3  end;
  4  /
```

Colored SQL 목록은 dba_hist_colored_sql 또는 wrm$_colored_sql 뷰를 통해 조회 가능하다.

```
SQL> select * from dba_hist_colored_sql ;

      DBID SQL_ID          CREATE_TIME
---------- --------------- -----------------
3982370468 803b7z0t84sq7   09/03/25 12:57:36
```

이처럼 sql_id에 색깔 표시를 해 두면 오라클이 AWR 정보를 수집할 때마다 Top SQL 선정기준과 상관없이 해당 SQL의 수행 통계를 저장한다.
단, 이 기능을 사용하더라도 스냅샷 시점에 캐시에서 밀려나고 없는 SQL 정보까지 저장할 수는 없다.
아쉬운 것은, Module과 Action 설정 값을 기준으로 색깔 표시하는 기능이 아직 제공되지 않는다는 점이며, 앞으로 추가될 것으로 기대해 본다.

Colored SQL 목록에서 제거할 때는 remove_colored_sql 프로시저를 이용하며, 이 명령을 수행하더라도 Top SQL 기준에 포함된 SQL은 AWR에 수집된다.

```
SQL> begin
  2    dbms_workload_repository.remove_colored_sql( '803b7z0t84sq7' )
  3  end;
  4  /
```

참고로, 앞에서 dbms_xplan 패키지를 소개하면서 언급했듯이 AWR에 저장된 SQL 실행계획은 다음과 같이 display_awr 프로시저를 통해 쉽게 출력해 볼 수 있다.

```
SQL> select * from table(dbms_xplan.display_awr(
  2                      '803b7z0t84sq7', NULL, NULL, 'basic rows bytes cost'));
```