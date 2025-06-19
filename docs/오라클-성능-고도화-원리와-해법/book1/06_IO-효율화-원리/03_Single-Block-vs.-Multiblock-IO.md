# 03. Single Block vs. Multiblock I/O

```
call     count      cpu    elapsed       disk      query    current       rows
------- ------ -------- ---------- ---------- ---------- ---------- ----------
Parse        1     0.00       0.00          0          0          0          0
Execute      1     0.00       0.00          0          0          0          0
Fetch        2     0.26       0.26         64         69          0          1
------- ------ -------- ---------- ---------- ---------- ---------- ----------
total        4     0.26       0.26         64         69          0          1
```

I/O Call 수행 원리에 대해 살펴보자. 위 Call 통계를 보면, 버퍼 캐시에서 69개 블록을 읽으면서 그 중 64개는 디스크에서 읽었다. 버퍼 캐시 히트율은 7.24%다.
디스크에서 읽은 블록 수가 64개라고 I/O Call까지 64번 발생했음을 의미하지는 않는다. 64번일 수도 있고, 그보다 작을 수도 있다.

읽고자 하는 블록을 버퍼 캐시에서 찾지 못했을 때, I/O Call을 통헤 데이터파일로부터 버퍼 캐시에 적재하는 방식에는 크게 두 가지가 있다.

- Single Block I/O
- Multiblock I/O

Single Block I/O는 말 그대로 한번의 I/O Call에 하나의 데이터 블록만 읽어 메모리에 적재하는 것을 말한다.
인덱스를 통해 테이블을 액세스할 때는, 기본적으로 인덱스와 테이블 블록 모두 이 방식을 사용한다.

Multiblock I/O는 I/O Call이 필요한 시점에 인접한 블록들을 같이 읽어 메모리에 적재하는 것을 말한다
오라클 블록 사이즈가 얼마건 간에 OS 단에서는 보통 1MB(=1,024KB) 단위로 I/O를 수행한다(OS마다 다름).
한번 I/O할 때 1MB 크기의 '그릇'을 사용하는 것이므로 테이블 Full Scan처럼 물리적으로 저장된 순서에 따라 읽을 때는 그릇이 허용하는 범위 내에서 인접한 블록들을 같이 읽는 것이 유리하다.
'인접한 블록'이란, 한 익스텐트 내에 속한 블록들을 말한다. 달리 말하면, Multiblock I/O 방식으로 읽더라도 익스텐트 범위를 넘지 못한다는 뜻이기도 하다.

Multiblock I/O 단위는 db_file_multiblock_read_count 파라미터에 의해 결정된다. 파라미터가 16이면 한 번에 최대 16개 블록을 버퍼 캐시에 적재한다.
만약 db_block_size가 8,192 바이트면 한 번에 최대 131,072 바이트를 읽는 셈이 된다. 파라미터를 128로 바꾸면 1,048,576 바이트씩 읽는다.
대개 OS 레벨에서 I/O 단위가 1MB이므로 db_block_size가 8,192일 때는 최대 설정할 수 있는 값은 128이 된다.
이 파라미터를 128 이상으로 설정하더라도 OS가 허용하는 I/O 단위가 1MB면 1MB씩만 읽는다.
<br/><br/>

디스크 I/O는 비용이 크므로 I/O Call 한번에 한 블록씩 읽는 것보다 여러 블록을 읽는 게 성능 향상에 도움이 되는데, 인덱스를 스캔할 때는 왜 한 블록씩 읽는 것일까?
인덱스 블록간 논리적 순서는 물리적으로 데이터파일에 저장된 순서와 다르다. 인덱스 블록간 논리적 순서란, 인덱스 리프 블록끼리 이중 연결 리스트(Double Linked List) 구조로 연결된 순서를 말한다.
물리적으로 한 익스텐트에 속한 블록들을 I/O Call 발생 시점에 같이 적재해 올렸는데, 그 블록들이 논리적 순서로는 한참 뒤쪽에 위치할 수 있다.
그러면 그 블록들은 실제 사용되지 못한 채 버퍼 상에서 밀려나는 일이 생길 수 있다. 하나의 블록을 캐싱하려면 다른 블록을 밀어내야 하는데, 이런 현상이 자주 발생한다면 버퍼 캐시 효율만 떨어뜨린다.
따라서 인덱스 스캔 시에는 Single Block I/O 방식으로 읽는 게 효율적이다.

Index Range Scan 뿐 아니라 Index Rull Scan 시에도 논리적인 순서에 따라 Single Block I/O 방식으로 읽는다.
인덱스의 논리적 순서를 무시하고 물리적인 순서에 따라 읽는 스캔 방식이 있는데, 이를 'Index Fast Full Scan'이라고 한다.
이때는 Table Full Scan과 마찬가지로 Multiblock I/O 방식을 사용하며, 한 번에 읽을 수 있는 최대 블록 수도 똑같이 db_file_multiblock_read_count 파라미터에 의해 결정된다.
<br/><br/>

서버 프로세스는 디스크에서 블록을 읽어야 하는 시점마다 I/O 서브시스템에 I/O 요청을 하고 대기 상태에 빠진다. 이때 발생하는 대표적인 대기 이벤트로는 아래 두 가지를 들 수 있다.

- db file sequential read 대기 이벤트 : Single Block I/O 방식으로 I/O를 요청할 때 발생
- db file scattered read 대기 이벤트 : Multiblock I/O 방식으로 I/O를 요청할 때 발생
  <br/><br/>

대량의 데이터를 Multiblock I/O 방식으로 읽을 때 Single Block I/O보다 성능상 유리한 것은 I/O Call 발생 횟수를 그만큼 줄여주기 때문이다. 직접 테스트하면서 그 의미를 살펴보자.

```
create table t
as
select * from all_objects;

alter table t add
constraint t_pk primary key(object_id);
```

테스트용 테이블 T와 PK 인덱스를 만들었다. 테이블과 인덱스를 만들자마자 아래 쿼리를 수행한다면 대부분 디스크 I/O를 통해 읽게 될 것이다.

```
select /*+ index(t) */ count(*)
from   t where object_id > 0

call     count      cpu    elapsed       disk      query    current       rows
------- ------ -------- ---------- ---------- ---------- ---------- ----------
Parse        1     0.00       0.00          0          0          0          0
Execute      1     0.00       0.00          0          0          0          0
Fetch        2     0.26       0.25         64         65          0          1
------- ------ -------- ---------- ---------- ---------- ---------- ----------
total        4     0.26       0.25         64         65          0          1

Rows     Row Source Operation
-------  --------------------------------------------------
      1  SORT AGGREGATE (cr=65 r=64 w=0 time=256400 us)
  31192   INDEX RANGE SCAN T_PK (cr=65 r=64 w=0 time=134613 us)

Elapsed times include waiting on following events:
  Event waited on                             Times    Max. Wait  Total Waited
  ----------------------------------------   Waited   ----------  ------------
  SQL*Net message to client                       2         0.00          0.00
  db file sequential read                        64         0.00          0.00
  SQL*Net message from client                     2         0.05          0.05
```

위 트레이스 결과를 보면, 논리적으로 65개 블록을 읽는 동안 64개의 디스크 블록을 읽었다. 이벤트 발생 현황을 보면 db file sequential read 대기 이벤트가 64번 발생했다.
즉, 64개 인덱스 블록을 Disk에서 읽으면서 64번의 I/O Call이 발생한 것이다.

이제 Multiblock I/O 방식으로 읽는 경우를 살펴볼 텐데, 그전에 db_block_size와 Multiblock I/O 단위를 확인한다.

```
SQL> show parameter db_block_size

NAME                                  TYPE                    VALUE
------------------------------------- ----------------------- -----------
db_block_size                         integer                 8192

SQL> show parameter db_file_multiblock_read_count

NAME                                  TYPE                    VALUE
------------------------------------- ----------------------- -----------
db_file_multiblock_read_count         integer                 16
```

db_block_size는 8,192이고, Multiblock I/O 단위는 16이다.
앞에서와 같은 양의 인덱스 블록을 Multiblock I/O 방식으로 읽도록 하기 위해 인덱스를 index fast full scan 방식으로 읽도록 유도해 보자. index_ffs 힌트를 사용하면 된다.
Multiblock I/O 단위가 16이므로 데이터파일에서 똑같이 64개 블록을 읽었을 때 4(=64/16)번의 I/O Call이 발생할 것으로 예상된다.
디스크 I/O가 발생하도록 하려면 먼저 테이블과 인덱스를 Drop 했다가 다시 생성해야 한다.

```
select /*+ index_ffs(t) */ count(*)
from   t where object_id > 0

call     count      cpu    elapsed       disk      query    current       rows
------- ------ -------- ---------- ---------- ---------- ---------- ----------
Parse        1     0.00       0.00          0          0          0          0
Execute      1     0.00       0.00          0          0          0          0
Fetch        2     0.26       0.26         64         69          0          1
------- ------ -------- ---------- ---------- ---------- ---------- ----------
total        4     0.26       0.26         64         69          0          1

Rows     Row Source Operation
-------  --------------------------------------------------
      1  SORT AGGREGATE (cr=69 r=64 w=0 time=267453 us)
  31192   INDEX FAST FULL SCAN T_PK (cr=69 r=64 w=0 time=143781 us)

Elapsed times include waiting on following events:
  Event waited on                             Times    Max. Wait  Total Waited
  ----------------------------------------   Waited   ----------  ------------
  SQL*Net message to client                       2         0.00          0.00
  db file scattered read                          9         0.00          0.00
  SQL*Net message from client                     2         0.35          0.36
```

똑같이 64개 블록을 디스크에서 읽었는데, I/O Call이 9번에 그쳤다. Single Block I/O 할 때보다는 크게 줄었지만 우리가 예상했던 4보다는 두 배 많은 수치다.
64/9 = 7.11 이므로 평균 7\~8개씩 읽은 셈이다. OS에서의 I/O 단위가 65,536(=8,192✕8) 바이트인 것일까? 트레이스 파일을 열어 확인해 보자.

```
EXEC #1:c=0,e=58,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=4,tim=1207457668427947
WAIT #1: nam='SQL*Net message to client' ela= 7 p1=1413697536 p2=1 p3=0
WAIT #1: nam='db file scattered read' ela= 73 p1=9 p2=80997 p3=4
WAIT #1: nam='db file scattered read' ela= 73 p1=9 p2=81001 p3=8
WAIT #1: nam='db file scattered read' ela= 72 p1=9 p2=81010 p3=7
WAIT #1: nam='db file scattered read' ela= 77 p1=9 p2=81017 p3=8
WAIT #1: nam='db file scattered read' ela= 70 p1=9 p2=81026 p3=7
WAIT #1: nam='db file scattered read' ela= 77 p1=9 p2=81033 p3=8
WAIT #1: nam='db file scattered read' ela= 74 p1=9 p2=81042 p3=7
WAIT #1: nam='db file scattered read' ela= 77 p1=9 p2=81049 p3=8
WAIT #1: nam='db file scattered read' ela= 74 p1=9 p2=81058 p3=7
FETCH
#1:c=262961,e=267473,p=64,cr=69,cu=0,mis=0,r=1,dep=0,og=4,tim=1207457668695516
WAIT #1: nam='SQL*Net message from client' ela= 4073 p1=1413697536 p2=1 p3=0
FETCH #1:c=0,e=4,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=0,tim=1207457668699756
WAIT #1: nam='SQL*Net message to client' ela= 6 p1=1413697536 p2=1 p3=0
WAIT #1: nam='SQL*Net message from client' ela= 356905 p1=1413697536 p2=1 p3=0
=====================
```

db file scattered read 대기 이벤트가 실제 9번 발생한 것을 볼 수 있고, 세 번째 파라미터(p3)를 보면 처음 것만 빼고 매번 7개 또는 8개씩을 읽었다.
테이블스페이스에 할당된 익스텐트 크기를 확인해 보면 그 이유를 쉽게 찾을 수 있다.

```
SQL> select extent_id, block_id, bytes, blocks
  2  from   dba_extents
  3  where  owner = USER
  4  and    segment_name = 'T_PK'
  5  and    tablespace_name = 'USERS'
  6  order by extent_id ;

EXTENT_ID    BLOCK_ID       BYTES     BLOCKS
---------- ---------- ----------- ----------
         0      80993       65536          8
         1      81001       65536          8
         2      81009       65536          8
         3      81017       65536          8
         4      81025       65536          8
         5      81033       65536          8
         6      81041       65536          8
         7      81049       65536          8
         8      81057       65536          8

9 개의 행이 선택되었습니다.
```

모든 익스텐트가 8개 블록으로 구성돼 있는 것이 원인이었다. Multiblock I/O 방식으로 읽더라도 익스텐트 범위를 넘지는 못하기 때문이다.
예를 들어, 모든 익스텐트에 20개 블록이 있고 db_file_multiblock_read_count가 8이면, 익스텐트마다 8, 8, 4개씩 세 번에 걸쳐 읽는다.
<br/><br/>

익스텐트 크기 때문에 예상보다 조금 더 많은 I/O Call이 발생하긴 했지만, Single Block I/O 때보다 훨씬 적은 양의 I/O Call이 발생하는 것을 알 수 있었다.

참고로, 위 테스트는 9i에서 수행한 것이다.
10g부터는 Index Range Scan 또는 Index Full Scan일 때도 Multiblock I/O 방식으로 읽는 경우가 있는데, 위처럼 테이블 액세스 없이 인덱스만 읽고 처리할 때가 그렇다.
인덱스를 스캔하면서 테이블을 Random 액세스할 때는 9i 이전과 동일학게 테이블과 인덱스 블록을 모두 Single Block I/O 방식으로 읽는다.
<br/><br/>

Single Block I/O 방식으로 읽은 블록들은 LRU 리스트 상 MRU 쪽(end)(버전마다 다르다. 예전에는 MRU end에 연결되었으나 최근 버전에서는 중간 정도에 위치한다고 이해하자.)으로 연결되므로 한번
적재되면 버퍼 캐시에 비교적 오래 머문다. 반면, Multiblock I/O 방식으로 읽은 블록들은 LRU 리스트에서 LRU 쪽(end)에 연결되므로 적재되고 얼마 지나지 않아 버퍼 캐시에서 밀려난다.
따라서 대량의 데이터를 Full Scan 했다고 해서 사용빈도가 높은 블록들이 버퍼 캐시에서 모두 밀려날 것을 우려하지 않아도 된다.