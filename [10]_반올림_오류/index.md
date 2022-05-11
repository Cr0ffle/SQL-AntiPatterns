# Chapter 10. 반올림 오류

## 1. 목표: 정수 대신 소수 사용

---

- 예제: 프로그래머 비용을 계산한 보고서
    
    ```sql
    SELECT b.bug_id, b.hours * a.hourly_rate AS cost_per_bug
      FROM Bugs AS b
      JOIN Accounts AS a ON (b.assigned_to = a.account_id);
    ```
    
- 비용을 정확하게 추적하기 위해, 두 칼럼 모두 소수를 지원해야 한다.
    - `FLOAT` 칼럼 추가 👉 정확하지 않음
        
        ```sql
        ALTER TABLE Bugs ADD COLUMN hours FLOAT;
        
        ALTER TABLE Accounts ADD COLUMN hourly_rate FLOAT;
        ```
        
- 목표는 정수가 아닌 수를 저장하고 이를 산술 연산에 사용하는 것

## 2. 안티패턴: `FLOAT` 데이터 타입 사용

---

- SQL 의 `FLOAT` 데이터 타입은 다른 프로그래밍 언어의 `float`와 마찬가지로 IEEE 754 표준에 따라 실수를 이진 형식으로 부호화한다.

### 필요에 의한 반올림

---

- 부동 소수점 수의 특성 👉 십진수로 표현된 모든 수를 이진수로 표현할 수는 없다.
    - ex> 1/3과 같은 유리수와 0.333...과 같은 순환소수로 표현된 수를 비교 → 십진수 소수로는 표현할 수 없다.
        - 1/3 + 1/3 + 1/3 = 1
        - 0.333 + 0.333 + 0.333 = 0.999
            - 유한소수를 사용하고 0.333과 같이 원래의 값에 가능한 가까운 값을 선택하는 것
- [IEEE 754](https://ko.wikipedia.org/wiki/IEEE_754) 는 부동 소수점 수를 밑수가 2인 형식으로 표현한다.
    - 이진수로 무한한 정도를 요하는 값과 십진수에서 무한한 정도를 요구하는 수는 다르다.
    - ex> 59.95 와 같이 십진수에서 유한한 정도를 가지는 값 → 이진수로 표현하려면 무한한 정도가 필요하다.
        - 밑수를 2로 하는 가장 가까운 값을 사용해 저장하는데, 이는 밑수를 10으로 했을 때 59.940000762929 와 같다.
- 데이터베이스에 `FLOAT`로 오차 없이 표현되는 값만 저장되어 있다고 할 수는 없으므로, 애플리케이션에서는 이런 칼럼에 있는 어떤 값이든 반올림되었다고 가정해야 한다.

### SQL에서 `FLOAT` 사용

---

- 어떤 데이터베이스는 부정확한 값을 보정해, 의도한 값을 표시해준다.
    
    ```sql
    -- 이쿼리는 59.95를 리턴한다.
    SELECT hourly_rate FROM Accounts WHERE account_id = 12;
    +-------------+
    | hourly_rate |
    +-------------+
    |       59.95 |
    +-------------+
    ```
    
    - 😰 그러나 `FLOAT`에 실제로 저장되어 있는 값은 이 값과 정확하게 같지는 않다 → 값이 IEEE 754 의 이진 형식에 따라 유한 정도로 표현 가능한 값으로 반올림되었음
        
        ```sql
        -- 이 값에 10억을 곱해보면 불일치를 확인할 수 있다.
        SELECT hourly_rate * 1000000000 FROM Accounts WHERE account_id = 12;
        +--------------------------+
        | hourly_rate * 1000000000 |
        +--------------------------+
        |        59950000762.93945 |
        +--------------------------+
        ```
        
- 동등 비교에 `FLOAT`를 사용
    - 😰 결과를 가져오는 데 실패
        
        ```sql
        -- 결과는 없다.
        SELECT * FROM Accounts WHERE hourly_rate = 59.95;
        
        SELECT * FROM Accounts WHERE hourly_rate = 59.95000076293945;
        +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
        | account_id | account_name | first_name | last_name | email | password_hash | portrait_image                 | hourly_rate |
        +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
        |         12 | account1     | NULL       | NULL      | NULL  | NULL          | NULL                           |       59.95 |
        +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
        ```
        
    - 💡 두 부동 소수점 값이 일정 수준 이상 충분히 가까우면 ‘사실상 같은’ 값으로 다루는 것
        - 두 값의 차를 구한 후 SQL 의 `ABS()`함수를 이용해 절대 값을 만든다.
            - 이 결과가 0 이면 두 값은 정확하게 같은것
            - 이 결과가 충분히 작다면 두 값을 사실상 같은 것으로 다룰 수 있다.
                
                ```sql
                SELECT * FROM Accounts WHERE ABS(hourly_rate - 59.95) < 0.000001;
                +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
                | account_id | account_name | first_name | last_name | email | password_hash | portrait_image                 | hourly_rate |
                +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
                |         12 | account1     | NULL       | NULL      | NULL  | NULL          | NULL                           |       59.95 |
                +------------+--------------+------------+-----------+-------+---------------+--------------------------------+-------------+
                ```
                
        - 😰 적절한 기준은 상황에 따라 다르다.
            
            ```sql
            -- 정도를 높이면 여전히 결과를 얻는데 실패한다.
            SELECT * FROM Accounts WHERE ABS(hourly_rate - 59.95) < 0.0000001;
            ```
            
- 많은 값을 집계해 계산할 때도 `FLOAT`의 부정확한 특성으로 인한 정확성 문제가 발생
    
    ```sql
    -- 59.95 * 3 = 179.85
    SELECT SUM( b.hours * a.hourly_rate ) AS project_cost
      FROM Bugs AS b
      JOIN Accounts AS a ON (b.assigned_to = a.account_id);
    +--------------------+
    | project_cost       |
    +--------------------+
    | 179.85000228881836 |
    +--------------------+
    ```
    
    - 😰 부정확한 부동 소수점 수를 사용하면 처음에는 오차가 아주 작지만, 계산을 반복할수록 오차가 누적되고 문제가 커지게 된다.

## 3. 안티패턴 인식 방법

---

- `FLOAT`, `REAL`, `DOUBLE PRECISION` 데이터 타입이 사용되는 곳이면 어디든 의심이 간다.
    - 부동 소수점 수를 사용하는 대부분의 애플리케이션에서는 IEEE 754 형식이 제공하는 넓은 범위의 값이 필요하지 않다.

## 4. 안티패턴 사용이 합당한 경우

---

- `INTEGER`나 `NUMERIC` 타입이 지원하는 것보다 큰 범위의 실수 값을 사용해야 할때는 `FLOAT`가 좋은 데이터 타입이다.
    - ex> 과학계산용 애플리케이션

## 5. 해법: `NUMERIC` 데이터 타입 사용

---

- 고정 소수점 수에는 `FLOAT`나 이와 비슷한 타입을 사용하지 말고, `NUMERIC` 또는 `DECIMAL` 타입을 사용해야 한다.
    
    ```sql
    ALTER TABLE Bugs MODIFY hours NUMERIC(9,2);
    ALTER TABLE Accounts MODIFY hourly_rate NUMERIC(9,2);
    ```
    
- 😃 칼럼 정의에서 지정한 정도까지 수치를 정확하게 표현한다.
    
    ```sql
    -- 반올림되지 않고 저장된다.
    SELECT hourly_rate FROM Accounts WHERE hourly_rate = 59.95;
    +-------------+
    | hourly_rate |
    +-------------+
    |       59.95 |
    +-------------+
    
    -- 10억을 곱해 크게 해도 기대하는 값을 얻을 수 있다.
    SELECT hourly_rate * 1000000000 FROM Accounts WHERE hourly_rate = 59.95;
    +--------------------------+
    | hourly_rate * 1000000000 |
    +--------------------------+
    |           59950000000.00 |
    +--------------------------+
    ```
    
    - `NUMERIC`과 `DECIMAL` 데이터 타입은 동일하게 동작한다.

<aside>
💡 가능하면 `FLOAT`를 사용하지 말라.

</aside>

- 참고
    - [http://wiki.gurubee.net/pages/viewpage.action?pageId=15794178](http://wiki.gurubee.net/pages/viewpage.action?pageId=15794178)
    - [https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-02물리.md](https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-02%EB%AC%BC%EB%A6%AC.md)
    - [https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md](https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md)