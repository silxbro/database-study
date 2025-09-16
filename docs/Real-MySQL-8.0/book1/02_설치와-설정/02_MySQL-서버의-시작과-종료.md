# [CH 2-2] MySQL 서버의 시작과 종료

이제 MySQL 서버의 설치 및 설정을 완료했으므로 MySQL 서버를 시작하고 종료하는 법, 그리고 mysql 클라이언트 프로그램을 이용해 간단한 접속 테스트를 해보자.
macOS와 윈도우에 설치된 MySQL 서버의 경우 이미 설치 과정에서 설정 파일의 경로에 대해 살펴봤으며, MySQL 서버를 시작하거나 종료하는 것은 GUI로 쉽게 제어할 수 있을 것이다.
여기서는 거의 대부분의 서비스 환경에서 사용되는 리눅스 운영체제에서 MySQL 서버의 설정 파일을 비롯해 MySQL 서버를 시작, 종료하는 방법을 살펴본다.

---
<br/>

## (1) 설정 파일 및 데이터 파일 준비
리눅스 서버에서 Yum 인스톨러나 RPM을 이용해 MySQL 서버를 설치하면 MySQL 서버에 필요한 프로그램들과 디렉터리들은 일부 준비되지만 트랜잭션 로그 파일과 시스템 테이블이 준비되지 않았기 때문에 아직
아직 MySQL 서버를 시작할 수 없다.
우선 MySQL 서버가 설치되면 /etc/`my.cnf` 설정 파일이 준비되는데, 이 설정 파일에는 MySQL 서버를 실행하는 데 꼭 필요한 3\~4개의 아주 기본적인 설정만 기록돼 있다.
실제 서비스용으로 사용하기에는 많이 부족한 상태지만 간단히 테스트용으로 MySQL 서버를 실행한다면 이 정도로도 충분히 MySQL 서버를 실행할 수는 있다.

여기서는 MySQL 서버를 끝까지 설치해보는 것이 목적이므로 RPM 패키지가 준비해 둔 MySQL 설정 파일을 그대로 이용해 설치를 진행해보겠다.
우선 다음과 같이 MySQL 서버를 실행하는 데 필요한 초기 데이터 파일(시스템 테이블이 저장되는 데이터 파일)과 트랜잭션 로그(리두 로그) 파일을 생성하자.

```
linux> mysqld --defaults-file=/etc/my.cnf --initialize-insecure
```

위와 같이 mysqld 명령에 --initialize-insecure 옵션을 사용하면, 필요한 초기 데이터 파일과 로그 파일들을 생성하고 마지막으로 비밀번호가 없는 관리자 계정인 root 유저를 생성한다.
만약 비밀번호를 가진 관리자 계정을 생성하고자 한다면 다음과 같이 --initialize 옵션을 사용하면 된다. --initialize 옵션을 사용하면 생성된 관리자 계정의 비밀번호를 에러 로그 파일로 기록한다.
에러 로그 파일의 기본 경로는 /var/log/mysqld.log 파일인데, 파일의 제일 마지막에 보면 관리자 계정인 root@localhost를 생성했으며, 비밀번호는 'DqguE(h5o>lS'라고 기록돼 있는 것을 확인할
수 있다.

```
linux> mysqld --defaults-file=/etc/my.cnf --initialize

linux> tail -n 4 /var/log/mysqld.log
2020-07-16T12:31:46.011759Z 0 [System] [MY-013169] /usr/sbin/mysqld (mysqld 8.0.21)
initializing of server in progress as process 34346
2020-07-16T12:31:46.017728Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2020-07-16T12:31:46.568716Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2020-07-16T12:31:47.493224Z 6 [Note] [MY-010454] [Server] A temporary password is generated for
root@localhost: DqguE(h5o>lS
```
<br/>

## (2) 시작과 종료
유닉스 계열 운영체제에서 RPM 패키지로 MySQL을 설치했다면 자동으로 /usr/lib/systemd/system/mysqld.service 파일이 생성되고, systemctl 유틸리티를 이용해 MySQL을 기동하거나 종료하는
것이 가능하다. 윈도우 인스톨러 버전의 MySQL을 설치했다면 설치 중 선택사항으로 윈도우의 서비스로 MySQL을 등록할 수 있다.

```
linux> systemctl start mysqld
```

시작된 MySQL 서버의 상태는 다음과 같이 더 자세히 확인할 수 있다.
```
linux> systemctl status mysqld
● mysqld.service - MySQL Server
  Loaded: loaded (/lib/systemd/system/mysqld.service; enabled; vendor ...)
  Active: active (running) since 2020-05-17 12:41:21 UTC; 1 months 26 days ago
    Docs: man:mysqld(8)
          http://dev.mysql.com/doc/refman/en/using-systemd.html
Main PID: 3976 (mysqld)
  Status: "Server is operational"
  CGroup: /system.slice/mysqld.service
              3976 /usr/sbin/mysqld
```

> 앞의 예제와 같이 MySQL 서버는 systemd를 이용해 시작하고 종료할 수도 있지만 MySQL 배포판과 함께 제공되는 mysqld_safe 스크립트를 이용해서 MySQL 서버를 시작하고 종료할 수도 있다.
> mysqld_safe 스크립트를 이용하면 MySQL 설정 파일(my.cnf)의 "[mysqld_safe]" 섹션의 설정들을 참조해서 MySQL 서버를 시작하게 되지만, 앞의 예제와 같이 systemd를 이용해서 MySQL
> 서버를 시작하면 mysqld_safe 스크립트를 사용하지 않고 MySQL 서버를 시작하고 종료하게 된다.
> 그래서 systemd를 이용하는 경우에는 MySQL 설정 파일의 "[mysqld_safe]" 섹션에만 설정 가능한 "malloc-lib" 같은 시스템 설정을 적용하고자 한다면 mysqld_safe 스크립트를 이용해 MySQL
> 서버를 시작해야 한다.
>
> 물론 systemd를 이용해 MySQL 서버를 시작하는 경우에도 메모리 할당자(Memory allocator)를 변경하고자 한다면 "LD_PRELOAD" 환경변수를 이용해서 MySQL 서버를 시작할 수도 있다.

```
linux> ps -ef | grep mysqld
mysql    3976      1  9  5월17 ?    5-08:10:40 /usr/sbin/mysqld
```

실행 중인 MySQL 서버를 종료하려면 시작과 동일하게 systemctl을 이용하되, 옵션을 stop으로 변경해서 실행하면 된다.

```
linux> systemctl stop mysqld
```

원격으로 MySQL 서버를 셧다운하려면 다음과 같이 MySQL 서버에 로그인한 상태에서 SHUTDOWN 명령을 실행하면 된다.
이렇게 원격으로 MySQL 서버를 셧다운하려면 SHUTDOWN 권한(Privileges)을 가지고 있어야 한다.

```
mysql> SHUTDOWN;
```

MySQL 서버에서는 실제 트랜잭션이 정상적으로 커밋돼도 데이터 파일에 변경된 내용이 기록되지 않고 로그 파일(리두 로그)에만 기록돼 있을 수 있다.
심지어 MySQL 서버가 종료되고 다시 시작된 이후에도 계속 이 상태로 남아있을 수도 있다. 사용량이 많은 MySQL 서버에서는 이런 현상이 더 일반적인데, 이는 결코 비정상적인 상황이 아니다.
하지만 MySQL 서버가 종료될 때 모든 커밋된 내용을 데이터 파일에 기록하고 종료하게 할 수도 있는데, 이 경우에는 다음과 같이 MySQL 서버의 옵션을 변경하고 MySQL 서버를 종료하면 된다.

```
mysql> SET GLOBAL innodb_fast_shutdown=0;
linux> systemctl stop mysqld.service

## 또는 원격으로 MySQL 서버 종료 시
mysql> SET GLOBAL innodb_fast_shutdown=0;
mysql> SHUTDOWN;
```

이렇게 모든 커밋된 데이터를 데이터 파일에 적용하고 종료하는 것을 클린 셧다운(Clean shutdown)이라고 표현한다.
클린 셧다운으로 종료되면 다시 MySQL 서버가 기동할 때 별도의 트랜잭션 복구 과정을 진행하지 않기 때문에 빠르게 시작할 수 있다.

> MySQL 서버가 시작되거나 종료될 때는 MySQL 서버(InnoDB 스토리지 엔진)의 버퍼 풀 내용을 백업하고 복구하는 과정이 내부적으로 실행된다.
> 실제 버퍼 풀의 내용을 백업하는 것이 아니라, 버퍼 풀에 적재돼 있던 데이터 파일의 데이터 페이지에 대한 메타 정보를 백업하기 때문에 용량이 크지 않으며, 백업 자체는 매우 빠르게 완료된다.
> 하지만 MySQL 서버가 새로 시작될 때는 디스크에서 데이터 파일들을 모두 읽어서 적재해야 하므로 상당한 시간이 걸릴 수도 있다.
> 혹시 MySQL 서버의 시작 시간이 오래 걸린다면 MySQL 서버가 버퍼 풀의 내용을 복구하고 있는지 확인해보는 것이 좋다.
<br/>

## (3) 서버 연결 테스트
MySQL 서버가 시작됐다면 서버에 직접 접속해보자. MySQL 서버에 접속하는 방법은 MySQL 서버 프로그램(mysqld)과 함께 설치된 MySQL 기본 클라이언트 프로그램인 mysql을 실행하면 된다.
다음과 같이 여러 가지 형태의 명령행 인자를 넣어 접속을 시도할 수 있다.

```
linux> mysql -uroot -p --host=localhost --socket=/tmp/mysql.sock
linux> mysql -uroot -p --host=127.0.0.1 --port=3306
linux> mysql -uroot -p
```

첫 번째 예제는 MySQL 소켓 파일을 이용해 접속하는 예제다. 두 번째 예제는 TCP/IP를 통해 127.0.0.1(로컬 호스트)로 접속하는 예제인데, 이 경우에는 포트를 명시하는 것이 일반적이다.
로컬 서버에 설치된 MySQL이 아니라 원격 호스트에 있는 MySQL 서버에 접속할 때는 반드시 두 번째 방법을 사용해야 한다.
MySQL 서버에 접속할 때는 호스트를 localhost로 명시하는 것과 127.0.0.1로 명시하는 것이 각각 의미가 다르다.
--host=localhost 옵션을 사용하면 MySQL 클라이언트 프로그램은 항상 소켓 파일을 통해 MySQL 서버에 접속하게 되는데, 이는 'Unix domain socket'을 이용하는 방식으로 TCP/IP를 통한 통신이
아니라 유닉스의 프로세스 간 통신(IPC; Inter Process Communication)의 일종이다.
하지만 127.0.0.1을 사용하는 경우에는 자기 서버를 가리키는 루프백(loopback) IP이기는 하지만 TCP/IP 통신 방식을 사용하는 것이다.

세 번째 방식은 별도로 호스트 주소와 포트를 명사하지 않는다.
이 경우에는 기본값으로 호스트는 localhost가 되며 소켓 파일을 사용하게 되는데, 소켓 파일의 위치는 MySQL 서버의 설정 파일에서 읽어서 사용한다.
MySQL 서버가 기동될 때 만들어지는 유닉스 소켓 파일은 MySQL 서버를 재시작하지 않으면 다시 만들어 낼 수 없기 때문에 실수로 삭제하지 않도록 주의한다.
유닉스나 리눅스에서 mysql 클라이언트 프로그램을 실행하는 경우에는 mysql 프로그램의 경로를 PATH 환경변수에 등록해 둔다.

MySQL 서버에 접속했다면 SHOW DATABASES 명령으로 데이터베이스의 목록을 확인할 수 있다.
처음 설치된 MySQL 서버에는 root라는 관리자 계정이 준비돼 있으며, --initialize-insecure 옵션으로 MySQL 서버가 초기화됐다면 비밀번호 없이 로그인할 수 있다.
만약 --initialize 옵션으로 MySQL 서버가 초기화됐다면 MySQL 서버의 로그 파일에 기록돼 있는 비밀번호를 이용해서 로그인하면 된다.

```
linux> mysql -h127.0.0.1 -uroot -p
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 44
Server version: 8.0.43-0ubuntu0.24.04.1 (Ubuntu)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)
```

MySQL 서버에 접속되면 위 예제의 마지막 줄과 같이 MySQL 프롬프트가 표시된다.
MySQL 프롬프트는 설정에 따라 표시 내용이 조금 다를 수 있는데, 아무런 프롬프트 설정이 없다면 예제에서와 같이 간단히 'mysql>'로 표시된다.
MySQL 서버에 로그인되면 위 예제와 같이 SHOW DATABASES 명령을 실행해 기본 생성된 데이터베이스의 목록을 확인할 수 있다.

때로는 MySQL 서버를 직접 로그인하지 않고, 원격 서버에서 MySQL 서버의 접속 가능 여부만 확인해야 하는 경우도 있다.
이처럼 커넥션이 가능한지만 확인하는 경우에는 MySQL 클라이언트를 설치하는 작업이 번거로울 수 있고, 때로는 보안상 이유로 MySQL 클라이언트 프로그램을 설치하지 못할 수도 있다.
네트워크 연결이 정상적인지 확인하는 경우에도 이러한 연결 테스트가 필요할 수 있다.
이 경우에는 간단히 telnet 명령이나 nc(Netcat) 명령을 이용해 원격지 MySQL 서버가 응답 가능한 상태인지 확인해볼 수 있다.

#### [Telnet 프로그램으로 확인하는 방법]
```
linux> telnet 10.2.40.61 3306
Trying 10.2.40.61...
Connected to prod1-db-mysqltest.bhero.io.
Escape character is '^]'.
S
8.0.19-log
...
```

#### [Netcat 프로그램으로 확인하는 방법]
```
linux> nc 10.2.40.61 3306
S
8.0.19-log...
```

Telnet과 Netcat 프로그램 모두 MySQL 서버로 접속해서 MySQL 서버가 보내준 메시지를 화면에 출력하는 것을 살펴볼 수 있다.
물론 글자가 깨져 정확한 내용을 알 수는 없지만 Telnet과 Netcat 프로그램이 출력한 내용에서 MySQL 서버의 버전 정보를 정상적으로 보여주고 있음을 알 수 있다.
이렇게 서버가 보내준 메시지를 출력한다면 네트워크 수준의 연결은 정상적임을 판단할 수 있다.
만약 Telnet이나 Netcat 프로그램이 서버의 버전 정보를 정상적으로 출력하는 상태에서도 응용 프로그램이 MySQL 서버에 접속하지 못한다면 이는 MySQL 서버의 계정 비밀번호가 일치하지 않거나 MySQL
서버 계정의 host 부분이 허용되지 않은 경우일 가능성이 높다.