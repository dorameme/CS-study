## MySQL EXPLAIN: Extra, Using index / Using index condition 설명

MySQL 실행 계획(EXPLAIN)에서 `Extra`와 `Using index`/`Using index condition`는 쿼리 실행 방식과 인덱스 활용 상태를 나타낸다.

---

## 1. Extra 컬럼

* EXPLAIN에서 `Extra`는 **쿼리 실행 시 MySQL이 추가로 수행하는 작업**을 보여준다.
* 예: `Using where`, `Using index`, `Using temporary`, `Using filesort` 등

---

## 2. Using index

* 의미: **커버링 인덱스(Covering Index) 사용**
* 테이블 데이터를 직접 읽지 않고, **인덱스만으로 SELECT 컬럼과 조건을 처리**했다는 뜻이다.
* 장점: 디스크 I/O 감소 → 조회 속도 빠름

### 예시

```sql
CREATE INDEX idx_emp_dept_salary ON Employees(dept_id, salary);

EXPLAIN SELECT dept_id, salary
FROM Employees
WHERE dept_id = 10 AND salary > 5000;
```

* Extra: `Using index`
* 의미: 인덱스만으로 결과 조회 가능 → 테이블 접근 불필요

---

## 3. Using index condition

* 의미: **Index Condition Pushdown (ICP)** 사용
* 조건 일부를 인덱스에서 바로 필터링하고, 나머지는 테이블에서 확인
* 장점: 전체 테이블을 읽는 것보다 효율적, 하지만 커버링 인덱스만큼 빠르지는 않음

### 예시

```sql
SELECT name
FROM Employees
WHERE dept_id = 10 AND salary > 5000;
```

* 인덱스 순서: (dept_id, salary)
* 조회 컬럼: name (인덱스에 없음)
* Extra: `Using index condition`
* 의미: dept_id, salary 조건은 인덱스에서 필터링 → name은 테이블에서 읽음

---

## 4. 차이 정리

| 구분                    | 의미         | 테이블 접근 여부    | 속도                |
| --------------------- | ---------- | ------------ | ----------------- |
| Using index           | 커버링 인덱스 사용 | 테이블 접근 없음    | 가장 빠름             |
| Using index condition | ICP 적용     | 일부 컬럼 테이블 접근 | 빠르지만 커버링 인덱스보다 느림 |
