# 07. Consistent vs. Current 모드 읽기

지금까지 설명한 내용을 바탕으로 Consistent 모드 읽기와 Current 모드 읽기의 차이점을 살펴보자.

<br/>

## (1) Consistent 모드 읽기와 Current 모드 읽기의 차이점
#### ✅ Consistent 모드 읽기(gets in consistent mode)는, SCN 확인 과정을 거치며 쿼리가 시작된 시점을 기준으로 일관성 있는 상태로 블록을 액세스하는 것을 말한다
- 이 모드로 데이터를 읽을 때는 항상 쿼리가 시작된 시점의 데이터를 가져온다.
- SQL 트레이스 Call 통계에서 볼 수 있는 'query' 항목과 AutoTrace에서의 'consistent gets' 항목이 지금 설명한 Consistent 모드에서 읽은 블록 수를 의미한다.
    - 읽는 중에 CR copy를 생성할 필요가 없어 Current 블럭을 읽더라도 Consistent 모드에서 읽었다면 'query' 항목에 집계된다.
    - select문에서 읽은 블록은 대부분 여기에 해당하며, 여기에는 CR 블록을 생성하려고 Undo 세그먼트로부터 읽어들이는 블록 수까지 더해진다.
#### ✅ Current 모드 읽기(gets in current mode)는, SQL문이 시작된 시점이 아니라 데이터를 찾아간 바로 그 시점의 최종 값을 읽으려고 블록을 액세스하는 것을 말한다
- 블록 SCN이 쿼리 SCN보다 높고 낮음을 따지지 않으며, 그 시점에 이미 커밋된 값이라면 그대로 받아들이고 읽는다.
- SQL 트레이스에서 Call 통계 레포트를 통해 볼 수 있는 'current' 항목과 AutoTrace에서의 'db block gets' 항목이 Current 모드에서 읽은 블록 수를 의미하며, 주로 다음과 같은 상황에서 나타난다.
    - DML 문을 수행할 때 주로 나타난다.
    - select for update문을 수행할 때도 Current 모드 읽기를 발견할 수 있다.
    - 8i 이전 버전에서는 Full 스캔을 포함하는 select 문에서도 Current 모드 읽기가 나타났는데, Full Scan 할 익스텐트 맵(Extent Map) 정보를 읽으려고 세그먼트 헤더에 접근할 때
      익스텐트에 대한 바로 현재 시점의 정보가 필요하기 때문이다.
      Locally Managed 테이블스페이스를 주로 사용하기 시작한 9i 이상부터는 Full Table Scan을 하더라도 Current 모드 읽기가 발생하지 않는다.
        - Index rowid를 이용한 테이블 액세스 시에는 테이블 익스텐트 정보가 불필요하므로 버전에 상관없이 Current 모드 읽기가 발생하지 않는다.
    - 디스크 소트가 필요할 정도로 대량의 데이터를 정렬할 때도 Current 모드 읽기가 나타난다.

<br/>

## (2) Consistent 모드로 갱신할 때 생기는 현상
EMP 테이블 7788 사원의 SAL 값이 현재 1,000인 상황에서 아래 TX1, TX2 두 개의 트랜잭션이 동시에 수행되었다. 양쪽 트랜잭션이 모두 완료된 시점에 7788 사원의 SAL 값은 얼마이어야 할까?

#### <상황 1>
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = sal + 100<br/>where empno = 7788;<br/><br/><br/>commit;<br/>|<br/>t1<br/><br/>t2<br/><br/>t3<br/>t4|<br/><br/><br/>update emp set sal = sal + 200<br/>where empno = 7788;<br/><br/>commit;|

물론 TX2 update는 t2 시점에 시작하지만 TX1에 의해 걸린 Lock을 대기하다가 t3 시점에 TX1이 커밋된 후에 진행을 계속한다.
아주 쉬운 문제처럼 보이지만 여러 사람에게 물어보면 확신하면서 선뜻 정답을 맞히는 경우는 흔치 않다. 그 이유를 곰곰이 생각해 보면 오라클이 특이하게도 두 가지 읽기 모드를 제공하는 데에서 비롯된다.
Consistent 읽기 모드 원리에 대해 세세한 내부 원리를 잘 모르더라도 오라클 매뉴얼에 간단히 언급된 기본적인 내용, 즉 쿼리 시작 시점을 기준으로 값을 읽는다는 사실은 대부분 알고 있다.
그러다 보니 오라클 사용자들은 항상 Consisent 모드 읽기 중심으로 생각한다.

만약 위에서 예시한 두 개의 update문이 Consistent 모드로 값을 읽고 갱신한다면 어떤 일이 발생할까?
Dirty Read를 허용하지 않는 한 t1과 t2 시점에 SAL 값은 1,000이었으므로 둘 다 1,000을 읽고 각각 100과 200을 더해 갱신을 완료한다.
최종 값은 1,200이 될 것이며, 트랜잭션 TX1의 처리 결과는 사라졌으므로 소위 말하는 Lost Update가 발생하는 결과를 초래한다.

이런 Lost Update 문제를 회피하려면 갱신 작업만큼은 Current 모드를 사용해야 한다.
정리하면, TX2 update는 Exclusive Lock 때문에 대기했다가 TX1 트랜잭션이 커밋한 후 Current 모드로 그 값을 읽어 진행을 계속한다. 그럼으로써 Lost Update 문제를 피할 수 있다.

<br/>

## (3) Current 모드로 갱신할 때 생기는 현상
Current 모드로 갱신해야 Lost Update를 회피할 수 있다고 했는데, Current 모드로만 처리했을 때 아래 두 트랜잭션이 모두 완료되고 나서 7788번 사원의 SAL 값은 얼마일지 예측해 보기 바란다.
t1 직전 시점에 7788 사원의 SAL 값이 1,000이다. (오라클에서의 실제 처리 결과가 아니라 Current 모드로 처리했을 때의 결과를 묻는 것이다.)

#### <상황 2>
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = 2000<br/>where empno = 7788<br/>and sal = 1000;<br/><br/><br/><br/>commit;<br/><br/><br/>|t1<br/><br/><br/>t2<br/><br/><br/>t3<br/><br/>t4|<br/><br/><br/>update emp set sal = 3000<br/>where empno = 7788<br/>and sal = 2000;<br/><br/><br/>commit;|

Current 모드로 처리한다면, TX2 트랜잭션은 TX1 트랜잭션이 커밋되기를 기다렸다가 SAL 값이 2,000으로 갱신되는 것을 확인하고 정상적으로 update를 수행한다.
따라서 최종 값은 3,000이 된다.

항상 Current 모드로만 작동하는 Sybase, SQL Server와 같은 DBMS에서 수행해 보면 실제 위와 같은 결과가 나온다. 이렇게 처리해도 문제가 없는 것일까? 이에 대한 판단에 앞서 한가지 경우를 더 살펴보자.

#### <상황 3>
|TX1||TX2|
|:---|:---:|:---|
|update t set no = no + 1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br/>where no > 50000 ;<br/><br/><br/><br/><br/><br/>commit;|t1<br/><br/><br/>t2<br/><br/>t3<br/><br/>t4|<br/><br/>insert into t values(100001, 100001);<br/><br/>commit;<br/><br/>|

TX1이 1 ~ 100,000까지의 Unique한 번호(no)를 가진 테이블에서 no > 50000 조건에 헤당하는 50,000개 레코드에 대해 인덱스를 경유해 순차적으로 갱신 작업을 진행하고 있다고 하자.
그런데 도중에 TX2 트랜잭션에서 no 값이 100,001인 레코드를 새로 추가하면 update되는 최종 결과건수는 50,000 이어야 할까? 아니면 50,001 이어야 할까?

실제 SQL Server에서 테스트 해 보면, 50,001건이 갱신된다. 즉, update가 진행되는 동안 새로 추가된 레코드까지도 값이 변경된 것이다.
만약, 인덱스를 이용해 no 값을 순차적으로 읽지 않고 Full Table Scan 방식으로 update를 진행한다면 insert 되는 100,001번째 레코드가 삽입되는 위치에 따라 update 결과 건수가 그때그때 달라진다.
즉, update가 이미 진행된 블록(=페이지)에 insert된 레코드는 갱신 대상에서 제외되고, 아직 update 되지 않은 블록(=페이지)에 insert된 레코드는 갱신 대상에 포함한다.

같은 이치로, 100,001번째 레코드를 insert하는 대신 TX2가 아래 update 문을 수행하면 변경된 레코드가 TX1의 update 대상 건에서 제외되므로 최종적으로 49,999건만 갱신된다.

```
TX2> update t set no = 0 where no = 100000 ;
TX2> commit;
```
또 다른 예로서, "delete from 로그" 문장이 수행되는 도중에 다른 트랜잭션에 의해 새로 추가된 로그 데이터까지 지워질 수도 있는데, update/delete 도중에 갱신/삭제 대상이 그때그때 변할 수 있다는 것은
오라클 사용자 입장에서는 참 놀라운 사실이 아닐 수 없다.

<br/>

## (4) Consistent 모드로 읽고, Current 모드로 갱신할 때 생기는 현상
Current 모드로 갱신을 수행할 때 어떤 현상이 생기는지 살펴보았다. 이 문제를 피하려고 오라클은 Consistent 모드로 읽고, Current 모드로 갱신한다.
오라클을 설명하는 자료에서 다음과 같은 내용을 흔히 접할 수 있다.
#### <설명 1>
```
오라클에서 update 문을 수행하면, 대상 레코드를 읽을 때는 Consistent 모드로 읽고 실제 값을 변경할 때는 Current 모드로 읽는다.
따라서 대상 레코드를 읽기 위한 블록 액세스는 SQL 트레이스에서 query 항목으로 계산되고, 값을 변경하기 위한 블록 액세스는 current 항목에 계산된다.
```

하지만 이 의미를 단순하게 이해한다면 또 다른 오류에 빠질 수 있다. 위 내용을 읽은 후 아래와 같은 상황에서 어떤 결과가 나올지 예측해 보기 바란다.

#### <상황 4>
|TX1||TX2|
|:---|:---:|:---|
|update emp set sal = sal + 100<br/>where empno = 7788<br/>and sal = 1000 ;<br/><br/><br/><br/>commit;<br/><br/><br/>|t1<br/><br/><br/>t2<br/><br/><br/>t3<br/><br/>t4|<br/><br/><br/>update emp set sal = sal + 200<br/>where empno = 7788<br/>and sal = 1000 ;<br/><br/><br/>commit;|

TX2는 TX1이 커밋되기를 기다렸다가 TX1이 끝나면 계속 진행한다. 하지만 이 때 7788 사원의 SAL 값은 이미 1,100으로 바뀐 상태이므로 TX2의 update는 실패하게 된다.

오라클 외 다른 DBMS는 항상 Current 모드 읽기만 지원하기 때문에 헷갈릴 이유가 없으며 너무 자연스럽게 이해가 된다.
하지만 <설명 1>을 접했다면 오라클에서 위 update가 실패하는 이유를 쉽게 이해하지 못할 수 있다.
TX2가 실제 값을 갱신할 때는 이미 1,100으로 바뀐 값을 읽겠지만 갱신 대상 레코드를 찾아갈 때는 Consistent 모드를 사용하기 때문이다.
즉, "empno = 7788이고 sal = 1000"인 레코드를 TX1이 변경하기 이전 값을 이용해 찾아가기 때문에 update를 실패할 이유가 없다고 생각한다.

아래 [1]번 update문을 [2]번 문장처럼 바꿔서 표현할 수 있는데, <설명 1>을 문장 그대로 받아들인다면 [2]번 update문에서 굵은 글씨체로 표시된 부분은 Consistent 모드로 읽고,
나머지 부분(→ sal = sal + 200에서 sal 값)은 Current 모드로 읽는다는 믿음을 갖게 된다.
```
[1] update emp set sal = sal + 200
    where empno = 7788
    and sal = 1000 ;

[2] update
    (
        select sal from emp
        where empno = 7788
        and sal = 1000
    )
    set sal = sal + 200 ;
```

실제 그런 식으로 처리한다면 <상황 4>의 TX2 update문은 'sal = 1000'인 레코드를 갱신하는 문장인데도, 이미 1,100으로 바뀐 레코드를 갱신하는 결과를 초래하게 된다.
하지만 다른 DBMS와 마찬가지로 오라클에서도 TX2의 갱신은 실패한다(갱신 레코드 건수 = 0).

<br/>

## (5) Consistent 모드로 갱신대상을 식별하고, Current 모드로 갱신
그럼 실제 오라클은 어떻게 두 개의 읽기 모드가 공존하면서 update를 처리하는 것일까? 이해를 돕기 위해 pseudo 코드로 표현해 보면 다음과 같다.
```
for c in
  (  ---> Consistent
      select rowid rid, empno, sal from emp
      where empno = 7788 and sal = 1000
  )
loop ---> Current
      update emp set sal = sal + 200
      where empno = c.empno
      and sal = c.sal
      and rowid = c.rid;
end loop;
```
위 pseudo 코드가 앞에서 인라인 뷰를 사용한 [2]번 update 문과 다른 점은 무엇일까? Consistent 모드에서 수행한 조건 체크를, Current 모드로 액세스하는 시점에 한 번 더 수행한다는 점이다.

pseudo 코드 내용을 두 단계로 나누어 설명해 보자.
```
단계1) where절에 기술된 조건에 따라 수정/삭제할 대상 레코등의 rowid를 Consistent 모드로 찾는다(→ DML 문이 시작된 시점 기준).
단계2) 앞에서 읽은 rowid가 가리키는 레코드를 찾아가 로우 Lock을 설정한 후에 Current 모드로 실제 update/delte 를 수행한다(→ 값이 변경되는 시점 기준).
      이 단계에서 Current 모드로 다시 한번 조건을 체크하고, 갱신할 값을 읽어 수정/삭제한다.
```
단계1을 수행해 update/delete 대상 건을 '모두' 추출하고 나서 단계2를 수행한다고 생각하면 안 된다.
위 pseudo 코드에서 보이고 있듯이 단계1에서 커서를 열어 Fetch하면서 단계2를 건건이 반복 수행한다.

단계1은 update/delete가 시작된 시점 기준으로 수정/삭제할 대상을 식별하려고 Consistent 모드 읽기를 사용할 뿐이며 거기서 읽은 값을 단계2에서 갱신하는 데 사용하지는 않는다.
단계1이 필요한 이유는, 갱신이 진행되는 동안 추가되거나 변경을 통해 범위 안에 새로 들어오는 레코드를 제외하고자 하는 것이다.
이미 범위 안에 포함돼 있던 레코드는, 단게2에서 변경이 이루어지는 바로 그 시점 기준으로 값을 읽고 갱신한다.
이때는 블록 SCN이 쿼리 SCN보다 높고 낮음을 따지지 않으며, 그 시점에 이미 커밋된 값이라면 그대로 받아들이고 읽는다.
이 때문에(update문이 시작된 이후에 1,000에서 1,100으로 값이 변경되었기 때문에) <상황 4>에서 TX2의 update가 실패하는 것이다.

짧게 요약하면, 설명2와 같다.
#### <설명 2>
```
1. select는 Consistent 모드로 읽는다.
2. insert, update, delete, merge는 Current 모드로 읽고 쓴다. 다만, 갱신할 대상 레코드를 식별하는 작업만큼은 Consistent 모드로 이루어진다.
```
이 두 가지 사실만 기억하면 그다지 어렵지 않게 동시 트랜잭션의 최종 결과를 예측할 수 있다. 정리하는 차원에서 앞서 본 <상황 2>와 <상황 3>을 다시 살펴보기 바란다.
처음엔 자신 없었을지 몰라도, 지금 보면 쉽게 처리 결과를 예측할 수 있을 것이다.

#### Write Consistency
앞선 사례(상황 4)에서, Consistent 모드와 Current 모드에서 읽은 값이 서로 달라 TX2의 upate는 실패했다.
TX2가 update를 시작한 t2 시점 기준으로는 empno = 7788 사원 레코드가 분명히 갱신 대상이었는데, 값이 달라졌다고 아무런 처리 없이 지나가도 상관없는 걸까? 그렇지 않다.
데이터 정합성에 문제가 생기는 경우가 있고, 이를 방지하려고 오라클은 'Restart 메커니즘'을 사용한다.
그때까지의 갱신을 롤백하고 update를 처음부터 다시 실행하는 것이며, Thomas Kyte는 그의 저서에서 이것을 'Write Consistency'라고 명명하고 있다.

이 기능은 데이터베이스 일관성을 유지하려고 오라클이 사용하는 아주 내부적인 메커니즘이므로 중요하게 다룰 내용은 아니다.
Consistent 모드로 찾은 레코드를 Current 모드로 읽어 갱신한다는 기본 컨셉만 이해해도 충분하다. 갱신 대상 레코드의 값이 중간에 바뀌었다고 항상 Restart 메커니즘이 작동하는 것도 아니다.
where 절에 사용된 컬럼 값이 바뀌었을 때만 작동한다.

그리고 Restart 메커니즘이 작동하더라도 대개는 처리 결과가 달라지지 않는다. 위 사례(상황 4)만 보더라도 그렇다.
데이터베이스 일관성에 문제가 생기는 사례를 찾기가 오히려 어렵다. 



















