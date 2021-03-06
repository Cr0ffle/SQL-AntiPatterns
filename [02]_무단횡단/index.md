# Chapter 2. λ¬΄λ¨ν‘λ¨

****

<aside>
π‘ κ° κ°μ μμ μ μΉΌλΌκ³Ό νμ μ μ₯νλΌ

</aside>

## 1. λͺ©ν: λ€μ€ κ° μμ± μ μ₯

---

![Untitled](./images/Untitled.png)

- νμ΄λΈμ μΉΌλΌμ΄ νλμ κ°μ κ°μ§ λ μ€κ³κ° μ½λ€. ν΄λΉ κ°μ μΈμ€ν΄μ€ νλλ₯Ό νννλ SQL λ°μ΄ν° νμμ μ νν  μ μλ€. πΒ μ νκ³Ό κ³μ μ λ€λμΌ κ΄κ³
    - ex> μ μ, λ μ§, λ¬Έμμ΄ κ°μ κ²
    
    ```sql
    CREATE TABLE Products (
      product_id   SERIAL PRIMARY KEY,
      product_name VARCHAR(1000),
      account_id   BIGINT UNSIGNED,
      FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
    );
    
    CREATE TABLE Accounts (
      account_id     SERIAL PRIMARY KEY,
      account_name   VARCHAR(20),
      first_name     VARCHAR(20),
      last_name      VARCHAR(20),
      email          VARCHAR(100),
      password_hash  CHAR(64),
      portrait_image BLOB,
      hourly_rate    NUMERIC(9,2)
    );
    
    INSERT INTO Accounts (account_id, account_name)
    VALUES (12, 'account1');
    
    INSERT INTO Products (product_id, product_name, account_id)
    VALUES (DEFAULT, 'Visual TurboBuilder', 12);
    ```
    
- μ νμ λ΄λΉμκ° μ¬λ¬ λͺμΌ μλ μλ€.

## 2. μν°ν¨ν΄: μΌνλ‘ κ΅¬λΆλ λͺ©λ‘μ μ μ₯

---

- λ°μ΄ν°λ² μ΄μ€ κ΅¬μ‘°μ λ³κ²½μ μ΅μννκΈ° μν΄, account_id μΉΌλΌμ `VARCHAR`λ‘ λ°κΎΈκ³  μ¬κΈ°μ μ¬λ¬ κ°μ κ³μ  μμ΄λλ₯Ό μΌνλ‘ κ΅¬λΆν΄ λμ΄νκΈ°λ‘ νλ€.
    
    ```sql
    create TABLE Products2 (
      product_id   SERIAL PRIMARY KEY,
      product_name VARCHAR(1000),
      account_id   VARCHAR(100) -- comma-separated list
    );
    
    INSERT INTO Products2 (product_id, product_name, account_id)
    VALUES (DEFAULT, 'Visual TurboBuilder', '12,34');
    ```
    

### μ΄ νμ΄λΈ μ€κ³λ‘λΆν° κ²ͺμ΄μΌ ν  μ±λ₯ λ¬Έμ μ λ°μ΄ν° μ ν©μ± λ¬Έμ 

---

- νΉμ  κ³μ μ λν μ ν μ‘°ν
    - λͺ¨λ  FK κ° νλμ νλμ κ²°ν©λμ΄ μμΌλ©΄ μΏΌλ¦¬κ° μ΄λ €μμ§λ€. λ μ΄μ κ°μμ§λ₯Ό λΉκ΅ν  μ μλ€.
    - μ΄λ€ [ν¨ν΄](https://dev.mysql.com/doc/refman/8.0/en/regexp.html#operator_regexp)μ λ§λμ§λ₯Ό κ²μ¬ν΄μΌ νλ€.
        
        ```sql
        SELECT * FROM Products2 WHERE account_id REGEXP '[[:<:]]' || 12 || '[[:>:]]';
        ```
        
    - π°Β ν¨ν΄ λ§€μΉ­μ μ¬μ©νλ©΄ μλͺ»λ κ²°κ³Όκ° λ¦¬ν΄λ  μλ μκ³  μΈλ±μ€λ νμ©νμ§ λͺ»νλ€.
    - π°Β λ²€λ μ€λ¦½μ μ΄μ§λ μλ€.
- μ£Όμ΄μ§ μ νμ λν κ³μ  μ λ³΄ μ‘°ν
    - μΌνλ‘ κ΅¬λΆλ λͺ©λ‘μ μ°Έμ‘°νλ νμ΄λΈμ λμλλ νκ³Ό μ‘°μΈνκΈ°λ λΆνΈν΄μ§κ³  λΉμ©μ΄ λ§μ΄ λ λ€.
        
        ```sql
        SELECT *
          FROM Products2 AS p
          JOIN Accounts AS a
            ON p.account_id REGEXP '[[:<:]]' || a.account_id || '[[:>:]]'
         WHERE p.product_id = 123;
        ```
        
    - π°Β μΈλ±μ€λ₯Ό νμ©ν  κΈ°νκ° μ¬λΌμ§λ€.
        - λ νμ΄λΈμ λͺ¨λ μ½μ΄ μΉ΄νμμ κ³±μ μμ±ν λ€μ, λͺ¨λ  νμ μ‘°ν©μ λν΄ μ κ· ννμμ νκ°ν΄μΌ νλ€.
- μ§κ³ μΏΌλ¦¬ λ§λ€κΈ°
    - μ§κ³ μΏΌλ¦¬λ νμ κ·Έλ£Ήμ λν΄ μ¬μ©νλλ‘ μ€κ³λμμ§, μΌνλ‘ κ΅¬λΆλ λͺ©λ‘μ λν΄ μ¬μ©νλλ‘ μ€κ³λ κ²μ΄ μλλ€.
        - `COUNT()`, `SUM()`, `AVG()`μ κ°μ ν¨μ
        
        ```sql
        SELECT product_id
             , LENGTH(account_id) - LENGTH(REPLACE(account_id, ',', '')) + 1
            AS contacts_per_product
          FROM Products2;
        +------------+----------------------+
        | product_id | contacts_per_product |
        +------------+----------------------+
        |          1 |                    2 |
        +------------+----------------------+
        ```
        
    - π°Β λͺννμ§ μλ€.
    - π°Β κ°λ°νλ λ° μκ°λ μ€λ κ±Έλ¦¬κ³  λλ²κΉνκΈ°λ μ΄λ ΅λ€.
- νΉμ  μ νμ λν κ³μ  κ°±μ 
    - λͺ©λ‘μ΄ μ λ ¬λ μνλ‘ μ μ§λμ§ μλλ€.
        
        ```sql
        UPDATE Products
           SET account_id = account_id || ',' || 56
         WHERE product_id = 123;
        ```
        
    - λͺ©λ‘μμ ν­λͺ©μ μ­μ νλ €λ©΄ λ κ°μ SQL μΏΌλ¦¬λ₯Ό μ€νν΄μΌ νλ€.
        
        ```php
        <?php
        
        $stmt = $pdo->query(
        // 1. μμ  λͺ©λ‘μ λΆλ¬μ΄
          "SELECT account_id FROM Products WHERE product_id = 123");
        $row = $stmt->fetch();
        $contact_list = $row['account_id'];
        
        // change list in PHP code
        $value_to_remove = "34";
        $contact_list = split(",", $contact_list);
        $key_to_remove = array_search($value_to_remove, $contact_list);
        unset($contact_list[$key_to_remove]);
        $contact_list = join(",", $contact_list);
        
        $stmt = $pdo->prepare(
        // 2. λͺ©λ‘μ κ°±μ 
          "UPDATE Products SET account_id = ?
           WHERE product_id = 123");
        $stmt->execute(array($contact_list));
        ```
        
    - π°Β λ§μ μ½λκ° νμνλ€.
- μ ν μμ΄λ μ ν¨μ± κ²μ¦
    - μ ν¨νμ§ μμ ν­λͺ©μ μλ ₯
        
        ```sql
        INSERT INTO Products2 (product_id, product_name, account_id)
        VALUES (DEFAULT, 'Visual TurboBuilder', '12,34,banana');
        ```
        
    - π°Β λ°μ΄ν°λ² μ΄μ€μμ μλ¬κ° λ°μνμ§λ μμ§λ§, λ°μ΄ν°λ μλ―Έ μλ κ²μ΄ λ  κ²μ΄λ€.
- κ΅¬λΆμ λ¬Έμ μ ν
    - λ¬Έμμ΄ λͺ©λ‘μ μ μ₯νλ κ²½μ° λͺ©λ‘μ μΌλΆ ν­λͺ©μ΄ κ΅¬λΆμ λ¬Έμλ₯Ό ν¬ν¨ν  μ μλ€.
- λͺ©λ‘ κΈΈμ΄ μ ν
    - μ μ₯ πΒ κ° ν­λͺ©μ κΈΈμ΄μ λ°λΌ λ€λ₯΄λ€.
        
        ```sql
        UPDATE Products2
           SET account_id = '10,14,18,22,26,30,34,38,42,46'
         WHERE product_id = 123;
        
        UPDATE Products2
           SET account_id = '101418,222630,343842,467790'
         WHERE product_id = 123;
        ```
        

## 3. μν°ν¨ν΄ μΈμ λ°©λ²

---

### λ¬΄λ¨ν‘λ¨ μν°ν¨ν΄μ΄ μ¬μ©λκ³  μμμ λνλ΄λ λ¨μ

---

- βμ΄ λͺ©λ‘μ΄ μ§μν΄μΌ νλ μ΅λ ν­λͺ© μλ μΌλ§λ λ κΉ?β
    
    β `VARCHAR` μΉΌλΌμ μ΅λ κΈΈμ΄λ₯Ό μ μ νλ € ν  λ μ§λ¬Έ
    
- βSQL μμ λ¨μ΄μ κ²½κ³λ₯Ό μ΄λ»κ² μμλ΄λμ§ μμ?β
    
    β λ¬Έμμ΄ μΌλΆλ₯Ό μ°Ύμλ΄κΈ° μν΄ μ κ· ννμμ μ¬μ©νλ€λ©΄, μ΄λ° λΆλΆμ λ³λλ‘ μ μ₯ν΄μΌ ν¨μ λ»νλ λ¨μ!
    
- βμ΄ λͺ©λ‘μμ μ λ λμ€μ§ μμ λ¬Έμλ?β
    
    β λͺ¨νΈνμ§ μμ λ¬Έμλ₯Ό κ΅¬λΆμλ‘ μ¬μ©νκ³  μΆκ² μ§λ§, μ΄λ€ κ΅¬λΆμλ₯Ό μ°λ  μΈμ  κ°λ κ·Έ λ¬Έμκ° λͺ©λ‘μ κ°μ λνλ  κ²
    

## 4. μν°ν¨ν΄ μ¬μ©μ΄ ν©λΉν κ²½μ°

---

- μ΄λ€ μ’λ₯μ μΏΌλ¦¬λ λ°μ΄ν°λ² μ΄μ€μ λ°μ κ·νλ₯Ό μ μ©ν΄ μ±λ₯μ ν₯μμν¬ μ μλ€.
    - ex> λͺ©λ‘μ μΌνλ‘ κ΅¬λΆλ λ¬Έμμ΄λ‘ μ μ₯νλ κ²
- μ νλ¦¬μΌμ΄μμμ μΌνλ‘ κ΅¬λΆλ νμμ λ°μ΄ν°λ₯Ό νμλ‘ νκ³ , λͺ©λ‘ μμ κ°λ³ ν­λͺ©μλ μ κ·Όν  νμκ° μμ μ μλ€.
- λ°μ΄ν°λ₯Ό λ°μ λ°μ΄ν°λ² μ΄μ€μ κ·Έλλ‘ μ μ₯νκ³  λμ€μ λμΌν νμμΌλ‘ λΆλ¬λ΄μΌ νλ©°, λͺ©λ‘ μμ κ°λ³ κ°μ λΆλ¦¬ν  νμκ° μλ€λ©΄.
- πΒ μ κ·ννλ κ²μ΄ λ¨Όμ !
    - μ νλ¦¬μΌμ΄μ μ½λλ₯Ό μ’λ μ΅ν΅μ± μκ² νκ³ , λ°μ΄ν°λ² μ΄μ€μ μ ν©μ±μ μ μ§ν  μ μκ² νλ€.

## 5. ν΄λ²: κ΅μ°¨ νμ΄λΈ μμ±

---

- κ΅μ°¨ νμ΄λΈ: μ΄λ€ νμ΄λΈμ΄ FKλ‘ λ νμ΄λΈμ μ°Έμ‘°ν  λ. μ°Έμ‘°λλ λ νμ΄λΈ μ¬μ΄μ λ€λλ€ κ΄κ³λ₯Ό κ΅¬ννλ€.
    
    ```sql
    CREATE TABLE Contacts (
      product_id  BIGINT UNSIGNED NOT NULL,
      account_id  BIGINT UNSIGNED NOT NULL,
      PRIMARY KEY (product_id, account_id),
      FOREIGN KEY (product_id) REFERENCES Products(product_id),
      FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
    );
    
    INSERT INTO Contacts (product_id, account_id)
    VALUES (123, 12), (123, 34), (345, 23), (567, 12), (567, 34);
    ```
    

### κ΅μ°¨ νμ΄λΈμ μ¬μ©

---

- κ³μ μΌλ‘ μ ν μ‘°ννκΈ°μ μ νμΌλ‘ κ³μ  μ‘°ννκΈ°
    
    ```sql
    -- productbyaccount
    SELECT p.*
      FROM Products AS p JOIN Contacts AS c ON (p.product_id = c.product_id)
     WHERE c.account_id = 34;
    
    -- accountbyproduct
    SELECT a.*
      FROM Accounts AS a JOIN Contacts AS c ON (a.account_id = c.account_id)
     WHERE c.product_id = 123;
    ```
    
    - πΒ ν¨μ¨μ μΌλ‘ μΈλ±μ€λ₯Ό μ¬μ©νλ€.
- μ§κ³ μΏΌλ¦¬ λ§λ€κΈ°
    
    ```sql
    -- accountsperproduct
    SELECT product_id, COUNT(*) AS accounts_per_product
      FROM Contacts
     GROUP BY product_id;
    
    -- productsperaccount
    SELECT account_id, COUNT(*) AS products_per_account
      FROM Contacts
     GROUP BY account_id;
    
    -- productwithmaxaccounts
    SELECT c.product_id, c.contacts_per_product
      FROM (
        SELECT product_id, COUNT(*) AS accounts_per_product
          FROM Contacts
         GROUP BY product_id
      ) AS c
     ORDER BY c.contacts_per_product DESC
     LIMIT 1
    ```
    
    - πΒ κ°λ¨νλ€. μ’λ λ³΅μ‘ν λ¦¬ν¬νΈλ₯Ό λ§λλ κ²λ κ°λ₯νλ€.
- νΉμ  μ νμ λν κ³μ  κ°±μ 
    - λͺ©λ‘μ ν­λͺ©μ μΆκ°νκ±°λ μ­μ νλ κ²μ κ΅μ°¨ νμ΄λΈμ νμ μ½μνκ±°λ μ­μ νλ λ°©λ²μΌλ‘ ν  μ μλ€.
        
        ```sql
        INSERT INTO Contacts (product_id, account_id) VALUES (456, 34);
        
        DELETE FROM Contacts
         WHERE product_id = 456
           AND account_id = 34;
        ```
        
    - πΒ ν λ²μ νλμ© μΆκ° λλ μ­μ ν  μ μλ€.
- μ ν μμ΄λ μ ν¨μ± κ²μ¦
    - πΒ FK β μ°Έμ‘° μ ν©μ±μ λ°μ΄ν°λ² μ΄μ€κ° κ°μ νλλ‘ ν  μ μλ€.
    - πΒ SQL λ°μ΄ν° νμ β λͺ¨λ  ν­λͺ©μ΄ ν΄λΉ νμμ μ ν¨ν κ°
- κ΅¬λΆμ λ¬Έμ μ ν
    - κ° ν­λͺ©μ λ³λμ νμΌλ‘ μ μ₯νλ―λ‘ κ΅¬λΆμλ₯Ό μ¬μ©νμ§ μλλ€.
    - πΒ κ΅¬λΆμ β ν­λͺ©μ ν¬ν¨λμ΄ μμμ§ κ±±μ ν  νμκ° μλ€.
- λͺ©λ‘ κΈΈμ΄ μ ν
    - ν νμ΄λΈμ λ¬Όλ¦¬μ μΌλ‘ μ μ₯ν  μ μλ ν μμλ§ μ νμ λ°λλ€.
    - πΒ ν­λͺ© μλ₯Ό μ ν β μ νλ¦¬μΌμ΄μμμ!
- κ΅μ°¨ νμ΄λΈμ λ€λ₯Έ μ₯μ 
    - πΒ μΈλ±μ€λ₯Ό νμ© β μ±λ₯μ΄ μ’μμ§λ€.
        - μΉΌλΌμ FK λ₯Ό μ μΈ β λ΄λΆμ μΌλ‘ ν΄λΉ μΉΌλΌμ λν μΈλ±μ€λ₯Ό μμ±νλ€.
    - πΒ κ΅μ°¨ νμ΄λΈμ μΉΌλΌμ μΆκ°ν΄ κ° ν­λͺ©μ μΆκ° μμ±μ λ£μ μ μλ€.

- μ°Έκ³ 
    - [https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md](https://github.com/pravusid/TIL/blob/2636a636c1b43ba44199e68914c4c674cb172814/Database/sql-anti-patterns.md)
    - [https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-01λΌλ¦¬.md](https://github.com/rewritech/til/blob/061e84726cb8a8f4efb0ebe09246c72b280bb6f7/SQL/SQL-antipatern-01%EB%85%BC%EB%A6%AC.md)
