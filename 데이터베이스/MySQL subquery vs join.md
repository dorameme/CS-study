

# 1. JOIN이 일반적으로 더 권장되는 이유

### 1) 옵티마이저가 더 많은 최적화 선택지를 가진다

JOIN은 테이블을 그대로 결합하는 구조라서 MySQL 옵티마이저가 다음을 자유롭게 선택할 수 있다.

* 어떤 테이블을 먼저 읽을 것인지(드라이빙 테이블 선정)
* 어떤 인덱스를 사용할 것인지
* 조인 버퍼 활용
* 범위 스캔, 인덱스 스캔, 해시 조인(8.0 이후), NL 조인 등 최적화된 전략 선택

반면 서브쿼리는 내부 쿼리의 결과를 **임시 테이블로 만들어 놓고** 외부에서 이용하는 경우가 많아 최적화 여지가 제한된다.

---

### 2) IN/EXISTS 구조에서는 내부 쿼리가 반복 실행될 수 있다

서브쿼리가 다음과 같은 구조일 때:

```sql
SELECT *
FROM Orders
WHERE customer_id IN (SELECT id FROM Customers WHERE country='USA');
```

MySQL 버전에 따라 내부 쿼리를 매 반복마다 수행하거나, 임시 테이블을 만든 뒤 작업해야 한다.
JOIN에서는 두 테이블을 결합하면서 한 번에 필터링하여 훨씬 효율적이다.

---

### 3) 임시 테이블 및 메모리 사용 증가

서브쿼리 결과를 MySQL이 물리적 임시 테이블로 만들면 다음 부하가 발생한다.

* 디스크 I/O
* 메모리 사용
* 인덱스 미사용

JOIN은 보통 임시 테이블이 필요 없다.

---

# 2. 그런데? 서브쿼리가 더 좋은 상황도 존재한다 (중요)

JOIN이 무조건 더 좋은 것은 아니다.
특정 상황에서는 **서브쿼리가 더 빠르고, 더 가독성이 좋고, 더 의미적으로 명확**하다.

---

## 상황 A. 집계 결과를 직접 조건으로 사용할 때

예:

```sql
SELECT *
FROM Customers
WHERE id = (SELECT MAX(customer_id) FROM Orders);
```

이걸 JOIN으로 억지로 바꾸면?

```sql
SELECT c.*
FROM Customers c
JOIN (
    SELECT MAX(customer_id) AS max_id FROM Orders
) t ON c.id = t.max_id;
```

JOIN이 오히려 더 복잡하며, 의도도 불분명해진다.
이 경우 **서브쿼리가 더 자연스럽고 빠르다.**

이유는 다음과 같다:

* 내부 서브쿼리는 단 1회 실행
* 하나의 스칼라 값만 반환 → 매우 효율적
* 옵티마이저가 별도 최적화 여부를 고민할 필요 없음

---

## 상황 B. EXISTS / NOT EXISTS

EXISTS는 다음과 같은 특징을 가진다.

* 첫 매칭되는 행만 발견하면 즉시 true/false 반환
* 조건이 만족되면 즉시 종료 → 매우 효율적
* 조인으로 변환하면 쓸데없이 전체 레코드를 조합할 수 있다

예:

```sql
SELECT c.name
FROM Customers c
WHERE EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.customer_id = c.id
);
```

JOIN으로 바꾸면?

```sql
SELECT DISTINCT c.name
FROM Customers c
JOIN Orders o ON o.customer_id = c.id;
```

문제:

* DISTINCT까지 필요할 수 있음
* 조인 과정에서 불필요한 결합이 발생
* EXISTS가 훨씬 더 빠르게 종료 가능

따라서 EXISTS / NOT EXISTS는 서브쿼리가 본질적으로 더 효율적이다.

---

# 3. 실제 JOIN vs 서브쿼리 실행 계획 비교 (EXPLAIN 예제)

테이블 구조 예시:

```sql
CREATE TABLE Customers (
    id INT PRIMARY KEY,
    country VARCHAR(20)
);

CREATE TABLE Orders (
    id INT PRIMARY KEY,
    customer_id INT,
    amount INT,
    INDEX idx_customer (customer_id)
);
```

---

# 예제 1. 서브쿼리 IN vs JOIN

## (1) 서브쿼리 IN 사용

```sql
EXPLAIN
SELECT *
FROM Orders
WHERE customer_id IN (
    SELECT id FROM Customers WHERE country='USA'
);
```

### 예상되는 실행 계획 동작

* 내부 Customers 서브쿼리를 먼저 수행
* 결과를 메모리 임시 테이블(temp table)에 저장
* Orders를 스캔하며 customer_id가 temp table에 포함되는지 체크

문제점

* temp table 접근
* 인덱스를 완벽히 활용하지 못할 가능성 있음

---

## (2) JOIN 사용

```sql
EXPLAIN
SELECT o.*
FROM Orders o
JOIN Customers c ON o.customer_id = c.id
WHERE c.country='USA';
```

### 예상 실행 계획

* Customers(country='USA') → 인덱스 가능
* Orders(customer_id) 인덱스 탐색 → 빠른 Nested Loop Join
* 임시 테이블 불필요
* 옵티마이저가 드라이빙 테이블을 최적 선택 가능

이 경우 **JOIN이 더 빠른 계획이 나온다.**

---

# 예제 2. EXISTS vs JOIN

## (1) EXISTS

```sql
EXPLAIN
SELECT c.id
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.customer_id = c.id
);
```

### 예상 실행 방식

* Customers에서 한 row를 읽는다
* 해당 고객의 주문이 존재하는지 Orders 인덱스(idx_customer)에서 즉시 검사
* 첫 매칭 row 있으면 즉시 TRUE
* 매우 빠른 조기 종료 가능

→ 이 경우 EXISTS가 최적이다.

---

## (2) JOIN 강제 변환

```sql
EXPLAIN
SELECT DISTINCT c.id
FROM Customers c
JOIN Orders o ON o.customer_id = c.id;
```

### 실행 방식

* Orders 전체에 대해 조인 발생
* DISTINCT로 중복 제거해야 함
* 조기 종료 불가
* 훨씬 더 많은 작업 필요

→ EXISTS가 현저히 더 효율적이다.

---

# 최종 정리

## JOIN이 일반적으로 더 좋은 이유

* 옵티마이저 최적화 선택 폭이 넓다
* 인덱스 활용 유리
* 임시 테이블 감소
* 반복 실행 감소
* 읽기 성능 우수

---

## 하지만 서브쿼리가 더 나은 명확한 경우

1. **스칼라 집계(subquery returning single value)**
2. **EXISTS / NOT EXISTS 구조**
3. **의도가 JOIN보다 더 명확한 경우**
4. **불필요한 중복 제거(DISTINCT) 없이 존재 여부만 확인하는 경우**
