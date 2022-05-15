

# #19 암묵적 칼럼

## 목표: 타이핑 줄이기

- 프로그래머들이 타이핑을 너무 많이 해야 한다고 불평하는 예 중 하나가 사용할 모든 칼럼을 SQL 쿼리에 써야 하는 경우다.

```sql
SELECT bug_id, date_reported, summary, description, resolution,
  reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;

-- 모든 칼럼을 조회한다면 아래와 같이 사용한다.

SELECT * FROM Bugs;
```

- INSERT 역시 칼럼을 암묵적으로 적용한다.

```sql
INSERT INTO Accounts (account_name, first_name, last_name, email,
  password, portrait_image, hourly_rate) VALUES
  ('bkarwin', 'Bill', 'Karwin', 'bill@example.com', SHA2('xyzzy'), NULL, 49.95);

INSERT INTO Accounts VALUES (DEFAULT,
  'bkarwin', 'Bill', 'Karwin', 'bill@example.com', SHA2('xyzzy'), NULL, 49.95);


```



## 안티패턴: 지름길만 좋아하면 길을 잃는다

- 칼럼 이름 지정 없이 와일드카드를 사용하면 타이핑을 줄이는 목적은 이룰 수 있겠지만 이런 습관은 몇가지 위험을 초래한다.

**리팩터링 방해**

- Bug 테이블에 일정 관리를 위해 date_due 칼럼을 추가해야 한다고 가정할때 아래 INSERT문은 에러를 발생한다.

```sql
INSERT INTO Bugs VALUES (DEFAULT, CURDATE(), 'New bug', 'Test T987 fails...',
    NULL, 123, NULL, NULL, DEFAULT, 'Medium', NULL);

-- SQLSTATE 21S01: Column count doesn't match value count at row 1

```

- 암묵적 칼럼을 사용하는 INSERT 문에서는 테이블의 모든 칼럼에 대한 값을 테이블에 정의된 순서로 나열해야 한다. 칼럼이 바뀌면 INSERT 문에서 에러가 발생하거나 심지어 엉뚱한 칼럼에 값을 할당할 수 있다.
- SELECT 쿼리를 사용할 때도 칼럼의 이름을 모르기 때문에 아래와 같이 칼럼의 순서위치로 참조하게 될 경우 의도하지 않은 값이 할당될 수 있다.

```sql
<?php
$stmt = $pdo->query("SELECT * FROM Bugs WHERE bug_id = 1234");
$row = $stmt->fetch();
$hours = $row[10];

```

- 칼럼 이름이 바뀌거나, 칼럼이 추가 또는 삭제되면 쿼리 결과가 애플리케이션 코드에서 지원하지 않는 방식으로 바뀔 수 있다.
- 이런 에러는 코드로 전파되고 어느 라인에서 실수가 발생했는지 추적하기가 어려워진다.

**숨겨진 비용**

- 쿼리에서 와일드카드를 사용하는 편리함은 성능과 확장적응성에 해를 끼칠 수 있다.
- 쿼리에서 더 많은 칼럼을 선택하면 데이터베이스 서버와 애플리케이션 사이의 네트워크를 통해 더 많은 데이터가 전달되어야 한다.



**요청한 것을 얻은 것이다**

- SQL에 원하지 않는 칼럼을 제외한 모든 칼럼을 지정하는 문법은 없다.
- 테이블의 모든 칼럼을 요청하는 와일드카드를 사용하거나 원하는 칼럼의 목록을 명시적으로 나열해야 한다.



## 안티패턴 인식 방법

- "애플리케이션이 여전히 데이터베이스의 칼럼을 예전 이름으로 참조해 동작하지 않아."
- "네트워크 병목을 추적하는 데 며칠이 걸렸어. 통계에 따르면 쿼리가 평균 2MB의 데이터를 가져가지만 표시하는 것은 그 10분의 1에 불과해"



## 안티패턴 사용이 합당한 경우

- 즉석 쿼리, 임시쿼리 등등 한번 사용할 쿼리라면 유지보수성은 중요하지 않다.
- 실행시의 효율보다 개발 효율을 중요하게 여겨서 너무 긴 칼럼 이름들을 입력하는 것 보다 중요한 것이 있다면 와일드카드를 사용할 수 있다. (IDE를 사용하자.)
- 긴 SQL 쿼리를 서버로 보내는 오버헤드보다 쿼리가 리턴하는 데이터가 훨씬 더 많은 네트워크 대역폭을 차지하는게 더 일반적이다.



### 해법: 명시적으로 칼럼 이름 지정하기

- 와일드카드나 암묵적 칼럼 목록에 의지하기보다는 항상 필요한 칼럼을 나열해야 한다.

```sql
SELECT bug_id, date_reported, summary, description, resolution,
  reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;

INSERT INTO Accounts (account_name, first_name, last_name, email,
  password_hash, portrait_image, hourly_rate)
VALUES ('bkarwin', 'Bill', 'Karwin', 'bill@example.com',
  SHA2('xyzzy'), NULL, 49.95);

```



**오류 검증**

- 쿼리의 SELECT 목록에 칼럼을 지정하면 SQL 쿼리가 에러나 앞에서 설명한 혼동에 대해 더 큰 저항력이 생긴다.
  - 테이블에서 칼럼의 위치가 바뀌어도 쿼리 결과에서는 바뀌지 않는다.
  - 테이블에 칼럼이 추가되어도, 쿼리 결과에는 나타나지 않는다.
  - 테이블에서 칼럼이 삭제되면 쿼리가 에러를 발생시킨다. 하지만 이 에러는 좋은 에러다. 코드를 고쳐야할 위치를 직접 알려주기 때문이다.



**그거 필요하지 않을 꺼야**

- 소프트웨어의 확장적응성과 작업처리량을 염려한다면, 네트워크 대역폭을 낭비할 가능성이 있는 곳을 살펴봐야 한다.
- 소프트웨어 개발과 테스트 단계에서는 SQL 쿼리의 대역폭이 문제없어 보일 수 있지만 초당 수천 개의 SQL 쿼리가 실행되는 실 환경에서는 문제가 될 수 있다.



**어쨌든 와일드카드를 포기해야 돼**

- 처음부터 와일드카드를 사용하지 않으면, 나중에 쿼리를 변경하기도 쉽다.



**SQL Antipatterns Tip**

- 원하는 대로 가져가라. 그러나 가져간 건 다 먹어야 한다.




