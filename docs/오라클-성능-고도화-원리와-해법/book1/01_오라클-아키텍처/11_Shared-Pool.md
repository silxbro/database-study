# 11. Shared Pool

지금까지 데이터 블록 I/O를 중심으로 설명하다 보니 DB 버퍼 캐시와 Redo 로그 버퍼에 대해 자연스럽게 설명이 이루어졌는데, 이제 SGA의 가장 중요한 구성요소 중 하나인 Shared Pool에 대해 간단히
살펴보도록 하자.
<br/>
<br/>
## (1) 딕셔너리 캐시
Shared Pool은 크게 딕셔너리 캐시(Dictionary Cache)와 라이브러리 캐시(Library Cache)로 나뉜다.
우선 딕셔너리 캐시에 대해 설명하면, 말 그대로 오라클 딕셔너리 정보를 저장해 두는 캐시영역으로서 Row 단위로 읽고 쓰기 때문에 '로우 캐시(Row Cache)' 라고도 불린다.
테이블, 인덱스 같은 오브젝트는 물론 테이블스페이스, 데이터파일, 세그먼트, 익스텐트, 사용자, 제약, Sequence, DB Link에 관한 정보들을 캐싱한다.
예를 들어, 사용자가 Sequence 객체를 하나 만들면 오라클 딕셔너리에 저장되고, 로우 캐시를 거쳐 읽고 쓰기가 이루어진다.
사용자가 Sequence로부터 새로운 값을 인출하기 위해 nextval을 호출할 때마다 로우 캐시를 통해 update가 이루어진다.

> #### [Sequence Cache 옵션]
> 잦은 채번은 로우 캐시에 경합을 발생시키게 된다. 이를 해소하기 위해 필수적으로 Cache 옵션을 사용해야 하며, Cache 크기가 10이면 Nocache 일 때보다 로우 캐시를 갱신하는 횟수가 1/10으로 준다.
> 동시에 채번이 많이 발생하는 Sequence 일수록 cache 사이즈를 크게 설정하는 것은 매우 중요한 튜닝 요소 중 하나이며, 기본 설정값은 20이다.

딕셔너리 캐시의 활동성에 대한 통계를 조회해 볼 수 있는 뷰가 v$rowcache인데, 여기서 히트율(Hit Ratio)를 조사했을 때 수치가 낮게 나오면 Shared Pool 사이즈를 늘리는 것을 고려해 볼 필요가 있다.

```
SQL> COLUMN HIT_RATIO FORMAT 990.99
SQL> SELECT ROUND((SUM(GETS - GETMISSES)) / SUM(GETS) * 100, 2) HIT_RATIO
  2  FROM   V$ROWCACHE;

HIT_RATIO
---------
    93.07

1 개의 행이 선택되었습니다.

SQL> select PARAMETER
  2       , GETS
  3       , GETMISSES
  4       , round((GETS - GETMISSES) / GETS * 100, 2) HIT_RATIO
  5       , MODIFICATIONS
  6  FROM   V$ROWCACHE
  7  WHERE  GETS > 0
  8  ORDER BY HIT_RATIO DESC ;

PARAMETER                               GETS  GETMISSES HIT_RATIO MODIFICATIONS
--------------------------------- ---------- ---------- --------- -------------
dc_tablespaces                         34802          7     99.98             0
dc_awr_control                          1059          1     99.91            39
dc_users                               40416         37     99.91             0
dc_profiles                             1013          1     99.90             0
dc_rollback_segments                   10876         21     99.81            31
dc_usernames                            2047         18     99.12             0
dc_global_oids                          2669         48     98.20             0
dc_histogram_data                       4921        130     97.36            42
dc_object_ids                          33451        941     97.19            70
dc_users                                 918         28     96.95             0
dc_tablespace_quotas                      20          1     95.00             0
dc_object_grants                        1734        112     93.54             0
dc_objects                             13259       1257     90.52           201
outstanding_alerts                      1775        192     89.18           358
dc_segments                             6682        728     89.11           103
dc_files                                  30          5     83.33             0
dc_histogram_data                      14387       2833     80.31          1604
dc_sequences                              40          8     80.00            40
dc_histogram_defs                      26626       7227     72.86          1692
dc_constraints                            56         20     64.29            56
dc_table_scns                              7          7      0.00             0

21 개의 행이 선택되었습니다.
```

v$rowcache에서 TYPE = 'PARENT'인 엔트리와 v$latch_children에서 이름이 'row cache objects'인 래치 개수를 조회해 보면, 항상 값이 일치하는 것을 알 수 있다.
즉, 로우 캐시에 관리되는 엔트리 각각에 대해 하나의 래치가 할당돼 있음을 짐작할 수 있다.

```
SQL> select *
  2  from  (select count(*) from v$rowcache where type = 'PARENT'),
  3        (select count(*) from v$latch_children where name = 'row cache objects')
  4  ;

  COUNT(*)   COUNT(*)
---------- ----------
        34         34
```
<br/>

## (2) 라이브러리 캐시
Shared Pool을 구성하는 두 번째 캐시 영역은 라이브러리 캐시인데, 이것은 지금까지 설명한 캐시 영역과는 조금 다른 역할을 맡고 있다. DB 버퍼 캐시, Redo 로그 버퍼 캐시, 딕셔너리 캐시 등은 데이터의
입출력을 빠르게 하기 위한 캐시영역인 반면 라이브러리 캐시는 사용자가 던진 SQL과 그 실행계획을 저장해 두는 캐시영역이다.
사용자가 SQL이라는 명령어를 통해 결과집합을 요청하면 이를 최적으로(→ 가장 적은 리소스를 사용하면서 가장 빠르게) 수행하기 위한 처리 루틴이 필요한데, 이를 실행계획(execution plan)이라고 한다.
빠른 쿼리 수행을 위해 내부적으로 생성한 일종의 프로시저와 같은 것이라고 이해하면 쉽다.

쿼리 구문을 분석해서 문법 오류 및 실행 권한 등을 체크하고, 최적화 과정을 거쳐 실행 계획을 만들고, SQL 실행엔진이 이해할 수 있는 형태로 포맷팅하는 전 과정을 하드파싱(Hard Parsing)이라고 한다.
특히 최적화(Optimization)는 하드파싱을 무겁게 만드는 가장 결정적 요인인데, 같은 SQL을 처리하려고 이런 무거운 작업을 반복 수행하는 것은 매우 비효율적이다.
그래서 같은 SQL에 대한 반복적인 하드파싱을 최소화하기 위한 새로운 캐시 공간을 두게 되었고, 그것이 바로 라이브러리 캐시 영역이다.
당연히 라이브러리 캐시 최적화 원리는 캐싱된 SQL과 그 실행계획의 재사용성을 높이는 데에 있다.