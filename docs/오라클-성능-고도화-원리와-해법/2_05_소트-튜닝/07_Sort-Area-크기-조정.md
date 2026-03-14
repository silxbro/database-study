# 07. Sort Area 크기 조정

세션 레벨에서 Sort Area 크기를 조정하거나, 시스템 레벨에서 각 세션에 할당될 수 있는 총 크기를 조정해야 할 때가 있다.
Sort Area 크기 조정을 통한 튜닝의 핵심은, 디스크 소트가 발생하지 않도록 하는 것을 1차 목표로 삼고 불가피할 때는 Onepass 소트로 처리되도록 하는 데에 있다.

오라클 9i부터 두 가지 PGA 메모리 관리 방식을 지원하므로 이에 대한 개념부터 살펴보자.
<br/>
<br/>
## (1) PGA 메모리 관리 방식의 선택
데이터 정렬, 해시 조인, 비트맵 머지, 비트맵 생성 등을 위해 사용하는 메모리 공간을 'Work Area'라고 부르며, sort_area_size, hash_area_size, bitmap_merge_area_size,
create_bitmap_area_size 같은 파라미터를 통해 조정한다.

8i까지는 이들 Work Area의 기본 값을 관리자가 지정하고, 프로그램의 작업 내용과 필요한 크기에 따라 세션 레벨에서 이들 값을 직접 조정해야만 했다.
하지만 9i부터는 '자동 PGA 메모리 관리(Automatic PGA Memory Management)' 기능이 도입되었기 때문에 사용자가 일일이 그 크기를 조정하지 않아도 된다.

DB 관리자는 pga_aggregate_target 파라미터를 통해 인스턴스 전체적으로 이용 가능한 PGA 메모리 총량을 지정하기만 하면 된다.
그러면 오라클이 시스템 부하 정도에 따라 자동으로 각 세션에 메모리를 할당해 준다. 그리고 이 파라미터의 설정 값은 인스턴스 기동 중에 자유롭게 늘리거나 줄일 수 있다.

자동 PGA 메모리 관리 기능을 활성화하려면 workarea_size_policy를 auto로 설정하면 되는데, 9i부터 기본적으로 auto로 설정돼 있다.
이 파라미터를 auto로 설정하면 *_area_size 파라미터는 모두 무시되며, 오라클이 내부적으로 계산한 값을 사용한다.

기본적으로 자동 PGA 메모리 관리 방식이 활성화되지만 시스템 또는 세션 레벨에서 '수동 PGA 메모리 관리' 방식으로 전환할 수 있다.

특히, 트랜잭션이 거의 없는 야간에 대량의 배치 Job을 수행할 때는 수동 방식으로 변경하고 직접 크기를 조정하는 것이 효과적일 수 있다.
왜냐하면, 자동 PGA 메모리 관리 방식 하에서는 프로세스당 사용할 수 있는 최대 크기가 제한되기 때문이다.
즉, Work Area를 사용 중인 다른 프로세스가 없더라도 특정 프로세스가 모든 공간을 다 쓸 수 없는 것이다. 결국 수 GB의 여유 메모리를 두고도 이를 충분히 활용하지 못해 작업 시간이 오래 걸릴 수 있다.

그럴 때 workarea_size_policy 파라미터를 세션 레벨에서 manual로 변경하고, 필요한 만큼(최대 2,147,483,647 바이트) Sort Area와 Hash Area 크기를 늘림으로써 성능을 향상시키고,
궁극적으로 전체 작업 시간을 크게 단축시킬 수 있다.
<br/>
<br/>
## (2) 자동 PGA 메모리 관리 방식 하에서 크기 결정 공식
auto 모드에서 단일 프로세스가 사용할 수 있는 최대 Work Area 크기는 인스턴스 기동 시 오라클에 의해 내부적으로 결정되며, _smm_max_size 파라미터(단위는 KB)를 통해 확인 가능하다.
```
SQL> connect / as sysdba
연결되었습니다.

SQL> select a.ksppinm name, b.ksppstvl value
  2  from   sys.x$ksppi a, sys.x$ksppcv b
  3  where  a.indx = b.indx
  4  and    a.ksppinm='_smm_max_size' ;

NAME                            VALUE
------------------------------- --------
_smm_max_size                   39116
```

이 파라미터의 값을 결정하는 내부 계산식은 버전에 따라 다르며, 오라클 9i부터 10gR1까지는 아래 계산식을 따른다.

> _smm_max_size = least((pga_aggregate_target✕0.05), (_pga_max_size✕0.5))


즉, DB 관리자가 지정한 pga_aggregate_target의 5%와 _pga_max_size 파라미터(단위는 Byte)의 50% 중 작은 값으로 설정된다.

10gR2부터는 조금 더 복잡하다.

- pga_aggregate_target <= 500MB 일 때

    - _smm_max_size = pga_aggregate_target ✕ 0.2

- 500MB < pga_aggregate_target <= 1000MB 일 때

    - _smm_max_size = 100MB

- 500MB < pga_aggregate_target > 1000MB 일 때

    - _smm_max_size = pga_aggregate_target ✕ 0.1


_pga_max_size는 거꾸로 _smm_max_size에 의해 결정된다.

- _pga_max_size = _smm_max_size ✕ 2

참고로, auto 모드에서 병렬 쿼리의 각 슬레이브(Slave) 프로세스가 사용할 수 있는 Work Area 총량은 _smm_px_max_size 파라미터(KB)에 의해 제한된다.

SGA는 sga_max_size 파라미터로 설정된 크기만큼 공간을 미리 할당한다.
이와 대조적으로, PGA는 자동 PGA 메모리 관리 기능을 사용한다고 해서 pga_aggregate_target 크기만큼의 메모리를 미리 할당해 두지는 않는다.
이 파라미터는 workarea_size_policy를 auto로 설정한 모든 프로세스들이 할당받을 수 있는 Work Area의 총량을 제한하는 용도로 사용된다.
<br/>
<br/>
## (3) 수동 PGA 메모리 관리 방식으로 변경 시 주의사항
manual 모드로 설정한 프로세스는 pga_aggregate_target 파라미터의 제약을 받지 않는다.
따라서 manual 모드로 설정한 많은 세션에서 Sort Area와 Hash Area를 아주 큰 값으로 설정하고 실제 매우 큰 작업을 동시에 수행한다면 가용한 물리적 메모리(Real Memory)가 고갈돼
페이징(Paging)이 발생하면서 시스템 전체 성능을 크게 떨어뜨릴 수 있다.

페이징이 심하면 시스템을 마비시킬 수도 있으므로 이들 파라미터를 수동으로 변경할 때는 그런 상황이 발생하지 않도록 세심한 주의를 기울여야 한다.
참고로, *_area_size를 위해 설정 가능한 값은 0부터 2,147,483,647(2GB에서 1Byte를 뺀 값)까지다.

특히, workarea_size_policy 파라미터를 manual로 설정한 상태에서 병렬 쿼리를 사용하면 각 병렬 슬레이브(Slave) 별로 sort_area_size 크기만큼의 Sort Area를 사용할 수 있다는 사실을
반드시 기억해야 한다. 만약 아래와 같은 쿼리를 수행할 때 어떤 일이 발생할지 상상해보라.
```
alter session set workarea_size_pollicy = manual;

alter session set sort_area_size = 2147483647;

select /*+ full(t) parallel(t 64) */ *
from   the_biggest_table t
order by object_name;
```

sort order by나 해시 조인 등을 수행할 때는 사용자가 지정한 DOP(the degree of parallelism)의 2배수만큼의 병렬 Slave가 떠서 작업을 수행하므로 위 쿼리가 수행되는 동안 128개 프로세스가
각각 최대 2GB의 Sort Area를 사용할 수 있다. 따라서 manual 모드에서 병렬 Degree를 크게 설정할 때는 sort_area_size와 hash_area_size를 반드시 확인해야 한다.
(물론 sort order by를 수행할 때 한쪽 서버 집합은 데이터 블록을 읽어 반대편 서버 집합에 분배하는 역할만 하므로 위 쿼리만으론 최대 64✕2GB의 Sort Area가 필요하다.)
<br/>
<br/>
## (4) PGA_AGGREGATE_TARGET의 적정 크기
pga_aggregate_target 파라미터 설정을 위한 적정 크기는 얼마일까? 오라클이 권고하는 값은 아래와 같다.

- OLTP 시스템 : (Total Physical Memory ✕ 80%) ✕ 20%
- DSS 시스템 : (Total Physical Memory ✕ 80%) ✕ 50%

예를 들어, 10GB의 물리적 메모리를 갖는다면 OS를 위해 2GB를 남겨두고 나머지 80%에 해당하는 8GB를 오라클에 할당한다.
OLTP 시스템이라면 그 중 6.4GB를, DSS 시스템이라면 4GB를 SGA에 할당한다. SGA에 할당하고 각각 남은 1.6GB와 4GB를 PGA 작업 공간으로 남겨 두라는 뜻이다.
(다시 말하지만, pga_aggregate_target 크기의 메모리를 미리 할당하지는 않는다.)

이것은 일반적인 권고사항일 뿐이며 애플리케이션 특성에 따라 모니터링 결과를 바탕으로 세밀한 조정이 필요하다.
대부분(90% 이상, 순수 OLTP 시스템에서는 100%) Optimal 소트 방식으로 수행되고 나머지 일부(10% 미만)만 Onepass 소트 방식으로 수행되는 것을 목표로 삼아야 한다.
시스템 Multipass 소트가 종종 발생하는 것으로 측정되면 크기를 늘리거나 튜닝이 필요한 것으로 받아들여야 하며, 이는 대용량 소트와 해시 조인이 많이 발생하는 DSS 시스템에서도 마찬가지다.
<br/>
<br/>
## (5) Sort Area 할당 및 해제
Sort Area가 할당되는 시점과 해제되는 시점에 대해 살펴보려고 한다.

예전에는 소트 오퍼레이션이 시작되는 시점에 sort_area_size 크기만큼의 메모리를 미리 할당했지만, 오라클 8.0부터는 db_block_size 크기에 해당하는 청크(Chunk) 단위로 필요한 만큼 조금씩 할당한다.
즉, sort_area_size는 할당할 수 있는 최대 크기를 지정하는 파라미터로 바뀐 것이다.

그리고 오라클 8i 이전에는 프로세스를 위해 할당된 PGA 공간을 프로세스가 해제될 때까지 OS에 반환하지 않았다.
9i에서 자동 PGA 메모리 관리 방식이 도입되면서부터는 프로세스가 더 이상 사용하지 않는 공간을 즉각 반환함으로써 다른 프로세스가 사용할 수 있도록 한다.
(버그로 인해 PGA 메모리가 반환되지 않는 경우는 지금도 종종 발견된다.)

실제 Sort Area가 할당되고 해제되는 과정을 측정해보자. 아래는 세션별로 현재 사용 중인 PGA와 UGA 크기, 가장 많이 사용했을 때의 크기를 측정하는 SQL문이다.

```
select
  round(MIN(decode(n.name, 'session pga memory', s.value))/1024)       "PGA(KB)"
, round(MIN(decode(n.name, 'session pga memory max', s.value))/1024)   "PGA_MAX(KB)"
, round(MIN(decode(n.name, 'session uga memory', s.value))/1024)       "UGA(KB)"
, round(MIN(decode(n.name, 'session uga memory max', s.value))/1024)   "UGA_MAX(KB)"
from   v$statname n, v$sesstat s
where  (name like '%uga%' or name like '%pga%')
and    s.sid = &SID
```

자동 PGA 메모리 관리 방식으로 시스템 레벨에서 사용할 수 있는 총량을 아래와 같이 24MB로 제한하고 테스트를 진행해 보자.
```
alter system set pga_aggregate_target = 24M;

create table t_emp
as
select *
from   emp, (select rownum no from dual connect by level <= 100000);
```

emp 테이블을 10만 번 복제한 t_emp 테이블을 생성했다. 이제 테스트 준비가 다 되었으므로 order by 절을 포함하는 아래 쿼리문이 수행될 때 PGA, UGA 크기가 어떻게 달라지는지 측정해 보자.
```
select * from t_emp order by empno;
```

위 쿼리문이 수행되는 동안 다른 세션에서 아래 4단계로 나눠 측정하면 된다.
참고로, 쿼리 완료 시점과 커서를 닫는 시점을 구분해 측정하려면, 사용자의 명시적인 요청이 있을 때만 Fetch를 진행하는 TOAD, Orange, PL/SQL Developer 같은 쿼리 툴을 사용하는 것이 편하다.

- 최초 : 쿼리 수행 직전
- 수행 도중 : 쿼리가 수행 중이지만 아직 결과가 출력되지 않은 상태(→ 값이 계속 변함)
- 완료 후 : 결과를 출력하기 시작했지만 데이터를 모두 Fetch 하지 않은 상태
- 커서를 닫은 후 : 정렬된 결과집합을 끝까지 Fetch하거나 다른 쿼리를 수행함으로써 기존 커서를 닫은 직후

측정 결과를 표로서 요약하면 아래와 같다.

|단계|PRA(KB)|PGA_MAX(KB)|UGA(KB)|UGA_MAX(KB)|
|:---|---:|---:|---:|---:|
|최초|376|632|153|401|
|수행 도중|5,560|6,584|4,308|5,331|
|완료 후|3,000|6,584|2,774|5,331|
|커서를 닫은 후|376|6,584|153|5,331|

PGA와 UGA를 각각 6,584KB, 5,331KB만큼 사용했지만 커서를 닫은 직후에는 모두 반환한다는 사실을 위 표를 통해 알 수 있다.

'수행 도중'과 '완료 후'에 UGA, PGA 크기가 Max 값을 밑도는 이유는, 소트해야 할 총량이 할당받을 수 있는 Sort Area 최대치를 초과하기 때문이다.
그때마다 중간 결과집합(Sort Run)을 디스크에 저장하고 메모리를 반환했다가 필요한 만큼 다시 할당받는다는 사실을 알 수 있다.

바로 이어서, 이번에는 workarea_size_policy를 manual로 변경하고 Sort Area는 50MB 크기로 설정한 상태에서 테스트를 진행해 보자.
```
alter session set workarea_size_size_policy = MANUAL;
alter session set sort_area_size = 52428800;
alter session set sort_area_retained_size = 52428800;
```

아래 표는 같은 쿼리를 수행하면서 각 단계별로 PGA, UGA 크기 변화를 측정한 것이다.

|단계|PRA(KB)|PGA_MAX(KB)|UGA(KB)|UGA_MAX(KB)|
|:---|---:|---:|---:|---:|
|최초|376|6,584|153|5,331|
|수행 도중|48,760|52,792|43,049|47,077|
|완료 후|4,792|52,792|4,315|47,077|
|커서를 닫은 후|440|52,792|153|47,077|

pga_aggregate_target를 24MB로 설정한 상태에서 한 세션의 Sort Area 크기가 50MB까지 도달했다. manual 모드로 설정한 프로세스는 이 파라미터의 제약을 받지 않음을 알 수 있다.

위 테스트는 10g 버전에서 수행한 것이며, 9i에서도 수치만 다를 뿐 같은 방식으로 동작한다.