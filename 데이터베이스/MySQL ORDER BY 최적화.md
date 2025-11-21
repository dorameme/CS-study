

# MySQL 조인에서 ORDER BY 최적화 가이드

## 1. 핵심 개념

* **드라이빙 테이블(Driving Table)**: 조인에서 먼저 읽는 테이블, WHERE 조건 적용 후 결과를 줄이는 역할
* **드리븐 테이블(Driven Table)**: 드라이빙 테이블 결과를 바탕으로 조회되는 테이블
* **최적화 원칙**: ORDER BY는 **드라이빙 테이블 컬럼 위주로 지정**해야 정렬 비용이 최소화된다.

---

## 2. 왜 드라이빙 테이블 기준이 중요한가

1. MySQL은 기본적으로 **드라이빙 테이블만 정렬** 후 드리븐 테이블과 조인한다.
2. 드리븐 테이블 컬럼이 ORDER BY에 포함되면 **임시 테이블 + Filesort**가 발생한다.
3. 인덱스를 활용하면 Single Pass 정렬 가능 → Two Pass 및 임시 테이블 비용 감소.

---

## 3. 예제 1: 드라이빙 테이블 기준 정렬

테이블 생성:

```sql
CREATE TABLE customer (
    id INT PRIMARY KEY,
    age INT
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    amount INT,
    created_at DATETIME,
    INDEX idx_customer_id(customer_id)
);
```

쿼리:

```sql
EXPLAIN
SELECT *
FROM customer c
JOIN orders o ON c.id = o.customer_id
WHERE c.age > 30
ORDER BY c.age;
```

예상 EXPLAIN 핵심 컬럼:

| id | table | type  | key             | rows | Extra       |
| -- | ----- | ----- | --------------- | ---- | ----------- |
| 1  | c     | range | PRIMARY         | 100  | Using where |
| 1  | o     | ref   | idx_customer_id | 2000 | Using index |

* **c가 드라이빙 테이블**, **c.age 기준 정렬** → Single Pass 가능
* **o는 인덱스 기반으로 참조**, 별도 정렬 필요 없음

---

## 4. 예제 2: 드리븐 테이블 컬럼 포함 시 비용 증가

쿼리:

```sql
EXPLAIN
SELECT *
FROM customer c
JOIN orders o ON c.id = o.customer_id
WHERE c.age > 30
ORDER BY o.created_at, c.age;
```

* `o.created_at`가 드리븐 테이블 컬럼이므로
  MySQL은 전체 조인 결과를 **임시 테이블로 만들고 Filesort 수행**
* 드라이빙 테이블 기준만 ORDER BY에 있는 경우보다 비용이 높다.

---

## 5. 최적화 전략

| 전략                  | 설명                                             |
| ------------------- | ---------------------------------------------- |
| 드라이빙 테이블 컬럼 우선      | ORDER BY는 드라이빙 테이블 기준으로 지정하면 성능이 가장 좋다         |
| LIMIT 활용            | 드리븐 테이블 컬럼 정렬이 필요한 경우, LIMIT으로 결과 줄이기          |
| 인덱스 활용              | 드라이빙 테이블 ORDER BY 컬럼에 인덱스를 추가 → Single Pass 가능 |
| STRAIGHT_JOIN 신중 사용 | 순서를 강제하면 옵티마이저 선택을 무시 → 비용 증가 가능               |

---

## 6. 실행 계획 활용 팁

1. EXPLAIN에서 **table, type, key, Extra** 확인
2. Driving/Driven 확인: 첫 번째 table이 일반적으로 드라이빙
3. ORDER BY와 드라이빙 테이블 컬럼 일치 여부 확인
4. Filesort나 Using temporary가 있으면 최적화 필요

---

## 7. 정리 표

| 항목             | 드라이빙 테이블 기준 ORDER BY         | 드리븐 테이블 컬럼 포함 ORDER BY |
| -------------- | ---------------------------- | ---------------------- |
| 정렬 위치          | 드라이빙 테이블만 정렬                 | 전체 조인 결과 임시 테이블로 정렬    |
| Filesort 발생 여부 | 없음/적음                        | 있음                     |
| 임시 테이블 사용 여부   | 없음                           | 있음                     |
| 성능             | 최고                           | 낮음                     |
| 인덱스 활용         | 가능 → Single Pass             | 제한적, Two Pass 가능       |
| 최적화 전략         | ORDER BY 컬럼 = 드라이빙 테이블 + 인덱스 | LIMIT 사용, 필요 최소화       |
