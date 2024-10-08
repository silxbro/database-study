# 04. 테이블 Random 액세스 부하

SQL 튜닝, 그 중에서도 인덱스 원리를 공부하면서 누구나 두 번 놀라게 된다.
첫 번째는 인덱스를 효과적으로 활용했을 때 쿼리 성능이 얼마나 빨라지는지를 느꼈을 때이고, 두 번째는 대량의 데이터를 인덱스를 통해 액세스할 때 쿼리 성능이 얼마나 느려지는지를 느꼈을 때이다.

본 절에서는 대량의 데이터를 처리할 때 테이블 Random 액세스가 가장 큰 부하 요인으로 작용하는 원인을 자세히 설명하고, 5절과 6절에서 이를 해소하기 위한 튜닝 방안을 소개한다.

<br/>

## (1) 인덱스 ROWID에 의한 테이블 액세스
앞 절에서 다양한 인덱스 스캔 방식에 대해 살펴보았는데, 쿼리에서 참조되는 컬럼이 인덱스에 모두 포함되는 경우가 아니라면 인덱스 스캔 이후 '테이블 Random 액세스'가 일어난다.
아래 실행계획에서 'Table Access By Index ROWID'라고 표시된 부분을 말하며, 지금부터 그 내부 메커니즘을 자세히 살펴보자.
```
SQL> select * from 고객 where 지역 = '서울';

Execution Plan
------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS
1  0     TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE)
2  1       INDEX (RANGE SCAN) OF '고객_지역_IDX' (INDEX)
```

### [물리적 주소? 논리적 주소?]
인덱스에 저장돼 있는 rowid는 흔히 '물리적 주소정보'라고 일컬어지는데, 오브젝트 번호, 데이터파일 번호, 블록 번호 같은 물리적 요소들로 구성돼 있기 때문일 것이다.

하지만 보는 시각에 따라서는 '논리적 주소정보'라고 표현하기도 한다. rowid가 물리적 위치 정보로 구성되지만 인덱스에서 테이블 레코드로 직접 연결되는 구조는 아니기 때문이다.
- IOT 테이블 로우에 대한 내부 식별자를 Logical ROWID라고 하는데 이것을 설명하려는 것이 아니며, 일반 힙 구조 테이블에서 사용되는 Physical ROWID가 내용적으로 논리적 의미를 담고 있다는 뜻이다.

### [메인 메모리 DB와의 비교]
메인 메모리 DB(MMDB)에 대해 들어 본 적이 있는가? 말 그대로 데이터를 모두 메모리에 로드해 놓고 메모리를 통해서만 I/O를 수행하는 DB라고 할 수 있다.
그런데 잘 튜닝된 OLTP성 오라클 데이터베이스라면 버퍼 캐시 히트율(BCHR, 읽고자 하는 데이터 블록이 버퍼 캐시에서 찾아지는 비율)이 99% 이상이므로 대부분 디스크를 경유하지 않고 메모리상에서 I/O를
수행할 텐데도 메인 메모리 DB만큼 빠르지는 않다. 특히 대량의 테이블을 인덱스를 통해 액세스할 때는 엄청난 차이가 난다. 왜 그럴까?
- 메모리 DB개발 업체들은 오라클 같은 DB를 '디스크 DB'라고 부르며, 최근에는 메인 메모리 DB도 대용량 데이터를 처리하기 위해 디스크를 이용하는 Hybrid 형태로 진화하고 있다.

메인 메모리 DB도 벤더에 따라 내부적으로 다른 아키텍처를 사용하겠지만 필자가 아는 어떤 제품의 아키텍처를 소개함으로써 방금 던진 질문에 대한 답을 찾고자 한다.

메인 메모리 DB에서 인스턴스를 기동하면 디스크에 저장된 데이터를 버퍼 캐시로 로딩하고 이어서 인덱스를 실시간으로 만든다.
이때 인덱스는 오라클처럼 디스크 상의 주소정보를 담는 게 아니라 메모리상의 주소정보, 즉 포인터(Pointer)를 담는다.
프로그래머라면 누구나 포인터 개념을 공부하므로 포인터에 의한 액세스가 얼마나 빠른지는 잘 알 것이다.

메모리상에서 데이터를 찾아가는 데 있어 포인터만큼 빠른 방법은 없으며 그 비용은 거의 0에 가깝다. 따라서 인덱스를 경유해 테이블을 액세스하는 데 대한 비용이 오라클과 비교할 수 없을 정도로 낮다.

질문에 대한 답이 무엇인지 이해했으리라 믿는다. 오라클은 테이블 블록이 수시로 버퍼 캐시에서 밀려났다가 다시 캐싱되며, 그때마다 다른 공간에 캐싱되기 때문에 인덱스에서 직접 포인터로 연결할 수 없는 구조다.
대신 디스크 상의 블록 위치 정보, 즉 DBA(Data Block Address)를 해시 키 값으로 삼아 해싱 알고리즘을 통해 버퍼 블록을 찾는다. 매번 위치가 달라지더라도 캐싱되는 해시 버킷만큼은 고정적이다.

여기서 메인 메모리 DB의 우월성을 강조하고자 하는 것은 아니며(메인 메모리 DB는 데이터 볼륨이 그리 크지 않으면서, 기존 디스크 DB로는 도저히 만족할 수 없을 정도의 빠른 트랜잭션 처리가 요구되는
업무에서만 제한적으로 사용되고 있음), 우리가 사용하는 오라클 데이터베이스에서 인덱스 rowid에 의한 테이블 액세스가 생각만큼 빠르지 않은 이유를 설명하려는 것이다.

### [rowid는 우편주소에 해당]
비유하자면, 오라클에서의 rowid는 우편주소에 해당하고, 메인 메모리 DB에서 사용하는 포인터는 전화번호에 해당한다.
전화는 물리적으로 연결된 구조여서 전화번호만 누르면 곧바로 상대방과 직접 통화할 수 있지만, 우편주소는 봉투에 적힌 대로 우체부 아저씨가 일일이 찾아다니는 구조이므로 전화와는 비교할 수 없이 느리다.

### [인덱스 rowid에 의한 테이블 액세스 구조]
오라클도 포인터로 빠르게 액세스하는 버퍼 Pinning 기법을 사용하지만 반복적으로 읽힐 가능성이 큰 블록에 대해서만 일부 적용하고 있다.
이 메커니즘의 도움을 받지 않은 일반적인 인덱스 rowid에 의한 테이블 액세스가 실제로 얼마나 고비용인지 알아보자.

- 인덱스에서 하나의 rowid를 읽고 DBA(Data Block Address, 디스크 상의 블록 위치 정보)를 해시 함수에 적용해 해시 값을 확인한다.
- 각 해시 체인은 래치(Latch)에 의해 보호되므로 해시 값이 가리키는 해시 체인에 대한 래치(→ cache buffers chains 래치)를 얻으려고 시도한다.
  하나의 cache buffers chains 래치가 여러 개 해시 체인을 동시에 관리한다는 사실도 기억할 필요가 있다.
- 다른 프로세스가 래치를 잡고 있으면 래치가 풀렸는지 확인하는 작업을 일정 횟수(기본 설정은 2,000번)만큼 반복한다.
- 그러고도 실패하면 CPU를 OS에 반환하고 잠시 대기 상태로 빠지는데, 이때 latch free (10g부터는 latch: cache buffers chains으로 세분화) 대기 이벤트가 나타난다.
- 정해진 시간 동안 잠을 자다가 깨어나서 다시 래치 상태를 확인하고, 계속해서 래치가 풀리지 않으면 또다시 대기 상태로 빠질 수도 있다.
- 래치가 해제되었다면 래치를 재빨리 획득하고 원하던 해시 체인으로 진입한다.
- 거기서 데이터 블록이 찾아지면 래치를 해제하고 바로 읽으면 되는데, 앞서 해당 블록을 액세스한 프로세스가 아직 일을 마치지 못해 버퍼 Lock을 쥔 상태라면 또다시 대기해야 한다.
  이때 나타나는 대기 이벤트가 buffer busy waits이다.
- 블록 읽기를 마치고 나면 버퍼 Lock을 해제해야 하므로 다시 해시 체인 래치를 얻으려고 시도한다. 이때 또다시 경합이 발생할 수 있다.

해시 체인을 스캔했는데 데이터 블록을 찾지 못했을 때는 일이 더 복잡해진다.

- 디스크로부터 블록을 퍼 올리려면 우선 Free 버퍼를 할당 받아야 하므로 LRU 리스트를 스캔한다.
  이를 위해서는 cache buffers lru chain 래치를 얻어야 하는데, 래치 경합이 심할 때는 여기서도 latch free 이벤트(10g부터 latch: cache buffers lru chain으로 세분화)가 나타날 수 있다.
- LRU 리스트를 정해진 임계치만큼 스캔했는데도 Free 상태의 버퍼를 찾지 못하면 DBWR에게 Dirty 버퍼를 디스크에 기록해 Free 버퍼를 확보해 달라는 신호를 보낸다.
  그런 후 해당 작업이 끝날 때까지 잠시 대기 상태에 빠지는데, 이때 나타나는 대기 이벤트가 free buffer waits이다.
- Free 버퍼를 할당 받은 후에는 I/O 서브시스템에 I/O 요청을 하고 다시 대기 상태에 빠지는데, 이때 db file sequential read 대기 이벤트가 나타난다.
- 마지막으로, 읽은 블록을 LRU 리스트 상에서 위치를 옮겨야 하기 때문에 다시 cache bufferes lru chain 래치를 얻어야 하는데, 이 또한 원활하지 못할 때는 latch free 이벤트가 나타난다.

오라클에서 하나의 레코드를 찾아가는데 있어 가장 빠르다고 알려진 rowid에 의한 테이블 액세스가 경우에 따라서는(→ 위 설명은 가장 최악의 상황을 가정한 것임) 이렇게까지 고비용이라는 사실을 상기시키기 위함이다.

요약하면, 인덱스 rowid는 테이블 레코드와 물리적으로 연결돼 있지 않기 때문에 인덱스를 통한 테이블 액세스는 생각보다 고비용 구조다.
설령 모든 데이터가 메모리에 캐싱돼 있더라도 테이블 레코드를 찾기 위해 매번 DBA를 해싱하고 래치 획득 과정을 반복해야 하기 때문이며, 동시 액세스가 심할 때는 래치와 버퍼 Lock에 대한 경합까지 발생한다.

앞으로 실행계획에서 아래와 같이 'Table Access By Index ROWID' 오퍼레이션을 볼 때면 위에서 설명한 복잡한 처리과정을 항상 머리 속에 떠올리기 바란다.
```
SQL> select * from 고객 where 지역 = '서울';

Execution Plan
------------------------------------------------------
0      SELECT STATEMENT Optimizer=ALL_ROWS
1  0     TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE)
2  1       INDEX (RANGE SCAN) OF '고객_지역_IDX' (INDEX)
```

<br/>

## (2) 인덱스 클러스터링 팩터
### [군집성 계수(= 데이터가 모여 있는 정도)
클러스터링 팩터(Clustering Factor, 이하 CF)는 '군집성 계수' 쯤으로 번역될 수 있는 용어로서, 특정 컬럼을 기준으로 같은 값을 갖는 데이터가 서로 모여있는 정도를 의미한다.
CF가 좋은 컬럼에 생성한 인덱스는 검색 효율이 매우 좋은데, 예를 들어 [거주지역 = '제주']에 해당하는 고객 데이터가 물리적으로 근접해 있다면 흩어져 있을 때보다 데이터를 찾는 속도가 빨라지게 마련이다.

인덱스 클러스터링 팩터가 가장 좋은 경우는, 인덱스 레코드 정렬 순서와 거기서 가리키는 테이블 레코드 정렬 순서가 100% 일치하는 경우이다.
반면 인덱스 클러스터링 팩터가 가장 안 좋은 경우는, 인덱스 레코드 정렬 순서와 테이블 레코드 정렬 순서가 전혀 일치하지 않는 경우이다.

### [클러스터링 팩터 조회]
CF는 데이터가 모여있는 정도를 의미한다고 했는데, 오라클은 이를 어떻게 수치화해서 표현할까? 테스트를 통해 확인해 보자.
```
SQL> create table t
  2  as
  3  select * from all_objects
  4  order by object_id;  → 테이블 레코드가 object_id 순으로 입력되도록 함

SQL> create index t_object_d_idx on t(object_id);

SQL> create index t_object_name_idx on t(object_name);
```
테이블 레코드가 object_id 순으로 입력되도록 t 테이블을 생성하고, object_id와 object_name 각각에 대한 인덱스를 하나씩 만들었다.

아래와 같이 통계정보를 생성하고 나면 오라클이 계산한 인덱스 CF를 뷰에서 확인할 수 있다.
```
SQL> exec dbms_stats.gather_table_stats(user, 'T');    -- 통계정보 수집

SQL> select i.index_name, t.blocks table_blocks, i.num_rows, i.clustering_factor
  2  from   user_tables t, user_indexes i
  3  where  t.table_name = 'T'
  4  and    i.tabile_name = t.table_name;

INDEX_NAME                TABLE_BLOCKS    NUM_ROWS CLUSTERING_FACTOR
------------------------- ------------ ----------- -----------------
T_OBJECT_ID_IDX                    709       50093               689
T_OBJECT_NAME_IDX                  709       50093             24936
```
clustering_factor 수치가 테이블 블록(table_blocks)에 가까울수록 데이터가 잘 정렬돼 있음을 의미하고, 레코드 개수(row_nums)에 가까울수록 흩어져 있음을 의미한다.
인덱스 통계를 수집할 때 clustering_factor 계산을 위해 오라클이 사용하는 아래 로직을 보고 나면 이유를 쉽게 알 수 있다.
- [1] counter 변수를 하나 선언한다.
- [2] 인덱스 리프 블록을 처음부터 끝까지 스캔하면서 인덱스 rowid로부터 블록 번호를 취한다.
- [3] 현재 읽고 있는 인덱스 레코드의 블록 번호가 바로 직전에 읽은 레코드의 블록 번호와 다를 때마다 counter 변수 값을 1씩 증가시킨다.
- [4] 스캔을 완료하고서, 최종 counter 변수 값을 clustering_factor로서 인덱스 통계에 저장한다.

t 테이블을 생성하면서 object_id 순으로 정렬했고, 이 컬럼에 대해 생성한 인덱스(t_object_id_idx)의 CF(=689)가 테이블 전체 블록 수(=709)에 근접한 것을 확인하기 바란다.

이런 식으로 측정된 CF 값은 옵티마이저가 (테이블 Full Scan과 비교해) 인덱스 Range Scan을 통한 테이블 액세스 비용을 평가하는 데에 사용된다.
참고로, (IO 비용 모델일 때) 인덱스를 이용한 테이블 액세스 비용을 계산하기 위해 오라클이 사용하는 공식은 다음과 같다.
```
비용 = blevel +                            -- 인덱스 수직적 탐색 비용
      (리프 블록 수 ✕ 유효 인덱스 선택도) +      -- 인덱스 수평적 탐색 비용
      (클러스터링 팩터 ✕ 유효 테이블 선택도)      -- 테이블 Random 액세스 비용
```
- blevel : 리프 블록에 도달하기 전 읽게 될 브랜치 블록 개수
- 유효 인덱스 선택도 : 전체 인덱스 레코드 중에서 조건절을 만족하는 레코드를 찾기 위해 스캔할 것으로 예상되는 비율(%)
- 유효 테이블 선택도 : 전체 레코드 중에서 인덱스 스캔을 완료하고서 최종적으로 테이블을 방문할 것으로 예상되는 비율(%)

### [클러스터링 팩터와 물리적 I/O]
"인덱스 CF가 좋다"고 하면 인덱스 정렬 순서와 테이블 정렬 순서가 서로 비슷하다는 것을 말한다.
인덱스 레코드는 항상 100% 정렬된 상태를 유지하는데, 테이블 레코드도 이와 비슷한 정렬 순서를 갖는다면 값이 같은 레코드들이 서로 군집해 있음을 뜻한다.
이는 궁극적으로 물리적인 디스크 I/O 횟수를 감소시키는 효과를 가져다준다.

오라클에서는 I/O는 블록 단위로 이루어지므로 인덱스를 통해 하나의 레코드를 읽으면 같은 블록에 속한 다른 레코드들도 함께 캐싱되는 결과를 가져올 것이고, CF가 좋은 인덱스였다면
(=같은 값들이 서로 군집해 있다면) 그 레코드들도 가까운 시점에 읽힐 가능성이 높다.
따라서 인덱스를 스캔하면서 읽는 테이블 블록들의 캐시 히트율이 높아지므로 물리적인 디스크 I/O 횟수가 감소하게 되는 것은 당연한 이치다.

반대로, 값이 같은 레코드들(예, 거주지역 = '제주')이 서로 멀리 떨어져 있다면 논리적으로 더 많은 블록을 읽어야 하므로 물리적인 디스크 I/O 횟수도 같이 증가하게 된다.
설상가상으로, 앞서 읽었던 테이블 블록을 다시 방문하고자 할 때 이미 캐시에서 밀려나고 없다면 같은 블록을 디스크에서 여러 번 읽게 되므로 I/O 효율은 더 나빠질 것이다.

### [클러스터링 팩터와 논리적 I/O]
인덱스 CF는 단적으로 말해, **인덱스를 경유해 테이블 전체 로우를 액세스할 때 읽을 것으로 예상되는 논리적인 블록 개수**를 말한다.
따라서 CF가 가장 좋을 때는 인덱스 통계에 나타나는 clustering_factor가 전체 테이블 블록 개수와 일치하고, 가장 안 좋을 때는 총 레코드 개수와 일치한다.
참고로, 모든 블록에 레코드 하나씩만 저장돼 있다면 clustering_factor는 항상 총 레코드 개수와 일치할 것이다.

- ### [물리적 I/O 계산을 위해 논리적 I/O를 측정]

  오라클은 인덱스가 가리키는 데이터의 흩어짐 정도를 인덱스 비용 계산식에 표현하기 위해 CF를 고안하였다.
  그런데 옵티마이저가 사용하는 비용 계산식은 기본적으로 물리적인 I/O만을 고려(논리적 I/O가 모두 물리적 I/O를 수반한다고 가정)하므로 CF도 궁극적으로 물리적 I/O 비용을 평가하기 위한 수단이라고
  말할 수 있다.

  하지만 논리적 I/O가 100% 물리적 I/O를 수반한다는 가정은 매우 비현실적이다. 특히, 앞서 읽었던 테이블 블록을 다시 읽을 때 실제로는 캐싱된 블록을 읽을 가능성이 훨씬 높다.

  한번 읽었던 테이블 블록이 버퍼 캐시에서 밀려나지 않도록 충분한 버퍼 캐시를 확보한 상태에서 인덱스를 통해 전체 테이블 레코드를 읽는 경우를 가정해 보자.
  그러면 위에서 본 t_object_name_idx 인덱스의 clustering_factor 수치(24,936)는 실제 발생할 수 있는 물리적 I/O 횟수와 전혀 동떨어진 값이 되고 만다.
  물리적으로 읽어야 할 전체 테이블 블록 개수는 고정(앞선 예제에서는 709개)돼 있기 때문이다.
  그럼에도 캐싱 효과를 예측하기가 매우 어렵기 때문에 옵티마이저는 CF를 통해 계산된 논리적 I/O 횟수를 그대로 물리적 I/O 횟수로 인정하고 인덱스 비용을 평가하는 것이다.

  결론적으로 말해, 인덱스 통계에서 볼 수 있는 clustering_factor는 인덱스를 통해 테이블을 액세스할 때 예상되는 논리적 I/O 개수를 더 정확히 표현하고 있다.

실제로 그런지 테스트를 통해 확인해 보자. 먼저, 앞서 생성한 t 테이블 전체 레코드를 CF가 좋은 t_object_id_idx 인덱스를 통해 액세스해 보자.
```
select /*+ index(t t_object_id_idx) */ count(*) from t
where  object_name >= ' '
and    object_id >= 0

Rows    Rows Source Operation
------- ---------------------------------------------------------------------------------
      1 SORT AGGREGATE (cr=801 pr=0 pw=0 time=115529 us)
  50093   TABLE ACCESS BY INDEX ROWID T (cr=801 pr=0 pw=0 time=1102123 us)
  50093     INDEX RANGE SCAN T_OBJECT_ID_IDX (cr=112 pr=0 pw=0 time=252664 us)
```
테이블을 액세스하는 단계에서 읽은 블록 수는 689(=801-112)개로서, 앞에서 인덱스 통계에서 보았던 clustering_factor 수치와 정확히 일치한다.

이번에는 CF가 안 좋은 t_object_name_idx 인덱스를 통해 액세스해 보자.
```
select /*+ index(t t_object_name_idx) */ count(*) from t
where  object_name >= ' '
and    object_id >= 0

Rows    Rows Source Operation
------- ---------------------------------------------------------------------------------
      1 SORT AGGREGATE (cr=25183 pr=0 pw=0 time=153224 us)
  50093   TABLE ACCESS BY INDEX ROWID T (cr=25183 pr=0 pw=0 time=1102085 us)
  50093     INDEX RANGE SCAN T_OBJECT_NAME_IDX (cr=247 pr=0 pw=0 time=252447 us)
```
테이블을 액세스하는 단계에서 읽은 블록 수는 24,936(=25,183-247)개로서, 역시 앞에서 보았던 clustering_factor 수치와 일치한다.

### [버퍼 Pinning에 의한 논리적 I/O 감소 원리]
똑같은 개수의 레코드를 읽는데 인덱스 CF에 따라 논리적인 블록 I/O 개수가 이처럼 심하게 차이 나는 이유는 무엇일까? 이유는, 인덱스를 통해 액세스되는 하나의 테이블 버퍼 블록을 Pinning하기 때문이다.

버퍼 Pinning에 대해 간단히 요약하면, 방금 액세스한 버퍼에 대한 Pin을 즉각 해제하지 않고 데이터베이스 Call 내에서 계속 유지하는 기능이라고 할 수 있다.
따라서 연속된 인덱스 레코드가 같은 블록을 가리킨다면, 래치 획득 과정을 생략하고 버퍼를 Pin한 상태에서 읽기 때문에 논리적인 블록 읽기(Logical Reads) 횟수가 증가하지 않는다.

이처럼 CF가 좋을 때는(완전히 같은 순서로 정렬돼 있을 필요는 없으며, 같은 테이블 블록에 모여 있기만 하면 됨) 인덱스를 통해 테이블 레코드를 읽는데 Random 블록 I/O가 적게 발생한다.

버퍼 Pinning 효과를 이용하거나 인위적으로 클러스터링 팩터를 높게 만들어 튜닝하는 사례를 다음 절 후반부에 소개하므로 참조하기 바란다.

<br/>

## (3) 인덱스 손익분기점
앞에서 설명한 것처럼 인덱스 rowid에 의한 테이블 액세스는 생각보다 고비용 구조이고, 따라서 일정량을 넘는 순간 테이블 전체를 스캔할 때보다 오히려 더 느려진다.
Index Range Scan에 의한 테이블 액세스가 Table Full Scan보다 느려지는 지점을 흔히 '손익 분기점'이라고 부른다.

인덱스에 의한 액세스가 Table Full Scan보다 더 느려지게 만드는 가장 핵심적인 두 가지 요인은 다음과 같다.
- 인덱스 rowid에 의한 테이블 액세스는 Random 액세스인 반면, Full Table Scan은 Sequential 액세스 방식으로 이루어진다.
- 디스크 I/O 시, 인덱스 rowid에 의한 테이블 액세스는 Single Block Read 방식을 사용하는 반면, Full Table Scan은 Multiblock Read 방식을 사용한다.

이런 요인에 의해 인덱스 손익분기점은 일반적으로 5 ~ 20%의 낮은 수준에서 결정되지만 CF에 따라 크게 달라진다.
인덱스의 CF가 나쁘면 같은 테이블 블록을 여러 번 반복 액세스하면서 논리적 I/O 개수가 증가하고 물리적 I/O 발생량도 증가하기 때문이다.
따라서 CF가 나쁘면 손익분기점은 5% 미만에서 결정되며, 심할 때는(BCHR가 매우 안 좋을 때) 1% 미만으로 떨어진다. 반대로 CF가 아주 좋을 때는 손익분기점이 90% 수준까지 올라가기도 한다.

인덱스 손익분기점은 데이터 상황에 따라 크게 달라지므로 5%, 10%, 20%처럼 정해진 수치 또는 범위를 정해 놓고 인덱스의 효용성을 판단하면 틀리기 쉽다.

오해하지 말 것은, 손익분기점은 인덱스가 항상 좋을 수 없음을 설명하려고 도입한 개념일 뿐 이를 높이기 위해 어떤 조치를 취하라는 뜻은 아니다.
즉, 테이블 스캔이 항상 나쁜 것은 아니며 바꿔 말해, 인덱스 스캔이 항상 좋은 것도 아니라는 사실을 인식시키기 위함이다.

물론 테이블을 Reorg 함으로써 CF를 좋게 만든다면 인덱스 손익분기점이 높아져 그 인덱스의 효용성이 증가하지만, 그런 방안은 최후의 수단이어야지 일상적인 튜닝 기법으로 남용하지 말아야 한다.

인덱스가 갖는 한계를 극복하려면 바로 이어서 설명할 DBMS 제공 기능들을 이용하거나 부분범위처리 원리를 적극 활용해야 한다.
인덱스 스캔 비효율이 없도록 잘 구성된 인덱스를 이용해 부분범위처리 방식으로 프로그램을 구현한다면 그 인덱스의 효용성은 100%가 된다. 무조건 인덱스를 사용하는 쪽이 유리하다는 것이다.

### [손익분기점을 극복하기 위한 기능들]
손익분기점 원리에 따르면 선택도가 높은 인덱스는 효용가치가 낮지만 대용량 테이블을 Full Scan 하는 것 역시 비효율이 크다.
그럴 때 오라클이 제공하는 기능들을 잘 활용하면 인덱스의 손익분기점 한계를 극복할 수 있다.

첫 번째 기능은 IOT(Index-Organized Table)로서, 테이블을 인덱스 구조로 생성하는 것을 말한다. 테이블 자체가 인덱스 구조이므로 항상 정렬된 상태를 유지한다.
그리고 인덱스 리프 블록이 곧 데이터 블록이어서 인덱스를 수직 탐색한 다음에 테이블 레코드를 읽기 위한 추가적인 Random 액세스가 불필요하다. IOT에 대한 자세한 설명은 6절을 참조하기 바란다.

두 번째는 클러스터 테이블(Clustered Table)이다. 키 값이 같은 레코드는 같은 블록에 모이도록 저장하기 때문에 클러스터 인덱스를 이용할 때는 테이블 Random 액세스가 키 값별로 한 번씩만 발생한다.
클러스터에 도달해서는 Sequential 방식으로 스캔하기 때문에 넓은 범위를 읽더라도 비효율이 없다. 클러스터에 대한 자세한 설명도 6절을 참조하기 바란다.

인덱스 손익분기점을 극복하기 위한 세 번째 기능은 파티셔닝이다.
읽고자 하는 데이터가 많을 때는 인덱스를 이용하지 않는 편이 낫다고 하지만, 수천만 건에 이르는 테이블을 Full Scan 해야 한다면 난감하기 그지없다.
그럴 때, 대량 범위 조건으로 자주 사용되는 컬럼 기준으로 테이블을 파티셔닝한다면 Full Table Scan 하더라도 일부 파티션만 읽고 멈추도록 할 수 있다.

클러스터는 기준 키 값이 같은 레코드를 블록 단위로 모아 놓지만 파티셔닝은 세그먼트 단위로 모아 놓는 점이 다르다. 자세한 내용은 6장에서 설명한다.