# MySQL NULL 처리 방식 총정리

## 1. NULL의 기본 개념

| 항목    | 설명                                 |
| ----- | ---------------------------------- |
| 정의    | 데이터가 존재하지 않거나 알 수 없는 상태를 나타냄       |
| 특징    | 0, 빈 문자열('')과 완전히 다른 개념            |
| 비교 방법 | `IS NULL` / `IS NOT NULL` 로만 검사 가능 |

```sql
-- NULL 비교
SELECT NULL = NULL;    -- NULL
SELECT NULL != NULL;   -- NULL
SELECT NULL = 1;       -- NULL
SELECT NULL = 'text';  -- NULL
```

---

## 2. NULL 비교 연산

### 2.1 올바른 비교

```sql
SELECT * FROM table WHERE column IS NULL;
SELECT * FROM table WHERE column IS NOT NULL;
```

잘못된 비교(항상 FALSE)

```sql
SELECT * FROM table WHERE column = NULL;
SELECT * FROM table WHERE column != NULL;
```

### 2.2 NULL-safe 비교 (`<=>`)

```sql
SELECT NULL <=> NULL;  -- 1
SELECT 1 <=> NULL;     -- 0
SELECT NULL <=> 1;     -- 0

SELECT * FROM table WHERE column1 <=> column2;
```

---

## 3. NULL과 논리 연산 (삼중값 논리)


## 3. NULL과 논리 연산 (Three-Valued Logic)

MySQL은 `TRUE`, `FALSE`, `NULL` 세 가지 논리 값을 사용한다. `NULL`이 포함된 논리 연산은 다음 규칙을 따른다.

### AND 연산

```sql
SELECT TRUE AND NULL;    -- 결과: NULL
SELECT FALSE AND NULL;   -- 결과: FALSE
SELECT NULL AND NULL;    -- 결과: NULL
```

### OR 연산

```sql
SELECT TRUE OR NULL;     -- 결과: TRUE
SELECT FALSE OR NULL;    -- 결과: NULL
SELECT NULL OR NULL;     -- 결과: NULL
```

### NOT 연산

```sql
SELECT NOT NULL;         -- 결과: NULL
```



---

## 4. NULL과 집계 함수

| 함수            | NULL 처리 방식 |
| ------------- | ---------- |
| COUNT(column) | NULL 제외    |
| COUNT(*)      | 모든 행 포함    |
| SUM(column)   | NULL 제외    |
| AVG(column)   | NULL 제외    |
| MAX/MIN       | NULL 제외    |

```sql
SELECT 
    COUNT(*) as total_rows,
    COUNT(column) as non_null_count,
    SUM(column) as total_sum
FROM table;
```
 `COUNT(*)`는 **NULL 여부와 상관없이 모든 행을 카운트**한다.

예를 들어, 컬럼 `column1`에 3개의 NULL 값이 있더라도:

```sql
CREATE TABLE test (column1 INT);

INSERT INTO test VALUES (NULL), (NULL), (NULL);

SELECT COUNT(*) FROM test;
```

결과는 `3`이 나온다.

반대로 `COUNT(column1)`은 **NULL을 제외**하므로 결과는 `0`이 된다.

정리하면:

| 함수             | NULL 처리 | 예시 결과 |
| -------------- | ------- | ----- |
| COUNT(*)       | 모든 행 포함 | 3     |
| COUNT(column1) | NULL 제외 | 0     |

---

## 5. NULL과 연산

### 5.1 산술 연산

```sql
SELECT NULL + 10;  -- NULL
SELECT 10 / NULL;  -- NULL
```

### 5.2 문자열 연산

```sql
SELECT CONCAT('Hello', NULL, 'World');  -- NULL
SELECT NULL || 'text';                  -- NULL
```

---

## 6. NULL 관련 함수

| 함수                          | 설명                     | 예시                               |
| --------------------------- | ---------------------- | -------------------------------- |
| IFNULL(expr1, expr2)        | expr1이 NULL이면 expr2 반환 | `IFNULL(NULL, '대체값') → '대체값'`    |
| ISNULL(expr)                | expr이 NULL이면 1, 아니면 0  | `ISNULL(NULL) → 1`               |
| COALESCE(expr1, expr2, ...) | 첫 번째 NULL이 아닌 값 반환     | `COALESCE(NULL, 'A', 'B') → 'A'` |
| NULLIF(expr1, expr2)        | expr1=expr2이면 NULL 반환  | `NULLIF(10, 10) → NULL`          |

---

## 7. NULL과 정렬 (ORDER BY)

| 정렬   | 기본 위치    |
| ---- | -------- |
| ASC  | NULL 먼저  |
| DESC | NULL 마지막 |

```sql
-- NULL 위치 지정
SELECT * FROM table ORDER BY column IS NULL, column ASC;
SELECT * FROM table ORDER BY column IS NOT NULL, column DESC;

-- MySQL 8.0+
SELECT * FROM table ORDER BY column ASC NULLS LAST;
```

---

## 8. NULL과 UNIQUE 제약조건

* MySQL에서는 UNIQUE 컬럼에 여러 개 NULL 값 허용

```sql
CREATE TABLE example (
    id INT,
    unique_column INT UNIQUE
);

INSERT INTO example VALUES (1, NULL);  -- OK
INSERT INTO example VALUES (2, NULL);  -- OK
INSERT INTO example VALUES (3, 100);   -- OK
INSERT INTO example VALUES (4, 100);   -- 실패
```

---

## 9. NULL과 인덱스

* 일반 인덱스: NULL 값 포함 가능
* WHERE 조건에서 인덱스 활용 가능하지만, `IS NOT NULL`이나 함수 사용 시 풀 테이블 스캔 가능

```sql
CREATE INDEX idx_column ON table(column);
SELECT * FROM table WHERE column IS NULL;  -- 인덱스 사용 가능
SELECT * FROM table WHERE IFNULL(column, '') = 'value'; -- 인덱스 사용 불가
```

---

## 10. NULL과 조인(JOIN)

| JOIN 종류    | NULL 처리             |
| ---------- | ------------------- |
| INNER JOIN | 양쪽 모두 NULL 아닌 행만 매칭 |
| LEFT JOIN  | 왼쪽 기준, 오른쪽 NULL 허용  |

```sql
SELECT * 
FROM table1 t1
LEFT JOIN table2 t2 ON t1.id = t2.id;
```

---

## 11. NULL과 그룹화(GROUP BY)

* NULL 값들은 하나의 그룹으로 묶임

```sql
SELECT column, COUNT(*)
FROM table
GROUP BY column;
```

---

## 12. 실전 예제

```sql
-- NULL-safe 비교
SELECT * FROM employees
WHERE COALESCE(department, '') = COALESCE(@input_department, '');

-- NULL 처리한 계산
SELECT employee_id,
       COALESCE(salary, 0) + COALESCE(bonus, 0) AS total_income
FROM employees;

-- NULL 포함 조건부 업데이트
UPDATE employees 
SET salary = COALESCE(salary, 0) + 1000
WHERE department = 'IT';
```

---

## 13. 주의사항

| 실수      | 설명                                       |
| ------- | ---------------------------------------- |
| NULL 비교 | `column = NULL` → 항상 FALSE               |
| IN 절    | `column IN (1, 2, NULL)` → NULL은 비교되지 않음 |
| NOT IN  | `column NOT IN (1, 2, NULL)` → 항상 FALSE  |
| 집계 함수   | AVG, SUM 등 NULL 제외되는 점 주의                |

* 권장 사항

  1. NULL 사용 여부 명확히 결정
  2. 애플리케이션 레벨에서 NULL 처리 고려
  3. COALESCE/IFNULL 사용
  4. NOT IN 대신 NOT EXISTS 활용

---
