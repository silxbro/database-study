# 09. Static vs. Dynamic SQL

하드 파싱 부하를 최소화하기 위해 Dynamic SQL 대신 Static SQL을 사용하라는 표현을 흔히 사용하는데, 용어를 제대로 사용하고 있는지 확인해 볼 필요가 있다.
그런 뜻에서 라이브러리 캐시 최적화를 위한 Static SQL 작성 기법들을 소개하기 전에 Static과 Dynamic SQL 용어를 명확히 정의하고자 한다.

먼저 \[그림 4-12\](p.308)의 두 쿼리 툴에서 작성한 SQL문이 Static SQL인지, 아니면 Dynamic SQL인지에 대해 생각해 보기 바란다.

왼쪽은 Dynamic이고 오른쪽은 Static이라고 답했다면 지금부터 설명하는 내용을 대충 읽고 넘어가서는 안 된다.

한 가지 예를 더 살펴보자. 아래는 JAVA에서 SQL문을 수행하는 예제인데, 이에 대해서는 이견 없이 Dynamic SQL이라고 답할 것이다.

```
SQLState = "select count(*) from emp where deptno = " + p_deptno;
stmt= conn.prepareStatement(SQLState);
rs = stmt.executeQuery();
```

그렇다면, 아래는 어떤가?

```
if (p_deptno == 10) {
  SQLState = "select count(*) from emp where deptno = 10";
} else if (p_deptno == 20) {
  SQLState = "select count(*) from emp where deptno = 20";
} else if (p_deptno == 30) {
  SQLState = "select count(*) from emp where deptno = 30";
} else if (p_deptno == 40) {
  SQLState = "select count(*) from emp where deptno = 40";
}
stmt= conn.prepareStatement(SQLState);
rs = stmt.executeQuery();
```

Static인가? 아니면 Dynamic인가? 튜닝 교육할 때 직접 질문을 던져보면 수강생마다 각기 다르게 대답한다.
그런데 대부분 개발 프로젝트에 가서 SQL 작성 표준 가이드 문서를 보면 Dynamic SQL을 사용하지 말라는 문구는 반드시 들어가 있다. 용어의 의미조차 명확하지 않은데, 이 표준이 잘 지켜질 리가 만무하다.
조건절에 바인드 변수를 사용하면 Static SQL, Literal 상수 값을 사용하면 Dynamic SQL로 분류하는 튜닝 교재들이 있어 이런 혼선이 빚어졌다고 생각한다.
<br/>
<br/>
## (1) Static SQL
Static SQL이란, String형 변수에 담지 않고 코드 사이에 직접 기술한 SQL문을 말한다. 다른 말로 'Embedded SQL'이라고도 한다. 아래는 Pro*C 구문으로 Static SQL을 작성한 예시다.

```
int main()
{
   printf("사번을 입력하십시오 : ");
   scanf("%d", &empno);
   EXEC SQL WHENEVER NOT FOUND GOTO notfound;
   EXEC SQL SELECT ENAME INTO :ename
            FROM   EMP
            WHERE  EMPNO = :empno;
   printf("사원명 : %s.\n", ename);

notfound:
  printf("%d는 존재하지 않는 사번입니다. \n", empno);
}
```

여기서 보듯이 SQL문을 String 변수에 담지 않고 마치 예약된 키워드처럼 C/C++ 코드 사이에 섞어서 기술하고 있다.

Pro*C, SQLJ와 같은 PreCompile 언어를 간단히 설명하면, 예를 들어 Pro\*C에서 소스 프로그램(.pc)을 작성해서 PreCompiler로 PreComplie하면 순수 C/C++ 코드가 만들어진다.
이를 다시 C/C++ Compiler로 Compile하면 실행파일이 만들어지고, 이것을 실행하게 된다.

PreCompiler가 PreCompile 과정에서 Static(=Embedded) SQL을 발견하면 이를 SQL 런타임 라이브러리에 포함된 함수를 호출하는 코드로 변환한다. 이 과정에서 결국은 String형 변수에 담긴다.
Static SQL이든 Dynamic SQL이든 PreCompile 단계를 거치고 나면 String 변수에 담기기는 마찬가지지만 Static SQL은 런타임 시에 절대 변하지 않으므로 PreCompile 단계에서 구문 분석, 유효
오브젝트 여부, 오브젝트 액세스 권한 등을 체크하는 것이 가능하다.

아래는 에디터로 작성한 .pc 파일을 각각 syntax, symantics 옵션을 주고 Pro*C로 PreCompile하는 예시다.

#### [test.pc]
```
EXEC SQL INCLUDE SQLCA.H;

int main()
{
  EXEC SQL
    UPDATE EMP SET SAL = SAL * 1.1
    WHER   EMPNO = 7900;

  return 0;
}
```

```
$ proc test.pc sqlcheck=syntax
```

위와 같이 sqlcheck=syntax 옵션을 주고 PreCompile하면 사전에 구문 분석을 실시한다. 여기서는 WHERE 키워드에 오타가 있으므로 Syntax 에러가 발생한다.

#### [test.pc]
```
EXEC SQL INCLUDE SQLCA.H;

int main()
{
  EXEC SQL
    UPDATE EMPs SET SAL = SAL * 1.1
    WHERE  EMPNO = 7900;
  return 0;
}
```

```
$ proc test.pc sqlcheck=syntax ---> Success!!
$ proc test.pc sqlcheck=full userid=scott/tiger --> Error!!
```

sqlcheck = full 옵션은 sqlcheck = semantics과 동일한 의미를 갖는다. 즉, 구문 분석 뿐 아니라 오브젝트가 유효한 것인지, 액세스 권한은 있는지까지 체크한다.
여기서는 SCOTT 계정에 없는 EMPs 테이블을 참조하였으므로 Semantic 에러가 발생한다.
<br/>
<br/>
## (2) Dynamic SQL
Dynamic SQL이란, String형 변수에 담아서 기술하는 SQL문을 말한다.
String 변수를 사용하므로 조건에 따라 SQL문을 동적으로 바꿀 수 있고, 또는 런타임 시에 사용자로부터 SQL문의 일부 또는 전부를 입력 받아서 실행할 수도 있다.
따라서 PreCompile 시 Syntax, Semantic 체크가 불가능하다.

Dynamic SQL을 만나면 PreCompiler는 그 내용을 확인하지 않고 그대로 통과시킨다.
Pro*C 환경에서 개발해봤다면 스칼라 서브쿼리, 분석 함수, ANSI 조인문 등을 사용했을 때 PreCompile 과정에서 에러가 나는 경험을 했을 것이다.
Semantic 체크는 DB 접속을 통해 이루어지지만 Syntax 체크만큼은 PreCompiler에 내장된 SQL 파서(Parser)를 이용하는데, 위 구문들을 사용하면 현재 사용 중인 PreCompiler가 그것들을 인식하지
못해 에러를 던지는 것이다. 해결방법은? Dynamic SQL을 사용하면 된다.

아래는 Pro*C에서 Dynamic SQL을 작성한 사례다. SQL을 String형 변수에 담아 실행하는 것에 주목하기 바란다. 바로 아래 주석 처리한 부분은 SQL을 런타임 시 입력 받는 방법을 예시한다.

```
int main()
{

  char select_stmt[ 50] = "SELECT ENAME FROM EMP WHERE EMPNO = :empno";
  // scanf("%c", &select_stmt); --> SQL문을 동적으로 입력 받을 수도 있음

  EXEC SQL PREPARE sql_stmt FROM :select_stmt;

  EXEC SQL DECLARE emp_cursor CURSOR FOR sql_stmt;

  EXEC SQL OPEN emp_cursor USING :empno;

  EXEC SQL FETCH emp_cursor INTO :ename;

  EXEC SQL CLOSE emp_cursor;

  prinf("사원명 : %s.\n", ename);
}
```

Pro*C에서 제공하는 Dynamic Method에는 4가지가 있고, 간단히 요약하면 아래와 같다.

#### [Method 1] 입력 Host 변수 없는 Non-Query
- Non-Query는 SELECT 이외 명령어를 말함

```
'DELETE FROM EMP WHERE DEPTNO = 20'
'GRANT SELECT ON EMP TO scott'
```

#### [Method 2] 입력 Host 변수 개수가 고정적인 Non-Query
```
'INSERT INTO EMP (ENAME, JOB) VALUES (:ename, :job)'
'DELETE FROM EMP WHERE EMPNO = :empno'
```

#### [Method 3] select-list 컬럼 개수와 입력 Host 변수 개수가 고정적인 Query
```
'SELECT DEPTNO, MAX(SAL) FROM EMP GROUP BY DEPTNO'
'SELECT DNAME, LOC FROM DEPT WHERE DEPTNO = 20'
'SELECT ENAME, EMPNO FROM EMP WHERE DEPTNO = :deptno'
```

#### [Method 4] select-list 컬럼 개수와 입력 Host 변수 개수가 가변적인 Query
```
'INSERT INTO EMP( <unknown> ) values ( <unknown> )'
'SELECT <unknown> FROM EMP WHERE DEPTNO = :deptno'
```
<br/>

## (3) 일반 프로그램 언어에서 SQL 작성법
지금까지 Pro*C 위주로 Static과 Dynamic SQL을 설명했는데, 가장 인기 있는 개발언어 중 하나인 JAVA에서는 SQL을 어떻게 작성하는지 살펴보자.

```
PreparedStatement stmt;
ResultSet rs;
StringBuffer SQLStmt = new StringBuffer();
SQLStmt.append("SELECT ENAME, SAL FROM EMP ");
SQLStmt.append("WHERE EMPNO = ?");

stmt = conn.prepareStatement(SQLStmt.toString());
stmt.setLong(1, txtEmpno.value);
rs = stmt.executeQuery();

// do anything

rs.close();
stmt.close();
```

그 다음은 Delphi에서 SQL을 작성하는 예시다.
```
begin
  Query1.Close;
  Query1.Sql.Clear;
  Query1.Sql.Add('SELECT ENAME, SAL FROM EMP ');
  Query1.Sql.Add('WHERE EMPNO = :empno');
  Query1.ParamByName('empno').AsString := txtEmpno.Text;
  Query1.Open;
end;
```

마지막으로, Visual Basic에서의 SQL문 작성 예시를 보자.
```
Dim comm As New ADODB.Command
Dim rs As ADODB.Recordset
Dim SQLStmt as String

SQLStmt = "SELECT ENAME, SAL FROM EMP "
SQLStmt = SQLStmt & "WHERE EMPNO = ?"
comm.CommandText = SQLStmt
comm.Parameters.Append comm.CreateParameter("empno", adNumeric, adParamInput)
comm.Parameters("empno").Value = txtEmpno.Text
Set rs = comm.Execute

' do anything

rs.Close
Set rs = Nothing
Set comm = Nothing
```

여기 3가지 사례에서 보듯이 Static SQL을 작성할 수 있는 방법이 제공되지 않는다. 모두 String 변수에 담아서 실행하는 것이다. 따라서 이들 언어에서 작성된 SQL은 모두 Dynamic SQL이다.
그런데도 Dynamic SQL로 작성하지 말라고 한다면 어불성설이다.

그리고 Toad, Orange, SQL*Plus과 같은 Ad-hoc 쿼리 툴에서 작성하는 SQL도 모두 Dynamic SQL이라고 보면 틀림없다.
이들 툴이 컴파일되는 시점에 SQL이 확정되지 않았으며, 사용자가 던지는 SQL을 런타임 시에 받아서 그대로 DBMS에 던지는 역할만 할 뿐이다.

Static(=Embedded) SQL을 지원하는 개발 언어로는 PowerBuilder, PL/SQL, Pro*C, SQLJ 정도가 있다.
- Static Embedded SQL과 Dynamic Embedded SQL로 분류하는 문서도 있음
  <br/><br/>

## (4) 문제의 본질은 바인드 변수 사용 여부
지금까지 설명한 Static, Dynamic SQL은 애플리케이션 개발 측면에서의 구분일 뿐이며, 데이터베이스 입장에서는 차이가 없다.
Static SQL을 사용하든 Dynamic SQL을 사용하든 오라클 입장에서는 던져진 SQL문 그 자체만 인식할 뿐이며, PL/SQL, Pro*C 등에서 애플리케이션 커서 캐싱 기능을 활용하고자 하는 경우 외에는
성능에도 전혀 영향이 없다.(PL/SQL, Pro\*C 등에서는 Static SQL일 때만 애플리케이션 커서 캐싱이 작동함)
애플리케이션 커서 캐싱 기능을 사용하지 않는다면 Dynamic, Static 구분은 라이브러리 캐시 효율과도 전혀 무관하다.

그러므로 라이브러리 캐시 효율을 논할 때 초점은 바인드 변수 사용 여부에 맞춰져야 한다. Dynamic SQL을 사용해 문제가 되는 것이 아니라 바인드 변수를 사용하지 않았을 때 문제가 되는 것이다.
바인드 변수를 사용하지 않고 Literal 값을 SQL 문자열에 결합하는 방식으로 개발했을 때, 반복적인 하드파싱으로 성능이 얼마나 저하되는지, 그리고 그 때문에 라이브러리 캐시에 얼마나 심한 경합이
발생하는지, 그 원리에 대해서는 앞에서 충분히 설명했다.

다시 한번 강조하지만, 바인드 변수 사용 여부로 Static과 Dynamic을 구분하는 것은 잘못된 것이므로 용어 사용에 주의하자.