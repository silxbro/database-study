# 05. 테이블 Random 액세스 최소화 튜닝

본 절에서는 테이블 Random 액세스 최소화를 위한 튜닝 방안과 사례를 소개한다.

<br/>

## (1) 인덱스 컬럼 추가
emp 테이블에 현재 PK 이외에 [deptno + job] 순으로 구성된 emp_x01 인덱스 하나만 있는 상태에서 아래 쿼리를 수행하려고 한다.
```
select /*+ index(emp emp_x01) */ *
from   emp
where  deptno = 30
and    sal >= 2000
```
위 조건을 만족하는 사원이 단 한 명뿐인데, 이를 찾기 위해 테이블 액세스는 6번 발생하였다고 하자.

인덱스 구성을 [deptno + sal] 순으로 바꿔주면 좋겠지만 실 운영 환경에서는 인덱스 구성을 함부로 바꾸기가 쉽지 않다. 기존 인덱스를 사용하는 SQL이 있을 수 있기 때문이다.

할 수 없이 인덱스를 새로 만들어야겠지만 이런 식으로 인덱스를 추가해 나가다 보면 테이블마다 인덱스가 수십 개씩 달려 배보다 배꼽이 더 커지게 된다.
인덱스 관리 비용이 증가함은 물론 DML 부하에 따른 트랜잭션 성능 저하가 생길 수 있응믈 쉽게 예상할 수 있다.

이럴 때, 기존 인덱스에 sal 컬럼을 추가하는 것만응로 큰 효과를 거둘 수 있다. 인덱스 스캔량은 줄지 않지만 테이블 Random 액세스 횟수를 줄여주기 때문이다.

인덱스에 컬럼을 추가해 튜닝했던 실제 사례를 살펴보자.
```
select 렌탈관리번호, 고객명, 서비스관리번호, 서비스번호, 예약접수일시
     , 방문국가코드1, 방문국가코드2, 방문국가코드3, 로밍승인번호, 자동로밍여부
from   로밍렌탈
where  서비스번호 Like '010%'
and    사용여부 = 'Y'

Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.010        0.012       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch        78    10.150       49.199   27830    266968          0   1909
-------- ------ --------- ------------ ------- --------- ---------- ------
Total        80    10.160       49.211   27830    266968          0   1909

Rows     Row Source Operation
-------- ------------------------------------------------------
    1909 TABLE ACCESS BY INDEX ROWID 로밍렌탈 (cr=266968 pr=27830 pw=0 time= ...)
  266476   INDEX RANGE SCAN 로밍렌탈_N2 (cr=1011 pr=900 pw=0 time=1893462 us)OF ...
```
위 SQL을 위해 '서비스번호' 하나로 구성된 단일 컬럼 인덱스(로밍렌탈_N2)가 사용됐는데, 테이블을 액세스하는 단계에서만 265,957(=266,968-1,011)개의 블록 I/O가 발생했고 이는 전체 I/O의
99.6%를 차지하는 양이다. 총 소요시간은 데이터베이스 구간에서만 49초에 이른다.

테이블을 총 266,476번(인덱스 스캔 단계 출력 개수) 방문하는 동안 블록 I/O가 265,957개 발생한 것을 보면, 인덱스 클러스터링 팩터가 아주 안 좋은 상태인 것도 알 수 있다.
데이터량이 워낙 많다보니 서비스번호 조건을 만족하는 데이터가 뿔뿔이 흩어져 있는 것이다.

클러스터링 팩터가 나빠 Random I/O가 평균 이상으로 많이 발생하는 현상은 테이블을 Reorg하지 않는 한 어쩔 수 없다.
문제는 테이블을 총 266,476번 방문했지만 최종 결과집합이 1,909건(테이블 액세스 단계 출력 건수)뿐이라는 데에 있다. 테이블을 방문하고서 사용여부 = 'Y' 조건을 체크하는 과정에서 대부분 버려진 것이다.

아래는 로밍렌탈_N2 인덱스에 '사용여부' 컬럼을 추가하고 나서의 SQL 트레이스 결과다.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.001       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch        78     0.140        0.154       0      2902          0   1909
-------- ------ --------- ------------ ------- --------- ---------- ------
Total        80     0.140        0.156       0      2902          0   1909

Rows     Row Source Operation
-------- ------------------------------------------------------
    1909 TABLE ACCESS BY INDEX ROWID 로밍렌탈 (cr=2902 pr=0 pw=0 time= ...)
    1909   INDEX RANGE SCAN 로밍렌탈_N2 (cr=1001 pr=0 pw=0 time=198557 us)OF ...
```
테이블을 1,909번 방문했지만 모두 결과집합에 포함되었으므로 비효율은 전혀 없다.

<br/>

## (2) PK 인덱스에 컬럼 추가
단일 테이블을 PK로 액세스할 때는 단 한 건만 조회하는 것이므로 테이블 Random 액세스도 단 1회 발생한다.
하지만 NL 조인할 때 Inner쪽(=right side)에서 액세스될 때는 Random 액세스 부하가 만만치 않다.
특히 Outer 테이블에서 Inner 테이블 쪽으로 조인 액세스가 많은 상황에서 Inner쪽 필터 조건에 의해 버려지는 레코드가 많다면 그 비효율은 매우 심각한 것일 수 있다.

아래 쿼리는 emp를 기준으로 NL 조인하고, 조인에 성공한 14건 중 loc = 'NEW YORK'인 레코드만 취하므로 최종 결과집합은 3건뿐이다.
```
select /*+ ordered use_nl(b) */ *
from   emp e, dept d
where  d.deptno = e.deptno
and    d.loc = 'NEW YORK'

Rows     Row Source Operation
-------- ------------------------------------------------------
       3 NESTED LOOPS (cr=25 pr=0 pw=0 time=442 us)
      14   TABLE ACCESS FULL EMP (cr=8 pr=0 pw=0 time=139 us)
       3   TABLE ACCESS BY INDEX ROWID DEPT (cr=17 pr=0 pw=0 time=568 us)
      14     INDEX UNIQUE SCAN DEPT_PK (cr=3 pr=0 pw=0 time=255 us)
```
dept_pk 인덱스에 loc 컬럼을 추가하면 불필요한 11번의 Random 액세스를 없앨 수 있지만 PK 인덱스에는 컬럼을 추가할 수 없다.
그러다 보니 [PK 컬럼 + 필터조건 컬럼] 형태의 새로운 Non-Unique 인덱스를 추가하는 경우가 종종 있다. 그럴 때 Non-Unique 인덱스를 이용해 PK 제약을 설정한다면 인덱스 개수를 줄일 수 있다.

PK 제약에는 중복 값 확인을 위한 인덱스가 반드시 필요하다. 인덱스가 없다면 값이 입력될 때마다 테이블 전체를 읽어 중복 값 존재 여부를 체크해야 하기 때문이다.
하지만 중복 체크를 위해 반드시 Unique 인덱스가 필요한 것은 아니며, Non-Unique 인덱스로도 가능하다.
Non-Unique 인덱스를 이용하면 중복 여부를 체크할 때 one-plus 스캔이 발생하는 약간의 비효율이 있기는 하지만 무시할 만하다.

PK 제약을 위해 Non-Unique 인덱스를 사용하도록 하는 방법은 다음과 같다.
```
alter table dept drop primary key;

create index dept_x01 on dept(deptno, loc);

alter table dept add
constraint dept_pk primary key(deptno) using index dept_x01;
```
아래는 위와 같이 PK 제약과 인덱스를 다시 만들고 나서 SQL을 다시 수행했을 때의 트레이스 결과다. 조인에 성공하고서 필터 조건 체크까지 완료된 3건만 dept 테이블을 액세스하였다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
       3 TABLE ACCESS BY INDEX ROWID DEPT (cr=14 pr=0 pw=0 time=302 us)
      18   NESTED LOOPS (cr=8 pr=0 pw=0 time=139 us)
      14     TABLE ACCESS FULL EMP (cr=8 pr=0 pw=0 time=145 us)
       3     INDEX RANGE SCAN DEPT_X01 (cr=4 pr=0 pw=0 time=220 us)
```
참고로, PK 제약을 위해 사용되는 인덱스는 PK 제약 순서와 서로 일치하지 않아도 상관없다. 중복 값 유무를 체크하는 용도이므로 PK 제약 컬럼들이 선두에 있기만 하면 된다.
예를 들어, PK 제약 컬럼 구성이 [고객번호, 상품번호, 거래일자]일 때 아래 1 ~ 3번 인덱스는 PK 제약을 위해 사용될 수 있지만 4와 같은 구성은 허용되지 않는다.
4와 같은 구성으로도 중복 값 유무를 확인할 수는 있지만 비효율적이기 때문에 허용하지 않는 것이다.

- [1] 거래일자 + 고객번호 + 상품번호
- [2] 상품번호 + 거래일자 + 고객번호 + 거래구분
- [3] 고객번호 + 거래일자 + 상품번호 + 매체구분 + 거래구분
- [4] 고객번호 + 상품번호 + 거래구분 + 거래일자

<br/>

## (3) 컬럼 추가에 따른 클러스터링 팩터 변화
인덱스에 컬럼을 추가함으로써 테이블 Random 액세스 부하를 줄이는 효과가 있지만 인덱스 클러스터링 팩터가 나빠지는 부작용을 초래할 수도 있다.

테스트를 위해 아래와 같이 테이블과 인덱스를 생성해 보자.
```
SQL> create table t
  2  as
  3  select * from all_objects
  4  order by object_type;

SQL> create index t_idx on t(object_type);

SQL> exec dbms_stats.gather_table_stats(user, 't');  -- 통계정보 수집

SQL> @cl_factor  -- 클러스터링 팩터를 조회하는 스크립트

INDEX_NAME            CL_FACTOR      TAB_BLKS      TAB_ROWS
---------------- -------------- ------------- -------------
T_IDX                       689           709        50,137
```
클러스터링 팩터가 매우 좋은 상태임을 알 수 있다. 이 인덱스를 경유해 테이블을 액세스할 때 실제 블록 I/O가 얼마나 발생하는지 확인해 보자.
```
select /*+ index(t t_idx) */ count(object_name), count(owner) from t
where  object_type > ' '

Rows     Row Source Operation
-------- ------------------------------------------------------
       1 SORT AGGREGATE (cr=829 pr=0 pw=0 time=37213 us)
   50137   TABLE ACCESS BY INDEX ROWID T (cr=829 pr=0 pw=0 time=75094 us)
   50137     INDEX RANGE SCAN T_IDX (cr=140 pr=0 pw=0 time=151867 us)
```
object_name과 owner는 null 허용 컬럼이므로 이를 읽으려고 테이블을 50,137번 방문하지만 다행히 블록 I/O는 689(=829-140)회만 발생하였다. 인덱스 클러스터링 팩터가 좋기 때문이다.

object_type과 object_name 두 조건으로 조회하는 또 다른 쿼리가 있어 성능 향상을 위해 아래와 같이 t_idx 인덱스에 object_name을 추가하기로 했다고 하자.
```
SQL> drop index t_idx;

SQL> create index t_idx on t(object_type, object_name);

-- 참고로, 10g부터는 인덱스를 생성할 때 통계정보가 자동 수집됨
SQL> exec dbms_stats.gather_index_stats(user, 't_idx');

SQL> @cl_factor

INDEX_NAME            CL_FACTOR      TAB_BLKS      TAB_ROWS
---------------- -------------- ------------- -------------
T_IDX                    33,843           709        50,137
```
그런데 컬럼을 추가하고 나니 인덱스 클러스터링 팩터가 689에서 33,843로 나빠졌다. 이유가 무엇일까?

1절에서 설명한 바와 같이 인덱스 내에서 키 값이 같은 레코드는 rowid 순으로 정렬된다.
그런데 여기에 변별력이 좋은 object_name 같은 컬럼을 추가하면 rowid 이전에 object_name 순으로 정렬되므로 클러스터링 팩터를 나쁘게 만드는 요인으로 작용한다.

실제 성능이 나빠지는지 확인하기 위해 앞서 수행한 SQL을 다시 수행해 보자.
참고로, object_name 컬럼이 추가되었으므로 count(object_name)을 위해서라면 테이블을 방문할 필요가 없지만 count(owner) 때문에 여전히 테이블을 방문해야만 한다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
       1 SORT AGGREGATE (cr=34153 pr=0 pw=0 time=148645 us)
   50137   TABLE ACCESS BY INDEX ROWID T (cr=34153 pr=0 pw=0 time=1052952 us)
   50137     INDEX RANGE SCAN T_IDX (cr=310 pr=0 pw=0 time=252653 us)
```
인덱스에 컬럼을 추가했더니 기존에 사용했던 쿼리 성능이 매우 나빠진 것을 볼 수 있다.
인덱스 스캔 단계에서의 블록 I/O 횟수가 140에서 310으로 증가하였고, 테이블 Random 액세스 단계에서의 블록 I/O 횟수는 689(=829-140)에서 33,843(=34,153-310)으로 증가하였다.

결론적으로, object_type처럼 **변별력이 좋지 않은 컬럼 뒤에 변별력이 좋은 다른 컬럼을 추가할 때는 클러스터링 팩터 변화에 주의를 기울여야 한다.**

<br/>

## (4) 인덱스만 읽고 처리
테이블을 액세스하고서 필터 조건에 의해 버려지는 레코드가 많을 때, 인덱스에 컬럼을 추가함으로써 얻는 성능 효과를 살펴보았다.
그런데 테이블 Random 액세스가 아무리 많더라도 필터 조건에 의해 버려지는 레코드가 거의 없다면 거기에 비효율은 없다. 이때는 어떻게 튜닝해야 할까?

이때는 아예 테이블 액세스가 발생하지 않도록 모든 필요한 컬럼을 인덱스에 포함시키는 방법을 고려해 볼 수 있다.
MS-SQL Server에서 사용하는 용어긴 하지만 그런 인덱스를 'Covered 인덱스'라 부르고, 인덱스만 읽고 처리하는 쿼리를 'Covered 쿼리'라고 한다.

### [사례 1]
아래 SQL 트레이스 결과를 보자.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.050        0.048       0         4          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2    11.130       31.320   10358    630340          0      7
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4    11.180       31.369   10358    630344          0      7

Rows     Row Source Operation
-------- ------------------------------------------------------
       1 INDEX RANGE SCAN COMM_CD_DTL_N2 (cr=2 pr=0 pw=0 time=52 us)
       7 SORT AGGREGATE (cr=623531 pr=10321 pw=0 time=30855289 us)
  707236   PARTITION RANGE ALL PARTITION (cr=623531 pr=10321 pw=0 time= ...)
  707236     PARTITION LIST SINGLE PARTITION (cr=623531 pr=10321 pw=0 ...)
  707236       TABLE ACCESS BY LOCAL INDEX ROWID CMPGN_OBJ_CUST (cr=623531 pr=10321 ...)    **
  707236         INDEX RANGE SCAN CMPGN_OBJ_CUST_N1 PARTITION (cr=10652 pr=6737 pw=0 ...)
       7 SORT AGGREGATE (cr=6773 pr=36 pw=0 time=450888 us)
    6757   PARTITION RANGE ALL PARTITION (cr=6773 pr=36 pw=0 time= ...)
    6757     PARTITION LIST SINGLE PARTITION (cr=6773 pr=36 pw=0 ...)
    6757       TABLE ACCESS BY LOCAL INDEX ROWID CMPGN_OBJ_CUST PARTITION (cr=6773 ...)
    6757         INDEX RANGE SCAN CMPGN_OBJ_CUST_N1 PARTITION (cr=196 pr=36 ...)
       7 SORT ORDER BY (cr=630340 pr=10358 pw=0 time=31320373 us)
       7   NESTED LOOPS (cr=34 pr=1 pw=0 time=13577 us)
       7     NESTED LOOPS OUTER (cr=18 pr=1 pw=0 time=13070 us)
       7       NESTED LOOPS (cr=8 pr=0 pw=0 time=706 us)
       1         INDEX RANGE SCAN CONT_CHNL_LST_N1 (cr=1 pr=0 pw=0 time=35 us)
       7         TABLE ACCESS BY INDEX ROWID CMPGN_SCHD (cr=7 pr=0 pw=0 time= ... )
      12           INDEX RANGE SCAN CMPGN_SCHD_N1 (cr=2 pr=0 pw=0 time=55 us)
       1       TABLE ACCESS BY INDEX ROWID ASGN_COND (cr=10 pr=1 pw=0 time=12216 us)
       1         INDEX UNIQUE SCAN ASGN_COND_PK (cr=9 pr=0 pw=0 time=244 us)O
       7     TABLE ACCESS BY INDEX ROWID CMPGN_BRIEF (cr=16 pr=0 pw=0 time= ...)
       7       INDEX UNIQUE SCAN CMPGN_BRIEF_PK (cr=9 pr=0 pw=0 time=238 us)
```
SQL을 보지 않더라도 어느 부분에서 비효율이 발생하는지 알 수 있다. 총 63만개 블록 I/O가 발생했는데, 그 중 98.9%가 위쪽 cmpgn_obj_cust 테이블을 액세스하는 단계에서 발생하고 있다.
실제 SQL문을 확인해 보자.
```
select * from (
  select ...
       , (select /*+ index (cmpg_obj_cust cmpgn_obj_cust_nl) */
                 count(cmpgn_obj_num)
          from   cmpgn_obj_cust
          where  cmpgn_num = zcs.cmpgn_num
          and    cmpgn_ts = zcs.strtgy_ts
          and    cont_chnl_cd = zcs.cont_chnl_cd
          and    cont_rslt_cl_cd is null
          and    csr_id is not null
          and    asgn_yn = 'Y') as utry_cnt
          ...
  from    cmpgn_brief zcb
        , cmpgn_schd zcs
        , asgn_cond zac
   where  zcs.eff_end_dtm = '99991231235959'
   and    zcs.cont_chnl_cd in (select cont_chnl_cd
                               from   cont_chnl_lst
                               where  sale_org_id = 'B101010000')
   and    zcs.asgn_oper_yn = 'Y'
   and    to_char(sysdate, 'yyyymmdd') between zcs.exec_sta_dt and zcs.exec_end_dt
   and    zcs.dialing_mode_st_cd = 'V'
   and    zcb.cmpgn_num    = zcs.cmpgn_num
   and    zcb.cmpgn_st_cd in ('ST207', 'ST310')
   and    zac.cmpgn_num(+) = zcs.cmpgn_num
   and    zac.strtgy_ts(+) = zcs.strtgy_ts
   and    zac.cont_chnl_cd(+) = zcs.cont_chnl_cd
     ) zcs_1
where      (zcs_1.utry_cnt > 0) or (zcs_1.retry_cnt > 0)
order by zcs_1.exec_sta_dt desc
```
라인 수를 줄이기 위해 쿼리 중 일부 내용을 생략하긴 했지만 트레이스에서 부하 지점으로 확인된 부분은 위에서 세 번째 줄 이하 스칼라 서브쿼리와 관련 있다.

이 스칼라 서브쿼리를 위해 사용된 기존 인덱스 구성은 다음과 같다.
```
cmpgn_obj_cust_nl : cmpgn_num + strtgy_ts + cont_chnl_cd
                  + cont_rslt_cl_cd + exec_end_dt + csr_id
```
스칼라 서브쿼리에 사용된 조건절 컬럼 중 asgn_yn만 빼고 모두 인덱스에 포함돼 있다.
asgn_yn = 'Y' 조건에 의해 최종적으로 버려지는 레코드는 하나도 없었지만(즉, 비효율은 없지만) 이 값을 체크하기 위한 테이블 Random 액세스 부하가 심각하다.

기존 인덱스가 이미 6개 컬럼으로 이루어졌지만 테이블 Random 액세스가 전혀 발생하지 않도록 뒤쪽에 asgn_yn 컬럼을 하나 더 추가해 보았다.
(참고로, select-list에서 count 함수에 사용된 cmpgn_obj_num 컬럼도 인덱스 구성에 포함되지 않았지만 not null 제약이 설정돼 있었다.
따라서 이 컬럼을 읽으려고 테이블을 액세스하지는 않으므로 인덱스 구성에 포함시키지 않아도 된다.)
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.050        0.048       0         4          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch         2     0.680        0.951      12     10883          0      7
-------- ------ --------- ------------ ------- --------- ---------- ------
Total         4     0.730        0.996      12     10887          0      7

Rows     Row Source Operation
-------- ------------------------------------------------------
       1 INDEX RANGE SCAN COMM_CD_DTL_N2 (cr=2 pr=0 pw=0 time=34 us)
       7 SORT AGGREGATE (cr=10651 pr=11 pw=0 time=894540 us)
  707098   PARTITION RANGE ALL PARTITION (cr=10651 pr=11 pw=0 time=708943 us)
  707098     PARTITION LIST SINGLE PARTITION (cr=10651 pr=11 pw=0 time= ...)
  707098       INDEX RANGE SCAN CMPGN_OBJ_CUST_N1 PARTITION (cr=10651 pr=11 pw=0 ...)
       7 SORT AGGREGATE (cr=196 pr=1 pw=0 time=55264 us)
    6790   PARTITION RANGE ALL PARTITION (cr=196 pr=1 pw=0 time=129616 us)
    6790     PARTITION LIST SINGLE PARTITION (cr=196 pr=1 pw=0 time= ...)
    6790       INDEX RANGE SCAN CMPGN_OBJ_CUST_N1 PARTITION: KEY KEY (cr=196 pr=1 ...)
       7 SORT ORDER BY (cr=10883 pr=12 pw=0 time=951229 us)
       7   NESTED LOOPS (cr=34 pr=0 pw=0 time=1372 us)
       7     NESTED LOOPS OUTER (cr=18 pr=0 pw=0 time=903 us)

........  이하는 전과 동일
```
612,879(=623,531-10,652)번이나 Random I/O를 발생시키던 cmpgn_obj_cust 테이블 액세스 단계가 사라졌다.
당연히 총 블록 I/O 개수도 630,344에서 10,887개로 줄었고, 소요시간도 31초에서 0.996초로 감소하였다.

### [사례 2]
사례를 하나 더 살펴보자. 아래는 모 통신 회사에서 관리하는 전체 서비스번호 중에서 신규 가입자를 위해 사용 가능한 번호를 조회하는 쿼리로서, 전국 대리점을 통해 가장 빈번하게 수행되는 쿼리 중 하나다.
```
select *
from  (select 식별번호 || 국번호 || 라인번호 as 서비스번호, 번호사용코드
            , 번호구분코드, 관리조직id, 사용자id
            , decode(선호번호여부, 'Y', '예', 'N', '아니오', '') 선호번호여부
       from   서비스번호
       where  식별번호 = :식별번호
       and    라인번호 between :라인시작번호 and :라인종료번호
       and    번호상태코드 = 'AV'
       and    번호사용코드 = :번호사용코드
       and    번호구분코드 = :번호구분코드
       and    선호번호여부 = :선호번호여부
       order by 서비스번호)
where rownum <= 500
```
바인드 변수에는 값이 아래와 같이 입력되는데, 식별번호(전화번호 앞 3자리)가 '010' 이면서 라인번호(가입자개별번호, 전화번호 끝 4자리)가 2000 ~ 5000 구간에 속하는, 사용 가능한 국번호
(전화번호 중간 3 ~ 4자리)를 조회하려는 것이다.
```
:식별번호          = '010'
:라인시작번호       = '2000'
:라인종료번호       = '5000'
:번호사용코드       = '01'
:번호구분코드       = 'D'
:선호번호여부       = 'N'
```
이 쿼리를 위해 사용된 인덱스 구성은 다음과 같다.
```
- 서비스번호_N4 : 식별번호 + 번호상태코드 + 번호사용코드 + 라인번호
              + 선호번호여부 + 번호구분코드
```
아래는 위와 같이 값을 입력하고서 실행했을 때의 트레이스 결과다.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.010        0.011       0         0          0      0
Execute       1     0.020        0.018       0         2          0      0
Fetch        21    18.780       96.381   11948    692666          0    500
-------- ------ --------- ------------ ------- --------- ---------- ------
Total        23    18.810       96.410   11948    692668          0    500

Rows     Row Source Operation
-------- ------------------------------------------------------
       0 STATEMENT
     500   COUNT STOPKEY (cr=692666 pr=11948 pw=0 time=96380650 us)
     500     VIEW (cr=692666 pr=11948 pw=0 time=96380140 us)
     500       SORT ORDER BY STOPKEY (cr=692666 pr=11948 pw=0 time=96379630 us)
  691237         FILTER (cr=692666 pr=11948 pw=0 time=65667547 us)
  691237           TABLE ACCESS BY INDEX ROWID 서비스번호 (cr=692666 pr=11948 pw=0 ...)
  691237             INDEX RANGE SCAN 서비스번호_N4 (cr=7223 pr=3731 pw=0 time= ...)
```
rownum 조건 때문에 최종 결과건수는 500건에 불과한데도 69만개 블록을 읽으면서 1분 36초가 소요되었다.

인덱스가 이미 6개 컬럼으로 구성돼 있지만 어쩔 수 없다. 워낙 자주 사용하는 쿼리이므로 서비스번호_N4 인덱스 뒤쪽에 아래처럼 3개 컬럼을 추가하기로 결정하였다.
```
- 서비스번호_N4 : 식별번호 + 번호상태코드 + 번호사용코드 + 라인번호
              + 선호번호여부 + 번호구분코드 + 국번호 + 관리조직ID + 사용자ID
```
아래는 인덱스를 다시 생성하고서 같은 SQL을 수행한 결과다.
```
Call      Count  CPU Time Elapsed Time    Disk     Query    Current   Rows
-------- ------ --------- ------------ ------- --------- ---------- ------
Parse         1     0.000        0.000       0         0          0      0
Execute       1     0.000        0.000       0         0          0      0
Fetch        21     1.700        1.728       0      7223          0    500
-------- ------ --------- ------------ ------- --------- ---------- ------
Total        23     1.700        1.728       0      7223          0    500

Rows     Row Source Operation
-------- ------------------------------------------------------
       0 STATEMENT
     500   COUNT STOPKEY (cr=7223 pr=0 pw=0 time=1727339 us)
     500     VIEW (cr=7223 pr=0 pw=0 time=1726826 us)
     500       SORT ORDER BY STOPKEY (cr=7223 pr=0 pw=0 time=1726816 us)
  691237         FILTER (cr=7223 pr=0 pw=0 time=1381986 us)
  691237           INDEX RANGE SCAN 서비스번호_N4 (cr=7223 pr=0 pw=0 time=691026 us)
```
인덱스 컬럼이 많아지면 그만큼 DML 속도가 느려지는 측면이 있지만 위에서처럼 사용빈도를 감안해서 결정한 것이라면 잃는 것보다 얻는 것이 많다.

참고로, 쿼리를 아래와 같이 구사한다면 인덱스 컬럼을 세 개가 아니라 '국번호' 하나만 추가하고도 튜닝이 가능하다.
국번호는 order by 조건에 포함되는 컬럼이므로 반드시 추가해야 하는 반면, 관리조직ID와 사용자ID는 참조정보이므로 최종 출력 건에 대해서만 읽으면 되기 때문이다.
```
select /*+ ordered use_nl(b) no_merge(b) rowid(b) */
       b.식별번호 || b.국번호 || b.라인번호 as 서비스번호
     , b.번호사용코드, b.번호구분코드, b.관리조직id, b.사용자id
     , decode(b.선호번호여부, 'Y', '예', 'N', '아니오', '') 선호번호여부
from  (select rowid rid
       from   서비스번호
       where  식별번호 = :식별번호
       and    라인번호 between :라인시작번호 and :라인종료번호
       and    번호상태코드 = 'AV'
       and    번호사용코드 = :번호사용코드
       and    번호구분코드 = :번호구분코드
       and    선호번호여부 = :선호번호여부
       order by 식별번호 || 국번호 || 라인번호) a, 서비스번호 b
where b.rowid = a.rid
and   rownum <= 500
```
대신 이 방식을 사용하면 최종 500건(rownum <= 500)에 대해 본 테이블과 rowid를 이용한 NL 조인이 발생하므로 인덱스를 3개 추가할 때보다 500개의 추가적인 Random I/O가 발생할 것이다.
지금 소개한 사례 2는 사용 빈도가 워낙 높은 쿼리여서 이 500개의 추가적인 Random I/O마저도 줄이려고 인덱스 컬럼을 3개 추가하는 방식으로 최종 결정하였다.

<br/>

## (5) 버퍼 Pinning 효과 활용
오라클의 경우, 한번 입력된 테이블 레코드는 절대 rowid가 바뀌지 않는다. 즉, 레코드 이동이 발생하지 않는다. 따라서 아래와 같이 미리 알고 있던 테이블 rowid 값을 이용해 레코드를 조회하는 것이 가능하다.
해당 레코드가 지워지지만 않는다면 말이다.
```
select * from emp where rowid = :rid
```
실행계획상에는 'Table Access By Index ROWID' 대신 아래와 같이 'Table Access By User ROWID'라고 표시된다.
```
Execution Plan
------------------------------------------------------------------------
0      SELECT STATMENET Optimizer=ALL_ROWS (Cost=1 Card=1 Bytes=37)
1   0    TABLE ACCESS (BY USER ROWID) OF 'EMP' (TABLE) (Cost=1 Card=1 ... )
```
미리 알고 있던 rowid 값이 아니더라도 아래처럼 인라인 뷰에서 읽은 rowid 값을 이용해 테이블을 액세스하는 것도 가능하다.
```
select /*+ ordered use_nl(b) rowid(b) */ b.*
from   (select /*+ index(emp emp_pk) no_merge */ rowid rid
        from   emp
        order by rowid) a, emp b
where   b.rowid = a.rid

Execution Plan
------------------------------------------------------------------------
0      SELECT STATMENET Optimizer=ALL_ROWS (Cost=16 Card=14 Bytes=686)
1   0    NESTED LOOPS (Cost=16 Card=14 Bytes=686)
2   1      VIEW (Cost=2 Card=14 Bytes=168)
3   2        SORT (ORDER BY) (Cost=2 Card=14 Bytes=168)
4   3          INDEX (FULL SCAN) OF 'EMP_PK' (INDEX (UNIQUE)) (Cost=1 ...)
5   1      TABLE ACCESS (BY USER ROWID) OF 'EMP' (TABLE) (Cost=1 Card=1 ...)
```
위 쿼리는 emp_pk 인덱스 전체를 스캔히 얻은 레코드를 rowid 순으로 정렬한 다음(→ 이 중간집합의 CF는 가장 완벽하게 좋은 상태가 됨) 한 건씩 순차적으로(→ NL 조인 방식) emp 테이블을 액세스하고
있는데, 여기서 재미있는 상상을 해 볼 수 있다. 만약 위처럼 User ROWID에 의한 테이블 액세스 시에도 버퍼 Pinning 효과가 나타난다면 어떨까?

Random 액세스 비효율은 한 건을 읽기 위해 블록을 통째로 읽기 때문에 발생하는 것인데, 위와 같은 쿼리에 버퍼 Pinning 효과까지 나타난다면 한 번 액세스로 블록 안에 있는 모든 레코드를 다 읽어 들이는
셈이 된다. CF가 가장 좋을 때 인덱스 손익분기점이 90% 이상에서 결정되는 것도 그 때문이었다.

따라서 그런 효과가 나타나기만 한다면 인덱스를 통해 아무리 많은 테이블 레코드를 액세스하더라도 Random 액세스에 의한 (I/O 측면에서의) 비효율은 거의 존재하지 않고, SQL 튜닝 시 가장 자주 사용되는
기법 중 하나가 될 것이다.

아래와 같이 1,000번 복제한 emp 테이블을 이용해 테스트해 보자.
```
create table emp
as
select * from scott.emp, (select rownum no from dual connect by level <= 1000)
order by dbms_random.value;

alter table emp add constraint emp_pk primary key(no, empno);
```
아래 결과에서 보듯 10g에선 버퍼 Pinning 효과가 나타나지 않는다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
   14000 NESTED LOOPS (cr=14036 pr=0 pw=0 time=453249 us)
   14000   VIEW (cr=36 pr=0 pw=0 time=131234 us)
   14000     SORT ORDER BY (cr=36 pr=0 pw=0 time=47229 us)
   14000       INDEX FULL SCAN EMP_PK (cr=36 pr=0 pw=0 time=42015 us)
   14000   TABLE ACCESS BY USER ROWID EMP (cr=14000 pr=0 pw=0 time=169445 us)
```
반갑게도 11g에선 우리가 기대했던 버퍼 Pinning 효과가 나타난다. 아래 결과를 확인하기 바란다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
   14000 NESTED LOOPS (cr=140 pr=0 pw=0 time=2316 us cost=14767 size=1816972 ... )
   14000   VIEW (cr=36 pr=0 pw=0 time=652 us cost=109 size=175836 card=14653)
   14000     SORT ORDER BY (cr=36 pr=0 pw=0 time=237 us cost=109 size=175836 ...)
   14000       INDEX FULL SCAN EMP_PK (cr=36 pr=0 pw=0 time=220 us cost=37 ...)
   14000   TABLE ACCESS BY USER ROWID EMP (cr=104 pr=0 pw=0 time=0 us cost=1 ...)
```
참고로, 위와 같은 방식의 테이블 액세스는 DB2에서 사용하는 테이터 블록 Prefetch 원리이기도 하다.

<br/>

## (6) 수동으로 클러스터링 팩터 높이기
테이블에는 데이터가 무작위로 입력되는 반면, 그것을 가리키는 인덱스는 정해진 키(key) 순으로 정렬되기 때문에 대개 CF가 좋지 않게 마련이다.
CF가 나쁜 인덱스를 이용해 많은 양의 데이터를 읽어야 할 때, SQL 튜닝이 가장 어렵다.

그럴 때, 해당 인덱스 기준으로 테이블을 재생성함으로써 CF를 인위적으로 좋게 만드는 방법을 생각해 볼 수 있고, 실제로 해 보면 그 효과가 매우 극적이다.

주의할 것은, 인덱스가 여러 개인 상황에서 특정 인덱스를 기준으로 테이블을 재정렬하면 다른 인덱스의 CF가 나빠질 수 있다는 점이다.
다행히 두 인덱스 키 컬럼 간에 상관관계가 높다면(예를 들어, 직급과 급여) 두 개 이상 인덱스의 CF가 동시에 좋아질 수 있지만 그런 경우가 아니라면 CF가 좋은 인덱스는 테이블당 하나뿐이다.

따라서 인위적으로 CF를 높일 목적으로 테이블을 Reorg할 때는 가장 자주 사용되는 인덱스를 기준으로 삼아야 하며, 혹시 다른 인덱스를 사용하는 중요한 쿼리 성능에 나쁜 영향을 주지 않는지 반드시
체크해 봐야 한다.

그리고 이 방법을 주기적으로 수행해야 한다면 데이터베이스 관리 비용이 증가하므로 테이블과 인덱스를 Rebuild하는 부담이 적고 효과가 확실할 때만 사용하는 것이 바람직하다.

필자가 이 방법을 사용해 별다른 부작용 없이 큰 효과를 보았던 사례를 하나 소개하려고 한다. 아래는 상품거래 테이블에 생성된 4개 인덱스의 컬럼 구성이다.
```
- 상품거래_PK : 상품번호 + 거래번호 + 시장구분코드 + 매매방법구분코드 + 거래일자
- 상품거래_X01 : 상품번호 + 거래시각 + 거래유형코드
- 상품거래_X02 : 매도고객번호 + 상품번호 + 거래시각
- 상품거래_X03 : 매수고객번호 + 상품번호 + 거래시각
```
그리고 각 인덱스의 CF를 조사해 보니 아래와 같았다.
```
인덱스명                  CL_FACTOR       TAB_BLKS          TAB_ROWS   RATIO
------------------ -------------- -------------- ----------------- -------
상품거래_PK             828,719,556     85,753,629     1,780,320,000     47%
상품거래_X01            866,183,444     85,753,629     1,780,320,000     49%
상품거래_X02          1,347,355,789     85,753,629     1,780,320,000     76%
상품거래_X03          1,028,188,086     85,753,629     1,780,320,000     58%
```
가장 빈번하게 사용되는 상품거래_PK 인덱스의 CF는 8억 2천만으로서, 총 테이블 건수 17억 8천만에 대한 Ratio(=cl_facor/tab_rows ✕ 100)가 47%이므로 평균적으로 두 개 레코드를 읽을 때마다
Random I/O가 한 번씩 일어나게 됨을 알 수 있다.

상품거래 테이블은 이 회사에서 가장 많이 조회가 일어나는 테이블이다. 조회뿐만 아니라 레코드 건수에서 알 수 있듯이 하루에만 수 백만 개 레코드가 입력되는 초대용량 테이블이다.
그래서 조회 성능과 관리의 편의성을 위해 거래일자 기준으로 일 단위 파티셔닝까지 돼 있는 상태다.(X01 ~ X03 인덱스에 거래일자 컬럼을 두지 않은 이유다.)

문제는, 대부분 쿼리들이 상품번호를 기준으로 조회한다는 데에 있다.
이 테이블에는 거래가 일어나는 순서대로 데이터가 입력되므로 상품번호를 선두 키로 갖는 PK와 X01 인덱스 기준으로는 데이터가 흩어지게 마련이다.
더구나 이 회사는 관리하는 상품 개수가 많지 않은 대신 상품번호별로 하루 평균 수만 개씩 레코드가 쌓이는 구조이기 때문에 테이블 조회 시 Random 액세스 부하가 심각할 수 밖에 없다.

개별 SQL 튜닝만으론 한계가 있고 구조적인 개선이 필요해 보인다. 뒤에서 설명할 IOT나 클러스터를 도입한다면 쿼리 성능은 분명히 개선되겠지만 insert 부하가 심각할 것이다. 어떻게 해야 할까?

이미 예상했겠지만 가장 많이 사용하는 PK 인덱스 컬럼 순으로 테이블을 정렬한다면 Random I/O 발생량을 상당히 줄일 수 있다.
게다가 인덱스 컬럼 구성 상 PK 인덱스와 X01 인덱스는 상관관계가 매우 높다. 거래번호와 거래시각 모두 순차적으로 증가하는 값이기 때문이다.
따라서 PK 인덱스 순으로 테이블을 정렬하면 X01 인덱스의 CF도 같이 좋아지는 일석이조의 효과가 생긴다.

또 한 가지, 테이블이 일 단위 파티셔닝 돼 있기 때문에 매일 야간에 마지막 파티션만 Reorg 할 수 있어 관리적 부담도 적다.
그리고 대부분 쿼리가 상품번호별로 과거 거래 데이터를 분석하는 용도이므로 실시간 정렬이 가능한 IOT가 요구되지도 않는다.

이런 여러 가지 상황을 종합적으로 고려해, 업무가 끝나고 야간 배치 프로그램이 수행되기 직전에 매일 테이블을 Reorg 하기로 결정했고, 그 결과 시스템 전반에 걸쳐 매우 긍정적인 효과가 나타났다.
아래는 테이블을 Reorg 하기 전 특정 쿼리의 트레이스 결과인데, 상품거래 테이블을 액세스하는 단계에서만 232,202(=241,218-9,016)번의 블록 I/O가 발생한 것을 볼 수 있다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
  940216 TABLE ACCESS BY LOCAL INDEX ROWID 상품거래 (cr=241218 pr=16
  943823   NESTED LOOPS (cr=9016 pr=7477 pw=0 time=1038208353 us)
     517     SORT UNIQUE (cr=2887 pr=2462 pw=0 time=7638243 us)
   27614       TABLE ACCESS BY INDEX ROWID 이상거래상세 (cr=2887 ... )
   27614         INDEX RANGE SCAN 이상거래상세_X01 (cr=1587 pr=1587 pw=0 ... )
  943305     PARTITION RANGE ITERATOR PARTITION: KEY KEY (cr=6129 pr=4915 ...
  943305       INDEX RANGE SCAN 상품거래_PK PARTITION: KEY KEY (cr=6129 ...)
```
아래는 테이블을 Reorg 하고 나서의 트레이스 결과인데, 블록 I/O가 50,229(=57,501-7,272)개로 줄었다. SQL 변경 없이도 블록 I/O가 기존의 1/5 수준으로 줄어든 것이다.
```
Rows     Row Source Operation
-------- ------------------------------------------------------
  940216 TABLE ACCESS BY LOCAL INDEX ROWID 상품거래 (cr=57501 pr=532
  943823   NESTED LOOPS (cr=7272 pr=3155 pw=0 time=24543204 us)
     517     SORT UNIQUE (cr=2887 pr=0 pw=0 time=101041 us)
   27614       TABLE ACCESS BY INDEX ROWID 이상거래상세 (cr=2887 pr=0 ... )
   27614         INDEX RANGE SCAN 이상거래상세_X01 (cr=1587 pr=0 pw=0 ... )
  943305     PARTITION RANGE ITERATOR PARTITION: KEY KEY (cr=4385 pr=3155 ...
  943305       INDEX RANGE SCAN 상품거래_PK PARTITION: KEY KEY (cr=4385 ...)
```
특정 SQL의 I/O만 줄어든 것이 아니다. 모니터링 결과, 이 테이블을 액세스하는 쿼리마다 I/O 발생량이 기존에 비해 평균 39% 수준(상품거래 이외 테이블에 대한 I/O까지 포함)으로 감소한 것을 관찰할 수 있었다.

### [차세대 시스템 구축 시 주의사항]
회사마다 차세대 시스템을 구축하는 목적이 다양하지만(예를 들어, 업무 프로세스 개선, 데이터 품질 개선 등) 그 중 빠지지 않는 것이 성능 개선이다.
차세대 시스템을 구축할 때는 그래서 기존보다 더 고급 사양의 하드웨어 시스템을 도입하고 SQL 튜닝도 실시해 보지만 오히려 성능이 예전만 못하는 경우가 의외로 많다.
심지어 모델 변경을 최소화했는데도 말이다. 왜 그럴까?

여기에 CF와 관련해서 주목해야 할 중요한 시사점이 한 가지 있는데, 원인을 분석해 보면 과거 시스템으로부터 데이터를 이관하는 과정에서 CF가 오히려 나빠진 데서 기인한다.
기존에 운영되던 시스템은 트랜잭션이 발생하는 순서대로 데이터가 입력되는 반면 데이터를 이관할 때는 병렬쿼리를 많이 활용하기 때문에 데이터를 무작위로 흩어놓는 경향이 있다.
이는 애플리케이션 전반에 걸쳐 테이블 Random I/O 횟수를 증가시키고, 결과적으로 디스크 I/O 발생량과 경합을 증가시키는 요인으로 작용한다.

따라서 데이터 이관 시에는 ASIS 대비 TOBE 시스템의 CF가 나빠지지 않았는지 조사하고, 그 결과에 따라 적절한 조치를 취해 주어야 한다.
모든 테이블을 대상으로 삼기는 현실적으로 어렵겠지만, 적어도 자주 사용하는 거래 데이터에 대해서는 반드시 필요한 조치라고 하겠다.