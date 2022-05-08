# #15 애매한 그룹

## 목표: 그룹당 최댓값을 가진 행 얻기

- GROUP BY절은 다양한 종류의 복잡한 보고서를 상대적으로 적은 코드로 만들 수 있게 해주는 막강한 기능이다.

```sql
-- START:standard
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
-- END:standard

```

- 하지만 아래 쿼리는 에러가 발생하거나 결과를 신뢰할 수 없다. (bug_id)

```sql
-- START:withbugid
SELECT product_id, MAX(date_reported) AS latest, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id; 
-- END:withbugid
```



## 안티패턴: 그룹되지 않은 칼럼 참조

**단일 값 규칙**

- 각 그룹의 행은 GROUP BY 절 뒤에 쓴 칼럼(또는 칼럼 목록)의 값이 같은 행이다. 예를 들어 아래 쿼리는 각 product_id 값 마다 그룹이 생긴다.

```sql
-- START:standard
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
-- END:standard

```

- 쿼리에서 SELECT 목록에 있는 모든 칼럼은 그룹당 하나의 값을 가져야 한다. (단일 값 규칙 Single-Value Rule)
- GROUP BY 절 뒤에 쓴 칼럼들은 얼마나 많은 행이 그룹에 대응되는지에 상관없이 각 그룹당 정확히 하나의 값만 나오는 것이 보장된다.
- MAX() 함수 또한 각 그룹당 하나의 값만 내보낸다는 것이 보장된다.
- 그러나 SELECT 목록에 있는 다른 칼럼에 대해서는 데이터베이스 서버가 이를 확신할 수 없다. 이런 다른 칼럼에 대해서는 그룹 안에 모든 행에 같은 값이 나오는지를 보장할 수 없다.

```sql
-- START:withbugid
SELECT product_id, MAX(date_reported) AS latest, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
-- END:withbugid
```

- bug_id 같은 여분의 칼럼에 대해서는 그룹 내에서 값이 단일함을 보장할 수 없기 때문에 데이터베이스는 이들이 단일 값 규칙을 위반했다고 가정한다.
- 대부분의 데이터베이스 제품에서 위 쿼리는 에러를 발생시킨다.



**내 뜻대로 동작하는 쿼리**

- 대부분의 사람들은 쿼리가 최댓값을 얻을 때, 자연히 다른 칼럼의 값도 그 최댓값을 얻은 행에서 가져올 것이라고 가정한다.

- 그러나 다음과 같은 경우에는 이런 추론을 할 수 없다.

  - 두 버그의 date_reported 값이 동일하고 이 값이 그룹 내 최댓값이라면 쿼리에서 어느 bug_id 값을 보여줘야 하는가?

  - 쿼리에서 두 가지 다른 집계 함수, 예를 들어 MAX()와 MIN()을 사용한다면 이는 그룹 안에서 두 개의 다른 행에 대응될 것이다 이 그룹에서는 쿼리가 어느 bug_id값을 리턴해야 하는가?

  - ```sql
    SELECT product_id, MAX(date_reported) AS latest,
      MIN(date_reported) AS earliest, bug_id
    FROM Bugs JOIN BugsProducts USING (bug_id)
    GROUP BY product_id;
    
    ```

  - 집계 함수가 리턴하는 값과 매치되는 행이 없는 경우에는 bug_id값을 어떻게 해야 하는가?

    ```sql
    SELECT product_id, SUM(hours) AS total_project_estimate, bug_id
    FROM Bugs JOIN BugsProducts USING (bug_id)
    GROUP BY product_id;
    
    ```



## 안티패턴 인식 방법

- 대부분의 데이터베이스 제품에서는 단일 값 규칙을 위반하는 쿼리에 대해 즉각적인 에러를 발생시킨다.
- SQLite와 MySQL에서는 모호한 칼럼에 예상치 못한 신뢰할 수 없는 값이 들어갈 수 있다. (MySQL은 sql_mode로 제어 가능)
- 가능하다면 모호성을 회피하도록 쿼리를 설계하는 것이 좋다.



## 안티패턴 사용이 합당한 경우

- 단일 값 규칙에 맞지 않는 경우 결과를 신뢰할 수 없다. 하지만 데이터베이스가 규칙을 덜 엄격하게 강제한다는 사실을 이점으로 활용할 수 있는 경우도 있다.

```sql
SELECT b.reported_by, a.account_name
FROM Bugs b JOIN Accounts a ON (b.reported_by = a.account_id)
GROUP BY b.reported_by;

```

- 이 쿼리에서 account_name 칼럼은 GROUP BY 절에도 나오지 않고 집계 함수안에 있는 것도 아니기 때문에 원칙적으로는 단일 값 규칙을 위반했다.
- 그러나 각 그룹에 대해 account_name에 가능한 값은 하나뿐이다. reported_by 값을 알면 account_name도 명확하게 알 수 있기 때문에 가능하다.
- 이런 종류의 명확한 관계를 함수 종속이라 부른다.
- 함수 종속의 가장 일반적인 예는 테이블의 PK와 테이블 속성 간의 관계다. (account_name은 PK인 account_id에 함수종속이다.)
- Bugs.reported_by도 Accounts 테이블의 PK를 참조하므로 Accounts 테이블에 종속된 속성과 비슷한 관계를 가진다.
- 쿼리에서 외래키인 reported_by 칼럼으로 그룹을 지으면 Accounts 테이블의 속성은 함수 종속이고 쿼리에 모호한 결과가 포함되지 않는다.
- 하지만 대부분의 데이터베이스는 여전히 에러를 발생시키고 이런 쿼리 실행 시 함수 종속성을 판단하는 것은 비용도 많이 든다.



## 해법: 칼럼을 모호하게 사용하지 않기

**함수 종속인 칼럼만 쿼리하기**

- 가장 간단한 방법은 모호한 칼럼을 쿼리에서 제거하는 것이다.

```sql
-- START:standard
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
-- END:standard
```



**상호 연관된 서브쿼리 사용하기**

- 상호 연관된 서브쿼리는 바깥쪽 쿼리에 대한 참조를 가지고 있어 바깥쪽 쿼리의 각 행에 대해 다른 결과를 생성할 수 있다.
- 이를 이용해 서브쿼리가 그룹 내에서 날짜가 큰 버그를 찾게 해 각 제품별로 가장 최근에 보고된 버그를 찾을 수 있다.

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
WHERE NOT EXISTS
  (SELECT * FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
   WHERE bp1.product_id = bp2.product_id
     AND b1.date_reported < b2.date_reported);

```

- 그러나 상호 연관된 서브쿼리는 바깥쪽 쿼리의 각 행에 대해 한 번씩 실행되기 때문에 성능상 최적의 방법은 아님을 명심해야 한다.



**유도 테이블 사용하기**

- 서브쿼리를 유도테이블(derived table)로 사용해 각 제품에 대한 proudct_id와 버그 보고일자의 최댓값만 포함하는 일시결과를 만들 수 있다.
- 그런 다음 이 결과를 테이블과 조인해 쿼리 결과가 각 제품당 가장 최근의 버그만포함하게 하면 결과를 얻을 수 있다.

```sql
SELECT m.product_id, m.latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
  JOIN (SELECT bp2.product_id, MAX(b2.date_reported) AS latest
   FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
   GROUP BY bp2.product_id) m
  ON (bp1.product_id = m.product_id AND b1.date_reported = m.latest);

```

- 유도 테이블을 사용하는 방법은 상호 연관된 서브쿼리를 사용하는 방법보다 확장적용성이 좋은 대안이다.
- 대부분의 데이터베이스에서는 서브쿼리가 한 번만 수행되지만 여전히 임시테이블을 사용하므로 성능상 최적의 방법은 아니다.



**조인 사용하기**

- 대응되는 행이 없을 수도 있는 행의 집합에 대해 대응을 시도하는 조인을 시도할 수 있다. (OUTER JOIN)

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 ON (b1.bug_id = bp1.bug_id)
LEFT OUTER JOIN (Bugs AS b2 JOIN BugsProducts AS bp2 ON (b2.bug_id = bp2.bug_id))
  ON (bp1.product_id = bp2.product_id AND (b1.date_reported < b2.date_reported
    OR b1.date_reported = b2.date_reported AND b1.bug_id < b2.bug_id))
WHERE b2.bug_id IS NULL;

```

- 서브쿼리보다 확장적응성이 뛰어나기 때문에 대량 데이터에 대한 쿼리에서 확장적응성이 중요한 경우에는 조인을 사용하는게 좋다. 
- 하지만 어떤 방법이 다른 방법보다 성능이 좋다고 가정하지 말고 여러 형태의 쿼리에 대해 성능을 측정해 확인해야한다는 점을 기억해야 한다.



**다른 칼럼에 집계 함수 사용하기**

- 다른 칼럼에 집계 함수를 적용해 단일 값 규칙을 따르게 할 수도 있다.

```sql
SELECT product_id, MAX(date_reported) AS latest,
  MAX(bug_id) AS latest_bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;

```

- bug_id가 시간순으로 생성되는 경우에만 이 방법을 사용할 수 있다.



**각 그룹에 대해 모든 값을 연결하기**

- MySQL과 SQLite는 그룹에 속한 모든 값을 하나의 값으로 연결하는 GROUP_CONCAT()함수를 지원한다. 이 함수는 디폴트로 쉼표로 구분된 문자열을 만든다.

```sql
SELECT product_id, MAX(date_reported) AS latest
  GROUP_CONCAT(bug_id) AS bug_id_list,
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;

```

| product_id | latest     | bug_id_list      |
| ---------- | ---------- | ---------------- |
| 1          | 2010-06-01 | 1234, 2248       |
| 2          | 2010-02-16 | 3456, 4077, 5150 |
| 3          | 2010-01-01 | 5678, 8063       |

- 이 방법의 단점은 표준이 아니라는 점이다.



**SQL Antipatterns Tip**

- 모호한 쿼리 결과를 피하기 위해 단일 값 규칙을 따라라.




