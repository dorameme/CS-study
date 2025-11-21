# 루스 인덱스 스캔(Loose Index Scan)

## 1. 개념

루스 인덱스 스캔은 MySQL 옵티마이저가 **인덱스의 전체 레코드를 읽지 않고, 필요한 인덱스 키만 건너뛰며 탐색하는 방식**이다.     
GROUP BY, DISTINCT 최적화에서 주로 사용되며, "중복되지 않는 인덱스 키만 순차적으로 스캔"하여 비용을 줄인다.      

즉, **필요한 그룹의 첫 번째 인덱스 엔트리만 읽고 나머지는 스킵**하는 방식이다.

또한 MIN()/MAX() 함수 최적화에도 사용되며, **인덱스의 시작점이나 끝점만 스캔**하여 빠르게 결과를 얻는다.       
GROUP BY 쿼리에서도 적용 가능하며, 필요에 따라 **부분적으로만 인덱스를 읽는 방식**으로 동작한다.

---

## 2. 동작 방식 (핵심 원리)

1. 인덱스에서 GROUP BY 또는 DISTINCT 대상 컬럼을 기준으로 정렬된 상태를 활용한다.
2. 같은 그룹에 속한 연속된 인덱스 엔트리를 모두 읽지 않고, **해당 그룹의 첫 번째 키만 읽는다.**
3. 그다음 그룹의 첫 번째 키로 점프하며 스캔한다.

예: GROUP BY col1

* 인덱스: (1, 1, 1, 2, 2, 3, 3)
* 루스 인덱스 스캔이 읽는 값: **1 → 2 → 3**

---

## 3. 루스 인덱스 스캔이 사용되는 조건

루스 인덱스 스캔은 특정 조건을 충족해야만 동작한다.

### 필수 조건

1. **GROUP BY 또는 DISTINCT 사용**
2. **GROUP BY/DISTINCT 컬럼이 단일 인덱스 또는 복합 인덱스의 왼쪽부터 사용됨**
3. WHERE 절이 있다면, 반드시 인덱스를 무너뜨리지 않아야 함

### 예시로 가능한 인덱스 구조

```sql
INDEX (a)
INDEX (a, b)
INDEX (a, b, c)
```

### 가능한 경우

```sql
SELECT a FROM t GROUP BY a;
SELECT a, b FROM t GROUP BY a, b;
SELECT DISTINCT a FROM t;
SELECT DISTINCT a, b FROM t;
```

### 불가능한 경우 (루스 인덱스 스캔 X)

* SELECT * 사용 (불필요한 컬럼 때문에 전체 인덱스가 필요)
* GROUP BY 순서가 인덱스를 따르지 않음
* WHERE 절이 인덱스를 깨뜨리는 경우

예:

```sql
SELECT a FROM t WHERE b = 10 GROUP BY a;  -- 인덱스 (a,b)라면 불가능
SELECT * FROM t GROUP BY a;               -- * 때문에 모든 컬럼 필요 -> 루스 인덱스 X
```

---

## 4. 실제 실행 계획에서 확인하는 방법

```sql
EXPLAIN SELECT a FROM t GROUP BY a;
```

실행 계획의 Extra 항목에서 다음 문구가 보이면 루스 인덱스 스캔이 동작하는 것이다.

* **Using index for group-by**
* **Using index for distinct**

특징:

* Using temporary, Using filesort가 없어야 한다.

---

## 5. 실제 활용 예시

### 1) DISTINCT 최적화

```sql
SELECT DISTINCT user_id
FROM logs;
```

**인덱스: (user_id)**
→ 루스 인덱스 스캔 사용됨

### 2) GROUP BY 최적화

```sql
SELECT user_id
FROM logs
GROUP BY user_id;
```

→ user_id 기준 인덱스가 있다면 가능

### 3) 복합 인덱스에서의 활용

인덱스: (user_id, action_type)

```sql
SELECT user_id, action_type
FROM logs
GROUP BY user_id, action_type;
```

→ 두 컬럼 모두 인덱스 왼쪽부터 조합이므로 루스 인덱스 스캔 가능

### 4) MIN()/MAX() 최적화

```sql
SELECT MIN(timestamp) FROM logs;
SELECT MAX(timestamp) FROM logs;
```

→ 인덱스의 시작점 또는 끝점만 읽고 결과 반환 가능

---

## 6. 사용 시 주의사항

1. SELECT 절에 인덱스에 없는 컬럼을 포함하면 루스 인덱스 스캔이 불가능하다.
2. GROUP BY 컬럼 순서는 인덱스 정의와 완전히 일치해야 한다.
3. WHERE 절 조건이 인덱스를 타지 못하면 루스 인덱스 스캔이 해제된다.
4. COUNT(*) 같은 집계함수는 루스 인덱스 스캔과 함께 사용할 수 없다.

---

## 7. 정리 표

| 구분    | 설명                                                     |
| ----- | ------------------------------------------------------ |
| 개념    | 인덱스의 모든 레코드를 읽지 않고 필요한 그룹의 첫 번째 키만 읽는 방식               |
| 등장 상황 | DISTINCT, GROUP BY 최적화, MIN()/MAX() 최적화                |
| 필수 조건 | 인덱스의 왼쪽부터 컬럼 일치, SELECT 컬럼이 인덱스 컬럼으로 제한                |
| 실행 계획 | "Using index for group-by", "Using index for distinct" |
| 장점    | 읽어야 할 인덱스 엔트리 수 감소 → 빠른 성능                             |
| 단점    | 제약이 많고 SELECT * 불가능                                    |
| 대표 사례 | DISTINCT user_id, GROUP BY user_id, MIN()/MAX()        |
| 특징    | GROUP BY 쿼리에서도 적용 가능, 부분적으로만 인덱스를 읽음                   |

---

## 8. 결론

루스 인덱스 스캔은 조건이 까다롭지만 **DISTINCT/GROUP BY/MIN()/MAX() 작업에서 성능을 크게 개선**할 수 있다.
다만 인덱스 구조와 SELECT 절 컬럼 구성에 따라 쉽게 비활성화될 수 있으므로,
SQL과 인덱스를 설계할 때 미리 이를 고려하는 것이 중요하다.
