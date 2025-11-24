# MySQL에서 OUTER JOIN 설명 및 LEFT JOIN + IS NULL 패턴 정리

## 1. OUTER JOIN이란

OUTER JOIN은 조인된 두 테이블 중 조건에 맞지 않는 행을 포함해서 결과를 반환하는 조인 방식이다.            
즉, 한쪽 테이블의 모든 행을 유지하면서 다른 쪽 테이블과 매칭되는 값이 없을 경우 NULL 값을 채워서 반환한다.

### OUTER JOIN 종류

* LEFT OUTER JOIN: 왼쪽 테이블의 모든 행을 유지한다.
* RIGHT OUTER JOIN: 오른쪽 테이블의 모든 행을 유지한다.
* FULL OUTER JOIN: 양쪽 테이블의 모든 행을 유지한다. (MySQL은 지원하지 않지만 UNION으로 구현 가능하다.)

## 2. LEFT JOIN + IS NULL 패턴이 OUTER JOIN인 이유

다음과 같은 쿼리를 자주 본다.

```sql
SELECT A.*
FROM A
LEFT JOIN B ON A.id = B.a_id
WHERE B.a_id IS NULL;
```

이 쿼리는 다음을 의미한다.

* LEFT JOIN으로 A의 모든 행을 유지한다.
* B와 조인 조건이 맞지 않는 경우 B.*는 NULL이 된다.
* WHERE 절에서 `B.a_id IS NULL` 조건을 주면, “B와 매칭되지 않은 A의 행”만 필터링된다.

즉, LEFT JOIN 자체가 OUTER JOIN이기 때문에 `LEFT JOIN + IS NULL`은 “조인 후 매칭 실패한 레코드만 필터링”하는 **안티 조인(Anti Join) 패턴**이다.

이 패턴의 본질은 다음 과정이다.

1. OUTER JOIN 수행
2. 매칭된 B가 없는 경우에만 A를 남김(= Anti Join 효과)

## 3. 왜 NOT IN / NOT EXISTS 대신 LEFT JOIN + IS NULL을 사용하는가

MySQL 최적화기에서 LEFT JOIN + IS NULL 패턴은 다음 장점이 있다.

1. 조인 순서를 더 유연하게 최적화할 수 있다.
2. 인덱스가 있는 경우, OUTER JOIN → NULL 체크 방식이 매우 빠르게 동작할 수 있다.
3. NULL 비교가 명확하고 예측 가능하다.

예를 들어 아래 두 쿼리는 같은 의미지만 실행 계획은 다를 수 있다.

```sql
SELECT A.* FROM A
WHERE A.id NOT IN (SELECT a_id FROM B);
```

```sql
SELECT A.* FROM A
LEFT JOIN B ON A.id = B.a_id
WHERE B.a_id IS NULL;
```

많은 상황에서 MySQL은 두 번째 방식(LEFT JOIN + IS NULL)을 더 효율적으로 최적화한다.

## 4. 예시로 보는 OUTER JOIN 동작 방식

### 데이터 준비

```sql
A:              B:
id | name       a_id | value
---+------      -----+------
 1 | a            1  | x
 2 | b            3  | y
 3 | c
```

### LEFT JOIN 수행 결과

```sql
SELECT A.id, A.name, B.value
FROM A
LEFT JOIN B ON A.id = B.a_id;
```

결과

```
1 | a | x
2 | b | NULL
3 | c | y
```

LEFT JOIN은 A의 모든 행을 유지한다.

### LEFT JOIN + IS NULL 사용

```sql
SELECT A.id, A.name
FROM A
LEFT JOIN B ON A.id = B.a_id
WHERE B.a_id IS NULL;
```

결과

```
2 | b
```

즉, B와 매칭되지 않는 A(=A.id=2)만 반환한다.




## 5. 정리

* OUTER JOIN은 조인 조건과 상관없이 한쪽 테이블의 모든 행을 유지한다.
* LEFT JOIN + IS NULL 패턴은 OUTER JOIN의 특성을 활용해 **안티 조인**을 구현하는 방식이다.
* 이 패턴은 MySQL에서 널리 사용되며, NOT IN이나 NOT EXISTS보다 더 안정적이거나 빠를 때가 많다.
* 최적화기 입장에서 OUTER JOIN 기반 비교는 조인 순서를 더 유연하게 선택할 수 있어 성능 면에서도 이점이 있다.

## 좀 더 자세한 설명

차근차근 풀어서 설명하면 이렇게 이해하면 된다.


# 1. 상황

```sql
SELECT A.*
FROM A
LEFT JOIN B ON A.id = B.a_id
WHERE B.a_id IS NULL;
```

* A: 드라이빙 테이블
* B: 조인 대상 테이블, `a_id` 컬럼에 인덱스 있음

우리는 **B에 매칭되는 값이 없는 A만 찾고 싶다.**

---

# 2. 인덱스가 없을 경우

* A의 각 row마다 B를 풀스캔해야 한다.
* B가 1,000,000건이면, 1건씩 비교 → 총 비용 매우 큼
* 즉, **nested loop + full scan** → 느림

---

# 3. 인덱스가 있을 경우

* B.a_id 에 인덱스가 있으면, **B에서 특정 값 조회가 O(log N)**로 가능
* A의 각 row를 가져오면서 B 인덱스를 탐색 → 매칭 여부 확인
* B와 매칭되지 않으면 NULL 반환 → 조건문 `B.a_id IS NULL` 체크
* Full table scan이 아니라 **index lookup**만 수행 → 훨씬 빠름

---

# 4. 동작 방식 그림 (단순화)

```
A 테이블: 1, 2, 3
B 테이블 (인덱스 있음): 1, 3

반복:
1) A.id=1 → B 인덱스 lookup → 존재 → skip
2) A.id=2 → B 인덱스 lookup → 없음 → 결과에 포함
3) A.id=3 → B 인덱스 lookup → 존재 → skip
```

* 인덱스 덕분에 **B 전체를 스캔하지 않고도 매칭 여부 확인 가능**
* 따라서 LEFT JOIN + IS NULL 패턴이 NOT IN / NOT EXISTS보다 더 빠른 경우가 많다.

---

# 5. 결론

* 인덱스가 있으면 OUTER JOIN → NULL 체크 방식은 **A는 풀스캔하더라도 B는 인덱스로 바로 탐색 가능**
* Anti Join 효과를 얻으면서 **조인 비용을 최소화**할 수 있다
* 즉, **속도 면에서 효율적**이다




## 추가 - LEFT JOIN 은 LEFT OUTER JOIN 과 동일하다

MySQL에서 `LEFT JOIN`은 SQL 표준의 `LEFT OUTER JOIN`과 완전히 동일한 의미를 가진다.
`OUTER` 키워드는 선택적으로 생략할 수 있으며, 기능적 차이는 없다.

```sql
SELECT ...
FROM A
LEFT JOIN B ON ...;
```

```sql
SELECT ...
FROM A
LEFT OUTER JOIN B ON ...;
```

두 쿼리는 MySQL 옵티마이저 입장에서 완전히 같은 조인으로 처리된다.

### 정리

* LEFT JOIN = LEFT OUTER JOIN
* RIGHT JOIN = RIGHT OUTER JOIN
* FULL OUTER JOIN은 MySQL에서 지원하지 않으며 UNION 기반으로 시뮬레이션해야 한다.
