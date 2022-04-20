# #3 순진한 트리

- 각 답글에 답을 달 수 있는 구조로 아래와 같이 테이블을 간단하게 설계했다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  parent_id    BIGINT UNSIGNED,
  comment      TEXT NOT NULL,
  FOREIGN KEY (parent_id) REFERENCES Comments(comment_id)
);

```



## 계층구조 저장 및 조회하기

- 데이터가 재귀적 관계를 가지는 것은 흔한 일이다. 데이터는 트리나 계층적 구조가 될 수 있다.
- 트리 데이터 구조에서 각 항목은 노드라 불리고 노드는 여러 개의 자식을 가질 수 있고 부모를 하나 가진다. 부모가 없는 최상위 노드를 root라고 하고 가장 아래에 있는 자식이 없는 노드를 leaf라고한다. 중간에 있는 노드는 그냥 노드다.
- 트리 데이터 구조를 가지는 예
  - 조직도
  - 글타래



## 안티패턴: 항상 부모에 의존하기

- 초보적 방법은 parent_id 칼럼을 추가하는 것이다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  parent_id    BIGINT UNSIGNED,
  bug_id       BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment      TEXT NOT NULL,
  FOREIGN KEY (parent_id) REFERENCES Comments(comment_id),
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

```





**인접 목록에서 트리 조회하기**

- 답글과 그 답글의 바로 아래 자식은 비교적 간단한 쿼리로 얻을 수 있지만 단 두 단계만 조회할 수 있다.

```sql
SELECT c1.*, c2.*
FROM Comments c1 LEFT OUTER JOIN Comments c2
  ON c2.parent_id = c1.comment_id;

```

- 트리의 특징 중 하나가 어느 깊이까지든 확장될 수 있다는 것이므로 단계에 상관없이 후손들을 조회할 수 있어야 한다.
- `COUNT()` 를 통해 답글 수를 계산하거나 `SUM()` 을 이용해 기계 조립에서 부품의 비용 합계를 구할 수 있어야한다.
- 또 다른 방법으로 글타래의 모든 행을 가져와서 애플리케이션에서 계층 구조를 만드는 방법인데 이 방법은 대량의 데이터를 애플리케이션으로 가져오는 것이기 때문에 비효율적이다.





**인접 목록에서 트리 유지하기**

- 새로운 노드를 추가하는 것과 같은 일부 연산은 간단하고 노드 하나 또는 서브트리를 이동하는 것 역시 쉽게 할 수 있다.

```sql
INSERT INTO Comments (bug_id, parent_id, author, comment)
  VALUES (1234, 7, 'Kukla', 'Thanks!');

UPDATE Comments SET parent_id = 3 WHERE comment_id = 6;
```

- 하지만 트리에서 노드를 삭제하는 작업은 복잡하고 비효율적이다.

```sql
SELECT comment_id FROM Comments WHERE parent_id = 4; -- returns 5 and 6
SELECT comment_id FROM Comments WHERE parent_id = 5; -- returns none
SELECT comment_id FROM Comments WHERE parent_id = 6; -- returns 7
SELECT comment_id FROM Comments WHERE parent_id = 7; -- returns none

DELETE FROM Comments WHERE comment_id IN ( 7 );
DELETE FROM Comments WHERE comment_id IN ( 5, 6 );
DELETE FROM Comments WHERE comment_id = 4;

```



## 안티패턴 인식 방법

- 다음과 같은 말을 듣는다면 순진한 트리 안티패턴이 사용되고 있음을 눈치챌 수 있다.
  - "트리에서 얼마나 깊은 단계를 지원해야 하지?"
  - "트리 데이터 구조를 관리하는 코드는 건드리는게 겁나"
  - "트리에서 고아 노드를 정리하기 위해 주기적으로 스크립트를 돌려야 해"



## 안티패턴 사용이 합당한 경우

- 인접 목록의 강점은 주어진 노드의 부모나 자식을 바로 얻어올 수 있고 새로운 노드를 추가하기도 쉽기 때문에 계층적 데이터로 작업하는 데 이 정도로만으로도 충분하다면 인접 목록은 적절한 방법이다.
- WITH 키워드, CTE(Common Table Expression)을 사용한 재귀적 쿼리 문법을 사용할 수 있는 DBMS라면 순진한 트리 구조를 사용해도 제약은 없어진다.

```sql
-- WITH절
WITH CommentTree
    (comment_id, bug_id, parent_id, author, comment, depth)
AS (
    SELECT *, 0 AS depth FROM Comments
    WHERE parent_id IS NULL
  UNION ALL
    SELECT c.*, ct.depth+1 AS depth FROM CommentTree ct
    JOIN Comments c ON (ct.comment_id = c.parent_id)
)
SELECT * FROM CommentTree WHERE bug_id = 1234;

-- START WITH ... CONNECT BY PRIOR
SELECT * FROM Comments
START WITH comment_id = 9876
CONNECT BY PRIOR parent_id = comment_id;
```

- 초판이 2011년 인걸 감안하면 이해가 가능하지만 2022년인 지금 재귀적 쿼리를 지원하지 않는 dbms는 거의 없다는게 함정



## 해법: 대안 트리 모델 사용

- 계층적 데이터를 저장하는 데는 인접 목록 모델(순진한 트리) 외에도 경로 열거(Path Enumeration), 중첩 집합(Nested Sets), 클로저 테이블(Closure Table)과 같은 몇가지 대안이 있다.



**경로 열거**

- 경로 열거 방법에서는 일련의 조상을 각 노드의 속성으로 저장해 이를 해결한다.
- 디렉토리 구조에서도 경로 열거 형태를 볼 수 있다.
  - /usr/local/lib 
    - /usr는 local의 부모
    - /local은 lib의 부모
- 기존 parent_id 칼럼 대신 path란 칼럼을 정의한다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  path         VARCHAR(1000),
  bug_id       BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment      TEXT NOT NULL,
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

```



| comment_id | path    | author | comment |
| ---------- | ------- | ------ | ------- |
| 1          | 1/      | A      | A       |
| 2          | 1/2/    | B      | B       |
| 3          | 1/2/3/  | C      | C       |
| 4          | 1/4/    | D      | D       |
| 5          | 1/4/5/  | E      | E       |
| 6          | 1/4/6/  | F      | F       |
| 7          | 1/4/6/7 | G      | G       |

- 위와 같은 형태에서 경로가 /1/4/6/7/ 인 답글 #7의 조상을 찾으려면 다음과 같이 할 수 있다.

```sql
SELECT *
FROM Comments AS c
WHERE '1/4/6/7/' LIKE c.path || '%';

-- 1/4/6/%, 1/4/%, 1/% 로 매칭
```

- 반대로 경로가 1/4/인 답글 #4의 후손을 찾으려면 다음과 같이 할 수 있다.

```sql
SELECT *
FROM Comments AS c
WHERE c.path LIKE '1/4/' || '%';

-- /1/4/5, 1/4/6, 1/4/6/7 매칭
```

- 특정 노드의 서브트리에서 글쓴이당 답글 수를 세는 집계함수역시 쉽게 사용할 수 있다.

```sql
SELECT COUNT(*)
FROM Comments AS c
WHERE c.path LIKE '1/4/' || '%'
GROUP BY c.author;

```

- 새로운 노드를 추가하는 방법은 인접 목록 모델에서와 비슷하게 부모 경로 + 새로운 노드의 아이디를 덧붙이는 방법으로 추가할 수 있다.

```sql
-- MySQL의 LAST_INSERT_ID()를 사용해서 path 수정
INSERT INTO Comments (author, comment) VALUES ('Ollie', 'Good job!');
UPDATE Comments 
  SET path = (SELECT path FROM Comments WHERE comment_id = 7)
    || LAST_INSERT_ID() || '/'
WHERE comment_id = LAST_INSERT_ID();
```

- 경로 열거 방법은 경로가 올바르게 형성되도록 하거나 값이 실제 노드에 대응되도록 강제하는 방법은 없다. 경로 문자열을 유지하는 것은 애플리케이션 코드에 종속되며 이를 검증하는데는 비용이 많이 든다.
- 경로 열거 방법은 VARCHAR 칼럼의 길이만큼 제한이 있기 때문에 무제한 확장은 불가능하다.



**중첩 집합**

- 중첩 집합은 각 노드가 자신의 부모를 저장하는 대신 자기 자손의 집합에 대한 정보를 저장한다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  nsleft       INTEGER NOT NULL,
  nsright      INTEGER NOT NULL,
  bug_id       BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment      TEXT NOT NULL,
  FOREIGN KEY (bug_id) REFERENCES Bugs (bug_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

```

- 이 정보는 트리의 각 노드를 두 개의 수로 부호화해 나타낼 수 있는데 여기에선 nsleft와 nsright로 구분한다.
- 각 노드의 nsleft와 nsright 수는 다음과 같이 주어진다.
  - nsleft 는 모든 자식 노드의 nsleft 보다 작아야 한다.
  - nsright 는 모든 자식의 노드의 nsright 보다 커야 한다.
  - 이 수들은 comment_id 값과는 아무런 상관이 없다.



![3-1](https://github.com/sup2is/dev-note/blob/master/db/images/sql-antipatterns/3-1.jpeg)



| comment_id | nsleft | nsright | author | comment |
| ---------- | ------ | ------- | ------ | ------- |
| 1          | 1      | 14      | A      | A       |
| 2          | 2      | 5       | B      | B       |
| 3          | 3      | 4       | C      | C       |
| 4          | 6      | 13      | D      | D       |
| 5          | 7      | 8       | E      | E       |
| 6          | 9      | 12      | F      | F       |
| 7          | 10     | 11      | G      | G       |

- nsleft 값이 현재 노드의 nsleft와 nsright 사이에 있는 노드를 검색하면 답글 #4 와 그 자손을 얻을 수 있다.

```sql
SELECT c2.*
FROM Comments AS c1
  JOIN Comments as c2
    ON c2.nsleft BETWEEN c1.nsleft AND c1.nsright
WHERE c1.comment_id = 4;

```

- 답글 #6과 그 조상은 nsright 값이 현재 노드의 숫자 사이에 있는 노드를 검색해 얻을 수 있다.

```sql
SELECT c2.*
FROM Comments AS c1
  JOIN Comment AS c2
    ON c1.nsleft BETWEEN c2.nsleft AND c2.nsright
WHERE c1.comment_id = 6;

```

- 중첩 집합 모델의 주요 강점 중 하나는 자식을 가진 노드를 삭제했을 때 그 자손이 자동으로 삭제된 노드 부모의 자손이 된다. 노드를 삭제해도 트리 구조에서는 아무런 문제가 없다.

```sql
-- Reports depth = 3
SELECT c1.comment_id, COUNT(c2.comment_id) AS depth
FROM Comment AS c1
  JOIN Comment AS c2
    ON c1.nsleft BETWEEN c2.nsleft AND c2.nsright
WHERE c1.comment_id = 7
GROUP BY c1.comment_id;

DELETE FROM Comment WHERE comment_id = 6;

-- Reports depth = 2
SELECT c1.comment_id, COUNT(c2.comment_id) AS depth
FROM Comment AS c1
  JOIN Comment AS c2
    ON c1.nsleft BETWEEN c2.nsleft AND c2.nsright
WHERE c1.comment_id = 7
GROUP BY c1.comment_id;

```

- 중첩 집합 모델의 단점은 부모 노드를 얻는 과정이 아래와 같이 복잡해진다.

```sql
SELECT parent.*
FROM Comment AS c
  JOIN Comment AS parent
    ON c.nsleft BETWEEN parent.nsleft AND parent.nsright
  LEFT OUTER JOIN Comment AS in_between
    ON c.nsleft BETWEEN in_between.nsleft AND in_between.nsright
    AND in_between.nsleft BETWEEN parent.nsleft AND parent.nsright
WHERE c.comment_id = 6
  AND in_between.comment_id IS NULL;

```

- 중첩 집합 모델에서는 노드를 추가, 이동하는 것과 같은 트리 조작도 다른 모델을 사용할 때보다 복잡하다. 새로운 노드를 추가할 때마다 새 노드의 왼쪽 값보다 큰 모든 노드의 왼쪽, 오른쪽 값을 다시 계산해야 한다.

```sql
-- make space for NS values 8 and 9
UPDATE Comment
  SET nsleft = CASE WHEN nsleft >= 8 THEN nsleft+2 ELSE nsleft END,
      nsright = nsright+2
WHERE nsright >= 7;

-- create new child of comment #5, occupying NS values 8 and 9
INSERT INTO Comment (nsleft, nsright, author, comment)
  VALUES (8, 9, 'Fran', 'Me too!');

```

- 중첩 집합 모델은 각 노드에 대해 조작하는 것 보다는 서브트리를 쉽고 빠르게 조회하는 것이 중요할 때 가장 잘 맞는다.
- 트리에 노드를 삽입하는 과정이 복잡하기 때문에 쓰기가 빈번하다면 중첩 집합은 좋은 선택이 아니다.



**클로저 테이블**

- 클로저 테이블은 부모-자식 관계에 대한 경로만을 저장하는 것이 아니라 트리의 모든 경로를 저장한다.

```sql
CREATE TABLE Comments (
  comment_id   SERIAL PRIMARY KEY,
  bug_id       BIGINT UNSIGNED NOT NULL,
  author       BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment      TEXT NOT NULL,
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

CREATE TABLE TreePaths (
  ancestor    BIGINT UNSIGNED NOT NULL,
  descendant  BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY(ancestor, descendant),
  FOREIGN KEY (ancestor) REFERENCES Comments(comment_id),
  FOREIGN KEY (descendant) REFERENCES Comments(comment_id)
);

```

- 트리 구조에 대한 정보를 Comments 테이블에 저장하는 대신 TreePaths를 사용한다.
- 이 테이블에는 트리에서 조상/자손 관계를 가진 모든 노드 쌍을 한 행으로 저장한다. 또한 각 노드에 대해 자기 자신을 참조하는 행도 추가한다.



![3-2](https://github.com/sup2is/dev-note/blob/master/db/images/sql-antipatterns/3-2.jpeg)



| ancestor | descendant |
| -------- | :--------- |
| 1        | 1          |
| 1        | 2          |
| 1        | 3          |
| 1        | 4          |
| 1        | 5          |
| 1        | 6          |
| 1        | 7          |
| 2        | 2          |
| 2        | 3          |
| 3        | 3          |
| 4        | 4          |
| 4        | 5          |
| 4        | 6          |
| 4        | 7          |
| 5        | 5          |
| 6        | 6          |
| 6        | 7          |
| 7        | 7          |



- 답글 #4의 자손을 얻으려면 TreePaths에서 ancestor가 4인 행을 가져오면 된다.

```sql
SELECT c.*
FROM Comments AS c
  JOIN TreePaths AS t ON c.comment_id = t.descendant
WHERE t.ancestor = 4;

```

- 답글 #6의 조상을 얻으려면 TreePath에서 descendant가 6인 행을 가져오면 된다.

```sql
SELECT c.*
FROM Comments AS c
  JOIN TreePaths AS t ON c.comment_id = t.ancestor
WHERE t.descendant = 6;

```

- 새로운 종말 노드, 예를 들어 답글 #5에 새로운 자식을 추가하려면 먼저 자기 자신을 참조하는 행을 추가하고 TreePaths에서 답글 #5를  descendant로 참조하는 모든 행을 복사해 descendant를 새로운 답글 아이디로 바꿔 넣는다.

```sql
INSERT INTO TreePaths (ancestor, descendant)
  SELECT t.ancestor, 8
  FROM TreePaths AS t
  WHERE t.descendant = 5
 UNION ALL
  SELECT 8, 8;

```

- 종말 노드를 삭제할 때, 예를 들어 #7을 삭제할 때는 TreePaths에서 descendant로 참조하는 모든 행을 삭제한다.

```sql
DELETE FROM TreePaths WHERE descendant = 7;

```

- 서브트리, 예를들어 답글#4와 그 자손을 삭제하려면 TreePaths에서 답글 #4를 descendant로 참조하는 모든 행과  답글 #4의 자손을 descendant로 참조하는 모든 행을 삭제한다.

```sql
DELETE FROM TreePaths
WHERE descendant IN (SELECT descendant
		     FROM TreePaths
		     WHERE ancestor = 4);

```

- 서브트리를 트리 내 다른 위치로 이동하고자 할 때는, 먼저 서브트리의 최상위 노드와 그 노드의 자손들을 참조하는 행을 삭제해 서브트리와 그 조상의 연결을 끊는다.
- 예를 들어 답글 #6을 답글 #4의 자식에서 #3의 자식으로 옮기려면 아래와 같이 한다.

```sql
-- START:delete
DELETE FROM TreePaths
WHERE descendant IN (SELECT descendant
		     FROM TreePaths
		     WHERE ancestor = 6)
  AND ancestor IN (SELECT ancestor
		   FROM TreePaths
		   WHERE descendant = 6
		     AND ancestor != descendant);
-- END:delete
-- 위 쿼리는 (1, 6), (1, 7), (4, 6), (4, 7) 을 삭제한다. 즉 자기 자신과 자기 자신이 갖고 있는 서브트리 외에 모든 참조를 제거한다.

-- START:reinsert
INSERT INTO TreePaths (ancestor, descendant)
  SELECT supertree.ancestor, subtree.descendant
  FROM TreePaths AS supertree
    CROSS JOIN TreePaths AS subtree
  WHERE supertree.descendant = 3
    AND subtree.ancestor = 6;
-- END:reinsert
-- 새로운 위치의 조상들과 서브트리의 자손들에 대응하는 행을 추가해서 고아가 된 서브트리를 붙인다.
-- 위 쿼리는 (1, 6), (2, 6), (3, 6), (1, 7), (2, 7), (3, 7) 경로를 새로 생성한다.

```

- 클로저 테이블 모델은 중첩 집합 모델보다 직관적이다. 조상과 자손을 조회하는 것은 두 방법 모두 빠르고 쉽지만 클로저 테이블이 계층구조 정보를 유지하기가 쉽다. 
- 두 방법 모두 인접 목록이나 경록 열거 방법보다 자식이나 부모를 조회하기 편리하다.
- 부모나 자식 노드를 더 쉽게 조회할 수 있도록 TreePaths에 path_length 속성을 추가하면 클로저 테이블을 개선할 수 있다. 
- 자기자신의 path_length는 0, 자식의 path_length는 1 ... 2 형태로 구성하면 아래와 같이 자식을 쉽게 조회할 수 있다.

```sql
SELECT *
FROM TreePaths
WHERE ancestor = 4 AND path_length = 1;

```



**어떤 모델을 사용해야 하는가?**

- 인접 목록은 가장 흔히 사용되는 모델로 많은 소프트웨어 개발자가 알고 있다.
- WITH나 CONNET BY PRIOR를 이용한 재귀적 쿼리는 인접 목록 모델을 좀 더 효율적으로 만들지만 이 문법을 지원하는 데이터베이스를 써야 한다.
- 경로 열거는 브레드크럼을 사용자 인터페이스에 보여줄 때는 좋지만 참조 정합성을 강제하지 못하고 정보를 중복 저장하기 때문에 깨지기 쉬운 구조다.
- 중첩 집합은 트리를 수정하는 일은 거의 없고 조회를 많이 하는 경우 적합하다. 역시 참조 정합성을 지원하지는 못한다.
- 클로저 테이블은 가장 융통성 있는 모델이고 한 노드가 여러 트리에 속하는 것을 허용하는 유일한 모델이다. 클로저 테이블은 별도의 저장공간을 사용하기 때문에 계산을 줄이는 대신 저장공간을 많이 사용하는 트레이드오프가 발생한다.

<br>

| 모델          | 테이블 | 자식 조회 | 트리 조회 | 삽입   | 삭제   |
| ------------- | ------ | --------- | --------- | ------ | ------ |
| 인접 목록     | 1      | 쉽다      | 어렵다    | 쉽다   | 쉽다   |
| 재귀적 쿼리   | 1      | 쉽다      | 쉽다      | 쉽다   | 쉽다   |
| 경로 열거     | 1      | 쉽다      | 쉽다      | 쉽다   | 쉽다   |
| 중첩 집합     | 1      | 어렵다    | 쉽다      | 어렵다 | 어렵다 |
| 클로저 테이블 | 2      | 쉽다      | 쉽다      | 쉽다   | 쉽다   |



**SQL Antipatterns Tip**

- 계층구조에는 항목과 관계가 있다. 작업에 맞도록 이 둘을 모두 모델링해야 한다.



