# [08] 다중 컬럼 속성

- 사람들의 연락처 정보를 저장하는 테이블에 들어갈 정보들
    - 이름
    - 호칭
    - 주소
    - 회사
    - :
    - 연락처
        - 집 전화번호
        - 회사 전화번호
        - 팩스 번호
        - 휴대폰 번호
        - 보조 휴대폰 번호
        - :
- *과연 컬럼은 얼마나 많아야 충분할까?*

## 목표 : 다중 값 속성 저장

- ‘속성’은 한 테이블에 들어가야 할 것처럼 보이지만, 여러 개의 값을 가진다
- 예제 : 버그 분류 시스템
    - 버그 데이터베이스에 태그를 허용하여 버그를 분류한다
    - 어떤 버그는 ‘인쇄', ‘리포트', ‘이메일' 같이 해당 버그가 영향을 미치는 소프트웨어 서브 시스템에 따라 분류된다
    - 어떤 버그는 결함의 성질에 따라 분류될 수도 있다
        - 프로그램이 죽는 버그는 ‘crash’ 태그를
        - 속도가 느린 문제는 ‘performance’ 태그를
        - 사용자 인터페이스에서 색상 선택이 부적절하다면 ‘cosmetic’ 태그를..
    - 태그는 상호 배타적일 필요가 없기 때문에 여러 태그를 다는 것도 가능하다

## 안티패턴

- 각 컬럼에는 하나의 값만 저장해야 한다
- 하지만, 속성에 여러 값이 들어갈 수 있다
- 따라서, 테이블에 여러 개의 컬럼을 만들고 각 컬럼에 하나의 태그를 저장해보자
    
    ```sql
    CREATE TABLE bugs (
    	bug_id       SERIAL PRIMARY KEY
    	descrpition  VARCHAR(1000),
    	tag1         VARCHAR(20),
    	tag2         VARCHAR(20),
    	tag3         VARCHAR(20)
    )
    ```
    
- 버그 태그를 달 때, 세 컬럼 중 하나에 값을 넣는다
- 사용되지 않으면 null이다
    
    
    | bug_id | description | tag1 | tag2 | tag3 |
    | --- | --- | --- | --- | --- |
    | 1234 | 저장 시 죽어버림 | crash | null | null |
    | 3456 | 성능을 개선해야 함 | printing | performance |  |
    | 5678 | XML 지원해야 함 | null | null | null |

### 값 검색

- 셋 중 어느 컬럼에 태그 문자열이 있는지 알 수 없다
- **따라서, 주어진 태그를 가진 버그를 찾으려면 세 컬럼 모두 확인해야 한다**
    - ‘performance’ 태그를 가진 버그를 조회하려면?
        
        ```sql
        SELECT *
        FROM bugs 
        WHERE tag1 = 'performance' OR tag2 = 'performance' OR tag3 = 'performance';
        ```
        
    - ‘performance’와 ‘printing’ 두 태그를 모두 가진 버그를 조회하려면?
        
        ```sql
        SELECT *
        FROM bugs 
        WHERE (tag1 = 'performance' OR tag2 = 'performance' OR tag3 = 'performance')
              AND (tag1 = 'printing' OR tag2 = 'printing' OR tag3 = 'printing');
        ```
        
    - ‘performance’나 ‘printing’ 태그를 하나라도 가진 버그를 조회하려면?
        
        ```sql
        SELECT *
        FROM bugs 
        WHERE 'performance' IN (tag1, tag2, tag3)
              AND 'printing' IN (tag1, tag2, tag3);
        ```
        

### 값 추가와 삭제

- 컬럼 집합에 값을 추가하거나 삭제할 때도, 확인을 위해 어플리케이션에서 해당 행을 조회해야 한다
- 업데이트
    - 어느 컬럼이 비어있는지 알 수 없기 때문에 단순 UPDATE를 사용해 컬럼 중 하나를 변경하는 것은 안전하지 않다
        
        ```sql
        SELECT * FROM bugs WHERE bug_id=3456;
        UPDATE bugs SET tag2 = 'performance' WHERE bug_id=3456;
        ```
        
    - 테이블을 조회하고 업데이트를 하기 전에 다른 클라이언트가 같은 행을 업데이트하기 위해 동일한 절차를 실해
        - 테이블을 조회하고 업데이트를 하기 전에 다른 클라이언트가 같은 행을 업데이트
    - 이는, 누가 먼저 업데이트를 했느냐에 따라 둘 중 하나는 업데이트에 실패하거나, 변경 내용을 덮어쓴다
- 삭제
    - `NULLIF()` 함수 : 두 인수 값이 같을 때, NULL을 반환한다
    - `NULLIF()` 함수를 사용해 컬럼 값이 특정 값과 같으면 NULL로 만드는 작업을 한다
        
        ```sql
        UPDATE bugs
        SET tag1 = NULLIF(tag1, 'performance'),
            tag2 = NULLIF(tag2, 'performance'),
            tag3 = NULLIF(tag3, 'performance')
        WHERE bug_id=3456;
        ```
        
- 추가
    - ‘performance’라는 태그를 태그 컬럼 중 처음으로 null인 컬럼에 기록한다
    - 만약, 모두 null이 아니면 새 태그가 추가되지 않는다
        
        ```sql
        UPDATE bugs
        SET tag1 = CASE WHEN 'performance' IN (tag2, tag3) THEN tag1 ELSE COALESCE(tag1, 'performance') END,
            tag2 = CASE WHEN 'performance' IN (tag1, tag3) THEN tag2 ELSE COALESCE(tag2, 'performance') END,
            tag2 = CASE WHEN 'performance' IN (tag2, tag3) THEN tag3 ELSE COALESCE(tag3, 'performance') END
        WHERE bug_id=3456;
        ```
        

### 유일성 보장

- 여러 컬럼에 동일한 태그가 나타나지 않아야 한다
- 하지만, 다중컬럼 속성 안티패턴을 사용한 경우에는, 데이터베이스에서 이를 예방하지 못한다
- 즉, 다음 같은 문장이 실행되는 것을 방지할 수 없다
    
    ```sql
    INSERT INTO bugs (description, tag1, tag2, tag3) VALUES ('printing is slow', 'printing', **'performance', 'performance'**);
    ```
    

### 값의 수 증가 처리

- 컬럼 세 개가 모자랄 수 있다
    - 즉, 더 많은 컬럼이 필요할 수 있다
- 한 컬럼에 하나의 값을 유지하기 위해서는, 버그가 가질 수 있는 태그의 최대개수만큼 컬럼을 정의해야 한다
- 과연, 태그의 최대 개수가 어떻게 될지 예측할 수 있을까?
- *‘모자르다면 컬럼을 추가하면 되지 않을까?’*
    - 데이터를 포함하고 있는 데이터베이스 테이블 구조를 변경하려면, 테이블 전체를 잠금 설정하고 다른 클라이언트의 접근을 차단하는 과정이 필요하다
    - 어떤 데이터베이스는 희망하는 구조의 새로운 테이블을 정의해 예전 테이블에서 모든 데이터를 복사한 다음, 예전 테이블을 삭제하는 식으로 테이블 변경을 구현한다
        - 테이블에 많은 데이터가 쌓여 있다면 작업에 많은 시간이 걸린다
    - **다중 컬럼 속성의 집합에 컬럼을 추가한 경우, 모든 어플리케이션에서 이 테이블을 참조하는 모든 SQL문을 확인해 새로운 컬럼을 지원하도록 수정해야 한다**
        - 수정이 필요한 쿼리를 하나라도 놓치면 발견하기 어려운 버그가 생긴다

## 안티패턴 인식 방법

- 사용자 인터페이스나 문서에는 여러 개의 값을 할당할 수 있지만, **최대 개수가 제한되어있는 속성이 기술되어 있다면**, 다중 컬럼 속성 안티패턴이다
- 이 안티패턴이 사용되고 있음을 나타내는 징조
    - *‘태그를 최대 몇개까지 붙일 수 있도록 지원해야 하지?’*
        - 태그와 같은 다중값 속성을 위해 테이블에 얼마나 많은 컬럼을 정의해야 하는지를 결정하려 하는 것이다
    - *‘SQL에서 여러 컬럼을 한꺼번에 검색하려면 어떻게 해야 하지?’*
        - 주어진 값을 여러 컬럼에 걸쳐 검색해야 한다면, 이들 컬럼을 실제로 하나의 논리적 속성으로 저장되어야 함을 나타낸다

## 안티패턴 사용이 합당한 경우

- 속성의 개수가 고정되고 선택의 위치나 순서가 중요한 경우
    - 주어진 버그가 여러 사용자 계정과 연관될 수 있지만, 각 연관은 본질적으로 유일하다
        - 버그를 보고한 사용자
        - 버그 수정을 위해 할당된 프로그래머
        - 수정 검증을 위해 할당된 QA 담당자
    - 각 컬럼에 들어가는 값은 같은 종류지만, 그 의미와 사용처가 달라 논리적으로 다른 속성이 된다
    - 이 경우에는, 세 속성을 따로따로 사용하기 때문에 세 개의 컬럼을 정의하는 것이 합당하다

## 해법: 종속 테이블 생성

- 다중 값 속성을 위한 컬럼을 하나 가지는 종속 테이블을 만든다
- **여러 개의 값을 여러 개의 컬럼 대신 여러 개의 행에 저장한다**
    
    ```sql
    CREATE TABLE tags (
    	bug_id      BIGINT UNSIGNED NOT NULL
    	tag         VARCHAR(20),
    	PRIMARY KEY (bug_id, tag),
    	FOREIGN KEY (bug_id) REFERENCES bugs(bug_id)
    )
    ```
    
- 하나의 버그에 연관된 모든 태그가 한 컬럼에 있으면, 주어진 태그에 대한 버그를 검색하는 작업이 좀 더 직관적이다
    
    ```sql
    SELECT *
    FROM bugs JOIN tags USING (bug_id)
    WHERE tag = 'performance';
    ```
    
- 연관을 추가,삭제 하는 것도 다중컬럼 안티패턴보다 훨씬 쉬워진다
    - 단순하게 종속 테이블에 행을 추가하거나 삭제하면 된다
        
        ```sql
        INSERT INTO tags (bug_id, tag) VALUES (1234, 'save');
        DELETE FROM tags WHERE bug_id=1234 AND tag = 'crash';
        ```
        
- 다중컬럼 안티패턴을 적용했을 때처럼, 태그의 개수가 컬럼의 개수에 의해 제한되지 않는다
- 버그에 필요한 만큼 태그를 적용할 수 있다

> ***같은 의미를 가지는 각각의 값은 하나의 컬럼에 저장하라***
>