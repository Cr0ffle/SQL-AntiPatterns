# Chapter 14. 모르는 것에 대한 두려움

- 예제: 사용자의 전체 이름을 하나의 칼럼처럼 포매팅 (Accounts)
    
    ```sql
    SELECT first_name || ' ' || last_name AS full_name FROM Accounts;
    +-----------+
    | full_name |
    +-----------+
    | 0         |
    | 0         |
    +-----------+
    
    SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM Accounts;
    +---------------+
    | full_name     |
    +---------------+
    | test person   |
    | second person |
    +---------------+
    ```
    
- 사용자의 중간 이름의 첫 글자를 테이블에 저장하도록 데이터베이스를 수정
    - 중간 이름 첫 자를 지정한 사용자의 이름은 정상적으로 표시되지만, 다른 사용자의 이름은 빈칸으로 표시된다.
        
        ```sql
        ALTER TABLE Accounts ADD COLUMN middle_initial CHAR(2);
        
        UPDATE Accounts SET middle_initial = 'J.' WHERE account_id = 123;
        
        SELECT concat(first_name, ' ', middle_initial, ' ', last_name) AS full_name
          FROM Accounts;
        +------------------+
        | full_name        |
        +------------------+
        | NULL             |
        | second J. person |
        +------------------+
        ```
        

## 1. 목표: 누락된 값을 구분하기

---

- 데이터베이스의 어떤 데이터에 값이 없는 것은 피할 수 없다 → SQL 은 특수한 값인 `NULL`을 지원한다.****
- 목표는 `NULL`을 포함하는 칼럼에 대한 쿼리를 작성하는 것

## 2. 안티패턴: `NULL`을 일반 값처럼 사용

---

- 대부분의 프로그래밍 언어와 달리, SQL 에서는 `NULL`을 0이나 `false` 또는 빈 문자열과 다른 특별한 값으로 취급한다.

### 수식에서 `NULL` 사용

---

- `NULL` 값을 가지는 칼럼이나 수식에 대해 산술 연산을 수행했을 때 👉 `NULL`이 리턴된다.
    - 😰 `NULL`은 0과 같지 않다. `NULL`은 길이가 0인 문자열과도 같지 않다.
        
        ```sql
        -- NULL + 10 = NULL
        -- '문자열: ' + NULL = NULL
        SELECT hours + 10 FROM Bugs;
        +------------+
        | hours + 10 |
        +------------+
        |      13.00 |
        |       NULL |
        +------------+
        ```
        
    - 😰 `NULL`은 `false`와도 같지 않다.
        - → `NULL`이 들어간 불리언 수식은 `AND`, `OR`, `NOT`을 사용하더라도 항상 `NULL`
            
            ```sql
            -- false AND NULL = NULL
            SELECT true AND NULL;
            +---------------+
            | true AND NULL |
            +---------------+
            |          NULL |
            +---------------+
            ```
            

### `NULL`을 가질 수 있는 칼럼 검색

---

- 검색 조건인 행만을 리턴하고, 다른 값을 가지거나 칼럼이 `NULL`인 행은 리턴하지 않는다.
    
    ```sql
    SELECT * FROM Bugs WHERE assigned_to = 123;
    +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
    | issue_id | bug_id | reported_by | product_id | priority | version_resolved | status | severity | version_affected | hours | assigned_to |
    +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
    |        1 |      1 |          12 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  3.00 |         123 |
    +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
    ```
    
- 다음 쿼리는 위 쿼리의 여집합, 즉 위 쿼리에서 리턴한 행 이외의 모든 행이 리턴되리라 생각할 것이다.
    - 😰 그러나 칼럼의 값이 `NULL`인 행은 리턴하지 않는다.
        
        ```sql
        SELECT * FROM Bugs WHERE NOT (assigned_to = 123);
        Empty set (0.00 sec)
        ```
        
- 👉 `NULL`과는 어떤 비교를 하든 그 결과는 `NULL`이다.
    - 심지어 `NULL`에 `NOT`을 해도 여전히 `NULL`이다.

- `NULL`인 행 또는 `NULL`이 아닌 행을 찾을 때 다음과 같은 실수를 한다.
    - 😰 `WHERE` 절 조건은 수식이 `true`일 때만 만족 → `NULL`과의 비교는 절대 `true`가 되는 법이 없다.
        
        ```sql
        -- 쿼리는 둘 다 assigned_to 가 NULL 인 행을 리턴하지 않는다.
        SELECT * FROM Bugs WHERE assigned_to = NULL;
        Empty set (0.00 sec)
        
        SELECT * FROM Bugs WHERE assigned_to <> NULL;
        Empty set (0.00 sec)
        ```
        

### 쿼리 파라미터로 `NULL` 사용

---

- 파라미터를 받는 SQL 에서는 `NULL`을 다른 일반적인 값처럼 사용하기가 어렵다.
    - 😰 일반 정수를 넣어주면 예측 가능한 결과를 리턴하지만, `NULL`을 파라미터로 사용할 수 없다.
        
        ```sql
        SELECT * FROM Bugs WHERE assigned_to = ?;
        ```
        

### 문제 회피하기

---

- 💡 데이터베이스에서 `NULL`을 허용하지 않도록 한다. 대신 뜻을 강조할 수 있는 다른 값을 선택한다.
    - 😰 각 칼럼마다 상황에 맞게 다른 값을 선택해야 한다.
        - -1 은 `SUM()`이나 `AVG()` 같은 계산을 할 때 포함된다.
            
            ```sql
            INSERT INTO Bugs (assigned_to, hours) VALUES (-1, -1);
            ```
            
            - 이런 값을 가진 행은 제외시켜야 하는데, 이는 `NULL` 사용을 금지하면서 피하려 했던 것
                
                ```sql
                SELECT AVG( hours ) AS average_hours_per_bug
                  FROM Bugs
                 WHERE hours <> -1;
                ```
                
        - 각 칼럼마다 사용되는 특별한 값을 기억하거나 문서화해야 한다 → 불필요한 작업을 추가하는 것
    - 😰 실제 사용자 계정에 대한 참조가 없다는 것을 표시하기 위해 계정을 생성해야 한다.(FK)
        - Accounts 에 ‘아무도 없음’ 또는 ‘할당 안 됨’과 같은 뜻을 가지는 행을 생성해줘야 한다.
- 칼럼을 `NOT NULL`로 선언 → 해당 칼럼에 값이 없는 상태로 행이 존재하는 것이 의미가 없기 때문이어야 한다.
    - Bugs.reported_by 칼럼은 값을 가져야 한다 → 모든 버그는 누군가에 의해 보고되었기 때문

## 3. 안티패턴 인식 방법

---

- "assigned_to(또는 다른) 칼럼에 아무 값도 설정되지 않은 행을 어떻게 찾을 수 있지?"
    
    → `NULL`에 대해서는 동등연산자를 사용할 수 없다.
    
- "애플리케이션에서 몇몇 사용자의 전체 이름이 표시되지 않아. 데이터베이스에서는 분명 볼 수 있는데."
    
    → 아마 문자열과 `NULL`을 연결해 `NULL`이 되어 버린 문제
    
- "이 프로젝트의 전체 작업시간 보고서에 우리가 완료한 몇몇 버그만 포함되어 있어! 그러니까 우선순위를 할당한 것들만 포함이 되어 있네"
    
    → 아마도 시간 합계를 구하는 집계 쿼리의 `WHERE` 절에, priority 칼럼 값이 `NULL`인 경우 `true`가 되지 않는 수식을 포함하고 있을 것이다.
    
- "Bugs 테이블에서 '알 수 없음'을 나타내는 데 예전에 사용하던 문자열을 사용할 수 없다는 것을 확인했기 때문에, 다른 어떤 값을 사용해야 할 지 그리고 데이터를 변환해서 우리 코드가 새로운 값을 사용하도록 하는데 개발 기간이 얼마나 필요할지 논의하기 이한 회의가 필요해."
    
    → 합당한 값이 될 수 있는 특별한 플래그 값을 할당했을 때 생길 수 있는 결과
    

## 4. 안티패턴 사용이 합당한 경우

---

- `NULL`을 사용하는 것은 안티패턴이 아니다.
- `NULL`을 일반적인 값처럼 사용하거나 일반적인 값을 `NULL`처럼 사용하는 것이 안티패턴이다.
    - `NULL`을 일반적인 값처럼 취급해야 하는 경우****
        - 외부 데이터를 불러오거나(import) 또는 데이터를 내보내기(export) 할 때
            - ex> MySQL 의 mysqlimport 에서는 \N 이 `NULL`을 나타낸다.
    - 일반적인 값을 `NULL`처럼 사용하는 것
        - 사용자 입력 또한 `NULL`을 직접 나타낼 수는 없다.
            - 애플리케이션에서는 어떤 특별한 입력이 `NULL`로 매핑되도록 할 수 있다.
            - ex> .NET 2.0 이상의 `ConvertEmptyStringToNull` 속성: 빈 문자열을 자동으로 `NULL`로 변환
        - 누락된 값에 여러 가지 구분이 있을 때는 `NULL`을 사용할 수 없다.
            - ex> 한 번도 할당되지 않은 버그 vs 할당되었지만 그 사람이 프로젝트를 떠나 할당되지 않은 상태로 바뀐 버그 → 각 상태에 대해 구별되는 값을 사용해야 한다.

## 5. 해법: 유일한 값으로 `NULL`을 사용하라

---

- `NULL`값과 관련된 대부분의 문제는 SQL 의 세 가지 값 로직의 동작을 제대로 이해하지 못한 데서 비롯된다.

### 스칼라 수식에서의 `NULL`

---

- 프로그래머가 기대하는 것과 다른 결과가 나오는 몇 가지 경우
    
    
    | 수식 | 기대 값 | 실제 값 | 이유 |
    | --- | --- | --- | --- |
    | NULL = 0 | TRUE | NULL | NULL은 0이 아니다. |
    | NULL = 12345 | FALSE | NULL | 지정된 값이 모르는 값과 같은지 알 수 없다. |
    | NULL <> 12345 | TRUE | NULL | 또한 다른지도 알 수 없다. |
    | NULL + 12345 | 12345 | NULL | NULL은 0이 아니다. |
    | NULL || ‘string’ | 'string' | NULL | NULL은 빈 문자열이 아니다. |
    | NULL = NULL | TRUE | NULL | 모르는 값과 모르는 값이 같은지 알 수 없다 |
    | NULL <> NULL | FALSE | NULL | 또한 다른지도 알 수 없다. |

### 불리언 수식에서의 `NULL`

---

- `NULL`이 `true`도 `false`도 아니라는 것!
- 프로그래머가 기대하는 것과 다른 결과가 나오는 몇 가지 경우
    
    
    | 수식 | 기대 값 | 실제 값 | 이유 |
    | --- | --- | --- | --- |
    | NULL AND TRUE | FALSE | NULL | NULL은 false가아니다. |
    | NULL AND FALSE | FALSE | FALSE | 어떤 진리 값이든 FALSE와 AND를 하면 false다 |
    | NULL OR FALSE | FALSE | NULL | NULL은 false가 아니다. |
    | NULL OR TRUE | TRUE | TRUE | 어떤 진리 값이든 TRUE와 OR를하면 true다 |
    | NOT (NULL) | TRUE | NULL | NULL은 false가 아니다. |

### `NULL` 검색하기

---

- SQL 표준에는 `IS NULL` 연산자가 정의되어 있는데, 피연산자가 `NULL`이면 `true`를 리턴한다.
    - 반대로 `IS NOT NULL`은 피연산자가 `NULL`이면 `false`를 리턴한다.
        
        ```sql
        SELECT * FROM Bugs WHERE assigned_to IS NULL;
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        | issue_id | bug_id | reported_by | product_id | priority | version_resolved | status | severity | version_affected | hours | assigned_to |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        |        2 |      2 |         123 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  NULL |        NULL |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        
        SELECT * FROM Bugs WHERE assigned_to IS NOT NULL;
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        | issue_id | bug_id | reported_by | product_id | priority | version_resolved | status | severity | version_affected | hours | assigned_to |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        |        1 |      1 |          12 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  3.00 |         123 |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        ```
        

- SQL-99 표준에서는 `IS DISTINCT FROM`이란 또 다른 비교연산자
    - 일반 비교연산자인 `<>`와 비슷하게 동작한다.
        - 같지 않다.
    - 다른 점은 피연산자가 `NULL`이더라도 항상 `true` 또는 `false`를 리턴한다는 것이다.
        
        ```sql
        -- 다음 두 쿼리는 동일하다.
        SELECT * FROM Bugs WHERE assigned_to IS NULL OR assigned_to <> 1;
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        | issue_id | bug_id | reported_by | product_id | priority | version_resolved | status | severity | version_affected | hours | assigned_to |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        |        1 |      1 |          12 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  3.00 |         123 |
        |        2 |      2 |         123 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  NULL |        NULL |
        +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
        
        -- 값을 비교하기 전에 IS NULL 로 확인을 해야 하는 지겨운 수식을 쓰지 않아도 된다.
        SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM 1;
        ```
        
        - MySQL 은 `IS DISTINCT FROM`처럼 동작하는 전용 연산자 `<=>`를 제공한다.
            - `NULL` safe equal 연산자 ([레퍼런스](https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html#operator_equal-to))
            
            ```sql
            -- 1
            SELECT 'a' <=> 'a';
            -- 0
            SELECT 'a' <=> 'b';
            SELECT * FROM Bugs WHERE NOT (assigned_to <=> 1);
            +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
            | issue_id | bug_id | reported_by | product_id | priority | version_resolved | status | severity | version_affected | hours | assigned_to |
            +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
            |        1 |      1 |          12 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  3.00 |         123 |
            |        2 |      2 |         123 |       NULL | NULL     | NULL             | NULL   | NULL     | NULL             |  NULL |        NULL |
            +----------+--------+-------------+------------+----------+------------------+--------+----------+------------------+-------+-------------+
            ```
            
    - 쿼리 파라미터로 리터럴 값이나 `NULL`을 보내고 싶을 때 이 연산자를 사용할 수 있다.
        
        ```sql
        SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM ?;
        ```
        

### 칼럼을 `NOT NULL`로 선언하기

---

- `NULL` 값이 애플리케이션 정책을 위반하거나 또는 의미가 없는 경우에는 칼럼에 `NOT NULL` 제약조건을 선언하는 것이 권장사항이다.
- 칼럼을 생략하더라도 `NULL` 디폴트 값이 들어가도록 모든 칼럼에 대해 `DEFAULT`를 정의하도록 권고한다.
    - 어떤 칼럼에 대해서는 좋은 충고가 되겠지만 모든 칼럼에 그런 것은 아니다.
    - 칼럼이 논리적인 디폴트 값을 가지지 않더라도 `NOT NULL` 제약조건이 필요한 경우는 정당하고 흔한 것이다.

### 동적 디폴트

---

- `coalesce()` 함수: 주어진 칼럼이나 수식에, 특히 특정 쿼리에서만 디폴트 값을 설정하는 방법
    - 표준 SQL 함수다.
    - 가변 인수를 받고 `NULL`이 아닌 첫 인수를 리턴한다.
        
        ```sql
        -- NULL 값으로 인해 전체 식이 NULL이 되지 않도록 coalesce()를 사용
        SELECT CONCAT( first_name
                     , COALESCE(CONCAT(' ', middle_initial, ' '), ' ')
                     , last_name ) AS full_name
          FROM Accounts;
        +------------------+
        | full_name        |
        +------------------+
        | test person      |
        | second J. person |
        +------------------+
        ```
        

<aside>
💡 어떤 데이터 타입에 대해서든 누락된 값을 뜻하는 데는 `NULL`을 사용하라.

</aside>

- 참고
    - [https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md](https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md)
    - [https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-03쿼리.md](https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-03%EC%BF%BC%EB%A6%AC.md)
    - [http://wiki.gurubee.net/pages/viewpage.action?pageId=15794210](http://wiki.gurubee.net/pages/viewpage.action?pageId=15794210)