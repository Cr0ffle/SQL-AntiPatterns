# #23 나쁜 것 안 보기

## 목표: 코드를 적게 작성하기

- 간결한 코드를 작성해야 하는 이성적인 이유
  - 작동하는 애플리케이션 코딩을 좀더 빨리 끝낼 수 있다.
  - 테스트하고 문서화하고 동료검토해야 할 코드 양이 줄어든다.
  - 코드가 적으면 버그도 적을 것이다.



## 안티패턴: 짚 없이 벽돌 만들기

- 개발자는 다음과 같은 두 가지 형태로 나쁜 것 안 보기 안티패턴을 사용한다.
- 하나는 데이터베이스 API의 리턴 값을 무시하는 것이고 다른 하나는 애플리케이션 코드에 산재해 있는 SQL 코드의 단편만을 보는것이다.



**진단 없는 진료**

```sql
<?php
$pdo = new PDO("mysql:dbname=test;host=db.example.com", //(1)
    "dbuser", "dbpassword");
$sql = "SELECT bug_id, summary, date_reported FROM Bugs
    WHERE assigned_to = ? AND status = ?";
$stmt = $dbh->prepare($sql);         //(2)
$stmt->execute(array(1, "OPEN"));    //(3)
$bug = $stmt->fetch();               //(4)

```

- 이 코드는 함수로부터 리턴된 상태값을 무시하고 있다. 리턴된 값을 확인하지 않으면 문제가 생겼는지 알 수 없다.
- (1) 에서 데이터베이스 커넥션 관련 처리가 없다.
- (2) 에서 prepare()는 입력 실수나 짝이 맞지 않는 괄호, 잘못된 칼럼 이름 등의 단순한 문법 에러로 false를 리턴할 수 있다.
- 이 상태에서 (3)에 execute를 호출하면 에러가 발생한다.
- (4)에서 호출하는 fetch()또한 DBMS에 대한 커넥션이 실패하는 것과 같은 에러가 발생할 경우 false를 리턴한다.
- 사용자는 코드를 보지 않는다. 사용자는 결과를 본다.



**읽기간 행**

- 나쁜 것 안 보기 안티패턴에 해당하는 흔한 나쁜 습관은 SQL 쿼리를 문자열로 생성하는 애플리케이션 코드를 뚫어져라 쳐다보는 것이다.

```sql
<?php
$sql  = "SELECT * FROM Bugs";
if ($bug_id) {
    $sql .= "WHERE bug_id = " . intval($bug_id);
}
$stmt = $pdo->prepare($sql);

-- BugsWHERE 사이에 띄어쓰기가 없다.
SELECT * FROM BugsWHERE bug_id = 1234
```

- 개발자들은 이런 문제를 디버깅할 때 SQL 자체를 보지 않고 SQL을 만들어내는 코드를 살펴보느라 엄청나게 많은 시간을 소비한다.



## 안티패턴 인식 방법

- 다음과 같은 말을 들으면 나쁜 것 안 보기 안티패턴에 직면한 것일 수 있다.
  - "데이터베이스를 조회한 다음에 프로그램이 죽어버려"
  - "내 SQL에서 에러 찾는 것을 도와주겠니? 내 코드는 여기 있어 ... "
  - " 내 코드를 에러 처리로 어지럽히고 싶지 않아"



## 안티패턴 사용이 합당한 경우

- 에러에 대해 정말 아무것도 할 것이 없다면 에러 검사를 생략할 수 있다.



## 해법: 에러에서 우아하게 복구하기

- 실수의 원인을 인지할 수 있는 기회를 가져야 한다.



**리듬 유지하기**

- 데이터베이스 API 호출 후 리턴 상태나 예외를 확인하는 것이 가장 좋은 방법이다.

```sql
<?php
try {
    $pdo = new PDO("mysql:dbname=test;host=localhost",
	"dbuser", "dbpassword");
} catch (PDOException $e) {                       //(1)
    report_error($e->getMessage());
    return;
}

$sql = "SELECT bug_id, summary, date_reported FROM Bugs
    WHERE assigned_to = ? AND status = ?";

if (($stmt = $pdo->prepare($sql)) === false) {    //(2)
    $error = $pdo->errorInfo();
    report_error($error[2]);
    return;
}

if ($stmt->execute(array(1, "OPEN")) === false) { //(3)
    $error = $stmt->errorInfo();
    report_error($error[2]);
    return;
}

if (($bug = $stmt->fetch()) === false) {          //(4)
    $error = $stmt->errorInfo();
    report_error($error[2]);
    return;
}

```



**스텝 되짚기**

- 문제를 디버깅하는 데 SQL 쿼리를 생성하는 코드를 보는 대신 실제 SQL 쿼리를 확인하는 것 또한 중요하다.
  - API 메서드의 인수에서 SQL 쿼리를 만들지 말고 변수를 사용해서 만든다. 이렇게 하면 사용하기 전에 변수 값을 확인할 수 있는 기회가 생긴다.
  - 로그 파일, IDE 디버거 콘솔 또는 디버깅 정보를 보여주는 블우저 확장 기능 등 애플리케이션 출력이 아닌 다른 곳에 SQL을 출력하도록 한다.
  - SQL 쿼리를 웹 애플리케이션의 HTML 주석으로 출력하지 않는다. 해커가 SQL 쿼리를 보면 데이터베이스 구조에 대한 많은 정보가 노출될 수 있다.
- 대부분의 데이터베이스 제품은 클라이언트 애플리케이션 코드를 사용하지 않고 데이터베이스 서버에서 자체 로깅 메커니즘을 제공한다.



**SQL Antipatterns Tip**

- 코드의 문제를 해결하는 것만으로도 이미 충분히 어렵다. 보지 않고 작업해 스스로를 방해하지 마라.




