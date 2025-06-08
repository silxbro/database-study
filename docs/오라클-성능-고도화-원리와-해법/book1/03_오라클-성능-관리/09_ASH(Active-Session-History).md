# 09. ASH(Active Session History)

Ratio 기반 분석 방법론의 한계점은, 시스템에 문제가 있는 것으로 진단되었을 때 그 원인을 찾아 실제 문제를 해결하는 데까지 많은 시간이 걸리는 데 있다고 했다.
이것은 대기 이벤트 기반 분석 방법론을 사용하더라도 마찬가지다.
대기 이벤트 발생량과 대기 시간을 통해 문제의 원인을 금방 알 수는 있지만, 실제 문제를 해결하려면 구체적으로 어떤 프로그램에서 문제를 일으켰고, 어떤 세션에서 성능 때문에 고생했는지를 확인할 수 있어야 한다.
물론 시스템 레벨에서 분석하고 해결할 수 있는, 구조적인 문제들도 있기는 하다.
(예를 들어, 최근에 급하게 방문 요청을 받아 진단했던 모 보험회사 시스템 분석 결과 gc cr block lost와 gc current block lost 이벤트에 의한 대기 시간이 가장 높게 나타나고 있었다.
이는 네트워크 설정이 잘못 됐거나 물리적인 이상이 발생했을 때 생기는 현상이다.)
하지만 그런 구조적인 문제가 흔히 발생하는 것은 아니므로 대개는 세션 레벨의 성능 분석을 요한다.

그런데 오라클이 기존에 제공하는 세션 레벨 동적 성능 뷰만으로는 문제를 빨리 찾기 어렵거나 아예 불가능한 경우가 대부분이었다.
SQL 트레이스를 통해 가장 상세한 세션 레벨 분석이 가능하지만 시스템에 주는 부하가 크고, 파일 단위로 정보가 수집되기 때문에 통계적 접근이 어려우며, 분석을 완료하는 데까지 시간이 오래 걸린다.
게다가 수동으로 활성화해야 하기 때문에 SQL 트레이스를 걸어 확인하려는 순간 이미 상황이 종료돼 버리는 경험을 DB 관리자라면 누구나 했을 것이다.
그리고 문제가 발생하기 직전 상황에 대한 분석은 아예 불가능하다.

이런 DB 성능 관리자들의 고충을 잘 이해한 오라클이 10g에서 ASH 기능을 탄생시켰다.
10g AWR은 데이터 수집을 아주 빠르게, 좀 더 많이 한다는 것 외에는 외형적으로 Statspack과 크게 달라진 것이 없다고 느낄 것이다. 하지만 ASH를 보는 순간 생각이 달라진다.
10g에서 개선되고 추가된 기능 중 가장 유용한 것을 하나 꼽으라면 단연 ASH를 들 수 있다.

이것은 별도의 Third Party 모니터링 도구 없이 오라클 내에서 세션 레벨 실시간 모니터링을 가능케 하는 강력한 기능으로서, OWI의 활용성을 극대화해준다.

```
SQL> select * from v$sgastat where name = 'ASH bufferes';

POOL         NAME                     BYTES
------------ ------------------- ----------
shared pool  ASH buffers           65011712

1 개의 행이 선택되었습니다.
```

오라클은 현재 접속해서 활동 중인 Active 세션 정보를 1초에 한번씩 샘플링해서 ASH 버퍼에 저장한다.
SGA Shared Pool에서 CPU당 2MB의 버퍼를 할당 받아 세션 정보를 기록하며, 1시간 혹은 버퍼의 2/3가 찰 때마다 디스크로 기록한다. 즉, AWR에 저장하는 것이다.

v$active_session_history 뷰를 이용해 ASH 버퍼에 저장된 세션 히스토리 정보를 조회할 수 있다. 우선 이 뷰에서 어떤 정보들이 열람 가능한지부터 살펴보자.

```
select
    sample_id, sample_time  ---------------------------------------------------------[1]
  , session_id, session_serial#, user_id, xid----------------------------------------[2]
  , sql_id, sql_child_number, sql_plan_hash_value------------------------------------[3]
  , session_state  ------------------------------------------------------------------[4]
  , qc_instance_id, qc_session_id  --------------------------------------------------[5]
  , blocking_session, blocking_session_serial#, blocking_session_status  ------------[6]
  , event, event#, seq#, wait_class, wait_time, time_waited -------------------------[7]
  , p1text, p1, p2text, p2, p3text, p3  ---------------------------------------------[8]
  , current_obj#, current_file#, current_block#  ------------------------------------[9]
  , program, module, action, client_id ---------------------------------------------[10]
from V$ACTIVE_SESSION_HISTORY
```

- [1] 샘플링이 일어난 시간과 샘플 ID
- [2] 세션정보, User명, 트랜잭션ID
- [3] 수행 중 SQL 정보
- [4] 현재 세션의 상태 정보, 'ON CPU' 또는 'WAITING'
- [5] 병렬 Slave 세션일 때, 쿼리 코디네이터(QC) 정보를 찾을 수 있게 함
- [6] 현재 세션의 진행을 막고 있는(=블로킹) 세션 정보
- [7] 현재 발생 중인 대기 이벤트 정보
- [8] 현재 발생 중인 대기 이벤트의 파라미터 정보
- [9] 해당 세션이 현재 참조하고 있는 오브젝트 정보. v$session 뷰에 있는 row_wait_obj#, row_wait_file#, row_wait_block# 컬럼을 가져온 것임
- [10] 애플리케이션 정보

[7]과 [8]번 '대기 이벤트' 정보는 두말할 것도 없고, [6]번 '블로킹 세션' 정보와 [9]번 '현재 액세스 중인 오브젝트' 정보도 매우 유용하다.
블로킹 세션 정보를 통해 현재 Lock을 발생시킨 세션을 빨리 찾아 해소할 수 있게 도와준다.

오브젝트 정보도 더할 나위 없이 유용하지만 현재 발생 중인 대기 이벤트의 Wait Class가 Application, Concurrency, Cluster, User I/O일 때만 의미 있는 값임을 알아야 한다.
예를 들어, ITL 슬롯 부족 때문에 발생하는 enq: TX - allocate ITL entry 대기 이벤트는 Configuration에 속하므로, v$active_session_history 뷰를 조회할 때 함께 출력되는 오브젝트에
Lock이 걸렸다고 판단해서는 안 된다. 대개 그럴 때는 오브젝트 번호가 -1로 출력되지만 직전에 발생한 이벤트의 오브젝트 정보가 계속 남아서 보이는 경우가 있으므로 잘못 해석하지 않도록 주의해야 한다.
아래 예시에서 파일번호와 블록번호가 그대로 남아있는 것을 볼 수 있다.

```
SQL> column current_obj#,  format 99999 heading 'CUR_|OBJ#'
SQL> column current_file#, format 999   heading 'CUR_|FIL#'
SQL> column current_block# format 999   heading 'CUR_|BLK#'
SQL> select to_char(sample_time, 'hh24:mi:ss') sample_tm, session_state
  2       , event, wait_class, current_obj#, current_file#, current_block#
  3  from   v$active_session_history
  4  where  session_id = 143
  5  and    session_serial# = 9
  6  order by sample_time ;

                                                                 CUR_ CUR_ CUR_
SAMPLE_T SESSION EVENT                          WAIT_CLASS       OBJ# FIL# BLK#
-------- ------- ------------------------------ -------------- ------ ---- ----
15:00:44 WAITING enq: TX - row lock contention  Application     55765    4  476
15:00:45 WAITING enq: TX - row lock contention  Application     55765    4  476
15:00:46 WAITING enq: TX - row lock contention  Application     55765    4  476
15:00:47 WAITING enq: TX - row lock contention  Application     55765    4  476
15:01:36 WAITING enq: TX - allocate ITL entry   Configuration      -1    4  476
15:01:37 WAITING enq: TX - allocate ITL entry   Configuration      -1    4  476
15:01:38 WAITING enq: TX - allocate ITL entry   Configuration      -1    4  476
15:01:39 WAITING enq: TX - allocate ITL entry   Configuration      -1    4  476
```

초 단위로 쓰기가 발생하는 ASH 버퍼를 읽을 때 래치를 사용한다면 경합이 생길 수 있다.
따라서 오라클은 ASH 버퍼를 읽는 세션에 대해서는 래치를 요구하지 않으며, 그 때문에 간혹 일관성 없는 잘못된 정보가 나타날 수도 있다.
<br/><br/>
ASH 기능을 이용하면 현재뿐 아니라 과거시점에 발생한 장애 및 성능 저하 원인까지 세션 레벨로 분석할 수 있게 도와준다.
Statspack을 이용할 때는 이튿날 아침에 분석 보고서를 생성해 문제점을 발견하더라도 이를 좀 더 세밀하게 분석해 볼 방법이 없었다. 이미 문제의 세션은 종료되고 없기 때문이다.

오라클 10g부터는 v$active_session_history 정보를 AWR 내에 보관하므로 과거치에 대한 세션 레벨 분석이 가능해졌다. 이것도 SGA를 DMA 방식으로 액세스하기 때문에 가능해진 일이라고 생각한다.
그렇더라도 그 방대한 정보를 다 저장하는 것은 무리가 되므로 1/10만 샘플링해서 저장한다.
문제가 되는 대기 이벤트는 일정간격을 두고 지속적으로 발생하기 때문에 샘플링된 자료만으로도 원인을 찾는데 큰 지장은 없다.

v$active_session_history를 조회했을 때 정보가 찾아지지 않는다면 이미 AWR에 쓰여진 것이므로 dba_hist_active_sess_history 뷰를 조회하면 된다.
AWR과 ASH를 활용하면 별도의 OWI 기반 모니터링 툴 없이도 아래와 같은 분석이 가능하다. (\[그림 3-9\](p.224))

- [1] AWR 뷰를 이용해 하루 동안의 이벤트 발생현황을 조회해 본다.
  [그림 3-9]의 그래프는 dba_hist_system_event를 이용해 그린 것인데, 08:15\~09:15 구간에서 enq: TM - contention 이벤트가 다량 발생한 것이 확인되었다.
- [2] dba_hist_active_sess_history를 조회해서 해당 이벤트를 많이 대기한 세션을 확인한다.
- [3] 블로킹 세션 정보를 통해 dba_hist_active_sess_history를 다시 조회한다. 블로킹 세션이 찾아지면 해당 세션이 그 시점에 어떤 작업을 수행 중이었는지 확인한다.
  sql_id를 이용해 그 당시 SQL과 실행계획까지 확인할 수 있다. v$sql과 v$sql_plan까지 AWR에 저장되기 때문이다.
  위 사례에서는 블로킹 세션이 Append Mode Insert를 수행하면서 Exclusive 모드 TM Lock에 의한 경합이 발생하고 있었다.
- [4] program, module, action, client_id 등 애플리케이션 정보를 이용해 관련 프로그램을 찾아 Append 힌트를 제거한다.
  그러고 나서, 다른 트랜잭션과 동시 DML이 발생할 수 있는 상황에서는 insert문에 Append 힌트를 사용해서는 안 된다는 사실을 개발팀 전체에 공지한다.

이처럼 부하를 최소화하면서, 세션 레벨의 상세한 분석이 가능하도록 오라클이 성능자료를 수집해 주므로 이제 AWR과 ASH를 잘 이용하면 전문 성능관리 툴의 도움 없이도 효과적으로 성능 분석을 할 수 있게
되었다.