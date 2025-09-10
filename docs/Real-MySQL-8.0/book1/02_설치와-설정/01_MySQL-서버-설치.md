# [CH 2-1] MySQL 서버 설치

MySQL 서버는 다음과 같이 다양한 형태로 설치할 수 있지만 가능하다면 리눅스의 RPM이나 운영체제별 인스톨러를 이용하기를 권장한다.

- Tar 또는 Zip으로 압축된 버전
- 리눅스 RPM 설치 버전(윈도우 인스톨러 및 macOS 설치 패키지)
- 소스코드 빌드

이 중 인스톨러를 이용한 MySQL 서버 설치 및 디렉터리 준비, 그리고 MySQL 서버의 시작과 종료에 대해 살펴보자.

---
<br/>

## (1) 버전과 에디션(엔터프라이즈와 커뮤니티) 선택
MySQL 서버의 버전을 선택할 때는 다른 제약 사항(기존 솔루션이 특정 버전만 지원하는 경우)이 없다면 가능한 한 최신 버전을 설치하는 것이 좋다.
기존 버전에서 새로운 메이저 버전(MySQL 5.1, 5.5, 5.6, 5.7, 8.0)으로 업그레이드하는 경우라면 최소 패치 버전이 15\~20번 이상 릴리즈된 버전을 선택하는 것이 안정적인 서비스에 도움이 될 것이다.
즉, MySQL 8.0 버전이라면 MySQL 8.0.15부터 8.0.20 사이의 버전부터 시작하는 것을 권장한다.
그리고 새로운 서비스라면 서비스 개발과 동시에 데이터베이스 서버를 함께 테스트해 나갈 수 있기 때문에 조금 더 빠른 패치 버전부터 시작해도 괜찮을 듯하다.
하지만 갓 출시된 메이저 버전을 선택하는 것은 조금 위험할 수 있다.
메이저 버전은 많은 변화를 거친 버전이므로 갓 출시된 상태에서는 치명적이거나 보완하는 데 많은 시간이 걸릴 만한 버그가 발생할 수도 있기 때문이다.

초기 버전의 MySQL 서버는 엔터프라이즈 에디션과 커뮤니티 에디션으로 나뉘어 있기는 했지만 실제 MySQL 서버의 기능에 차이가 있었던 것이 아니라 기술 지원의 차이만 있었다.
하지만 MySQL 5.5 버전부터는 커뮤니티와 엔터프라이즈 에디션의 기능이 달라지면서 소스코드도 달라졌고, MySQL 엔터프라이즈 에디션의 소스코드는 더이상 공개되지 않는다.

하지만 MySQL 서버의 상용화 전략의 핵심 내용은 엔터프라이즈 에디션과 커뮤니티 에디션 모두 동일하며, 특정 부가 기능들만 상용 버전인 엔터프라이즈 에디션에 포함되는 방식이다.
이런 상용화 방식을 오픈 코어 모델(Open Core Model)이라고 한다.
즉, MySQL 엔터프라이즈 에디션과 커뮤니티 에디션의 핵심 기능은 거의 차이가 없으며, 다음과 같은 부가적인 기능과 서비스들은 엔터프라이즈 에디션에서만 지원된다.
- Thread Pool
- Enterprise Audit
- Enterprise TDE(Master Key 관리)
- Enterprise Authentication
- Enterprise Firewall
- Enterprise Monitor
- Enterprise Backup
- MySQL 기술 지원

지금까지의 경험상 Percona에서 출시하는 Percona Server 백업 및 모니터링 도구 또는 Percona Server에서 지원하는 플러그인(Thread Pool과 Audit 플러그인 등)을 활용하면 MySQL 커뮤니티
에디션의 부족한 부분을 메꿀 수 있었기 때문에 MySQL 엔터프라이즈 에디션의 필요성은 그다지 크지 않았다.
물론 기술 지원은 별개의 문제인데, MySQL 엔터프라이즈 에디션과 커뮤니티 에디션의 기본 성능이 다르다거나 한 것은 아니므로 엔터프라이즈 에디션에서 지원하는 것들이 꼭 필요한지 검토해 보는 것이 좋다.
<br/>
<br/>
## (2) MySQL 설치
MySQL 서버의 다양한 설치 방법 중에서 운영체제별로 설치 프로그램을 이용하는 방법 위주로 살펴보자.

### [1] 리눅스 서버의 Yum 인스톨러 설치
Yum 인스톨러를 이용하려면 MySQL 소프트웨어 리포지토리(Repository)를 등록해야 하는데, 이를 위해서는 [MySQL 다운로드 페이지](https://dev.mysql.com/downloads/repo/yum/)에서
RPM 설치 파일을 직접 받아서 설치해야 한다.

#### [그림 2.1] Yum 리포지토리 설치용 RPM 다운로드
<img src="https://github.com/user-attachments/assets/a949e27a-b385-421e-9cd5-7c44fa3f0322" width="600"/><br/>

각 운영체제의 버전에 맞는 RPM 파일을 다운로드해서 MySQL 서버를 설치하고자 하는 리눅스 서버에서 다음과 같이 Yum 리포지토리 정보를 등록한다.

```
linux> sudo rmp -Uvh mysql80-community-release-el7-3.noarch.rpm
준비 중...                           ################################# [100%]
Updating / installing...
   1:mysql80-community-release-el7-3  ################################# [100%]
```

Yum 리포지토리가 등록되면 다음과 같이 MySQL 설치용 RPM 파일들이 저장된 경로를 가진 파일이 생성된 것을 확인할 수 있다.
```
linux> ls -alh /etc/yum.repos.d/*mysql*
-rw-r--r-- 1 root root 2.1K  4월 24  2019 /etc/yum.repos.d/mysql-community-source.repo
-rw-r--r-- 1 root root 2.1K  4월 24  2019 /etc/yum.repos.d/mysql-community.repo
```

이제 다음과 같이 Yum 인스톨러 명령을 이용해 버전별로 설치 가능한 MySQL 소프트웨어 목록을 확인할 수 있다.
```
linux> sudo yum search mysql-community
...
============== N/S matched: mysql-community ==============
mysql-community-client.i686 : MySQL database client applications and tools
mysql-community-client.x86_64 : MySQL database client applications and tools
mysql-community-common.i686 : MySQL database common files for server and client libs
mysql-community-common.x86_64 : MySQL database common files for server and client libs
mysql-community-libs.i686 : Shared libraries for MySQL database client applications
mysql-community-libs.x86_64 : Shared libraries for MySQL database client applications
mysql-community-release.noarch : MySQL repository configuration for yum
mysql-community-server.x86_64 : A very fast and reliable SQL database server
mysql-community-test.x86_64 : Test suite for the MySQL database server
...

linux> sudo yum --showduplicates list mysql-community-server
...
Available Packages
mysql-community-server.x86_64  8.0.11-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.12-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.13-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.14-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.15-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.16-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.16-2.el7  mysql80-community
mysql-community-server.x86_64  8.0.17-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.18-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.19-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.20-1.el7  mysql80-community
mysql-community-server.x86_64  8.0.21-1.el7  mysql80-community
```

> yum 명령어 앞에 사용된 sudo 명령은 yum 명령을 root 권한으로 실행하게 해준다.
> MySQL 서버를 설치하는 과정에서 리눅스 서버의 관리자만 접근할 수 있는 디렉터리에 파일들을 복사하기 때문에 반드시 root 권한이 필요하다.
> 그래서 만약 현재 사용자가 root가 아니라면 sudo 명령을 yum 명령과 함께 사용해야 한다.

yum search 명령의 결과로 어떤 RPM 패키지가 있는지 확인할 수 있으며, yum --showduplicates list 명령으로 설치 가능한 모든 버전을 확인할 수 있다.

이제 MySQL 8.0의 마지막 버전인 8.0.20 버전을 설치해보자.
```
linux> sudo yum install mysql-community-server-8.0.21
...
Dependencies Resolved
=========================================================================
Package                    Arch    Version       Repository         Size
=========================================================================
Installing:
 mysql-community-server    x86_64  8.0.21-1.el7  mysql80-community  499 M
Installing for dependencies:
 mysql-community-client    x86_64  8.0.21-1.el7  mysql80-community   48 M
 mysql-community-common    x86_64  8.0.21-1.el7  mysql80-community  617 K
 mysql-community-libs      x86_64  8.0.21-1.el7  mysql80-community  4.5 M

Transaction Summary
=========================================================================
Install  1 Package (+3 Dependent packages)

Total download size: 551 M
Installed size: 2.5 G
Is this ok [y/d/N]:
```

Is this ok [y/d/N]: 프롬프트에 'y'를 입력하면 나머지 설치가 모두 진행된다. Yum 인스톨러를 이용하는 경우 설치하고자 하는 버전을 패키지 이름 뒤에 '-'으로 구분해서 입력하면 된다.

리눅스 서버에서는 Yum 인스톨러나 RPM 설치를 하더라도 MySQL 서버를 바로 시작할 수 있는 준비가 되지는 않는다.
주로 서비스용 MySQL 서버는 리눅스 서버에서 많이 사용하므로 리눅스 서버에서의 설정 파일과 시스템 테이블을 준비해야 한다.

### [2] 리눅스 서버에서 Yum 인스톨러 없이 RPM 파일로 설치
Yum 인스톨러를 사용하지 않고 RPM 패키지로 직접 설치하려면 설치에 필요한 RPM 패키지 파일들을 직접 다운로드해야 한다.
[MySQL RPM 다운로드 페이지](https://dev.mysql.com/downloads/mysql/)에서 운영체제의 버전과 CPU 아키텍처를 선택한 후, RPM 패키지 파일을 다운로드하면 된다.
최신 버전이 아닌 이전 버전을 다운로드하고 싶다면 'Looking for previous GA versions?' 링크를 클릭하면 원하는 버전을 선택할 수 있다.

#### [그림 2.2] MySQL RPM 패키지 다운로드
<img src="https://github.com/user-attachments/assets/30673c13-1dc5-4ccc-95b6-7ca2b1118951" width="570"/><br/>

RPM 패키지 다운로드 페이지에서 다음의 RPM 패키지 파일들을 다운로드한 후 의존 관계 순서대로(표 2.1의 순서가 의존 관계 순서이므로 이 순서대로 설치) 설치하면 된다.
Development Libraries 패키지는 C/C++ 언어로 MySQL 서버에 접속하는 프로그램을 개발하고 빌드할 때 필요한 파일들을 담고 있으므로 굳이 필요치 않다면 설치하지 않아도 된다.

#### [표 2.1] RPM 패키지 목록
<img src="https://github.com/user-attachments/assets/540b63fb-875c-40cc-8465-dff233cb671d" width="600"/><br/>

```
linux> rpm -Uvh mysql-community-devel-8.0.21-1.el7.x86_64.rpm
linux> rpm -Uvh mysql-community-libs-8.0.21-1.el7.x86_64.rpm
linux> rpm -Uvh mysql-community-libs-compat-8.0.21-1.el7.x86_64.rpm
linux> rpm -Uvh mysql-community-common-8.0.21-1.el7.x86_64.rpm
linux> rpm -Uvh mysql-community-server-8.0.21-1.el7.x86_64.rpm
linux> rpm -Uvh mysql-community-client-8.0.21-1.el7.x86_64.rpm
```

### [3] macOS용 DMG 패키지 설치
macOS에서 인스톨러로 설치하려면 설치에 필요한 DMG 패키지 파일들을 직접 다운로드해야 한다.
[MySQL 다운로드 페이지](https://dev.mysql.com/downloads/mysql/)에서 운영체제의 버전을 선택한 후, DMG 패키지 파일을 다운로드하면 된다.
최신 버전이 아닌 이전 버전을 다운로드하고 싶다면 'Looking for previous GA versions?' 링크를 클릭하면 원하는 버전을 선택할 수 있다.

#### [그림 2.3] MySQL DMG 패키지 다운로드
<img src="https://github.com/user-attachments/assets/91439e16-5730-4bc4-9709-68c035e6a365" width="600"/><br/>

다운로드한 DMG 파일을 실행하면 패키지 실행 화면이 나타나고, 패키지 파일을 더블클릭해서 설치를 진행하면 아래와 같이 설치 옵션을 변경하는 화면이 나타난다.

#### [그림 2.4] 설치 옵션 변경
<img src="https://github.com/user-attachments/assets/2cbd8057-404d-4ac5-9aab-15a82a2f487c" width="470"/><br/>

설치 위치는 기본 디렉터리로 그대로 유지하고, '설치' 버튼을 클릭해서 다음으로 넘어가면 라이선스 동의 화면이 표시된다. 그다음으로 사용자 인증 방식을 선택하는 화면이 표시된다.
여기서 'Use Strong Password Encryption'을 선택하면 Caching SHA-2 Authentication을 사용하게 되고, 'Use Legacy Password Encryption'을 선택하면 Native
Authentication 방식을 사용하게 된다.

#### [그림 2.5] 사용자 인증 방식 선택
<img src="https://github.com/user-attachments/assets/833decbe-4427-456d-9e24-6b55d44217be" width="470"/><br/>

MySQL 서버를 사설 네트워크에서만 사용한다면 'Use Legacy Password Encryption'을 선택해도 괜찮지만 인터넷을 경유해서 MySQL 서버에 접속하게 된다면 'Use Strong Password
Encryption'을 선택하자.

#### [그림 2.6] macOS의 관리자 계정 비밀번호 입력
<img src="https://github.com/user-attachments/assets/abced497-4094-4b67-bfe3-6636b848823c" width="470"/><br/>

MySQL 서버를 설치할 때 기본 설정으로 설치하면 데이터 디렉터리와 로그 파일들을 /usr/local/mysql 디렉터리 하위에 생성하고 관리자 모드로 MySQL 서버 프로세스를 기동하기 때문에 아래와 같이
관리자 계정에 대한 비밀번호 설정이 필요하다.

설치가 완료되면 다음과 같이 MySQL 서버가 자동으로 실행된다.

```
macos> ps -ef | grep mysqld
74 54322    1    0  8:27PM ??        0:00.59 /usr/local/mysql/bin/mysqld --user=_mysql
    --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
    --plugin-dir=/usr/local/mysql/lib/plugin --log-error=/usr/local/mysql/data/mysqld.local.err
    --pid-file=/usr/local/mysql/data/mysqld.local.pid --keyring-file-data=/usr/local/mysql/keyring/keyring
    --early-plugin-load=keyring_file.so --default_authentication_plugin=mysql_native_password
```

MySQL 서버가 설치된 디렉터리는 /usr/local/mysql이며, 하위의 각 디렉터리 정보는 다음과 같다.
여기에 나열되지 않은 디렉터리도 있는데, 최소한 다음 디렉터리들은 절대 삭제하면 안 되는 디렉터리들이다.

- bin: MySQL 서버와 클라이언트 프로그램, 유틸리티를 위한 디렉터리
- data: 로그 파일과 데이터 파일들이 저장되는 디렉터리
- include: C/C++ 헤더 파일들이 저장된 디렉터리
- lib: 라이브러리 파일들이 저장된 디렉터리
- share: 다양한 지원 파일들이 저장돼 있으며, 에러 메시지나 샘플 설정 파일(my.cnf)이 있는 디렉터리

macOS에 설치된 MySQL 서버의 설정 파일(my.cnf) 등록 및 시작과 종료는 모두 '시스템 환경설정'의 최하단에 있는 'MySQL'을 클릭하면 실해오디는 MySQL 관리 프로그램에서 수행할 수 있다.

macOS에 설치된 MySQL 서버의 관리 프로그램은 아래와 같은데, 이 화면에서 MySQL 서버를 시작하거나 종료할 수 있다.
macOS의 터미널에서 MySQL 서버를 실행하거나 종료하고 싶다면 다음과 같은 명령으로 MySQL 서버를 시작하고 종료할 수도 있다.
```
## MySQL 서버 시작
macos> sudo /usr/local/mysql/support-files/mysql.server start

## MySQL 서버 종료
macos> sudo /usr/local/mysql/support-files/mysql.server stop
```

#### [그림 2.8] MySQL 서버 관리 화면
<img src="https://github.com/user-attachments/assets/e810e421-5624-4fca-af6a-55d710f55c11" width="470"/><br/>

최상단에 있는 'Configuration' 탭을 클릭하면 MySQL 서버의 설정을 변경할 수 있는 화면이 표시된다.

#### [그림 2.9] MySQL 서버 설정
<img src="https://github.com/user-attachments/assets/ac159687-5468-4ff8-a2a5-d425fdf23517" width="470"/><br/>

MySQL 서버의 각종 디렉터리와 로그 파일들의 경로는 설정돼 있지만 MySQL 서버의 설정 파일(Configuration File)은 아직 준비돼 있지 않다는 것을 알 수 있다.
MySQL 서버의 기본 설정 파일 없이 실행 프로그램이 저장된 기본 디렉터리와 데이터 디렉터리 정도만 설정된 것을 알 수 있다.
우선 /usr/local/mysql 디렉터리에 my.cnf라는 빈 파일을 생성하고, 설정 파일(Configuration File) 항목에 /usr/local/mysql/my.cnf라고 입력한 후, 최하단의 'Apply' 버튼을 클릭해서
MySQL 설정 파일을 등록해두자. 많은 내용이 설정 파일의 변경과 연관이 있으므로 이 설정 파일의 경로는 반드시 기억해 둔다.

macOS에서 설치된 MySQL 서버가 정상 작동하지 않거나 다른 부분을 변경해야 한다면 [MySQL 설치 매뉴얼](https://dev/.mysql/doc/refman/8.0/en/osx-nstallation.html)에서 더 자세한
내용을 참고할 수 있다.

### [4] 윈도우 MSI 인스톨러 설치
윈도우에서 인스톨러로 MySQL 서버를 설치하려면 설치에 필요한 윈도우 인스톨 프로그램을 직접 다운로드해야 한다.
[MySQL 다운로드 페이지](https://dev.mysql.com/downloads/mysql/)에서 운영체제의 버전을 선택하면 MSI 설치 프로그램을 다운로드할 수 있는 링크를 제공하며, 해당 링크를 클릭해 MSI 인스톨
프로그램을 다운로드하면 된다. 최신 버전이 아닌 이전 버전을 다운로드하고 싶다면 아래 그림 우측 상단에 있는 'Looking for previous GA versions?' 링크를 클릭하면 원하는 버전을 선택할 수 있다.

#### [그림 2.10] 윈도우 MSI 인스톨러 다운로드
<img src="https://github.com/user-attachments/assets/2b145d12-a81a-45e7-9d31-fb02719bce22" width="550"/><br/>

다운로드된 MSI 인스톨러 파일을 실행하면 설치 유형을 선택하는 화면이 나타난다.

#### [그림 2.11] 설치 유형 선택
<img src="https://github.com/user-attachments/assets/8698115b-0ba2-42ff-9f83-4de589e541a1" width="550"/><br/>

'Developer Default'를 선택하면 MySQL 서버와 클라이언트 도구, 그리고 MySQL Workbench 같은 GUI 클라이언트 도구가 모두 설치된다.
여기서는 꼭 필요한 소프트웨어만 선택하기 위해 'Custom'을 선택하고 다음으로 넘어가자.

#### [그림 2.12] 설치 소프트웨어 선택
<img src="https://github.com/user-attachments/assets/72fc1091-106d-41b2-ad7a-6d8ec45ea276" width="550"/><br/>

설치할 소프트웨어를 직접 선택할 수 있는데, 꼭 필요한 소프트웨어인 MySQL 서버(MySQL 클라이언트 프로그램이 포함돼 있음)와 MySQL Shell, MySQL Router만 선택하고 다음 화면으로 넘어가자.
다음 화면에서 필요한 라이브러리들을 모두 설치한 후, 그다음 화면으로 넘어가자.

#### [그림 2.13] 고가용성(High Availability) 옵션 선택
<img src="https://github.com/user-attachments/assets/72435831-e33b-4f63-91a6-f9cb744933fe" width="550"/><br/>

MySQL 서버의 고가용성 옵션을 선택할 수 있다. 여기서는 복제 없이 단일 서버 실행 모드인 'Standalone MySQL Server / Classic MySQL Replication' 옵션을 선택한다.

#### [그림 2.14] 네트워크 옵션 선택
<img src="https://github.com/user-attachments/assets/b3878cab-4ff2-445a-8b35-b571664bc34a" width="550"/><br/>

MySQL 서버를 어떤 방식으로 접속하게 할지 설정한다. 지금 설치하는 MySQL 서버는 테스트용이므로 'Config Type'을 'Development Computer'로 선택한다.
'Development Computer' 옵션을 선택하면 MySQL 서버가 허용하는 커넥션의 개수를 적게 설정하게 되므로 서비스용 MySQL 서버를 설치한다면 그에 맞게 옵션을 변경하자.
'Connectivity' 옵션은 일반적으로 많이 사용되는 TCP/IP로 선택하고, Port는 MySQL 서버의 기본 포트인 3306으로, X Protocol Port도 기본 포트인 3306 그대로 유지한다.
이처럼 설정하고 다음 화면으로 넘어가자.

#### [그림 2.15] 비밀번호 인증 방법 선택
<img src="https://github.com/user-attachments/assets/b70eb0eb-cf4d-4671-b2da-781d4d7cde0d" width="550"/><br/>

위 그림에서는 사용자 로그인 시점에 사용할 비밀번호 인증 방식을 선택한다.
'Strong Password Encryption'은 Caching SHA-2 Authentication 플러그인을 사용하는 것이며, 'Legacy Authentication Method'는 Native Authentication 플러그인을 사용하는
것이다. 지금 설치하는 MySQL 서버는 테스트용이므로 'Legacy Authentication Method'를 선택한다.

#### [그림 2.16] 관리자 계정 비밀번호 입력
<img src="https://github.com/user-attachments/assets/fb313dd4-0de2-4ea3-aa59-79ceeb2845cb" width="550"/><br/>

위 그림에서는 MySQL 서버의 관리자 계정(root 계정)의 비밀번호를 입력한다. 필요하다면 하단의 'Add User' 버튼을 이용해 추가 계정을 더 등록할 수 있다.

#### [그림 2.17] 설정 내용 적용
<img src="https://github.com/user-attachments/assets/496f2b73-ec06-436e-83e3-032850361739" width="550"/><br/>

위 그림에서는 지금까지 설정한 내용을 이용해 MySQL 설정 파일과 데이터 디렉터리 및 기본 시스템 테이블을 생성한다.
설정 내용에 따라 1\~2분 내에 완료될 수도 있고, 더 오랜 시간이 걸릴 수도 있으니 응답이 없다고 해서 설치를 취소하지 않도록 주의하자.

MySQL 서버의 설정이 모두 완료되면 MySQL Router의 옵션을 설정하는 화면이 나타나는데, 이 설정 화면은 취소하고 다음으로 넘어간다.

이제 MySQL 서버의 설치가 완료됐다. MySQL 서버의 프로그램과 설정 파일(my.ini)의 위치는 아래와 같이 윈도우 서비스 화면에서 'MySQL80' 서비스의 등록 정보를 통해 확인할 수 있다.

#### [그림 2.18] 윈도우 서비스의 MySQL80 서비스 속성
<img src="https://github.com/user-attachments/assets/17b5f8f2-d978-444c-abe3-6f135be79036" width="650"/><br/>

윈도우 서비스에서 'MySQL 80' 서비스의 등록 정보를 살펴보면 MySQL 서버의 설치 디렉터리와 설정 파일의 위치는 다음과 같다.

<img src="https://github.com/user-attachments/assets/3ad26497-a58e-47f7-9b16-6ce8c9630d0c" width="420"/><br/>

지금까지 진행한 설치 과정을 그대로 따라왔다면 동일한 결과가 보일 것이다.
MySQL 서버의 설정 파일의 경로는 '실행 파일 경로' 항목의 --defaults-file 옵션에 지정된 파일로 확인할 수 있는데, 기본 파일 경로로 "C:\ProgramData\MySQL\MySQL Server
8.0\Data\my.ini"를 사용한다.
- MySQL 서버의 설정 파일은 리눅스나 macOS, 유닉스 계열에서는 my.cnf이지만 윈도우 운영체제에서는 my.ini라는 파일명을 사용한다.

MySQL 서버 디렉터리의 구조는 다음과 같다. 다음 디렉터리들이 삭제되면 MySQL 서버가 정상적으로 실행되지 않을 수 있으니 주의하자.

- bin: MySQL 서버와 클라이언트 프로그램, 그리고 유틸리티를 위한 디렉터리
- include: C/C++ 헤더 파일들이 저장된 디렉터리
- lib: 라이브러리 파일들이 저장된 디렉터리
- share: 다양한 지원 파일들이 저장돼 있으며, 에러 메시지나 샘플 설정 파일(my.ini)이 있는 디렉터리

윈도우 운영체제에서 설치한 MySQL 서버가 정상적으로 작동하지 않거나 추가 변경이 더 필요한 경우에는 [MySQL 설치 매뉴얼](https://dev.mysql.com/doc/refman/8.0/en/windows-installation.html)을
참고한다.