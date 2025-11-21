
# MySQL 조인에서 Driving & Driven 테이블

## 1. 개념 정의

### 드라이빙 테이블(Driving Table)

조인에서 **가장 먼저 읽는 테이블**이다.    
MySQL이 조인을 수행할 때 기준이 되는 테이블로,    
WHERE 조건으로 먼저 필터링을 수행하며 결과 집합을 최소화하는 것이 중요하다.

### 드리븐 테이블(Driven Table)

드라이빙 테이블의 결과를 바탕으로 **두 번째로 읽는 테이블**이다.   
드라이빙 테이블의 각 행마다 조인 조건을 이용해 조회된다.    
효율적인 조인을 위해 드리븐 테이블에는 조인 키에 인덱스가 있어야 한다.

---

## 2. 옵티마이저 역할

MySQL 옵티마이저는 조인 순서를 자동으로 결정한다.     
선택 기준은 다음과 같다.

* WHERE 조건으로 결과를 최소화할 수 있는 테이블을 드라이빙 테이블로 선택한다.
* 드라이빙 결과가 작으면 드리븐 테이블 접근 횟수가 줄어 전체 조인 비용이 낮아진다.
* 조인 키에 인덱스가 있으면 옵티마이저가 이를 고려하여 비용을 계산한다.
* 사용자가 명시적으로 `STRAIGHT_JOIN` 힌트를 주면 순서를 강제로 지정할 수 있다.

즉, 대부분의 경우 옵티마이저가 알아서 드라이빙/드리븐을 결정하지만,
실행 계획(EXPLAIN)을 확인하여 성능에 영향을 주는 순서를 검토하는 것이 중요하다.

---

## 3. EXPLAIN 예제

테이블:

```sql
CREATE TABLE customer (
    id INT PRIMARY KEY,
    age INT
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    amount INT,
    INDEX idx_customer_id(customer_id)
);
```

쿼리 1:

```sql
EXPLAIN
SELECT *
FROM customer c
JOIN orders o ON c.id = o.customer_id
WHERE c.age > 30;
```

예상 EXPLAIN 출력 (핵심 컬럼만):

| id | select_type | table | type  | possible_keys   | key             | rows | Extra       |
| -- | ----------- | ----- | ----- | --------------- | --------------- | ---- | ----------- |
| 1  | SIMPLE      | c     | range | PRIMARY         | PRIMARY         | 100  | Using where |
| 1  | SIMPLE      | o     | ref   | idx_customer_id | idx_customer_id | 2000 | Using index |

설명:

* `c`가 드라이빙 테이블: range 스캔으로 조건(age > 30) 필터링
* `o`가 드리븐 테이블: c.id를 이용해 인덱스 idx_customer_id 참조
* 드라이빙 결과가 작을수록 o 조회 횟수가 줄어 성능이 향상된다.

---

쿼리 2: STRAIGHT_JOIN으로 순서 강제

```sql
EXPLAIN
SELECT *
FROM orders o
STRAIGHT_JOIN customer c ON c.id = o.customer_id
WHERE c.age > 30;
```

* `o`가 강제로 드라이빙 테이블이 된다.
* customer를 조회할 때 age > 30 필터링 전에 orders 전체를 탐색해야 하므로 비용 증가 가능.

---

## 4. 정리 표

| 항목       | Driving Table                     | Driven Table                   |
| -------- | --------------------------------- | ------------------------------ |
| 정의       | 조인에서 **먼저 읽는 테이블**                | 드라이빙 결과를 바탕으로 **두 번째로 읽는 테이블** |
| 역할       | WHERE 조건 적용, 결과 최소화               | 조인 조건 기반으로 조회                  |
| 선택 기준    | 결과 집합이 적은 테이블, 인덱스 활용             | 조인 키 인덱스 존재 시 효율적              |
| 성능 영향    | 작을수록 조인 전체 비용 감소                  | 드라이빙 결과 개수에 따라 접근 횟수 결정        |
| 옵티마이저 결정 | 자동 결정, 필요 시 STRAIGHT_JOIN으로 강제 가능 | 드라이빙 기준으로 자동 결정                |
| 실행 계획 확인 | EXPLAIN의 table, type, key 컬럼      | EXPLAIN의 table, type, key 컬럼   |
