# #7 다형성 연관

- 사용자는 버그에 댓글을 달 수 있는 요구사항이다. Bugs와 Comments 사이의 관계는 일대다 관계다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  bug_id       BIGINT UNSIGNED NOT NULL,
  author_id    BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment      TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES Accounts(account_id),
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

```

- 그러나 댓글이 달릴 수 있는 테이블이 두개 일 수 있다. FeatureRequest는 별도 테이블에 저장하기는 하지만 Bugs와 비슷한 테이블이고 버그나 기능 요청 중 어느 이슈 타입과 관계되든 Comments를 하나의 테이블에 저장하고 싶은 상황이다.
- 그러나 여러개의 부모 테이블을 참조하는 FK를 만들 수는 없다.

```sql
-- START:constraint
  ...
  FOREIGN KEY (issue_id)
      REFERENCES Bugs(issue_id) OR FeatureRequests(issue_id)
);
-- END:constraint
```



## 목표: 여러 부모 참조

![7-1](https://github.com/sup2is/dev-note/blob/master/db/images/sql-antipatterns/7-1.jpeg)

## 안티 패턴: 이중 목적의 FK 사용

- 이런 경우에 대한 해법은 다형성 연관이란 이름이 붙을 정도로 널리 알려져 있다.
- 여러 테이블을 참조하기 때문에 난잡한 연관이라 불리기도 한다.

**다형성 연관 정의**

- 다형성 연관을 작동하게 하려면 참조하는 부모 테이블이름을 뜻하는 issue_type이라는 칼럼을 추가해야한다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  issue_type   VARCHAR(20),     -- "Bugs" or "FeatureRequests"
  issue_id     BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME,
  comment      TEXT,
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

```

- 이런 설계에서는 FK를 지정할 수 없다. FK는 하나의 테이블만 참조할 수 있기 때문에 다형성 연관을 사용할 경우에는 이 연관을 메타데이터에 선언할 수 없다.
- 그러므로 데이터 정합성을 강제하는 것이 불가능하다.
- 이 패턴은 이전에 봤던 EAV 안티 패턴과 비슷한점이 많다. (EAV도 참조정합성 강제 불가능)



**다형성 연관에서의 조회**

- 자식 테이블을 부모 테이블과 조인할 때 issue_type을 정확하게 사용해서 조회해야 한다.

```sql
-- START:issue
SELECT *
FROM Bugs AS b JOIN Comments AS c
  ON (b.issue_id = c.issue_id AND c.issue_type = 'Bugs')
WHERE b.issue_id = 1234;
-- END:issue
```

- 댓글을 조회할때는 아래와 같이 OUTER JOIN을 사용해서 조회해야 한다.

```sql
-- START:comment
SELECT *
FROM Comments AS c
  LEFT OUTER JOIN Bugs AS b
    ON (b.issue_id = c.issue_id AND c.issue_type = 'Bugs')
  LEFT OUTER JOIN FeatureRequests AS f
    ON (f.issue_id = c.issue_id AND c.issue_type = 'FeatureRequests');
-- END:comment

```



**비 객체지향 예제**

- 다형성 연관은 부모 테이블이 서로 아무런 관계가 없을 때도 사용할 수 있다.
- Orders 테이블, Users 테이블 모두 Address와 연관될 수 있다.
- 이런 설계는 한 주소를 사용자와 주문 둘 다에 연관시킬 수 없다.

```sql
-- START:table
CREATE TABLE Addresses (
  address_id   SERIAL PRIMARY KEY,
  parent       VARCHAR(20),     -- "Users" or "Orders"
  parent_id    BIGINT UNSIGNED NOT NULL,
  address      TEXT
);
-- END:table

```

- 또한 사용자가 배송지 주소 뿐 아니라 청구지 주소도 갖고 있다면 이를 구별하기 위한 방법이 Address  테이블에 있어야 하고 이런 표시는 잡초처럼 퍼져 나간다.

```sql
-- START:weeds
CREATE TABLE Addresses (
  address_id   SERIAL PRIMARY KEY,
  parent       VARCHAR(20),     -- "Users" or "Orders"
  parent_id    BIGINT UNSIGNED NOT NULL,
  users_usage  VARCHAR(20),     -- "billing" or "shipping"
  orders_usage VARCHAR(20),     -- "billing" or "shipping"
  address      TEXT
);
-- END:weeds
```



## 안티패턴 인식 방법

- 다음과 같은 말이 들린다면 다형성 연관 안티패턴이 사용되고 있음을 나타낸다.
  - "이 태깅 스키마는 데이터베이스 내의 어떤 리소스에도 태그(또는 다른 속성)를 달 수 있다.
  - "우리 데이터베이스 설계에서는 FK를 선언할 수 없어."
  - "entity_type 칼럼의 용도가 뭐지?"
- Ruby on Rails 에서는 액티브 레코드 클래스에 :polymorphic 속성을 선언하면 다형성 연관을 사용할 수 있다.

```sql
class Comment < ActiveRecord::Base
  belongs_to :commentable, :polymorphic => true
end

class Bug < ActiveRecord::Base
  has_many :comments, :as => :commentable
end

class FeatureRequest < ActiveRecord::Base
  has_many :comments, :as => :commentable
end

```

- Java를 위한 Hibernate 프레임워크도 다양한 스키마 선언을 통해 다형성 연관을 지원한다.
  - [https://javabeat.net/polymorphic-association-mapping-relationship-hibernate/](https://javabeat.net/polymorphic-association-mapping-relationship-hibernate/)
  - [https://thorben-janssen.com/polymorphic-association-mappings-of-independent-classes/](https://thorben-janssen.com/polymorphic-association-mappings-of-independent-classes/)
  - [https://www.concretepage.com/hibernate/hibernate-any-manytoany-and-anymetadef-annotation-example](https://www.concretepage.com/hibernate/hibernate-any-manytoany-and-anymetadef-annotation-example)



## 안티패턴 사용이 합당한 경우

- 다형성 연관 안티패턴은 사용을 피하고 FK와 같은 제약조건을 사용해 참조 정합성을 보장해야 한다.
- 다형성 연관은 메타데이터 대신 애플리케이션 코드에 지나치게 의존하게 만드는 경우가 많다.
- Hibernate와 같은 객체-관계 프로그래밍 프레임워크를 사용하는 경우 이 안티패턴 사용이 불가피할 수 있다. 이런 프레임워크는 참조 정합성 유지를 위한 애플리케이션 로직을 캡슐화해 다형성 연관으로 인해 생기는 위험을 완화해줄 수 있다.



## 해법: 관계 단순화

- 다형성 연관의 단점을 피하면서 필요한 데이터 모델을 지원하기 위해서는 데이터베이스를 다시 설계하는 게 낫다.



**역 참조**

`교차 테이블 생성`

- 자식 테이블 Comments에 있는 FK는 여러 부모 테이블을 참조할 수 없으므로, 대신 Comments 테이블을 참조하는 여러 개의 FK를 사용하도록 한다. 각 부모에 대해 별도의 교차 테이블을 생성하고 교차 테이블에는 각 부모 테이블에 대한 FK 뿐 아니라 Comments에 대한 FK도 포함시킨다.

```sql
CREATE TABLE BugsComments (
  issue_id    BIGINT UNSIGNED NOT NULL,
  comment_id  BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (issue_id, comment_id),
  FOREIGN KEY (issue_id) REFERENCES Bugs(issue_id),
  FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
);


CREATE TABLE FeaturesComments (
  issue_id    BIGINT UNSIGNED NOT NULL,
  comment_id  BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (issue_id, comment_id),
  FOREIGN KEY (issue_id) REFERENCES FeatureRequests(issue_id),
  FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
);

```

- 이 방법을 사용하면 issue_type 같은 칼럼이 불필요해지고 FK를 통해 데이터 정합성을 강제할 수 있다.



![7-2](https://github.com/sup2is/dev-note/blob/master/db/images/sql-antipatterns/7-2.jpeg)



`신호등 설치`

- 각 댓글은 하나의 버그 또는 하나의 기능요청에만 연결되어야 한다. 이 규칙을 부분적으로 강제할 수 있는데 각 교차테이블의 comment_id 칼럼에 UNIQUE 제약조건을 선언하면 된다.

```sql
CREATE TABLE BugsComments (
  issue_id    BIGINT UNSIGNED NOT NULL,
  comment_id  BIGINT UNSIGNED NOT NULL,
  UNIQUE KEY (comment_id),
  PRIMARY KEY (issue_id, comment_id),
  FOREIGN KEY (issue_id) REFERENCES Bugs(issue_id),
  FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
);

```

- 위 방법은 하나의 댓글이 하나의 버그에만 매핑되게 강제할 수 있지만 버그와 기능요청 양쪽에 동시에 연관되는 것은 방지하지 못한다. 이를 방지하는 것은 애플리케이션 코드의 책임으로 남는다.

`양쪽 다 보기`

- 특정 버그 또는 기능 요청에 대한 댓글은 교차테이블을 이용해 간단히 조회할 수 있다.

```sql
-- START:issue
SELECT *
FROM BugsComments AS b
  JOIN Comments AS c USING (comment_id)
WHERE b.issue_id = 1234;
-- END:issue
-- START:comment
SELECT *
FROM Comments AS c
  LEFT OUTER JOIN (BugsComments JOIN Bugs AS b USING (issue_id))
    USING (comment_id)
  LEFT OUTER JOIN (FeaturesComments JOIN FeatureRequests AS f USING (issue_id))
    USING (comment_id)
WHERE c.comment_id = 9876;
-- END:comment

```



`차선 통합`

- 때론 여러 부모 테이블에 대해 조회한 결과를 하나의 테이블에서 조회한 것처럼 보이게 할 필요가 있을때 아래와 같은 방법을 사용할 수 있다.

```sql
SELECT b.issue_id, b.description, b.reporter, b.priority, b.status,
    b.severity, b.version_affected,
    NULL AS sponsor
  FROM Comments AS c
    JOIN (BugsComments JOIN Bugs AS b USING (issue_id))
      USING (comment_id)
  WHERE c.comment_id = 9876;
  
UNION
  SELECT f.issue_id, f.description, f.reporter, f.priority, f.status,
    NULL AS severity, NULL AS version_affected,
    f.sponsor
  FROM Comments AS c
    JOIN (FeaturesComments JOIN FeatureRequests AS f USING (issue_id))
      USING (comment_id)
  WHERE c.comment_id = 9876;

```

- 한쪽 테이블에만 존재하는 칼럼에 대해서는 그에 대응해 NULL로 대응시켜준다.
- 다른 방법은 COALESCE() 함수를 사용하는 방법이다.
- COALESCE()은 파라미터의 순서대로 NULL이 아닌 값을 가진 인자를 리턴한다.

```sql
SELECT c.*,
  COALESCE(b.issue_id,    f.issue_id   ) AS issue_id,
  COALESCE(b.description, f.description) AS description,
  COALESCE(b.reporter,    f.reporter   ) AS reporter,
  COALESCE(b.priority,    f.priority   ) AS priority,
  COALESCE(b.status,      f.status     ) AS status,
  b.severity,
  b.version_affected,
  f.sponsor
  
FROM Comments AS c
  LEFT OUTER JOIN (BugsComments JOIN Bugs AS b USING (issue_id))
    USING (comment_id)
  LEFT OUTER JOIN (FeaturesComments JOIN FeatureRequests AS f USING (issue_id))
    USING (comment_id)
WHERE c.comment_id = 9876;

```

- 위 방법을 사용해서 뷰로 만들어두면 애플리케이션에서 간단하게 사용할 수 있다.



**공통 수퍼테이블 생성**

- 객체지향 다형성에서는 서브타입이 공통의 수퍼타입을 공유하기 때문에 두 서브타입을 비슷하게 참조할 수 있다.
- 모든 부모 테이블이 상속할 베이스테이블을 생성해 문제를 해결할 수 있다.

```sql
CREATE TABLE Issues (
  issue_id     SERIAL PRIMARY KEY
);

CREATE TABLE Bugs (
  issue_id     BIGINT UNSIGNED PRIMARY KEY,
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id),
  . . .
);

CREATE TABLE FeatureRequests (
  issue_id     BIGINT UNSIGNED PRIMARY KEY,
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id),
  . . .
);

CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  issue_id     BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME,
  comment      TEXT,
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id),
);

```

![7-3](https://github.com/sup2is/dev-note/blob/master/db/images/sql-antipatterns/7-3.jpeg)



- 특정 댓글이 참조하는 버그 또는 기능요청을 비교적 간단한 쿼리를 통해 조회할 수 있다.
- Issue 테이블에 속성을 정의하지 않았다면 쿼리에 이 테이블을 포함시킬 필요가 없다. Bugs 테이블과 Issues 테이블의 PK 값이 같기 때문에 Bugs를 Comments와 직접 조인할 수 있다. 
- 데이터베이스에서 칼럼이 서로 유사한 정보를 표현하는 경우에는 FK 제약조건으로 직접 연결되어 있지 않더라도 두 테이블을 조인할 수 있다.

```sql
-- START:comment
SELECT *
FROM Comments AS c
  LEFT OUTER JOIN Bugs AS b USING (issue_id)
  LEFT OUTER JOIN FeatureRequests AS f USING (issue_id)
WHERE c.comment_id = 9876;
-- END:comment
-- START:issue
SELECT *
FROM Bugs AS b
  JOIN Comments AS c USING (issue_id)
WHERE b.issue_id = 1234;
-- END:issue

```



**SQL Antipatterns Tip**

- 모든 테이블 관계에는 참조하는 테이블 하나, 참조되는 테이블 하나가 있다.

