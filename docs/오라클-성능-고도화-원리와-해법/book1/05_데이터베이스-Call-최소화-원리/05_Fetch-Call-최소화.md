# 05. Fetch Call 최소화

지금부터 설명할 Fetch Call 최소화 원리 내용을 요약하면 다음과 같다.

- 부분범위처리 원리
- OLTP 환경에서 부분범위처리에 의한 성능개선 원리
- ArraySize 조정에 의한 Fetch Call 감소 및 블록 I/O 감소 효과
- 프로그램 언어에서 Array 단위 Fetch 기능 활용
  <br/><br/>

## (1) 부분범위처리 원리
클릭했을 때 아래 JAVA Method를 호출하는 실행 버튼이 있다고 하자. SQL문에 사용된 BIG_TABLE이 1억 건에 이르는 대용량 테이블인데도 실제 테스트해 보면 버튼을 클릭하자마자 곧바로 결과를 리턴한다.
1억 건의 테이블을 그렇게 빨리 읽을 수 있는 원리는 무엇일까?

```
private void execute(Connection con) throws Exception {
    Statement stmt = con.createStatement();
    ResultSet rs = stmt.executeQury("select name from big_table");

    if( rs.next() ) {
      System.out.println(rs.getString(1));
    }

    rs.close();
    stmt.close();
}
```

한 가지 테스트를 더 해 보자.

```
SQL> create table t (
  2    x NUMBER   not null
  3    y NUMBER   not null ) ;

테이블이 생성되었습니다.

SQL> insert into t
  2  select *
  3  from (
  4    select rownum x, rownum y
  5    from   dual
  6    connect by level <= 5000000
  7  )
  8  order by dbms_random.value  --> 테이블과 인덱스 정렬 순서를 다르게 함
  9  ;

5000000 개의 행이 만들어졌습니다.

SQL> alter table t add
  2  constraint t_pk primary key (x);

테이블이 변경되었습니다.

SQL> alter system flush buffer_cache;

시스템이 변경되었습니다.

SQL> set arraysize 5
SQL> set timing on
SQL> select /*+ index(t t_pk) */ x, y
  2  from   t
  3  where  x > 0
  4  and    y <= 6 ;

           X            Y
------------ ------------
           1            1
           2            2
           3            3
           4            4
           5            5  --> 엔터를 치자마자 여기까지 출력하고 멈춤
           6            6  --> 33초 경과한 후에 이 라인을 출력하고 수행 종료

6 개의 행이 선택되었습니다.

경   과: 00:00:33.25
```

x,y 두 컬럼에는 같은 값을 입력했고, x 컬럼으로만 구성된 인덱스가 생성돼 있다.

x > 0 조건으로 인덱스를 스캔하도록 했는데, 모든 레코드가 0보다 크기 때문에 5백만 건을 스캔하면서 테이블을 5백만 번 액세스하게 된다.
테이블 필터 조건으로는 y <= 6을 사용했고, 전체 레코드 중 여기에 해당하는 것은 6개뿐이므로 최종 결과 집합은 6건이다.
이런 상황에서 쿼리를 수행해보면, 엔터를 입력하자마자 5건을 출력하고 잠시 멈춘 듯하다가 33초가 경과한 후에 마지막 6번째 레코드를 출력하고 수행을 종료한다. 왜 이런 현상이 발생하는 걸까?

ArraySize를 5로 설정한 데서 해답을 찾을 수 있는데, 아래 예시를 보면서 이해해 보자.

공사장에서 미장공이 시멘트를 이용해 벽돌을 쌓는 동안 운반공이 벽돌을 실어 나르고 있다. 아마 집을 짓는 듯하다.
쌓여 있는 벽돌을 한 번에 모두 실어 나를 수 없기 때문에 수레를 이용해 일정량씩 나누어 운반하는 모습을 볼 수 있다.
운반공은 미장공이 벽돌을 더 가져오라는 요청(→ Fetch Call)이 있을 때만 벽돌을 실어 나른다. 추가 요청이 없으면 운반작업은 거기서 멈춘다.

DBMS도 이처럼 데이터를 클라이언트에게 전송할 때 일정량씩 나누어 전송하며, 오라클의 경우 ArraySize(또는 FetchSize) 설정을 통해 운반단위를 조절한다.
그리고 전체 결과집합 중 아직 전송하지 않은 분량이 많이 남아있어도 클라이언트로부터 추가 Fetch Call을 받기 전까지는 그대로 멈춰 서서 기다린다.
여기에 OLTP 환경에서 대용량 데이터를 빠르게 핸들링할 수 있는 아주 중요한 원리가 숨어있다.

앞에서 1억 건짜리 BIG_TABLE을 쿼리하는 데 1초가 채 걸리지 않는다고 했는데, 그것도 Array 단위 Fetch 때문에 나타나는 효과다.
물론 1억 건 전체를 읽고 처리한 것은 아니며, rs.next()를 한 번만 호출하고 곧바로 ResultSet과 Statement를 닫아 버렸다.
rs.next()를 호출하는 순간 오라클은 FetchSize(자바에서 기본값은 10)만큼을 전송했고, 클라이언트는 그 중 한 건만 콘솔에 출력하고는 곧바로 커서를 닫은 것이다.
추가 요청 없이 그대로 커서를 닫아버리면 오라클은 데이터를 더는 전송하지 않고 바로 일을 마친다.
이처럼 쿼리 결과집합을 전송할 때, 전체 데이터를 쉼 없이 연속적으로 처리하지 않고 사용자로부터 Fetch Call이 있을 때마다 일정량씩 나누어서 전송하는 것을 이른바 '부분범위처리'라고 한다.
(어떤 오라클 튜닝 서적을 보면, 쿼리 수행 시 그 결과 집합을 버퍼 캐시에 모두 적재하고 나서 사용자에게 전송한다고 설명하고 있다.
그 책을 보고 지금까지 그렇게 이해하고 있다면 여기서 개념을 다시 정립하기 바란다.)
<br/><br/><br/>
그러면 적정한 데이터 운반단위는 얼마일까? 항상 결론은 '적당한 게 좋다'이다. 공사장을 다시 예로 들어 보자.
힘을 적게 들이면서 한 번에 실어 나를 수 있는 적당량을 운반단위로 정하고 그만큼씩만 실어 나르는 것이 가장 효과적인데, 적당량이라고 하는 게 공사장 성격에 따라 다르다.
작은 집 담장을 쌓고 있다면 작은 수레를 이용해 소량씩 나르다가 멈춰야 할 시점에 멈추는 것이 유리하고, 아파트를 짓고 있다면 가능한 한 큰 수레를 이용해 한 번에 많은 양을 실어 나르는 것이 유리하다.
만약 담장을 쌓는 정도의 공사인데 10,000개씩 실어 나르면 그 중 1,000개만 쓰고 나머지 9,000개는 버리는 일이 생길지도 모른다. 그럼, 한 개씩 실어 나르는 것이 효과적일까?
아마 어깨, 팔은 멀쩡한데 다리만 아프고 작업 속도는 훨씬 늦어질 것이다.

지금까지 설명한 부분범위처리 원리를 이해했다면, 네트워크를 통해 전송해야 할 데이터양에 따라 ArraySize를 조절할 필요가 있음을 직감했을 것이다.
예를 들어, 대량 데이터를 파일로 내려받는다면 어차피 전체 데이터를 전송해야 하므로 가급적 그 값을 크게 설정해야 한다.
ArraySize를 조정한다고 해서 전송해야 할 총량이 변하지는 않지만, Fetch Call 횟수를 그만큼 줄일 수 있다.
반대로 앞쪽 일부 데이터만 Fetch하다가 멈추는 프로그램이라면 ArraySize를 작게 설정하는 것이 유리하다. 불필요하게 많은 데이터를 전송하고 버리는 비효율을 줄일 수 있기 때문이다.

ArraySize를 5로 설정하면, 서버 측에서는 Oracle Net으로 데이터를 내려 보내다가 5건당 한 번씩 전송 명령을 날리고는 클라이언트로부터 다시 Fetch Call이 올 때까지 대기한다.
클라이언트 측에는 서버로부터 전송받은 5개 레코드를 담을 Array 버퍼가 할당되며, 그곳에 서버로부터 받은 데이터를 담았다가 한 건씩 꺼내 화면에 출력하거나 다른 작업들을 수행한다.

> #### [SDU, TDU]
> 흔히 Array Fetch를 얘기할 때, Array 버퍼가 서버 측(Server Side)에 할당된다고 생각한다. 서버 측 Array 버퍼에 데이터가 차면 전송한다는 설명이다.
> 하지만 Array 버퍼는 클라이언트 측(Client Side)에 위치하며, 서버 측에서는 SDU에 버퍼링이 이루어진다. Array Fetch를 수행하는 내부 메커니즘에 대해 자세히 알아보자.
>
> 오라클에서 데이터를 전송하는 단위는 ArraySize에 의해 결정된다. 하지만 내부적으로 데이터는 네트워크 패킷 단위로 단편화되어 여러 번에 나누어 전송된다.
> 부분범위처리 내에 또 다른 부분범위처리가 작동하는 것이다. 이것은 네트워크 프로그램을 해 보았다면 너무 상식적인 얘기다.
> 예를 들어, ArraySize가 100이고 한 레코드당 1MB를 차지한다면 한번 Fetch 할 때마다 100MB를 전송해야 하는데, 이를 하나의 패킷으로 묶어 한 번에 전송하지는 않는다.
> 네트워크를 통해 큰 데이터를 전송할 때는 작은 패킷들로 단편화해야 하며, 그래야 유실이나 에러가 발생했을 때 부분 재전송을 통해 복구할 수 있다.
>
> OSI 7 레이어는 Application, Presentation, Session, Transport, Network, Data Link, Physical 이렇게 7개 레이어로 이루어진다.
> 오라클 서버와 클라이언트는 Application 레이어에 위치하며, 그 아래에 있는 레이어를 통해 서로 데이터를 주고받는다.
>
> **SDU(Session Data Unit)** 는 Session 레이어 데이터 버퍼에 대한 규격으로서, 네트워크를 통해 전송하기 전에 Oracle Net이 데이터를 담아 두려고 사용하는 버퍼다.
> 예컨대, ArraySize를 5로 설정하면 클라이언트 측에는 서버로부터 전송받은 5개 레코드를 담을 Array 버퍼를 할당한다.
> 서버 측에서는 Oracle Net으로 데이터를 내려보내다가 5건당 한 번씩 전송 명령을 날리고는 매번 클라이언트로부터 다음 Fetch Call을 기다리는데, Oracle Net이 서버 프로세스로부터 전송명령을
> 받을 때까지 데이터를 버퍼링하는 곳이 SDU다.
> Oracle Net은 서버 프로세스로부터 전송요청을 받기 전에라도 SDU가 다 차면 버퍼에 쌓인 데이터를 전송하는데, 이 때는 클라이언트로부터 Fetch Call을 기다리지 않고 곧이어 데이터를 받아 SDU를
> 계속 채워 나간다.
>
> **TDU(Transport Data Unit)** 는 Transport 레이어 데이터 버퍼에 대한 규격이다.
> 물리적인 하부 레이어로 내려보내기 전에 데이터를 잘게 쪼개어 클라이언트에게 전송되는 도중에 유실이나 에러가 없도록 제어하는 역할을 한다.
>
> SDU와 TDU 사이즈는 TNSNAMES.ORA, LISTENER.ORA 파일에서 아래와 같이 설정 가능하며, 이들의 기본 설정 값은 2KB다.
>
> ```
> (SDU=2048)(TDU=2048)
> ```
>
> 예를 들어, 결과집합이 18건이고 각 로우당 900바이트를 차지한다고 가정하자.
> 그러면 총 16,200바이트를 전송해야 하는데, 만약 ArraySize를 5로 설정하면 한번 Fetch 할 때마다 4,500(=900✕5)바이트씩 3번을 전송하고 4번째는 2,700(=900✕3) 바이트를 전송하게 된다.
>
> ```
> fetch1 : 900 ✕ 5 = 4,500
>
> fetch2 : 900 ✕ 5 = 4,500
>
> fetch3 : 900 ✕ 5 = 4,500
>
> fetch4 : 900 ✕ 3 = 2,700
> ```
>
> 이때, SDU를 2,048로 설정하면 총 11개 패킷으로 단편화되어서 Transport 레이어로 전달된다.
>
> ```
> fetch1 : 2,048 + 2,048 + 404 = 4,500 (→ 3개 패킷)
>
> fetch2 : 2,048 + 2,048 + 404 = 4,500 (→ 3개 패킷)
>
> fetch3 : 2,048 + 2,048 + 404 = 4,500 (→ 3개 패킷)
>
> fetch4 : 2,048 + 652 = 2,700         (→ 2개 패킷)
> ```
>
> TDU는 1,024로 설정했다고 가정하자. 그러면 총 18개 패킷으로 단편화되어 클라이언트에게 전송이 이루어진다.
>
> ```
> fetch1 : (1,024 + 1,024) + (1,024 + 1,024) + 404 = 4,500 (→ 5개 패킷)
>
> fetch2 : (1,024 + 1,024) + (1,024 + 1,024) + 404 = 4,500 (→ 5개 패킷)
>
> fetch3 : (1,024 + 1,024) + (1,024 + 1,024) + 404 = 4,500 (→ 5개 패킷)
>
> fetch4 : (1,024 + 1,024) + 652 = 2,700                   (→ 3개 패킷)
> ```
>
> 참고로, 각 패킷은 헤더 정보를 포함하므로 패킷 단편화를 줄이면 네트웍 트래픽도 줄어들게 된다.
<br/>

## (2) OLTP 환경에서 부분범위처리에 의한 성능개선 원리
이제 부분범위처리 원리에 대해 이해했으므로 직전에 수행한 테스트에서 왜 5건만 출력하고 한동안 멈춰 서 있었는지 설명할 수 있다.

'x > 0 and y <= 6' 조건으로 쿼리를 수행하면, 첫 번째 Fetch Call에서는 인덱스를 따라 x 컬럼 값이 1\~5인 5개 레코드를 전송받아 Array 버퍼에 담는다.
x와 y 컬럼 값을 같게 입력해 놓았으므로 이들 5개 레코드는 테이블 필터 조건인 y <= 6 조건도 만족한다.
오라클 서버는 이 5개 레코드를 아주 빠르게 찾았으므로 지체 없이 전송 명령을 통해 클라이언트에게 전송하고, 클라이언트는 Array 버퍼에 담긴 5개 레코드를 곧바로 화면에 출력한다.

문제는 두 번째 Fetch Call에서 발생한다. 사용자로부터 두 번째 Fetch Call 명령을 받자마자 x = y = 6인 레코드를 찾아 Oracle Net으로 내려보낸다.
이제 'x > 0 and y <= 6' 조건을 만족하는 레코드가 더 없다는 사실을 오라클은 모르기 때문에 계속 인덱스를 스캔하면서 테이블을 액세스해 본다.
끝까지 가 본 후에야 더는 전송할 데이터가 없음을 인식하고 그대로 한 건만 전송하도록 Oracle Net에 명령을 보낸다. Oracle Net은 한 건만 담은 패킷을 클라이언트에게 전송한다.
클라이언트는 Array 버퍼가 다시 채워지기를 기다리면서 30여 초 이상을 허비했지만 결국 한 건밖에 없다는 신호를 받고 이를 출력한 후에 커서를 닫는다.

조건절을 아래처럼 바꿔서 다시 수행해 보면 화면에 아무것도 출력하지 않은 채 30초 이상을 대기하다가 한 건밖에 없다는 메시지를 그제야 출력하고 수행을 마친다.
지금까지 설명한 부분범위처리 원리를 이해했다면 아래 현상에 대해선느 굳이 설명이 필요 없으리라고 믿는다.

```
SQL> alter system flush buffer_cache;

시스템이 변경되었습니다.

경   과: 00:00:06.65
SQL> select /*+ index(t t_pk) */ x, y
  2  from   t
  3  where  x > 0
  4  and    y <= 1 ;

           X            Y
------------ ------------
           1            1  --> 37초 후에 이 라인을 출력하고 곧바로 수행 종료

1 개의 행이 선택되었습니다.

경   과: 00:00:37.59
```

앞에서 6건을 Fetch 할 때는 엔터를 치자마자 결과가 뿌려지기 시작했지만 방금 쿼리에서는 1건 Fetch 하려고 30초 이상 경과한 후에 처음 결과가 뿌려졌다.
이런 부분범위처리 원리 때문에 OLTP 환경에서는 결과집합이 많을수록 오히려 성능이 더 좋아진다.

이 말의 의미를 좀 더 이해하기 쉽게 설명해 주는 테스트를 한 가지 더 수행해 보자.
앞서 만들어 놓은 데이터를 이용해 SQL*Plus에서 아래 [1]번 쿼리를 수행해 보면 처음부터 끝까지 쉬지 않고 결과를 출력하는 반면, [2]번 쿼리는 화면에 레코드를 뿌리는 도중에 잠깐씩 멈칫하는 현상을
보인다.

```
[1] select /*+ index(t t_pk) */ * from t where x > 0 and mod(y, 50) = 0
[2] select /*+ index(t t_pk) */ * from t where x > 0 and mod(y, 50000) = 0
```

ArraySize가 10일 때, [1]번 쿼리는 500건을 스캔할 때마다 한 번씩 전송 명령을 날리지만 [2]번 쿼리는 500,000건당 한 번씩이므로 그만큼 클라이언트 측 Array 버퍼를 채우는 데 시간이 많이
소요되어 출력 도중 끊기는 현상이 생기는 것이다.

여기에 한 가지를 더해, OLTP성 업무에서는 쿼리 결과 집합이 아주 많더라도 그 중 일부만 Fetch하고 멈출 때가 자주 있다.
따라서 출력 대상 건이 많을수록 Array를 빨리 채울 수 있어 쿼리 응답 속도도 그만큼 빨라진다.
인덱스와 부분범위처리 원리를 잘 활용하면 OLTP 환경에서 극적인 성능개선 효과를 얻을 수 있는 원리가 여기에 숨어 있다.

어부가 배를 타고 바다에 나가 낚시를 하는 것에 비유하면, 물 반 고기 반인 황금어장에서는 배를 금방 채워 몇 시간 되지 않아 집으로 돌아갈 수 있지만 고기가 얼마 없는 어장에서는 온종일 낚시를 하고도
배를 다 채우지 못한 채 귀가하는 것과 같은 이치다.

데이터가 많을수록 더 빨라진다는 의미는 부분범위처리가 가능한 업무에 한하며, 서버 가공 프로그램이나 DW성 쿼리처럼 결과 집합 전체를 Fetch 해야 한다면 결과 집합이 많을수록 더 빨라지는 일은 있을 수
없다.

> #### [One-Row Fetch]
> 사실 위에서 정확히 설명하지 않고 넘어간 부분이 있다. 지금까지 설명한 원리만 잘 전달됐다면 굳이 밝힐 필요가 없지만 일일이 테스트 해 볼 때 시행착오를 겪지 않도록 하려고 설명하는 것이다.
> SQL*Plus를 잘 사용하지 않는다면, 그리고 지금껏 설명한 내용을 직접 테스트해 볼 요량이 아니라면 넘어가도 무방하다.
>
> SQL*Plus에서 ArraySize를 5로 설정하고 'x > 0 and y <= 6' 조건으로 쿼리를 수행했을 때, 첫 번째 Fetch Call에서 5개를 빠르게 전송 받기 때문에 대기 없이 곧바로 출력이 이루어진다고
> 설명했다. 그렇다면 'x > 0 and y <= 5' 조건을 사용했을 때도 Array 버퍼가 첫 번째 Fetch Call에서 금방 채워지므로 쿼리를 시작하자마자 5건이 바로 출력되어야 한다.
> 그런데 SQL\*Plus에서 테스트 해 보면 'x > 0 and y <= 1' 조건을 사용했을 때처럼 한참 후에야 5건을 출력하고 수행을 마친다.
>
> 이것은 오라클 서버와는 무관하게 SQL*Plus에서 나타나는 특징인데, SQL\*Plus에서 쿼리를 수행하면 첫 번째 Fetch에서는 항상 한 건만 요청(이하 One-Row Fetch)하고, 두 번째부터 ArraySize
> 만큼을 요청하기 때문에 생기는 현상이다.
> 'x > 0 and y <= 5' 조건일 때, 결과건수는 총 5건이므로 첫 번째 Fetch에서 1건을 우선 가져왔지만 두 번째 Fetch에서 4건 밖에 찾을 수 없어 끝까지 대기하느라고 오래 걸리는 것이다.
> 첫 번째 Fetch에서 1건을 빠르게 리턴 받지만 곧바로 출력하지 않는 이유는, Fetch 해 오는 방식과는 무관하게 Array 버퍼로는 5개를 할당하기 때문이며 클라이언트는 이 버퍼가 다 채워져야 출력을
> 시작한다.
>
> 같은 이치로, 'x > 0 and y <= 1' 조건으로 수행할 때는 첫 번째 Fetch Call에서 대기할 것 같지만 실상은 두 번째 Fetch Call에서 대기한다.
> 첫 번째 Fetch에서 이미 1건을 가져왔지만 클라이언트 쪽 Array 버퍼 크기가 5이므로 나머지 4개를 마저 채우려고 두 번째 Fetch Call을 날리고 기다리는 것이다.
> 아래 SQL 트레이스에서 1건만 가져오는데도 불구하고 2번의 Fetch Call이 발생한 것을 확인할 수 있다.
> SQL*Plus가 아닌 다른 쿼리 툴에서는 One-Row Fetch가 없으므로 Fetch Count가 1로 나타난다(참고로, Orange 같은 쿼리 툴은 SQL\*Plus처럼 작동함).
>
> ```
> call     count     cpu    elapsed     disk      query    current   rows
> ------- ------ ------- ---------- -------- ---------- ---------- ------
> Parse        1    0.00       0.02        2          2          0      0
> Execute      1    0.00       0.00        0          0          0      0
> Fetch        2   13.23      37.24    22048    5010564          0      1
> ------- ------ ------- ---------- -------- ---------- ---------- ------
> total        4   13.23      37.27    22050    5010566          0      1
> ```
>
> 아래는 TKProf를 돌리기 전 원시 trc 파일에서 발췌한 것인데, 이를 통해 더 분명하게 One-Row Fetch의 정체를 확인할 수 있다.
>
> ```
> PARSE #11:c=31250,e=734963,p=750,cr=83,cu=0,mis=1,r=0,dep=0,og=1,tim=17270212161
> EXEC #11:c=0,e=83,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,tim=17270212425
> FETCH #11:c=0,e=25231,p=13,cr=4,cu=0,mis=0,r=1,dep=0,og=1,tim=17270237752
> FETCH #11:c=13234375,e=37220332,p=22035,cr=5010560,cu=0,mis=0,r=0,dep=0,og=1, ...
> ```
>
> 첫 번째 Fetch에서 분명 1건(r=1)을 가져왔고, 두 번째 Fetch에서는 0건(r=0)을 가져왔다.
>
> SQL*Plus가 아닌 다른 쿼리 툴에서 트레이스를 걸어보면, 아래처럼 1건 Fetch를 위해 한번만 Call이 발생한다.
>
> ```
> PARSE #4:c=46875,e=52061,p=0,cr=83,cu=0,mis=1,r=0,dep=0,og=1,tim=24109588612
> EXEC #4:c=0,e=36,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,tim=24109595053
> FETCH #4:c=8640625,e=8699849,p=0,cr=5010563,cu=0,mis=0,r=1,dep=0,og=1, ...
> ```
>
> 'x > 0 and y <= 6' 조건일 때는 어떤가? 첫 번째 Fetch Call에서 5건을 가져오고 두 번째 Fetch에서 1건을 얻은 다음 대기할 것으로 예상된다.
> 그러나 실제로는 첫 번째 Fetch에서는 1건만 가져오고, 두 번째 Fetch에서 5건을 가져와 클라이언트 측 Array 버퍼를 채운다.
> 5개가 채워졌으므로 일단 화면에 출력하고, 남은 1건 외에 4건을 더 가져와 Array 버퍼를 채우려고 세 번째 Fetch를 수행하는데, 이 때 지연이 발생하게 된다.
> 아래 트레이스 결과를 통해 6건을 가져오려고 3번 Fetch Call이 발생했음을 알 수 있다.
>
> ```
> call     count     cpu    elapsed     disk      query    current   rows
> ------- ------ ------- ---------- -------- ---------- ---------- ------
> Parse        1    0.00       0.00        0          0          0      0
> Execute      1    0.00       0.00        0          0          0      0
> Fetch        3   11.42      33.59    22796    5010565          0      6
> ------- ------ ------- ---------- -------- ---------- ---------- ------
> total        5   11.42      33.59    22796    5010565          0      6
> ```
>
> 아래 트레이스 원시파일 내용을 통해 각각 1, 5, 0건을 Fetch 했음을 확인할 수 있다.
>
> ```
> PARSE #3:c=0,e=81,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,tim=29655520663
> EXEC #3:c=0,e=95,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,tim=29655541041
> FETCH #3:c=0,e=57507,p=24,cr=4,cu=0,mis=0,r=1,dep=0,og=1,tim=29655601091
> FETCH #3:c=0,e=63588,p=40,cr=6,cu=0,mis=0,r=5,dep=0,og=1,tim=29655667566
> FETCH #3:c=11421875,e=33477896,p=22732,cr=5010555,cu=0,mis=0,r=0,dep=0,og=1, ...
> ```
>
> SQL*Plus가 아닌 다른 쿼리 툴에서 트레이스를 걸어보면, 아래처럼 Fetch Call이 두 번만 발생하며, 첫 번째 Fetch에서 5건, 두 번째 Fetch에서 1건을 가져온다.
>
> ```
> PARSE #2:c=109375,e=154193,p=0,cr=83,cu=0,mis=1,r=0,dep=0,og=1,tim=24086976950
> EXEC #2:c=0,e=73,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,tim=24086997667
> FETCH #2:c=0,e=149,p=0,cr=8,cu=0,mis=0,r=5,dep=0,og=1,tim=24087000985
> FETCH #2:c=8593750,e=8598242,p=0,cr=5010556,cu=0,mis=0,r=1,dep=0,og=1, ...
> ```
>
> Fetch 관련 테스트를 진행하고자 할 때, 쿼리 툴마다 이처럼 다른 방식으로 데이터를 Fetch 한다는 사실을 알 필요가 있다. Array 단위로 멈추지 않고 모든 데이터를 다 Fetch하는 쿼리 툴도 있다.
> 하지만 어떤 클라이언트 툴을 사용하든 서버 측에서는 항상 Array 단위로 전송한다는 사실만큼은 변함이 없다.
<br/>

## (3) ArraySize 조정에 의한 Fetch Call 감소 및 블록 I/O 감소 효과
대량 데이터를 내려받을 때 ArraySize를 크게 설정할수록 그만큼 Fetch Call 횟수가 줄어 네트워크 부하가 감소하고, 쿼리 성능이 향상된다. 그뿐만이 아니다.
서버 프로세스가 읽어야 할 블록 개수까지 줄어드는 일거양득의 효과를 얻게 된다. ArraySize를 조정하는데 왜 블록 I/O가 줄어드는 것일까? 직접 테스트를 통해 ArraySize 조정에 따른 효과를 확인해 보자.

```
SQL> create table test as select * from all_objects;

테이블이 생성되었습니다.

SQL> set autotrace traceonly statistics;

세션이 변경되었습니다.

SQL> set arraysize 2
SQL> select * from test;

49839 개의 행이 선택되었습니다.

Statistics
---------------------------------------------------------------------
          0  recursive calls
          0  db block gets
      25366  consistent gets
       1382  physical reads
          0  redo size
    5155080  bytes sent via SQL*Net to client
     274494  bytes received via SQL*Net from client
      24921  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
      49839  rows processed
```

ArraySize를 2로 설정한 상태에서 49,839 로우를 가져오려고 읽은 블록 개수는 25,366개다.
그리고 Fetch 횟수는 24,921번이므로 한번 Fetch 할 때마다 2개 로우씩(49,839/24,921 = 1.9998796) 읽은 것을 알 수 있다.

SQL 트레이스를 이용해도 같은 테스트를 진행할 수 있는데, 각 항목을 매칭시켜보면 아래와 같다.

```
call     count    cpu    elapsed    disk    query    current     rows
------- ------ ------ ---------- ------- -------- ---------- --------
Parse        1   0.00       0.00       0        0          0        0
Execute      1   0.00       0.00       0        0          0        0
Fetch    24921   0.46       0.33    1382    25366          0    49839
------- ------ ------ ---------- ------- -------- ---------- --------
total    24921   0.46       0.33    1382    25366          0    49839

  * db block gets                      =  current
  * consistent gets                    =  query
  * physical reads                     =  disk
  * SQL*Net roundtrips to/from client  =  fetch count
  * rows processed                     =  fetch rows
```

ArraySize를 늘리면서 테스트를 계속해 보자.

```
SQL> set arraysize 5
SQL> select * from test;
...

      10509  consistent gets
       9969  SQL*Net roundtrips to/from client
      49839  rows processed

SQL> set arraysize 10
SQL> select * from test;
...

       5613  consistent gets
       4985  SQL*Net roundtrips to/from client
      49839  rows processed

SQL> set arraysize 100
SQL> select * from test;
...

       1184  consistent gets
        500  SQL*Net roundtrips to/from client
      49839  rows processed

SQL> set arraysize 5000
SQL> select * from test;
...

        700  consistent gets
         11  SQL*Net roundtrips to/from client
      49839  rows processed
```

ArraySize를 키울수록 Fetch Count 횟수가 줄고 더불어 Block I/O까지 주는 것을 볼 수 있다. 즉, 반비례 관계다.

|ArraySize|Fetch Count<br/>(SQL*Net roundtrips)|Block I/O|
|---:|---:|---:|
|2|24,921|25,366|
|5|9,969|10,509|
|10|4,985|5,613|
|20|2,493|3,153|
|50|998|1,677|
|100|500|1,184|
|200|251|939|
|500|101|789|
|1,000|51|740|
|2,000|26|715|
|5,000|11|700|

눈에 띄는 것은, ArraySize를 키운다고 해서 같은 비율로 Fetch Count와 Block I/O가 줄지는 않는다는 점이다.
따라서 무작정 크게 설정한다고 좋은 것만은 아니며 일정 크기 이상이면 오히려 리소스만 낭비하게 된다. 데이터 크기에 따라 다를 텐데, 위 데이터 상황에서는 100 정도로 설정하는 게 적당해 보인다.

ArraySize가 늘면서 블록 I/O까지 감소하는 원리를 설명해 보자.

\[그림 5-8\](p.384)처럼 10개 행으로 구성된 3개의 Block이 있다고 하자. 총 30개 레코드이므로 ArraySize를 3으로 설정하면 Fetch 횟수는 10이고, 이때 Block I/O는 12번이나 발생하게 된다.
왜냐하면, 10개 레코드가 담긴 블록들을 각각 4번에 걸쳐 반복 액세스해야 하기 때문이다. 그림에서 보듯, 첫 번째 Fetch에서 읽은 1번 블록을 2\~4번째 Fetch에서도 반복 액세스하게 된다.
2번 블록은 4\~7번째 Fetch, 3번 블록은 7\~10번 Fetch에 의해 반복적으로 읽힌다. 만약 ArraySize를 10으로 설정한다면 3번의 Fetch와 3번 블록 I/O로 줄일 수 있다.
그리고 ArraySize를 30으로 설정하면 Fetch 횟수는 1로 줄어든다.
<br/>
<br/>
## (4) 프로그램 언어에서 Array 단위 Fetch 기능 활용
지금까지 SQL*Plus 중심으로만 설명했는데, PL/SQL을 포함한 프로그램 언어에서 어떻게 ArraySize를 제어하는지 확인해 보자.

PL/SQL에서 커서를 열고 레코드를 Fetch하면, (Array Processing에서 보았던 Bulk Collect 구문을 사용하지 않는 한) 9i까지는 한 번에 한 로우씩만 처리(Single-Row Fetch)했었다.
10g부터는 자동으로 100개씩 Array Fetch가 일어난다. 단, 아래처럼 Cursor FOR Loop 구문을 이용할 때만 작동한다.

```
  for item in cursor
  loop
    ......
  end loop;
```

Cursor FOR Loop 문은 커서의 Open, Fetch, Close가 내부적으로 이루어지는 것이 특징이며, Implicit Cursor FOR Loop와 Explicit Cursor FOR Loop 두 가지 형태가 있다.
둘 다 Array Fetch 효과를 얻을 수 있으며, 아래 예제를 통해 구문의 차이점을 확인하자.

```
# Implicit Cursor FOR Loop

declare
  l_object_name big_table.object_name%type;
begin
  for item in ( select object_name from big_table where rownum <= 1000 )
  loop
    l_object_name := item.object_name;
    dbms_output.put_line(l_object_name);
  end loop;
end;
/

# Explicit Cursor FOR Loop

declare
  l_object_name big_table.object_name%type;
  cursor c is select object_name from big_table where rownum <= 1000;
begin
  for item in c
  loop
    l_object_name := item.object_name;
    dbms_output.put_line(l_object_name);
  end loop;
end;
/
```

SQL 트레이스 결과는 둘 다 아래와 같이 동일하다.

```
SELECT OBJECT_NAME
FROM BIG_TABLE WHERE ROWNUM <= 1000

call     count      cpu    elapsed    disk    query    current    rows
------- ------ -------- ---------- ------- -------- ---------- -------
Parse        1     0.00       0.00       0        0          0       0
Execute      1     0.00       0.00       0        0          0       0
Fetch       11     0.00       0.00       0       25          0    1000
------- ------ -------- ---------- ------- -------- ---------- -------
total       13     0.00       0.00       0       25          0    1000

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 64      (recursive depth: 1)

Rows     Row Source Operation
-------  ---------------------------------------------------
   1000  COUNT STOPKEY (cr=25 pr=0 pw=0 time=1049 us)
   1000   TABLE ACCESS FULL BIG_TABLE (cr=25 pr=0 pw=0 time=1045 us)
```

아래는 Cursor FOR Loop가 아닌 일반 커서를 이용하는 예제와 그때의 SQL 트레이스 결과다.
위에서 11이던 Fetch 횟수가 1,001로 늘어난 것을 통해 Array 단위 Fetch가 작동하지 않았음을 알 수 있고, 이는 sys_refcursor를 사용해도 마찬가지 결과가 나타난다.

Array 단위 Fetch가 작동하지 못함으로 인해 블록 I/O까지 25에서 1,003으로 40배가량 늘어났다.

```
declare
cursor c is
  select object_name
  from big_table where rownum <= 1000;
  l_object_name big_table.object_name%type;
begin
  open c;
  loop
    fetch c into l_object_name;
    exit when c%notfound;
    dbms_output.put_line(l_object_name);
  end loop;
  close c;
end;
/
*************************************************************************

SELECT OBJECT_NAME
FROM BIG_TABLE WHERE ROWNUM <= 1000

call     count      cpu    elapsed    disk    query    current    rows
------- ------ -------- ---------- ------- -------- ---------- -------
Parse        1     0.00       0.00       0        0          0       0
Execute      1     0.00       0.00       0        0          0       0
Fetch     1001     0.01       0.00       0     1003          0    1000
------- ------ -------- ---------- ------- -------- ---------- -------
total     1003     0.01       0.00       0     1003          0    1000

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 64      (recursive depth: 1)

Rows     Row Source Operation
-------  ---------------------------------------------------
   1000  COUNT STOPKEY (cr=1003 pr=0 pw=0 time=4058 us)
   1000   TABLE ACCESS FULL BIG_TABLE (cr=1003 pr=0 pw=0 time=2049 us)
```

일반 프로그램 언어에서는 어떻게 ArraySize를 조정하는지 JAVA 예를 통해 확인해 보자.

```
String sql = "select custid, name from customer";
PrparedStatement stmt = conn.prepareStatement(sql);
stmt.setFetchSize(100);    -- Statement에서 조정
ResultSet rs = stmt.executeQuery();
// rs.setFetchSize(100);   -- ResultSet에서 조정할 수도 있다.

while ( rs.next() ) {
    int empno = rs.getInt(1);
    String ename = rs.getString(2);
    System.out.println(empno + ":" + ename);
}

rs.close();
stmt.close();
```

setFetchSize 메서드를 이용해 FetchSize를 조정하는 예는 앞 절에서 잠깐 본 적이 있다.
JAVA에서 FetchSize 기본 값은 10이다. 대량 데이터를 Fetch 할 때 이 값을 100\~500 정도로 늘려 주면 기본 값을 사용할 때보다 데이터베이스 Call 부하를 1/10\~1/50으로 줄일 수 있다.
예를 들어, FetchSize를 100으로 설정했을 때 데이터를 Fetch 해오는 메커니즘은 아래와 같다.

- [1] 최초 rs.next() 호출 시 한꺼번에 100건을 가져와서 클라이언트 Array 버퍼에 캐싱한다.
- [2] 이후 rs.next() 호출할 때는 데이터베이스 Call을 발생시키지 않고 Array 버퍼에서 읽는다.
- [3] 버퍼에 캐싱 돼 있던 데이터를 모두 소진한 후 101번째 rs.next() 호출 시 다시 100건을 가져온다.
- [4] 모든 결과집합을 다 읽을 때까지 2\~3번 과정을 반복한다.