## `range checked for each record`에 대한 설명

`range checked for each record`는 MySQL의 EXPLAIN 실행 계획에서 나타나는 메시지로, 인덱스를 효율적으로 사용하지 못하고 있는 상황을 나타낸다.

## 의미

이 메시지는 MySQL이 각 레코드마다 인덱스 범위 스캔을 수행해야 함을 의미한다.    
즉, 테이블의 레코드를 하나씩 처리할 때마다 해당 레코드에 대한 인덱스 범위 검사를 추가로 수행하는 비효율적인 작업 방식이다.

## 발생 조건

주로 다음과 같은 상황에서 발생한다:

1. 조인 조건에서 인덱스 사용 불가

```sql
-- 예시: t1.col1에 인덱스가 있지만, t2.col2에는 인덱스가 없는 경우
SELECT * FROM t1, t2 WHERE t1.col1 = t2.col2;
```

2. 상관 서브쿼리

```sql
-- 외부 쿼리의 각 행마다 서브쿼리 실행
SELECT * FROM employees e
WHERE salary > (SELECT AVG(salary) FROM salaries s WHERE s.emp_no = e.emp_no);
```

3. 인덱스되지 않은 컬럼을 사용한 조인

## 성능 영향

* Nested Loop Join에서 특히 문제된다.
* O(n × m)의 시간 복잡도를 가질 수 있다.
* 대량의 데이터 처리 시 성능 저하가 심각하다.

## 해결 방법

### 1. 적절한 인덱스 생성

```sql
-- 조인에 사용되는 컬럼에 인덱스 생성
ALTER TABLE t2 ADD INDEX idx_col2 (col2);
```

### 2. 쿼리 최적화

```sql
-- 서브쿼리를 조인으로 변경
SELECT e.*
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE e.salary > s.avg_salary;
```

### 3. 임시 테이블 활용

```sql
-- 서브쿼리 결과를 임시 테이블로 추출
CREATE TEMPORARY TABLE avg_salaries AS
SELECT emp_no, AVG(salary) as avg_salary FROM salaries GROUP BY emp_no;

SELECT e.*
FROM employees e
JOIN avg_salaries a ON e.emp_no = a.emp_no
WHERE e.salary > a.avg_salary;
```

### 4. 설정 최적화

```sql
-- 임시 테이블 메모리 크기 증가
SET tmp_table_size = 256 * 1024 * 1024;
SET max_heap_table_size = 256 * 1024 * 1024;
```

## 실제 예시

```sql
-- 문제 있는 쿼리
EXPLAIN SELECT * FROM orders o, customers c
WHERE o.customer_name = c.name AND o.amount > 1000;
-- 결과: 'range checked for each record' 발생
```

```sql
-- 해결 방법: customers.name에 인덱스 추가
ALTER TABLE customers ADD INDEX idx_name (name);
```

## 모니터링 방법

```sql
-- 실행 계획 확인
EXPLAIN FORMAT=JSON YOUR_QUERY;

-- 성능 모니터링
SHOW STATUS LIKE 'Handler_read%';
SHOW PROFILE;
```

## 결론

`range checked for each record`는 인덱스 최적화가 필요함을 나타내는 중요한 신호이다. 적절한 인덱스 생성과 쿼리 최적화를 통해 이 문제를 해결하면 성능을 크게 개선할 수 있다.
