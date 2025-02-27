# 02. 병렬 Order By와 Group By

P->P 데이터 재분배는 주로 병렬 order by, 병렬 group by, 병렬 조인을 포함한 SQL에서 나타난다.
아주 단순한 SQL이 아니고서야 대부분 이들 오퍼레이션을 포함하므로 거의 모든 병렬 SQL에서 Inter-Operation Parallelism이 일어난다고 보면 틀림없다.
<br/>
<br/>
## (1) 병렬 Order By
병렬 order by와 병렬 group by를 설명하기 위해 사용할 테스트 데이터를 먼저 만들어 보자.
```
SQL> create table 고객
  2  as
  3  select rownum 고객ID
  4       , dbms_random.string('U', 10) 고객명
  5       , mod(rownum, 10) + 1 고객등급
  6       , to_char(to_date('20090101','yyyymmdd') + (rownum - 1), 'yyyymmdd') 가입일
  7  from   dual
  8  connect by level <= 1000;

SQL> exec dbms_stats.gather_table_stats(user, '고객');
```

방금 만든 고객 테이블을 병렬로 읽어 고객명 순으로 정렬하는 쿼리와 실행계획은 다음과 같다.
```
SQL> set autotrace traceonly exp
SQL> select /*+ full(고객) parallel(고객 2) */
  2         고객ID, 고객명, 고객등급
  3  from   고객
  4  order by 고객명;

---------------------------------------------------------------------------------------
| Id  | Operation                      | Name     | Rows  |   TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |          |  1000 |       |      |            |
|   1 |   PX COORDINATOR               |          |       |       |      |            |
|   2 |     PX SEND QC (ORDER)         | :TQ10001 |  1000 | Q1,01 | P->S | QC (ORDER) |
|   3 |       SORT ORDER BY            |          |  1000 | Q1,01 | PCWP |            |
|   4 |         PX RECEIVE             |          |  1000 | Q1,01 | PCWP |            |
|   5 |           PX SEND RANGE        | :TQ10000 |  1000 | Q1,00 | P->P | RANGE      |
|   6 |             PX BLOCK ITERATOR  |          |  1000 | Q1,00 | PCWC |            |
|   7 |               TABLE ACCESS FULL| 고객      |  1000 | Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------
```

v$pq_tqstat 뷰(Parallel Query Table Queue Statistics)의 활용법을 소개하려고 한다.

order by를 병렬로 수행하려면 테이블 큐를 통한 데이터 재분배가 필요한데, 쿼리 수행이 완료된 직후에 같은 세션에서 v$pq_tqstat를 쿼리해 보면 아래와 같이 테이블 큐를 통한 데이터 전송 통계를
확인해 볼 수 있다.
```
SQL> set autotrace off
SQL> /
     ......

1000 개의 행이 선택되었습니다.

SQL> break on dfo_no on tq_id on server_type
SQL> select tq_id, server_type, process, num_rows, bytes, waits
  2  from   v$pq_tstat
  3  order by
  4         dfo_number
  5       , tq_id
  6       , decode(substr(server_type, 1, 4), 'Rang', 1, 'Prod', 2, 'Cons', 3)
  7       , process;

  TQ_ID SERVER_TYPE PROCESS       NUM_ROWS        BYTES      WAITS
------- ----------- ---------- ----------- ------------ ----------
      0 Ranger      QC                 182        7,552          2
        Producer    P002               455        9,603          5
                    P003               545       11,612          4
        Consumer    P000               479       10,125          6
                    P001               521       11,032          5
      1 Producer    P000               479       10,122          0
                    P001               521       11,612          0
        Consumer    QC               1,000       21,131          1

8 개의 행이 선택되었습니다.
```

병렬 쿼리 수행 속도가 예상만큼 빠르지 않다면 테이블 큐를 통한 데이터 전송량에 편차가 크지 않은지 확인해 볼 필요가 있는데, 그럴 때 v$pq_tqstat 뷰가 유용하게 쓰인다.
<br/>
<br/>

## (2) 병렬 Group By
병렬 group by는 어떻게 수행되는지 바로 앞서 만들어 둔 고객 테이블을 이용해 살펴 보자.
```
SQL> set autotrace traceonly exp
SQL> select /*+ full(고객) parallel(고객 2) */
  2         고객명, count(*)
  3  from   고객
  4  group by 고객명 ;

---------------------------------------------------------------------------------------
| Id  | Operation                      | Name     | Rows  |   TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |          |   969 |       |      |            |
|   1 |   PX COORDINATOR               |          |       |       |      |            |
|   2 |     PX SEND QC (RANDOM)        | :TQ10001 |   969 | Q1,01 | P->S | QC (RAND)  |
|   3 |       HASH GROUP BY            |          |   969 | Q1,01 | PCWP |            |
|   4 |         PX RECEIVE             |          |  1000 | Q1,01 | PCWP |            |
|   5 |           PX SEND HASH         | :TQ10000 |  1000 | Q1,00 | P->P | HASH       |
|   6 |             PX BLOCK ITERATOR  |          |  1000 | Q1,00 | PCWC |            |
|   7 |               TABLE ACCESS FULL| 고객      |  1000 | Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------
```

10gR2에서 hash group by가 소개되었고, 위 쿼리에 order by를 추가하면 아래와 같이 sort group by로 바뀐다.
```
---------------------------------------------------------------------------------------
| Id  | Operation                      | Name     | Rows  |   TQ  |IN-OUT| PQ Distrib |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |          |   969 |       |      |            |
|   1 |   PX COORDINATOR               |          |       |       |      |            |
|   2 |     PX SEND QC (ORDER)         | :TQ10001 |   969 | Q1,01 | P->S | QC (ORDER) |
|   3 |       SORT GROUP BY            |          |   969 | Q1,01 | PCWP |            |
|   4 |         PX RECEIVE             |          |  1000 | Q1,01 | PCWP |            |
|   5 |           PX SEND RANGE        | :TQ10000 |  1000 | Q1,00 | P->P | RANGE      |
|   6 |             PX BLOCK ITERATOR  |          |  1000 | Q1,00 | PCWC |            |
|   7 |               TABLE ACCESS FULL| 고객      |  1000 | Q1,00 | PCWP |            |
---------------------------------------------------------------------------------------
```

병렬 order by 실행계획을 방금 것과 비교해 보면, 3번 단계의 'sort order by'가 'sort group by'로 바뀌는 것 말고는 모두 같다.
즉, order by와 group by를 병렬로 처리하는 내부 수행원리는 기본적으로 같다는 뜻이다.

그리고 병렬 sort group by와 병렬 hash group by의 차이점은 데이터 분배 방식에 있다. 즉, group by 키 정렬 순서에 따라 분배하느냐 해시 함수 결과 값에 따라 분배하느냐의 차이다.
group by 결과를 QC에게 전송할 때도, sort group by는 값 순서대로(QC ORDER) 진행하지만 hash group by는 먼저 처리가 끝난 순서대로(QC RANDOM) 진행한다.

앞서 예로 들었던 명함 관리를 다시 생각해 보자. 이번에는 명함을 그룹핑해서 고객사별로 연락 가능한 담당자가 몇 명인지를 집계하려고 한다.

우선, 8명의 영업 사원이 각자 관리하던 명함을 집계하고 나면 그 결과를 영업부장(QC에 해당)이 최종적으로 집계하는 경우를 보자.
OO물산 고객이 첫 번째 영업사원에게서 10명, 두 번째 영업사원에게서 5명으로 집계된다면 영업부장은 이를 합쳐 15명으로 최종 합계를 구해야 한다.
만약 각 영업 사원이 집계한 고객사에 중복이 없다면(즉, 고객사별로 명함이 한 장씩만 있다면) 영업 사원들이 한 것과 같은 일량의 집계 작업을 영업부장이 한 번 더 수행하는 셈이 된다.
병렬 order by도 각각 정렬된 결과집합을 최종 머지(Merge)하는 작업의 복잡성 때문에 이와 같은 방식은 그다지 효과적이지 않다.

order by와 마찬가지로, 병렬 group by도 두 집합으로 나눠 한 쪽은 명함을 읽어서 분배하고 다른 한쪽은 그것을 받아 집계하도록 해야 병렬 처리 효과를 극대화할 수 있다.

바로 앞서 보았던 group by 쿼리 예제로 다시 돌아와, 직접 쿼리를 수행하고서 v$pq_tqstat 결과를 통해 병렬 group by 과정을 설명해 보자. (hash group by를 기준으로 설명한다.)
```
SQL> set autotrace off
SQL> /
     .....

969 개의 행이 선택되었습니다.

SQL> @pq_tq -> v$pq_tqstat를 쿼리하는 스트립트

  TQ_ID SERVER_TYPE PROCESS       NUM_ROWS        BYTES      WAITS
------- ----------- ---------- ----------- ------------ ----------
      0 Producer    P002               422        2,150          2
                    P003               578        2,930          2
        Consumer    P000               513        2,605          4
                    P001               487        2,475          3
      1 Producer    P000               496        4,524          2
                    P001               473        4,317          2
        Consumer    QC                 969        8,841          3

7 개의 행이 선택되었습니다.
```

P002와 P003은 각각 422와 578개 로우를 읽어 해시 함수를 적용하고, 거기서 반환된 값에 따라 두 번째 서버 집합으로 데이터를 분배하였다.
그 과정에서 P000은 513개 로우를 받아 group by한 결과 496개 로우를 QC에게 전송하였다. P001으로 487개 로우를 받아 group by한 결과 473개 로우를 QC에게 전송하였다.

해시 값에 따라 데이터를 분배하였으므로 P000과 P001은 서로 배타적인 집합을 가지고 group by를 수행한다. 따라서 QC가 한 번 더 집계하는 과정 없이 받은 데이터를 그대로 클라이언트에게 전송하면 된다.

### [Group By가 두 번 나타날 때의 처리 과정]
병렬 group by 실행계획에 아래와 같이 group by가 두 번(ID 3, ID 6) 나타나는 경우가 있다. group by를 두 번 수행하는 것일까?
```
SQL> set autotrace traceonly exp
SQL> select /*+ full(고객) parallel(고객 2) */
  2         고객등급, count(*)
  3  from   고객
  4  group by 고객등급 ;

-----------------------------------------------------------------------------------------
| Id  | Operation                        | Name     | Rows  |   TQ  |IN-OUT| PQ Distrib |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |          |    10 |       |      |            |
|   1 |   PX COORDINATOR                 |          |       |       |      |            |
|   2 |     PX SEND QC (RANDOM)          | :TQ10001 |    10 | Q1,01 | P->S | QC (RAND)  |
|   3 |       HASH GROUP BY              |          |    10 | Q1,01 | PCWP |            |
|   4 |         PX RECEIVE               |          |    10 | Q1,01 | PCWP |            |
|   5 |           PX SEND HASH           | :TQ10000 |    10 | Q1,00 | P->P | HASH       |
|   6 |             HASH GROUP BY        |          |    10 | Q1,00 | PCWP |            |
|   7 |               PX BLOCK ITERATOR  |          |  1000 | Q1,00 | PCWC |            |
|   8 |                 TABLE ACCESS FULL| 고객      |  1000 | Q1,00 | PCWP |            |
-----------------------------------------------------------------------------------------
```

비밀은 group 기준 컬럼의 선택도(selectivity)에 있다. 앞서 만들었던 고객 테이블에 있는 각 컬럼의 선택도와 카디널리티를 조회해 보자.
```
SQL> set autotrace off
SQL> select column_name
  2       , num_distinct
  3       , num_nulls
  4       , 1/num_distinct selectivity
  5       , round(1/numdistinct * t.num_rows, 2) cardinality
  6  from   user_tables t, user_tab_columns c
  7  where  t.table_name = '고객'
  8  and    c.table_name = t.table_name
  9  order by column_id ;

COLUMNS_NAME          NUM_DISTINCT  NUM_NULLS SELECTIVITY CARDINALITY
--------------------- ------------ ---------- ----------- -----------
고객ID                         1000          0        .001           1
고객명                           969          0  .001031992        1.03
고객등급                          10          0          .1         100
가입일                          1000          0        .001           1
```

고객명의 선택도는 0.001(=0.1%)로서 매우 낮고, 고객등급의 선택도는 0.1(=10%)로서 비교적 높다. 즉, 고객등급으로 group by 한 결과집합은 원래의 데이터 집합과 비교하면 1/10 크기다.
따라서 첫 번째 서버 집합이 읽은 데이터를 먼저 group by 하고 나서 두 번째 서버 집합에 전송한다면 프로세스 간 통신량이 1/10로 줄어 그만큼 병렬 처리 과정에서 생기는 병목을 줄일 수 있다.

아래는 쿼리를 직접 수행하고 나서 v$pq_tqstat 뷰를 조회한 결과다.
```
SQL> select /*+ full(고객) parallel(고객 2) */
  2         고객등급, count(*)
  3  from   고객
  4  group by 고객등급 ;

   고객등급  COUNT(*)
--------- ----------
        1        100
        6        100
      ...      .....

10 개의 행이 선택되었습니다.

SQL> @pq_tq -> v$pq_stat를 쿼리하는 스크립트

  TQ_ID SERVER_TYPE PROCESS       NUM_ROWS        BYTES      WAITS
------- ----------- ---------- ----------- ------------ ----------
      0 Producer    P002                10          200          0
                    P003                10          200          0
        Consumer    P000                10          200         39
                    P001                10          200         39
      1 Producer    P000                 5           60          2
                    P001                 5           60          1
        Consumer    QC                  10          120          3

7 개의 행이 선택되었습니다.
```

앞에서 group by가 한 번 나타날 때와 비교했을 때 프로세스 간에 주고받은 데이터 건수가 현격히 준 것을 확인할 수 있다.

P002와 P003이 group by한 결과집합을 해시 값에 따라 분배하고 나면 P000과 P001은 서로 배타적인 집합을 갖게 된다.
하지만 받은 데이터가 최종 집계된 값이 아니므로 P000과 P001 프로세스는 한 번 더 group by를 수행해야만 한다.
예를 들어, P000 프로세스가 6등급 데이터를 P002와 P003 프로세스 모두로부터 받을 수 있기 때문이다.

두 번째 서버집합의 최종 집계가 끝나고 나면 QC는 이를 받아서 그대로 클라이언트에게 전송하면 된다.

참고로, 선택도가 낮은 컬럼(위 사례에서 '고객명')으로 group by 할 때도 강제로 이 방식을 사용하도록 하려면 _groupby_nopushdown_cut_ratio 파라미터를 0으로 세팅(세션 레벨 변경도 가능)하면 된다.
이 파라미터의 기본 값인 3은 group by 기준 컬럼의 선택도에 따라 옵티마이저가 방식을 결정하도록 하는 것이다.

참고로, 11g에서는 gby_pushdown, no_gby_pushdown 힌트가 추가돼 파라미터 변경 없이도 사용자가 group by 방식을 조정할 수 있게 되었다.