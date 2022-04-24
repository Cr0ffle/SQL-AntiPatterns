# [04] 아이디가 필요해

## 데이터 중복 이슈

---

- Articles 테이블과 Tags 테이블 간의 매핑 테이블
    
    ```sql
    CREATE TABLE ArticleTags (
    	id          SERIAL PRIMARY KEY,
    	article_id  BIGINT UNSIGNED NOT NULL,
    	tag_id      BIGINT UNSIGNED NOT NULL,
    	FOREIGN KEY (article_id) REFERENCES Articles (id),
    	FOREIGN KEY (tag_id)     REFERENCES Tags (id)
    );
    ```
    
- 문제 상황
    - 현재, ‘경제' Tag가 달린 Article은 총 5개다
    - 하지만 count를 조회해보면 7개가 나온다
    - 그 이유는 **데이터 중복이슈!**
    - 동일한 연관을 나타내는 데이터가 여러 번 들어간 것이다
        
        
        | id | tag_id | article_id |
        | --- | --- | --- |
        | 22 | 327 | 1234 |
        | 23 | 327 | 1234 |
        | 23 | 327 | 1234 |
    - **이 테이블은 PK를 가지고 있지만 중요 컬럼의 중복을 막지 못한다**
- 해결방법 : 두 컬럼에 대해 UNIQUE 제약 조건을 생성한다
- 의문 : **그럼 대체 id 컬럼은 왜 필요할까?**

## 목표:PK 관례 확립

---

- 목표 : **모든 테이블이 PK를 갖도록 한다**
- 하지만, PK의 본질을 혼동하면 안티패턴이 초래된다
- 누군가는..
    - *“PK는 필요하지 않아요!”*
    - *“UNIQUE 인덱스를 유지하는 데 오버헤드가 더 많아요!”*
    - *“딱히 PK 목적으로 사용할만한 컬럼이 없어요!”*
    - *“정말 PK가 필요해?!”*
- 왜 필요하냐
    1. 테이블 내의 모든 행이 유일함을 보장한다
        - 따라서 각 행에 접근하는 논리적 매커니즘이 되고 중복 행이 저장되는 것을 방지한다
    2. PK는 관계를 생성할 때 FK로부터 참조된다
- PK는 아무런 의미도 가지지 않는 인위적인 값을 저장해야 한다
    - PK로 적절하지 않은 것들
        - 이름
        - 이메일 주소
        - 주민등록번호
        - :
    - 이 컬럼을 PK로 사용하면, 다른 속성 컬럼에는 중복 값이 들어가는 것을 허용하는 반면에, 특정 행에 유일하게 접근할 수 있게 된다
- 각 행의 PK가 유일하게 할당되는 것을 보장하기 위해 여러 DBMS에서는 트랜잭션 격리 범위 밖에서 유일한 정수 값을 생성하는 매커니즘을 제공한다
    
    
    | DBMS | 기능 |
    | --- | --- |
    | MySQL | AUTO_INCREMENT, SERIAL |
    | Oralce | SEQUENCE |
    | PostgreSQL |  SEQUENCE, SERIAL |
    | Firebird | GENERATOR, SEQUENCE |
    | : | : |

## 안티패턴: 만능키

---

- 문화적 관례
    - PK 컬럼의 이름은 id다
    - PK 컬럼의 데이터 타입은 32/64 비트 정수다
    - 유일값은 자동생성된다
- 모든 테이블에 id란 이름의 컬럼이 있는 것은 너무도 흔해져 이게 PK와 동의어가 되어버렸다
    - 여기서 잘못된 정의를 내리게 되는데..
        - PK란, 다음처럼 정의되는 컬럼이다! (잘못된 정의)
            
            ```sql
            CREATE TABLE Bugs (
            	**id SERIAL PRIMARY KEY,**
              :
            )
            ```
            

### 중복키 생성

- 테이블 안에 다른 컬럼이 자연키로 사용될 수 있는 상황에서도 통념에 따라 id 컬럼을 PK로 정의한다
- 그리고 그 다른 컬럼에 UNIQUE 제약 조건을 설정한다
- 예시
    
    ```sql
    CREATE TABLE Bugs (
    	id     SERIAL      PRIMARY KEY,
    	bug_id VARCHAR(10) UNIQUE,
    	:
    )
    
    INSERT INTO Bugs (bug_id, ...) VALUES ('VIS_078', ...);
    ```
    
    - bug_id 컬럼은 각 행을 유일하게 식별할 수 있도록 해준다는 면에서 id와 사용 목적이 동일하다

### 중복행 허용

- 복합키는 여러 컬럼을 포함한다
- Bug와 Product의 매핑테이블인 BugProducts는 bug_id와 product_id의 쌍이 한번만 있다는 것을 보장해야 한다
- 하지만, id 컬럼을 PK로 사용하는 경우에 이런 제약조건이 적용되지 않는다
    
    ```sql
    CREATE TABLE BugProducts (
    	id     SERIAL PRIMARY KEY,
      bug_id BIGINT UNSIGNED NOT NULL,
      product_id BIGINT UNSIGNED NOT NULL,
    	:
    );
    ```
    
- 중복을 방지하기 위해서는 id 뿐 아니라 두 컬럼에도 제약조건을 걸어줘야 한다
    
    ```sql
    CREATE TABLE BugProducts (
    	id     SERIAL PRIMARY KEY,
      bug_id BIGINT UNSIGNED NOT NULL,
      product_id BIGINT UNSIGNED NOT NULL,
    	:
      UNIQUE KEY (bug_id, product_id),
    );
    ```
    
- 그러나 이 두 컬럼에 UNIQUE 제약 조건을 걸어야 한다면 id 컬럼은 불필요하다

### 모호한 키의 의미

- 프로그래밍에서 ‘코드’란 ‘의미를 명확하게 한다’의 목표를 가져야 한다
- id란 이름은 너무 일반적이라, 아무런 의미를 갖지 못한다
- 두 테이블 간의 조인할 때 문제가 된다
    
    ```sql
    **SELECT b.id, a.id**
    FROM Bugs b
    JOIN Accounts a ON (b.assigned_to=a.id)
    WHERE b.status='OPEN'l
    ```
    
    - 컬럼 이름이 bug_id, account_id로 되어있다면 쿼리 결과를 읽기 훨씬 쉽다

### USING 사용

- Bugs와 BugProducts의 조인
    
    ```sql
    SELECT *
    FROM Bugs AS b
    JOIN BugProducts AS bp ON (b.bug_id = bp.bud_id);
    ```
    
- 양쪽 테이블에서 컬럼 이름이 같다면 앞의 쿼리는 다음처럼 작성할 수 있다
    
    ```sql
    SELECT *
    FROM Bugs
    JOIN BugProducts USING (bug_id);
    ```
    
- 그러나, id란 이름의 가상키를 PK로 정의해야 한다면, 종속된 테이블에서 해당 PK를 참조하는  컬럼의 이름과 같을 수 없게 되어, 약간 장황한 ON 문법을 사용해야 한다

### 어려운 복합키

- 어떤 개발자는 사용하기 어렵다는 이유로 복합키를 거부한다
- 키를 비교할 때 모든 컬럼을 비교해야 하며,
복합 PK를 참조하는 FK는 자신도 복합 FK가 되어야 한다
- 이런 거부는 수학자가 2차원 3차원 좌표계를 거부하고 1차원만으로 모든 계산을 수행하는 것과 같다

## 안티패턴 인식 방법

---

- 테이블에 PK 컬럼 이름으로 id가 사용되고 있으면 안티패턴의 징후로 볼 수 있다
- 안티패턴의 중거가 될 수 있는 말들
    - ‘이 테이블에는 PK가 없어도 될 것 같은데’
        - PK와 가상키 용어는 의미가 다르다
        - 모든 테이블은 중복 행을 방지하고 각 행을 유일하게 식별하기 위해 PK 제약조건을 가져야 한다
        - 이런 경우는 자연키나 복합키를 사용해볼 수 있다
    - ‘다대다 연결에서 왜 중복이 발생했지?’
        - 교차 테이블에는 FK 컬럼을 묶어 PK 제약조건을 걸거나, UNIQUE 제약조건이라도 걸어줘야 한다
    - ‘내가 원하는 실제 값을 얻으려면 매번 조인을 해야 하네.. DB 이론처럼 값들을 색인 테이블로 옮기고 ID로 참조하는 형태로 쓰고 싶지 않아’
        - 이것은 정규화와 관련된 내용이다
        - 정규화는 가상키와 상관이 없다

## 안티패턴 사용이 합당한 경우

---

- 객체-관계 프레임워크에서는 Convention over Configuration을 통해 개발을 단순화한다
- 이런 프레임워크에서는 모든 테이블이 동일한 방식으로 PK를 정의한다고 가정한다
    - 우리는 프레임워크를 사용하여 다른 원하는 기능을 사용하고 싶다
- 물론, 가상키를 사용하고 자동 증가하는 정수를 사용해 키 값을 할당하는 것은 잘못이 아니다
- 하지만, 모든 테이블에 가상키가 필요한 것도 아니고, 모든 가상키 컬럼 이름을 id로 해야 하는 것도 아니다
- 가상키는 지나치게 긴 자연키를 대체하기 위해 사용할 수 있다
    - ex) 파일 경로는 좋은 자연키가 될 수 있지만, 긴 문자열이라 인덱스를 유지하는데 많은 비용이 든다

## 해법 : 상황에 맞추기

---

- **PK는 제약조건이다**
    - 데이터 타입이 아니다
- 데이터 타입이 인덱스를 지원하기만 하면, 어떤 컬럼에 대해서는 PK를 선언할 수 있다
- 또한, 테이블의 특정 컬럼을 PK로 잡지 않고도 자동 증가하는 정수값을 가지도록 정의할 수 있다
- **이 두 개념은 서로 독립적이다**

### 있는 그대로 말하기

- PK에 의미있는 이름을 선택해야 한다
    - 예) Bugs 테이블에는 bug_id
- FK에서도 가능하다면 같은 컬럼의 이름을 사용해야 한다
    - 이는 곧, 동일한 PK 이름이 다른 테이블에 나오면 안된다는 것을 의미한다
- 그러나 예외는 있다
    - 연결의 본직을 더 잘 표현하는 경우라면, FK를 자신이 참조하는 PK이름과 다르게 하는 것도 괜찮다
    
    ```sql
    CREATE TABLE Bugs(
    	...
    	reported_by BIGINT UNSIGNED NOT NULL,
    	FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
    );
    ```
    

### 자연키와 복합키 포용

- 유일함이 보장되고, NULL 값을 가지는 경우가 없고, 행을 식별하는 용도로 사용할 수 있는 속성의 테이블이 있다면
    - 굳이 통념을 따르기 위해 가상키를 추가해야 한다는 의무감을 느끼지 않아도 된다
- 실제로, 테이블에 있는 각 속성은 변하기 마련이고, 유일하지 않게 될 수도 있다
    - 데이터베이스는 프로젝트 기간동안 변화하고, 결정권자들이 자연키를 존중하지 않을 수도 있다
    - 처음에는 자연키로 손색이 없어 보이던 컬럼이 나중에는 중복을 허용하는 것으로 밝혀질 수도 있다
    - 이런 경우에는 가상 키를 사용할 수 있다
- 복합키가 적절한 경우에는 복합키를 사용하자
    - BugProducts 테이블과 같이 여러 컬럼의 조합으로 행을 가장 잘 식별할 수 있다면, 이를 복합키로 사용할 수 있다
    - 복합 PK를 참조하는 FK 또한 복합키가 되어야 한다
    - 종속되는 테이블에 이렇게 컬럼 조합을 중복해야 하는 것은 안좋아 보일 수 있다

> 관례는 도움이 될 때만 좋은 것이다
>