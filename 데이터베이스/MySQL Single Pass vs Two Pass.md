# MySQL Single Pass vs Two Pass 정리

## 1. 개념 요약

### Single Pass

Single Pass(싱글 패스)는 **데이터를 한 번만 스캔하여 결과를 생성하는 방식**이다.      
MySQL이 테이블이나 인덱스를 스캔하면서 **동시에 결과를 만들 수 있을 때** 적용된다.

특징:

* 한 번의 스캔으로 정렬 없이 결과 생성 가능
* ORDER BY가 인덱스 순서와 일치하면 사용 가능
* GROUP BY가 인덱스 순서와 일치하면 사용 가능
* 정렬 파일 정렬(sort buffer) 불필요
* 성능이 빠르고 메모리 비용이 낮다

---

### Two Pass

Two Pass(투 패스)는 **데이터를 두 번 이상 처리하는 방식**이다.
정렬 또는 그룹 처리를 위해 **임시 버퍼(sorting buffer)나 temporary table**에 데이터를 저장한 뒤
한 번 더 처리해야 하는 경우 발생한다.

특징:

* 정렬 또는 그룹화를 위해 데이터를 한 번 모으고, 다시 정렬/집계하는 두 단계 필요
* ORDER BY가 인덱스 의미와 다르면 발생
* GROUP BY가 인덱스 순서와 다르면 발생
* temporary table 생성 가능
* 성능 오버헤드 발생

---

## 2. Single Pass vs Two Pass 비교 표

| 항목          | Single Pass         | Two Pass         |
| ----------- | ------------------- | ---------------- |
| 스캔 횟수       | 1회                  | 2회               |
| 정렬 필요 여부    | 없음                  | 있음               |
| GROUP BY 처리 | 인덱스 순서와 일치하면 정렬 불필요 | 인덱스와 불일치하면 정렬 필요 |
| ORDER BY 처리 | 인덱스 순서와 일치해야 함      | 인덱스 순서 불일치 시 발생  |
| 임시 테이블 사용   | 거의 없음               | 생성 가능            |
| 성능          | 빠르고 비용 낮음           | 느리고 비용 높음        |
| 옵티마이저 판단    | 인덱스 기반으로 즉시 실행 가능   | 중간 정렬 단계 필요      |

---

## 3. 예시로 이해하는 Single Pass / Two Pass

테이블:

```sql
CREATE TABLE log_data (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    created_at DATETIME,
    INDEX idx_user_created (user_id, created_at)
);
```

---

### 예시 1: Single Pass

```sql
SELECT *
FROM log_data
WHERE user_id = 10
ORDER BY created_at;
```

이 쿼리는 Single Pass가 가능하다.

이유:

* WHERE user_id = 10 은 인덱스 idx_user_created의 선두 컬럼을 사용함
* created_at은 인덱스의 두 번째 컬럼이며 순서대로 정렬되어 있음
* 인덱스 범위 스캔으로 이미 정렬된 순서로 데이터를 읽을 수 있음
* 추가 정렬 단계가 필요하지 않음

실행 방식:

1. 인덱스 범위 스캔
2. 결과 순서 유지
3. 바로 반환

---

### 예시 2: Two Pass

```sql
SELECT *
FROM log_data
WHERE user_id = 10
ORDER BY id;
```

이 쿼리는 Two Pass가 발생한다.

이유:

* WHERE user_id = 10 은 인덱스를 이용해 범위 스캔 가능
* 하지만 ORDER BY id는 인덱스 순서와 관련 없음
* 결과를 모두 뽑은 후 id 기준으로 다시 정렬해야 함
* 따라서 정렬 버퍼 또는 temporary table을 사용하여 정렬 필요

실행 방식:

1. 인덱스 또는 테이블에서 데이터 모음
2. id 기준으로 정렬
3. 정렬된 결과 반환

---

### 예시 3: GROUP BY에서 Two Pass

```sql
SELECT user_id, COUNT(*)
FROM log_data
GROUP BY user_id;
```

가능한 경우:

* idx_user_created(user_id, created_at)을 사용하면 user_id 기준으로 이미 정렬되어 있으므로
  Single Pass로 GROUP BY 처리 가능

하지만 다음처럼 다른 조건이 들어가면 Two Pass 발생:

```sql
SELECT user_id, COUNT(*)
FROM log_data
GROUP BY user_id
ORDER BY created_at;
```

이 경우:

* user_id로 그룹핑
* created_at은 그룹 순서와 정렬 순서가 다름
* 다시 정렬 필요 → Two Pass

---

## 4. 핵심 정리

1. Single Pass는 **정렬 없이 인덱스 스캔만으로 결과를 만들 수 있을 때** 발생한다.
2. Two Pass는 **정렬 필요** 또는 **GROUP BY/ORDER BY 방향이 인덱스와 맞지 않을 때** 발생한다.
3. Single Pass가 가능한 쿼리를 만드는 것이 성능적으로 유리하다.
4. 정렬이나 그룹화가 필요한 경우 MySQL은 임시 테이블 또는 정렬 버퍼를 사용해 Two Pass로 처리한다.

---
