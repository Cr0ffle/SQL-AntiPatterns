# #11 31가지 맛

- 아래와 같은 데이터타입을 강제하는 스키마가 있다.

```sql
CREATE TABLE Bugs (
  -- other columns
  status   VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
);

```

- 변경이 없을것만 같았던 status에 변경이 필요해졌고 해당 테이블을 수정하기 위해서 테이블 락이 이뤄져야 한다.



## 목표: 칼럼을 특정 값으로 제한하기

- 칼럼의 값을 고정된 집합의 값으로 제한하는 것은 매우 유용하다 해당 칼럼이 유효하지 않은 항목을 절대로 포함하지 않는다고 보장할 수 있으면 칼럼을 사용하는 것이 단순해진다.



## 안티패턴: 칼럼 정의에 값 지정

- 칼럼 정의는 메타데이터, 즉 테이블 구조 정의의 일부다.
- 칼럼에 CHECK 제약조건을 정의할 수 있고 이 조건은 제약조건을 실패하게 하는 INSERT, UPDATE를 허용하지 않는다.

```sql
CREATE TABLE Bugs (
  -- other columns
  status   VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
);

```

- MySQL은 칼럼을 특정 값의 제한으로 제한하는 ENUM이라 불리는 비표준 데이터타입을 지원한다.

```sql
CREATE TABLE Bugs (
  -- other columns
  status ENUM('NEW', 'IN PROGRESS', 'FIXED'),
);

```

- 다른 방법은 도메인이나 사용자 정의 타입을 사용하는 방법이다. 이를 이용해 칼럼에 미리 지정한 값만 허용하도록 제한하는 방법을 적용할 수 있다.
- 또 다른 방법으로 트리거를 사용하는 방법이 있다.
- 위 방법들은 모두 단점을 갖고 있다.



**중간에 있는 게 뭐지?**

- status에 해당하는 값을 가져오는 쿼리가 꽤 복잡해진다. (애플리케이션에서 관리하지 않고 스키마로도 관리되지 않을 때)
- MySQL의 ENUM을 사용한다면 아래와 같이 가져올 수 있긴 하지만 이 역시 좋은 방법은 아니다.

```sql
SELECT column_type
FROM information_schema.columns
WHERE table_schema = 'bugtracker_schema'
  AND table_name = 'bugs'
  AND column_name = 'status';

```

- 결국 값의 목록을 애플리케이션 코드에 동일하게 유지하는 단순한 방식을 취하게 되는데 애플리케이션 데이터와 데이터베이스 메타데이터가 서로 맞지 않게 되면 문제가 발생한다.

**새로운 맛 추가하기**

- 가장 흔한 변경은 허용된 값을 추가하거나 삭제하는 것이다. 

```sql
ALTER TABLE Bugs MODIFY COLUMN status
  ENUM('NEW', 'IN PROGRESS', 'FIXED', 'DUPLICATE');

```

- 어떤 데이터베이스 제품에서는 테이블이 비어 있지 않으면 칼럼 정의를 변경할 수 없다.
- 이런 작업은 복잡하고 비용이 많이 드는 작업이다.
- 정책적으로 메타데이터를 변경하는 것은 드물어야하고 주의를 요해야 한다. 



**예전 맛은 절대 없어지지 않는다.**

- 값을 더이상 사용되지 않게 만들면 과거 데이터가 망가질 수 있다.

```sql
ALTER TABLE Bugs MODIFY COLUMN status
  ENUM('NEW', 'IN PROGRESS', 'CODE COMPLETE', 'VERIFIED');

```

- 기존에 있던 FIXED를 삭제하면 상태가 FIXED인 버그들은 난감해질 수 있다.
- 없어질 값이라도 과거 행이 참조하는 한 그대로 유지해야 할 수도 있다. 그러나 이런 경우에 더 이상 사용되지 않는 값을 식별하는 마땅한 방법이 없다.



**포팅이 어렵다.**

- 체크 제약조건, ENUM, 사용자 정의 타입은 모든 SQL 데이터베이스 제품에서 균일하게 지원되는 기능이 아니다.
- 이런 차이로 인해 여러 데이터베이스 제품을 지원할 필요가 있는 경우 이런 안티패턴을 적용하기 어렵다.



## 안티패턴 인식 방법

- ENUM이나 체크 제약조건의 문제는 값의 집합이 고정되지 않았을 때 나타난다.
- ENUM 사용을 고려하고 있다면 먼저 값의 집합이 변할 것 같은지 스스로에게 물어보고 변할 것 같으면 ENUM을 사용하지 않는게 좋다.
  - "데이터베이스를 내려야 애플리케이션 메뉴의 선택항목을 추가할 수 있어. 길어야 30분이면 충분할거야. 모든게 잘 되면 말이지"
  - "status 칼럼은 다음 값 중 하나만 가질 수 있어. 이 목록을 바꿀 일이 생기면 안 돼."
  - "애플리케이션 코드에 있는 목록 값이 데이터베이스에 있는 비즈니스 규칙과 또 틀어졌어."



## 안티패턴 사용이 합당한 경우

- 값의 집합이 변하지 않는다면 ENUM을 사용해도 문제가 별로 없다.



## 해법: 데이터로 값을 지정하기

- 칼럼의 값을 제한하는 것보다 Bugs.status 칼럼에 들어갈 수 있는 각 값을 행으로 하는 색인 테이블을 만드는 것이다.
- 그리고 Bugs.status가 새로 만든 테이블을 참조하도록 FK 제약조건을 선언한다.

```sql
CREATE TABLE BugStatus (
  status  VARCHAR(20) PRIMARY KEY
);

INSERT INTO BugStatus (status) VALUES ('NEW'), ('IN PROGRESS'), ('FIXED');

CREATE TABLE Bugs (
  -- other columns
  status  VARCHAR(20),
  FOREIGN KEY (status) REFERENCES BugStatus(status)
    ON UPDATE CASCADE
);

```

- 위와 같은 방법을 사용하면 ENUM, 체크제약조건처럼 status 값을 제한할 수 있다.



**값의 집합 쿼리하기**

- 이제 허용된 값은 메타데이터로 관리되는 것이 아니라 데이터로 저장된다.
- 값의 집합을 데이터로 얻어올 수 있기 때문에 허용된 값을 조회하기 쉽다.

```sql
SELECT status FROM BugStatus ORDER BY status;

```



**색인 테이블의 값 갱신하기**

- 색인 테이블을 사용하면 평범한 INSERT 문으로 값을 추가할 수 있다.
- 칼럼을 재정의하거나 다운타임 일정을 세울 필요도 없다. 

```sql
INSERT INTO BugStatus (status) VALUES ('DUPLICATE');

UPDATE BugStatus SET status = 'INVALID' WHERE status = 'BOGUS';

```



**더 이상 사용하지 않는 값 지원하기**

- Bugs에 있는 행이 참조하는 한 색인 테이블에서 행을 삭제할 수는 없다.
- 그러나 색인 테이블에 또 다른 속성 칼럼을 추가해 더 이상 사용되지 않는 값을 표시할 수는 있다.

```sql
ALTER TABLE BugStatus ADD COLUMN active
  ENUM('INACTIVE', 'ACTIVE') NOT NULL DEFAULT 'ACTIVE';

```

- 값을 DELETE 하는 대신 UPDATE해서 유연하고 융통성있게 사용할 수 있다.

```sql
UPDATE BugStatus SET active = 'INACTIVE' WHERE status = 'DUPLICATE';

SELECT status FROM BugStatus WHERE active = 'ACTIVE';
```



**포팅이 쉽다**

- ENUM, 체크제약조건과 달리 색인 테이블을 사용하는 방법은 FK 제약조건을 사용한 참조 정합성이란 표준 SQL 기능만 사용하기 때문에 포팅이 쉬워진다.
- 각 값을 별도의 행으로 저장하기 때문에 행의 개수에 제한이 없다.



**SQL Antipatterns Tip**

- 고정된 값의 집합에 대한 유효성 확인을 할 때는 메타데이터를 사용하라.
- 유동적인 값의 집합에 대한 유효성 확인을 할 때는 데이터를 사용하라.


