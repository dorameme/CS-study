# MySQL 서브쿼리와 파생 테이블 정리

## 1. 서브쿼리 개념

서브쿼리는 쿼리 내부에 포함된 또 다른 쿼리이다. 하나의 값, 여러 값, 임시 구조를 반환할 수 있으며 위치에 따라 의미와 실행 방식이 달라진다.

서브쿼리는 다음 위치에 모두 올 수 있다.

* SELECT 절 (스칼라 서브쿼리)
* WHERE 절 (조건 서브쿼리)
* HAVING 절 (집계 조건)
* ORDER BY 절
* FROM 절 (파생 테이블)

## 2. 위치별 서브쿼리 종류

### 2.1 SELECT 절 서브쿼리 (스칼라 서브쿼리)

```sql
SELECT name, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;
```

하나의 결과값을 반환하며 컬럼처럼 사용된다.

### 2.2 WHERE 절 서브쿼리

```sql
SELECT *
FROM users
WHERE id IN (SELECT user_id FROM payments WHERE amount > 1000);
```

조건 필터링용으로 사용된다. IN, EXISTS, NOT EXISTS 등과 주로 결합된다.

### 2.3 HAVING 절 서브쿼리

집계된 결과에 조건을 적용할 때 사용한다.

```sql
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id
HAVING COUNT(*) > (SELECT AVG(order_count) FROM monthly_order_stats);
```

### 2.4 ORDER BY 절 서브쿼리

정렬용 표현식으로 사용 가능하다.

```sql
SELECT *
FROM users
ORDER BY (SELECT MAX(amount) FROM payments p WHERE p.user_id = users.id) DESC;
```

### 2.5 FROM 절 서브쿼리 (파생 테이블)

FROM 절에 올 때만 특별히 파생 테이블(derived table)이라고 부른다.

```sql
SELECT d.user_id, d.cnt
FROM (
    SELECT user_id, COUNT(*) AS cnt
    FROM orders
    GROUP BY user_id
) AS d;
```

이 쿼리는 MySQL 내부에서 임시 테이블을 생성하여 그 테이블을 기반으로 이후 조인을 실행한다.

## 3. 파생 테이블(derived table)

파생 테이블은 FROM 절에 위치한 서브쿼리이다. MySQL은 이를 먼저 실행하고 결과를 임시 테이블(temporary table)에 담는다. 이후 이 임시 테이블을 실제 테이블과 동일하게 취급하여 나머지 쿼리를 수행한다.

특징은 다음과 같다.

* 반드시 임시 테이블을 사용한다.
* 인덱스가 자동 생성되지 않기 때문에 성능 문제가 발생할 수 있다.
* 옵티마이저가 전체 쿼리를 재배치할 수 없으므로 최적화가 제한된다.
* 필요 시 materialized 형태로 결과를 저장한 뒤 재사용한다.

## 4. 왜 파생 테이블은 FROM 절에서만 가능한가?

서브쿼리는 어디든 위치할 수 있지만, FROM 절에 있을 때만 테이블처럼 동작해야 한다. MySQL은 FROM 절에 있는 서브쿼리를 테이블로 다뤄야 하므로 별도의 임시 테이블 구조가 필요하고, 이 구조를 파생 테이블이라고 부른다.

WHERE, SELECT 등의 서브쿼리는 단순한 표현식이며 테이블처럼 사용되지 않는다. 그러므로 파생 테이블이 아니다.

## 5. 서브쿼리의 실행 방식

MySQL은 상황에 따라 다음과 같은 최적화를 적용할 수 있다.

* Semi-Join 최적화 (IN, EXISTS 최적화)
* IN-to-EXISTS 변환
* Subquery materialization
* Index subquery 최적화

이 최적화들은 WHERE 절 기반 서브쿼리에 주로 적용되며, FROM 절 파생 테이블에는 적용되지 않는다.

## 6. 파생 테이블과 임시 테이블의 관계

파생 테이블은 항상 임시 테이블을 생성한다. 다만 모든 임시 테이블이 파생 테이블인 것은 아니다. 임시 테이블은 다음 상황에서도 만들어진다.

* GROUP BY 처리
* ORDER BY 처리
* DISTINCT 처리
* UNION 처리

즉, 파생 테이블은 임시 테이블을 필요로 하는 여러 상황 중 하나일 뿐이다.

## 7. 파생 테이블이 비추천되는 이유

* 인덱스가 없어 조인이 비효율적이다.
* 메모리 또는 디스크 임시 테이블을 생성하는 비용이 크다.
* 옵티마이저가 쿼리 재배치(Join Reorder)를 할 수 없어서 전체 계획이 비효율적이 될 수 있다.

가능하면 JOIN 또는 윈도우 함수로 대체하는 것이 좋다.

## 8. 서브쿼리가 오히려 유리한 경우

특정 상황에서는 파생 테이블이 아닌 서브쿼리가 더 효율적일 수 있다.

### 8.1 집계값 비교 목적

```sql
SELECT * FROM users
WHERE score > (SELECT AVG(score) FROM users);
```

전체 집계 결과를 단 한 번만 계산하면 된다.

### 8.2 EXISTS 기반 존재 확인

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

EXISTS는 조건이 충족되면 즉시 중단되므로 매우 빠르다.

### 8.3 덜 복잡한 실행 계획

WHERE 절 기반 서브쿼리는 Semi-Join 최적화를 받을 수 있으나, 파생 테이블은 그런 최적화를 받기 어렵다.

## 9. 정리

* 서브쿼리는 어디든 올 수 있다.
* FROM 절 서브쿼리만 파생 테이블이며 임시 테이블 기반으로 동작한다.
* 파생 테이블은 성능 제한이 있으며 가능하면 JOIN이나 윈도우 함수로 대체하는 것이 좋다.
* WHERE, SELECT 절 서브쿼리는 다양한 최적화가 가능하며 존재 확인, 집계 비교 같은 경우에 매우 효율적이다.
