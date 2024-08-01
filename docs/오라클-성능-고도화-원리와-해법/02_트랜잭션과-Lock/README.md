# 02. 트랜잭션과 Lock

1장에서 아키텍처를 설명하면서 오라클만의 독특한 읽기 일관성 모델을 비교적 자세히 설명하였다.
DB2, SQL Server, Sybase 등은 Lock을 통해 읽기 일관성을 구현하지만, 오라클은 Undo 데이터를 이용해 읽기 일관성을 구현한다는 게 핵심이다.
즉, Undo에 저장된 정보를 이용해 쿼리가 시작된 시점을 기준으로 일관성 있는 결과집합을 생성해 낸다.

오라클은 데이터를 읽을 때 Lock을 사용하지 않다 보니 확실히 동시성 측면에서는 다른 DBMS보다 우월하다고 평가할 수 있다.
SQL Server도 최근 버전에서 Snapshot 모드로 설정하면 오라클과 비슷한 읽기 일관성 모델로 데이터베이스를 운영할 수 있는 기능을 지원하기 시작했는데, 동시성과 읽기 일관성 측면에서 오라클이 우월한
모델임을 반증한 것이 아닌가 싶다.

그래서일까? 오라클 환경의 개발자들은 다른 DBMS를 사용하는 개발자들에 비해 Lock과 트랜잭션 동시성 제어(Concurrency Control)에 대한 개념이 상당히 부족하다는 것을 현장에서 많이 느낀다.
다른 DBMS를 다루는 튜닝 관련 서적과 교육강좌들을 보면 Lock과 트랜잭션 동시성 제어를 항상 첫 번째 주제로 다루는 데 반해 오라클 측에서는 아예 다루지 않거나 짧게 소개하는 데에 그친다.

오라클이 동시성 측면에서 우월하다지만 DBA들은 Lock 때문에 긴장을 늦추지 못한다. 특히, 새로 개발된 시스템을 가동하고 안정화되는 시기까지는 Lock과의 전쟁을 벌이는 일이 다반사다.
오라클도 Lock으로부터 자유롭지 못하다는 얘기다.

오라클은 10g부터 대기 이벤트(Wait Event)를 아래와 같이 분류해 놓았다.
```
SQL>  select wait_class, count(*)
  2   from   v$event_name
  3   gropu by wait_class
  4   order by 1 ;

WAIT_CLASS                       COUNT(*)
-----------------------------  ----------
Administrative                        46
Application                           12
Cluster                               47
Commit                                 1
Concurrency                           24
Configuration                         23
Idle                                  62
Network                               26
Other                                588
Scheduler                              2
System I/O                            24
User I/O                              17

12 개의 행이 선택되었습니다.
```
주목할 것은, 오라클에서 발생하는 Lock 경합의 대부분을 차지하는 enq: TM - contention 이벤트와 enq: TX - row lock contention 이벤트가 Concurrency가 아닌 Application으로
분류돼 있다는 사실이다.

```
SQL>  select event#, name, wait_class from v$event_name
  2   where name in (  'enq: TM - contention'
  3                  , 'enq: TX - row lock contention'  )  ;

    EVENT#  NAME                          WAIT_CLASS
----------  ----------------------------- ------------------
       175  enq: TM - contention          Application
       183  enq: TX - row lock contention Application

2개의 행이 선택되었습니다.
```
- enq: TM - connnection : DML 테이블 Lock 경합 시 발생한다.
- enq: TX - row lock contention : DML 로우 Lock 경합 시 발생한다.

같이 Application으로 분류된 대기 이벤트 중 SQL*Net break/reset to client 가 있다.
이 이벤트는 사용자가 수행한 SQL문이, 존재하지 않는 테이블을 참조하거나 사용자 정의 함수/프로시저에서 처리(catch)하지 않은 예외상황(Exception)을 만났을 때 나타난다.
테이블 Lock과 로우 Lock 관련 이벤트를 이런 유형의 프로그램 오류와 같이 분류한 것은 이들 문제가 DBA 이슈가 아니라 개발자 이슈임을 분명히 밝히고 있는 것이다.

Lock이 해제되지 않는 상황이 지속될 때 DBA가 할 수 있는 일은, Lock을 소유한 세션을 찾아 프로세스를 강제로 중지시키는 일뿐이다. 근본적인 해법은 애플리케이션 로직에서 찾아야 한다.

Lock은 각 데이터베이스의 특징을 결정짓는 가장 핵심적인 원리다. 따라서 사용하는 데이터베이스만의 독특한 Lock 메커니즘을 이해하지 못한다면 고품질・고성능의 애플리케이션을 구축하기 어렵다.
트랜잭션과 동시성 제어에 대한 개념 또한 데이터베이스 프로그래머의 필수 과목이다.

본 장은 성능을 논하기에 앞서, 기본적인 트랜잭션 개념을 상기시키고, 개발자 및 설계자가 반드시 알아야 할 동시성 제어 기법들을 설명한다. 그와 함께 일관성과 동시성을 같이 높이는 방안들을 제시하려고 한다.
마지막으로, 개발자・DBA가 반드시 알아야 할 오라클만의 독특한 Lock 메커니즘에 대해서도 비교적 자세히 설명한다.