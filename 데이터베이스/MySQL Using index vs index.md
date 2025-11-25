# Extra : Using index  vs type: index 

이 두 가지는 **서로 다른 차원**의 실행 계획 정보.

## **Using index Extra 컬럼**

### **의미**
- **커버링 인덱스Covering Index**가 사용됨
- **인덱스만 읽고 쿼리를 완료**할 수 있음
- 실제 데이터 파일(테이블) 접근 불필요

### **발생 조건**
```sql
-- 예시 1: 모든 컬럼이 인덱스에 포함된 경우
CREATE INDEX idx_covering ON employees (department_id, salary);

-- 커버링 인덱스 사용 (Using index)
EXPLAIN SELECT department_id, salary FROM employees 
WHERE department_id = 5;
-- Extra: Using index
-- type: ref (인덱스 조회)
```

```sql
-- 예시 2: PK만 조회하는 경우
EXPLAIN SELECT emp_id FROM employees;
-- Extra: Using index  (PK는 항상 인덱스에 포함)
-- type: index
```

## **type: index (접근 방식)**

### **의미**
- **인덱스 풀 스캔(Index Full Scan)** 수행
- 인덱스의 **처음부터 끝까지 모두 읽음**
- WHERE 조건이 인덱스 컬럼을 사용하지 않거나, 범위가 너무 넓은 경우

### **발생 조건**
```sql
-- 예시 1: 인덱스 컬럼으로 정렬 but WHERE 조건 없음
CREATE INDEX idx_dept_salary ON employees (department_id, salary);

EXPLAIN SELECT * FROM employees ORDER BY department_id, salary;
-- type: index  (인덱스 풀 스캔)
-- Extra: Using index? → NO! (인덱스만으로 처리 불가)
```

```sql
-- 예시 2: 인덱스의 일부 컬럼만 조건으로 사용
EXPLAIN SELECT * FROM employees WHERE salary > 5000;
-- type: index  (department_id 없어서 인덱스 효율적 사용 불가)
-- key: idx_dept_salary 사용 but 풀 스캔
```

## **조합별 시나리오**

### **시나리오 1: Using index O, type: index**
```sql
-- 인덱스 풀 스캔 but 커버링 인덱스
CREATE INDEX idx_covering ON products (category_id, product_name, price);

EXPLAIN SELECT category_id, product_name FROM products;
-- type: index        (인덱스 풀 스캔)
-- Extra: Using index (커버링 인덱스)
-- key: idx_covering
```

**설명:** 인덱스 전체를 스캔하지만, 필요한 데이터가 모두 인덱스에 포함됨

### **시나리오 2: Using index X, type: index**
```sql
-- 인덱스 풀 스캔 + 데이터 접근 필요
EXPLAIN SELECT * FROM products ORDER BY category_id, product_name;
-- type: index        (인덱스 풀 스캔)
-- Extra: NULL        (커버링 인덱스 아님)
-- key: idx_covering
```

**설명:** 인덱스 전체 스캔 후, 실제 데이터에도 접근해야 함

### **시나리오 3: Using index O, type: ref**
```sql
-- 효율적인 인덱스 범위 스캔 + 커버링 인덱스
EXPLAIN SELECT category_id, product_name 
FROM products 
WHERE category_id = 'ELECTRONICS';
-- type: ref         (인덱스 범위 스캔)
-- Extra: Using index (커버링 인덱스)
-- key: idx_covering
```

**설명:** 가장 이상적인 경우 - 효율적인 인덱스 사용 + 데이터 접근 불필요

### **시나리오 4: Using index X, type: ref**
```sql
-- 효율적인 인덱스 범위 스캔 but 데이터 접근 필요
EXPLAIN SELECT * FROM products 
WHERE category_id = 'ELECTRONICS';
-- type: ref         (인덱스 범위 스캔)
-- Extra: NULL       (커버링 인덱스 아님)
-- key: idx_covering
```

**설명:** 인덱스는 효율적으로 사용 but 실제 데이터 접근 필요

## **성능 비교**

### **성능 랭킹 (좋음 → 나촄)**
```sql
1. type: ref + Using index    -- 최상: 효율적 인덱스 스캔 + 커버링
   SELECT indexed_col FROM table WHERE indexed_col = 'value';

2. type: ref                  -- 양호: 효율적 인덱스 스캔
   SELECT * FROM table WHERE indexed_col = 'value';

3. type: index + Using index  -- 보통: 인덱스 풀 스캔 but 커버링
   SELECT indexed_col FROM table;

4. type: index               -- 나쁨: 인덱스 풀 스캔 + 데이터 접근
   SELECT * FROM table ORDER BY indexed_col;

5. type: ALL                 -- 최악: 테이블 풀 스캔
   SELECT * FROM table;
```

## **실제 예제 분석**

```sql
-- 테이블 생성
CREATE TABLE user_actions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    action_type VARCHAR(20),
    action_date DATETIME,
    details TEXT,
    INDEX idx_user_action (user_id, action_type, action_date)
);

-- 케이스 1: type=ref + Using index (최적)
EXPLAIN SELECT user_id, action_type 
FROM user_actions 
WHERE user_id = 100 AND action_type = 'LOGIN';
-- type: ref, Extra: Using index

-- 케이스 2: type=index + Using index (보통)
EXPLAIN SELECT user_id, action_type, action_date 
FROM user_actions;
-- type: index, Extra: Using index

-- 케이스 3: type=index (나쁨)
EXPLAIN SELECT * FROM user_actions 
ORDER BY user_id, action_type;
-- type: index, Extra: NULL

-- 케이스 4: type=ref (양호)
EXPLAIN SELECT * FROM user_actions 
WHERE user_id = 100;
-- type: ref, Extra: NULL
```

## **최적화 팁**

### **Using index를 활용하려면**
```sql
-- 1. 자주 조회하는 컬럼을 인덱스에 포함
CREATE INDEX idx_covering ON table (condition_col, select_col1, select_col2);

-- 2. 필요한 컬럼만 SELECT
-- 나쁨: SELECT * (커버링 불가)
-- 좋음: SELECT indexed_col1, indexed_col2 (커버링 가능)

-- 3. INCLUDED 컬럼 사용 (MySQL 8.0+)
CREATE INDEX idx_include ON table (condition_col) 
INCLUDE (select_col1, select_col2);
```

### **type: index를 피하려면**
```sql
-- 1. WHERE 조건을 추가하여 범위 축소
-- 나쁨: SELECT ... ORDER BY indexed_col
-- 좋음: SELECT ... WHERE indexed_col = value ORDER BY indexed_col

-- 2. 인덱스 선두 컬럼 조건 사용
-- 나쁨: WHERE indexed_col2 = value  (선두 컬럼 없음)
-- 좋음: WHERE indexed_col1 = value AND indexed_col2 = value
```

## **결론**

- **`Using index`**: "인덱스만으로 처리 가능" (커버링 인덱스)
- **`type: index`**: "인덱스 전체를 스캔" (인덱스 풀 스캔)

**이상적인 조합**: `type: ref` + `Using index`
**주의 필요한 조합**: `type: index` (데이터 많을수록 성능 저하)

