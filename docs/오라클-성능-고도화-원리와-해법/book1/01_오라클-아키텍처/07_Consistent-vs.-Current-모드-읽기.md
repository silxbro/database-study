# 07. Consistent vs. Current 모드 읽기

SQL 튜닝할 때 트레이스를 통해 항상 접하게 되는 이 두 가지 블록 읽기 모드의 차이점을 설명하려고 지금까지 먼 길을 돌아왔다고 해도 과언이 아니다.
<br/>
<br/>
## (1) Consistent 모드 읽기와 Current 모드 읽기의 차이점
먼저 **Consistent 모드 읽기(gets in consistent mode)는, SCN 확인 과정을 거치며 쿼리가 시작된 시점을 기준으로 일관성 있는 상태로 블록을 액세스하는 것**을 말한다.
이 모드로 데이터를 읽을 때는 쿼리가 1시간 걸리든 10시간 걸리든 항상 쿼리가 시작된 시점의 데이터를 가져온다.

SQL 트레이스 Call 통계에서 볼 수 있는 '**query**' 항목과 AutoTrace에서의 '**consistent gets**' 항목이 지금 설명한 Consistent 모드에서 읽은 블록 수를 의미한다.
읽는 중에 CR copy를 생성할 필요가 없어 **Current 블록을 읽더라도 Consistent 모드에서 읽었다면 'query' 항목에 집계**된다.
select 문에서 읽은 블록은 대부분 여기에 해당하며, 여기에는 **CR 블록을 생성하려고 Undo 세그먼트로부터 읽어들이는 블록 수까지 더해진다**.

```
[  SQL Trace 결과  ]

call     count    cpu    elapsed    disk     query   current    rows
------- ------ ------ ---------- ------- --------- --------- -------
Parse        1   0.00       0.00       0         0         0       0
Execute      1   0.41       1.15      66     28691      7235    7071
Fetch        0   0.00       0.00       0         0         0       0
------- ------ ------ ---------- ------- --------- --------- -------
total        2   0.41       1.16      66     28691      7235    7071

[  AutoTrace 결과  ]

Statistics
--------------------------------------------------------
          0  recursive calls
       7235  db block gets
      28691  consistent gets
         66  physical reads
     663872  redo size
        935  bytes sent via SQL*Net to client
        979  bytes received via SQL*Net from client
          6  SQL*Net roundtrips to/from client
          1  sorts (memory)
          0  sorts (disk)
       7071  rows processed
```
<br/>

**Current 모드 읽기(gets in current mode)는, SQL 문이 시작된 시점이 아니라 데이터를 찾아간 바로 그 시점의 최종 값을 읽으려고 블록을 액세스하는 것**을 말한다.
블록 SCN이 쿼리 SCN보다 높고 낮음을 따지지 않으며, 그 시점에 이미 커밋된 값이라면 그대로 받아들이고 읽는다.

SQL 트레이스에서 Call 통께 레포트를 통해 볼 수 있는 '**current**' 항목과 AutoTrace에서의 '**db block gets**' 항목이 Current 모드에서 읽은 블록 수를 의미하며, 주로 다음과 같은 상황에서 나타난다.

- DML문을 수행할 때 주로 나타난다.
- select for update문을 수행할 때도 Current 모드 읽기를 발견할 수 있다.
- 8i 이전 버전에서는 Full 스캔을 포함하는 select 문에서도 Current 모드 읽기가 나타났는데, Full Scan 할 익스텐트 맵(Extent Map) 정보를 읽으려고 세그먼트 헤더에 접근할 때 익스텐트에 대한
  바로 현재 시점의 정보가 필요하기 때문이다. Locally Managed 테이블스페이스를 주로 사용하기 시작한 9i 이상부터는 Full Table Scan을 하더라도 Current 모드 읽기가 발생하지 않는다.
  Index rowid를 이용한 테이블 액세스 시에는 테이블 익스텐트 정보가 불필요하므로 버전에 상관없이 Current 모드 읽기가 발생하지 않는다.

- 디스크 소트가 필요할 정도로 대량의 데이터를 정렬할 때도 Current 모드 읽기가 나타난다.

Consistent 모드 읽기에 대한 원리는 지금껏 상세히 설명해 왔지만 Current 모드 읽기는 여기서 처음 소개하므로 좀 더 자세히 설명하려고 한다.
<br/>
<br/>
## (2) Consistent 모드로 갱신할 때 생기는 현상
EMP 테이블 7788번 사원의 SAL 값이 현재 1,000인 상황에서 아래 TX1, TX2 두 개의 트랜잭션이 동시에 수행되었다. 양쪽 트랜잭션이 모두 완료된 시점에 7788 사원의 SAL 값은 얼마이어야 할까?

### [상황 1]
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = sal + 100<br/>where empno = 7788;<br/><br/><br/>commit;<br/><br/>|t1<br/><br/>t2<br/><br/>t3<br/>t4|<br/><br/>update emp set sal = sal + 200<br/>where empno = 7788;<br/><br/>commit;|

물론 TX2 update는 t2 시점에 시작하지만 TX1에 의해 걸린 Lock을 대기하다가 t3 시점에 TX1이 커밋된 후에 진행을 계속한다. 분명히 답은 1,200과 1,300 둘 중 하나인데, 설명은 제각각이다.
재미있는 것은, 오라클만의 독특한 읽기 일관성 모드에 익숙한 사람들이 더 시간을 끌면서 고민하고 심지어 틀린 답을 내기도 한다는 사실이다.

그 이유를 곰곰이 생각해 보면 오라클이 특이하게도 두 가지 읽기 모드를 제공하는 데에서 비롯된다.
앞 절에서 Consistent 모드 읽기 원리에 대해서 집중적으로 설명해 왔는데, 지금까지 설명한 세세한 내부 원리를 잘 모르더라도 오라클 매뉴얼에 간단히 언급된 기본적인 내용, 즉 쿼리 시작 시점을 기준으로
값을 읽는다는 사실은 대부분 알고 있다. 그러다 보니 오라클 사용자들은 항상 Consistent 모드 읽기 중심으로 생각한다.

만약 위에서 예시한 두 개의 update 문이 Consistent 모드로 값을 읽고 갱신한다면 어떤 일이 발생할까?
Dirty Read를 허용하지 않는 한 t1과 t2 시점에 SAL 값은 1,000이었으므로 둘 다 1,000을 읽고 각각 100과 200을 더해 갱신을 완료한다.
최종 값은 1,200이 될 것이며, 트랜잭션 TX1의 처리 결과는 사라졌으므로 소위 말하는 **Lost Update가 발생**하는 결과를 초래한다.

이런 Lost Update 문제를 회피하려면 갱신 작업만큼은 Current 모드를 사용해야 한다.
정리하면, <상황1>에서 TX2 update는 Exclusive Lock 때문에 대기했다가 TX1 트랜잭션이 커밋한 후 Current 모드로 그 값을 읽어 진행을 계속한다. 그럼으로써 Lost Update 문제를 피할 수 있다.
<br/>
<br/>
## (3) Current 모드로 갱신할 때 생기는 현상
Current 모드로 갱신해야 Lost Update를 회피할 수 있다고 했는데, Current 모드로만 처리했을 때 아래 두 트랜잭션이 모두 완료되고 나서 7788 사원의 SAL 값은 얼마일지 예측해 보기 바란다.
t1 직전 시점에 7788 사원의 SAL 값이 1,000이었다.(오라클에서의 실제 처리 결과가 아니라 Current 모드로 처리했을 때의 결과를 묻는 것이다.)

### [상황 2]
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = 2000<br/>where empno = 7788<br/>and sal = 1000 ;<br/><br/><br/><br/>commit;<br/><br/><br/>|t1<br/><br/><br/>t2<br/><br/><br/>t3<br/><br/>t4|<br/><br/><br/>update emp set sal = 3000<br/>where empno = 7788<br/>and sal = 2000 ;<br/><br/><br/>commit;|

Current 모드로 처리한다면, TX2 트랜잭션은 TX1 트랜잭션이 커밋되기를 기다렸다가 SAL 값이 2,000으로 갱신되는 것을 확인하고 정상적으로 update를 수행한다. 따라서 최종 값은 3,000이 된다.

항상 Current 모드로만 작동하는 Sybase, SQL Server 같은 DBMS에서 수행해 보면 실제 위와 같은 결과가 나온다. 이렇게 처리해도 문제가 없는 것일까?
이에 대한 판단에 앞서 한가지 경우를 더 살펴보자.

### [상황 3]
|TX1||TX2|
|:---|:---:|:---|
|update t set no = no + 1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br/>where no > 50000 ;<br/><br/><br/><br/><br/><br/>commit;|t1<br/><br/><br/>t2<br/><br/>t3<br/><br/>t4|<br/><br/>insert into t values(100001, 100001);<br/><br/>commit;<br/><br/>|

TX1이 1\~100,000까지의 Unique한 번호(no)를 가진 테이블에서 no > 50000 조건에 해당하는 50,000개 레코드에 대해 인덱스를 경유해 순차적으로 갱신 작업을 진행하고 있다고 하자.
그런데 도중에 TX2 트랜잭션에서 no 값이 100,001인 레코드를 새로 추가하면 update되는 최종 결과건수는 50,000이어야 할까? 아니면 50,001이어야 할까?

실제 SQL Server에서 테스트 해보면, 50,001건이 갱신된다. 즉, update가 진행되는 동안 새로 추가된 레코드까지도 값이 변경된 것이다.
만약, 인덱스를 이용해 no 값을 순차적으로 읽지 않고 Full Table Scan 방식으로 update를 진행한다면 insert 되는 100,001번째 레코드가 삽입되는 위치에 따라 update 결과 건수가 그때그때
달라진다. 즉, update가 이미 진행된 블록(=페이지)에 insert된 레코드는 갱신 대상에서 제외되고, 아직 update 되지 않은 블록(=페이지)에 insert된 레코드는 갱신 대상에 포함된다.

같은 이치로, 100,001번째 레코드를 insert하는 대신 TX2가 아래 update문을 수행하면 변경된 레코드가 TX1의 update 대상 건에서 제외되므로 최종적으로 49,999건만 갱신된다.

```
TX2> update t set no = 0 where no = 100000;

TX2> commit;
```

또 다른 예로서, "delete from 로그" 문장이 수행되는 도중에 다른 트랜잭션에 의해 새로 추가된 로그 데이터까지 지워질 수도 있는데, **update/delete 도중에 갱신/삭제 대상이 그때그때 변할 수
있다**는 것은 오라클 사용자 입장에서는 참 놀라운 사실이 아닐 수 없다.
<br/>
<br/>
## (4) Consistent 모드로 읽고, Current 모드로 갱신할 때 생기는 현상
Current 모드로 갱신할 수행할 때 어떤 현상이 생기는지 살펴보았다. 이 문제를 피하려고 오라클은 Consistent 모드로 읽고, Current 모드로 갱신한다.
오라클을 설명하는 자료에서 다음과 같은 내용을 흔히 접할 수 있다.

> #### <설명1>
> 오라클에서 update문을 수행하면, 대상 레코드를 읽을 때는 Consistent 모드로 읽고 실제 값을 변경할 때는 Current 모드로 읽는다.
> 따라서 대상 레코드를 읽기 위한 블록 액세스는 SQL 트레이스에서 query 항목으로 계산되고, 값을 변경하기 위한 블록 액세스는 current 항목에 계산된다.

하지만 이 의미를 단순하게 이해한다면 또 다른 오류에 빠질 수 있다. 위 내용을 읽은 후 아래와 같은 상황에서 어떤 결과가 나올지 예측해 보기 바란다.

### [상황 4]
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = sal + 100<br/>where empno = 7788<br/>and sal = 1000 ;<br/><br/><br/><br/>commit;<br/><br/><br/>|t1<br/><br/><br/>t2<br/><br/><br/>t3<br/><br/>t4|<br/><br/><br/>update emp set sal = sal + 200<br/>where empno = 7788<br/>and sal = 1000 ;<br/><br/><br/>commit;|

TX2는 TX1이 커밋되기를 기다렸다가 TX1이 끝나면 계속 진행한다. 하지만 이 때 7788 사원의 SAL 값은 이미 1,100으로 바뀐 상태이므로 TX2의 update는 실패하게 된다.

오라클 외 다른 DBMS는 항상 Current 모드 읽기만 지원하기 때문에 헷갈릴 이유가 없으며 너무 자연스럽게 이해가 된다.
하지만 <설명1>을 접했다면 오라클에서 위 update가 실패하는 이유를 쉽게 이해하지 못할 수 있다.
TX2가 실제 값을 갱신할 때는 이미 1,100으로 바뀐 값을 읽겠지만 갱신 대상 레코드를 찾아갈 때는 Consistent 모드를 사용하기 때문이다.
즉, "empno = 7788이고 sal = 1000"인 레코드를 TX1이 변경하기 이전 값을 이용해 찾아가기 때문에 update를 실패할 이유가 없다고 생각한다.

아래 [1]번 update문을 [2]번 문장처럼 바꿔서 표현할 수 있는데, <설명1>을 문장 그대로 받아들인다면 [2]번 update문에서 *로 표시한 부분은 Consistent 모드로 읽고, 나머지 부분(→ sal =
sal + 200에서 sal 값)은 Current 모드로 읽는다는 믿음을 갖게 된다.

```
[1] update emp set sal = sal + 200
    where empno = 7788
    and sal = 1000 ;

[2] update
    (
      select sal from emp    *
      where  empno = 7788    *
      and    sal = 1000      *
    )
    set sal = sal + 200;
```

실제 그런 식으로 처리한다면 <상황4>의 TX2 update문은 'sal = 1000'인 레코드를 갱신하는 문장인데도, 이미 1,100으로 바뀐 레코드를 갱신하는 결과를 초래하게 된다.
하지만 다른 DBMS와 마찬가지로 오라클에서도 TX2의 갱신은 실패한다(갱신 레코드 건수 = 0).
<br/>
<br/>
## (5) Consistent 모드로 갱신대상을 식별하고, Current 모드로 갱신
그럼 실제 오라클은 어떻게 두 개의 읽기 모드가 공존하면서 update를 처리하는 것일까? 이해를 돕기 위해 pseudo 코드로 표현해 보면 다음과 같다.

```
for c in
  (
      -- Consistent
      select rowid rid, empno, sal from emp
      where  empno = 7788 and sal = 1000
  )
loop
      -- Current
      update emp set sal = sal + 200
      where  empno = c.empno
      and    sal = c.sal
      and    rowid = c.rid ;
end loop;
```

위 pseudo 코드가 앞에서 인라인 뷰를 사용한 [2]번 update문과 다른 점은 무엇일까? Consistent 모드에서 수행한 조건 체크를, Current 모드로 액세스하는 시점에 한 번 더 수행한다는 점이다.

pseudo 코드 내용을 두 단계로 나누어 설명해 보자.

- 단계1) where절에 기술된 조건에 따라 수정/삭제할 대상 레코드의 rowid를 **Consistent 모드**로 찾는다(→ DML문이 시작된 시점 기준).
- 단계2) 앞에서 읽은 rowid가 가리키는 레코드를 찾아가 로우 Lock을 설정한 후에 **Current 모드**로 실제 update/delete를 수행한다(→ 값이 변경되는 시점 기준).
  이 단계에서 Current 모드로 다시 한번 조건을 체크하고, 갱신할 값을 읽어 수정/삭제한다.

단계1을 수행해 update/delete 대상 건을 '모두' 추출하고 나서 단계2를 수행한다고 생각하면 안 된다.
위 pseudo 코드에서 보이고 있듯이 단계1에서 커서를 열어 Fetch하면서 단계2를 건건이 반복 수행한다.

단계1은 udpate/delete가 시작된 시점 기준으로 수정/삭제할 대상을 식별하려고 Consistent 모드 읽기를 사용할 뿐이며 거기서 읽은 값을 단계2에서 갱신하는 데 사용하지는 않는다.
단계1이 필요한 이유는, 갱신이 진행되는 동안 추가되거나 변경을 통해 범위 안에 새로 들어오는 레코드를 제외하고자 하는 것이다.
이미 범위 안에 포함돼 있던 레코드는, 단계2에서 변경이 이루어지는 바로 그 시점 기준으로 값을 읽고 갱신한다.
이때는 블록 SCN이 쿼리 SCN보다 높고 낮음을 따지지 않으며, 그 시점에 이미 커밋된 값이라면 그대로 받아들이고 읽는다.
이 때문에(update문이 시작된 이후에 1,000에서 1,100으로 값이 변경되었기 때문에) <상황4>에서 TX2의 update가 실패하는 것이다.

> #### <설명2>
> - [1] select는 Consistent 모드로 읽는다.
> - [2] insert, update, delete, merge는 Current 모드로 읽고 쓴다. 다만, 갱신할 대상 레코드를 식별하는 작업만큼은 Consistent 모드로 이루어진다.

이 두 가지 사실만 기억하면 그다지 어렵지 않게 동시 트랜잭션의 최종 결과를 예측할 수 있다. 정리하는 차원에서 앞서 본 <상황2>와 <상황3>을 다시 살펴보기 바란다.
처음엔 자신 없었을지 몰라도, 지금 보면 쉽게 처리 결과를 예측할 수 있을 것이다.

> #### [Write Consistency]
> 앞선 사례(상황4)에서, Consistent 모드와 Current 모드에서 읽은 값이 서로 달라 TX2의 update는 실패했다.
> TX2가 update를 시작한 t2 시점 기준으로는 empno = 7788 사원 레코드가 분명 갱신 대상이었는데, 값이 달라졌다고 아무런 처리 없이 지나가도 상관없는 걸까? 그렇지 않다
> 데이터 정합성에 문제가 생기는 경우가 있고, 이를 방지하려고 오라클은 'Restart 메커니즘'을 사용한다.
> 그때까지의 갱신을 롤백하고 update를 처음부터 다시 실행하는 것이며, Thomas Kyte는 그의 저서에 이것을 "Write Consistency"라고 명명하고 있다.
>
> 이 기능은 데이터베이스 일관성을 유지하려고 오라클이 사용하는 아주 내부적인 메커니즘이므로 중요하게 다룰 내용은 아니다.
> Consistent 모드로 찾은 레코드를 Current 모드로 읽어 갱신한다는 기본 컨셉만 이해해도 충분하다. 갱신 대상 레코드의 값이 중간에 바뀌었다고 항상 Restart 메커니즘이 작동하는 것도 아니다.
> where절에 사용된 컬럼 값이 바뀌었을 때만 작동한다.
>
> 그리고 Restart 메커니즘이 작동하더라도 대개는 처리 결과가 달라지지 않는다. 위 사례(상황4)만 보더라도 그렇다. 데이터베이스 일관성에 문제가 생기는 사례를 찾기가 오히려 어렵다.
> 이것은 동시 트랜잭션 유형, 데이터 저장 위치, 그리고 갱신 시점의 미묘한 차이 때문에 일관성 없게 값이 갱신되는 현상을 방지하려고 구현된 기능인데, 어떨 때 그런 현상이 발생하는지 비교적 쉬운 예를
> 하나 들어 보자. 아래와 같은 t 테이블이 있다.
>
>   &nbsp;&nbsp;&nbsp;id&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col2<br/>
>   &nbsp;&nbsp;\-------------------<br/>
>   &nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;A<br/>
>   &nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B<br/>
>   &nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;C
>
> 먼저 트랜잭션 TX1에서 아래 update문을 실행하였다.
>
> ```
> TX1> update t set col2 = 'X' where col1 <= 5;
> ```
>
> 1번(id = 1) 레코드를 갱신하고 아직 2번(id = 2)으로 진입하지 않은 상황에서 두 번째 트랜잭션 TX2가 아래 update문을 실행하고 커밋까지 성공하였다.
> (이미 update를 진행한 영역은 Lock이 설정돼 있으므로 문제가 생기지 않는다. 문제는 항상 앞으로 진행할, Lock이 설정되지 않은 영역에서 발생한다.)
>
> ```
> TX2> update t set col1 = (case when col1 = 5 then 6 else 5 end) where id in (2, 3);
> TX2> commit;
> ```
>
> 이후 TX1이 처리를 계속할 때 2번과 3번 레코드는 갱신 대상에서 제외된다. Restart가 작동하지 않는다면 말이다. 3번 레코드는 원래 갱신 대상이 아니었다.
> 2번은 대상이었으나 갱신하기 직전에 col1 값이 6으로 바뀌어 대상에서 제외된다.
>
> Restart 없이 처리했을 때, 일관성에 문제가 있다고 느끼는가? 읽기 작업이 쿼리 시작 시점 기준으로 일관성 있게 진행하는 것처럼, 쓰기 작업도 기준 시점에 존재해야 한다.
> 그런데 조금 전 트랜잭셔녀 처리 결과는 그렇지가 못하다.
> 2번과 3번 두 레코드가 모두 갱신 대상에서 제외되었는데, 어느 시점으로 보더라도 이 두 레코드가 동시에 'col1 <= 5' 조건을 만족하지 않았던 적이 없었기 때문이다.
> "TX1의 시작 시점"으로 보면, 2번은 TX1의 갱신 대상이었고 3번은 대상이 아니었다. "TX1의 완료 시점" 또는 "TX2의 완료 시점"으로 보면, 3번은 TX1의 갱신 대상이었고 2번은 대상이 아니었다.
>
> 이 문제를 해결하려고 오라클은 Restart 방식을 선택하였고, Restart 시점이 일관성 기준 시점이 된다.
> 물론, 기준 시점이 바뀌었으므로 처음 update 시작 시점과 Restart 시점 사이에 제3의 트랜잭션이 레코드를 추가/변경/삭제했다면 그것도 최종 update 결과에 반영된다.
> 그리고 상당히 많은 갱신 작업이 이루어진 이후에 이 기능이 작동함으로써 겪는 성능상 불이익은 사용자가 감내해야 할 몫이다.
>
> 참고로, 이후 update 과정에서 Restart가 또다시 발생하는 불상사를 막기 위해 오라클은 조건에 부합하는 레코드를 모두 SELECT FOR UPDATE 모드로 Lock을 설정하고 나서 update를 재시작한다.
>
> 데이터를 일관성 있게 갱신하려면 처음부터 SELECT FOR UPDATE 모드로 Lock을 설정하고 나서 진행해야 안전하지만(→ 비관적 동시성 제어) 대상 범위를 두 번 액세스하는 부하 및 동시성 저하가 발생한다.
> 따라서 오라클은 일단 update를 진행해 보고(→ 낙관적 동시성 제어) 일관성을 해칠만한 사유가 발생(→ 조건절 컬럼 값이 변경됐음을 발견)한 때만, 그때까지의 처리를 롤백하고 원안대로 다시 시작하는 것이라고
> 이해하면 쉽다.
>
> 결과적으로 t 테이블의 최종 상태는 아래 좌측과 같다. Write Consistency 개념을 설명하기 직전까지만 이해한 독자라면 아래 우측과 같이 예상했을 것이다.
>
>   &nbsp;&nbsp;&nbsp;id&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;id&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;col2<br/>
>   &nbsp;&nbsp;\-------------------&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;\-------------------<br/>
>   &nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;X&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;X<br/>
>   &nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B<br/>
>   &nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;X&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;C
>
> Restart 메커니즘이 필요한 예를 한 가지 더 들어 보겠다. 이번에는 emp 레코드에 갱신이 발생할 때마다 로그 테이블에 기록을 남기는 before update each row 트리거가 활성화돼 있다고 하자.
> 그러면 상황 4에서, t3 시점에 TX2에 대한 블로킹이 해제되면서 로그 테이블에 insert가 진행되지만 정작 emp 레코드에 대한 갱신은 실패해 데이버베이스가 비일관된 상태에 놓이게 된다.
> 이때 Restart 메커니즘이 작동해 이제까지의 갱신(로깅 데이터도 포함)을 롤백하고 update를 다시 진행한다면 그런 비일관성을 해결할 수 있다.
>
> 이 기능을 좀 더 연구하고 싶다면, Thomax Kyte가 저술한 Expert Oracle Database Architecture 7장을 참조하기 바란다. 그의 블로그에서도 같은 내용을 열람할 수 있다.
<br/>

## (6) 오라클에서 일관성 없게 값을 갱신하는 사례
오라클만의 독특한 Consistency 모델을 이해하지 못한 채 프로그램을 작성하고 그로 말미암아 종종 일관성 없는 상태로 값이 갱신되는 오류(주로 사용자 정의 함수/프로시저, 트리거 등을 사용할 때 발생)가
발견되곤 하는데, 여기서 그런 사례 중 하나를 소개하려고 한다.

```
update 계좌2
set    총잔고 = 계좌2.잔고 +
                (select 잔고 from 계좌1 where 계좌번호 = 계좌2.계좌번호)
where  계좌번호 = 7788;

update 계좌2
set    총잔고 = (select 계좌2.잔고 + 잔고 from 계좌1
                where 계좌번호 = 계좌2.계좌번호)
where  계좌번호 = 7788;
```

스칼라 서브쿼리는 특별한 이유가 없는 한 항상 Consistent 모드로 읽기를 수행한다. 따라서 첫 번째 문장에서 계좌2.잔고는 Current 모드로 읽는 반면 계좌1.잔고는 Consistent 모드로 읽는다.
위 update 문장이 진행되는 도중에 계좌1에서 변경이 발생했더라도 update문이 시작되는 시점의 값을 찾아 읽고, delete가 발생했더라도 지워지기 이전 값을 찾아 읽는다.

반면 두 번째 문장은, Current 모드로 읽어야 할 계좌2의 잔고 값을 스칼라 서브쿼리 내에서 참조하기 때문에 스칼라 서브쿼리까지도 Current 모드로 작동하게 된다.
따라서 위 update 문장이 진행되는 도중에 계좌1에서 변경이 발생했다면 그 새로운 값을 읽고, delete가 발생했다면 조인에 실패해 NULL 값으로 update 될 것이다.

따라서 update 문이 수행되는 동안 두 테이블로부터 잔고를 변경하는 트랜잭션이 동시에 진행할 수 있는 상황이라면 업무 특성에 맞게 SQL을 작성해야만 한다.
코딩 스타일에 따라 실제 미묘한 차이가 발생할 수 있다는 사실을 테스트를 통해 확인해 보자.

```
TX1> create table 계좌1
  2  nologging
  3  as
  4  select empno 계좌번호, ename 계좌명, 1000 잔고 from emp;

TX1> create table 계좌2
  2  nologging
  3  as
  4  select empno 계좌번호, ename 계좌명, 1000 잔고, 2000 총잔고 from emp;

TX1> alter table 계좌1 add constarint 계좌1_pk primary key(계좌번호);

TX1> alter table 계좌2 add constraint 계좌2_pk primary key(계좌번호);

TX1> select 계좌1.잔고, 계좌2.잔고, 계좌2.총잔고
  2       , 계좌1.잔고+계좌2.잔고 총잔고2
  3  from   계좌1, 계좌2
  4  where  계좌1.계좌번호 = 7788
  5  and    계좌2.계좌번호 = 계좌1.계좌번호 ;

       잔고        잔고       총잔고      총잔고2
---------- ---------- ---------- ----------
      1000       1000       2000       2000
```

테스트용 데이터를 만들었고, 계좌2 테이블에 미리 구해 둔 총잔고와 실시간 쿼리해서 얻은 총잔고(→ 총잔고2)가 같음을 확인하였다.
이제 TX1에서 아래처럼 계좌1에 100원을 입금하고 계좌2에 200원을 입금하는 트랜잭션이 시작되었다.

```
TX1> update 계좌1 set 잔고 = 잔고 + 100 where 계좌번호 = 7788;

1 행이 갱신되었습니다.

TX1> update 계좌2 set 잔고 = 잔고 + 200 where 계좌번호 = 7788;

1 행이 갱신되었습니다.
```

그런데 TX1 트랜잭션이 진행되는 동안 문제가 생겼다.
TX1이 두 테이블 잔고를 변경하는 도중에 계좌1과 계좌2의 잔고를 더해 계좌2 테이블 총잔고 컬럼에 값을 갱신해 주는 배치 프로그램이 두 번째 트랜잭션(TX2)에서 돌기 시작했다(아래 update문).
잠시 Lock이 걸려 대기하겠지만 첫 번째 트랜잭션(TX1)이 곧이어 커밋을 하므로 Lock은 금방 풀린다.

```
TX2> update 계좌2 set 총잔고 = 계좌2.잔고
  2            + (select 잔고 from 계좌1 where 계좌번호 = 계좌2.계좌번호)
  3  where 계좌번호 = 7788;

TX1> commit;

TX2> commit;

TX1> select 계좌1.잔고, 계좌2.잔고, 계좌2.총잔고
  2       , 계좌1.잔고+계좌2.잔고 총잔고2
  3  from   계좌1, 계좌2
  4  where  계좌1.계좌번호 = 7788
  5  and    계좌2.계좌번호 = 계좌1.계좌번호 ;

       잔고        잔고       총잔고      총잔고2
---------- ---------- ---------- ----------
      1100       1200       2200       2300
```

어떤 일이 발생했는지 확인해 보자. TX1 트랜잭션이 시작되기 전 시점에 7788번 계좌의 총잔고는 2,000이었고, 트랜잭션이 끝난 시점의 총잔고는 2,300이어야 한다.
그런데 배치 프로그램을 통해 총잔고 컬럼에 저장된 것은 2,200이라는 엉뚱한 값이다. 왜 이런 일이 발생하는 걸까?
계좌2.잔고를 읽을 때는 Current 모드에서 읽지만 계좌1.잔고를 읽을 때는 Consistent 모드에서 읽었기 때문이다.

계좌1과 계좌2 테이블을 다시 만들고 같은 테스트를 진행해 보자. 대신 이번에는 TX2 트랜잭션의 update문을 아래와 같이 수행해 보았다.

```
TX2> update 계좌2 set 총잔고 =
  2    (select 잔고 + 계좌2.잔고 from 계좌1 where 계좌번호 = 계좌2.계좌번호)
  3  where 계좌번호 = 7788;
```

update 결과를 확인해 보자.

```
TX1> select 계좌1.잔고, 계좌2.잔고, 계좌2.총잔고
  2       , 계좌1.잔고+계좌2.잔고 총잔고2
  3  from   계좌1, 계좌2
  4  where  계좌1.계좌번호 = 7788
  5  and    계좌2.계좌번호 = 계좌1.계좌번호 ;

       잔고        잔고       총잔고      총잔고2
---------- ---------- ---------- ----------
      1100       1200       2300       2300
```

계좌1.잔고와 계좌2.잔고를 모두 Current 모드로 읽었기 때문에 일관성 있게 갱신된 것을 알 수 있다. 이처럼 오라클에서도 일관성 없게 값을 갱신할 가능성은 존재한다.
하지만 사용자가 오라클만의 독특한 읽기 모드를 정확히 이해하고 주의 깊게 SQL을 작성한다면 피해갈 수 있는 문제다.

평소 DBMS마다 다른 Lock 메커니즘과 Consistency 모델에 대한 이해가 부족했다면 지금까지의 설명과 예시가 다소 어렵고 생소하게 느껴질 수 있다.
하지만 본 절에서 설명하고자 한 핵심 내용은 Consistent 모드와 Current 모드 읽기의 차이점을 밝히는 데에 있으므로 이에 대한 어느 정도의 이해가 생겼다면 계속 책을 읽어 내려가는 데에 전혀 지장이
없으므로 안심해도 된다.
그렇더라도 향후 고급 데이터베이스 프로그래머로 성장하려면 Lock과 이를 지원하는 내부 메커니즘, 그리고 트랜잭션이 동시에 진행되는 상황에서 발생할 수 있는 미묘한 차이점들을 이해하는 것이 필수적이므로,
앞으로 이에 대한 개인적인 고민과 연구가 계속 뒤따라야 할 것이다.