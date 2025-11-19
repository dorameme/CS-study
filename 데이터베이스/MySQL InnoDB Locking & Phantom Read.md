# MySQL InnoDB Locking & Phantom Read

## 1. InnoDB에서 사용되는 잠금 종류

### 1.1 레코드 락(Record Lock)

이미 존재하는 특정 행만 잠근다.
특정 PK 또는 Unique Key로 정확히 한 행을 조회하면 레코드 락만 걸린다.

예시

```sql
SELECT * FROM users WHERE id = 5 FOR UPDATE;
```

id=5인 기존 행만 잠긴다.
INSERT는 허용된다.

---

### 1.2 갭 락(Gap Lock)

행과 행 사이의 빈 공간을 잠근다.
범위 조건 검색 시 해당 범위 안에 새로운 행이 삽입되는 것을 막는다.

예시

```sql
SELECT * FROM users WHERE age > 30 FOR UPDATE;
```

조건 범위 내의 빈 공간이 잠기므로 새로운 INSERT가 차단된다.

---

### 1.3 넥스트 키 락(Next-Key Lock)

레코드 락 + 갭 락이 결합된 형태이다.
행과 그 앞 구간까지 함께 잠가서 범위 내 INSERT를 완전히 차단한다.
InnoDB의 기본 잠금 방식이다.

예시

```sql
SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
```

---

## 2. InnoDB의 팬텀 리드 발생 여부

격리 수준에 따라 팬텀 리드 가능 여부가 달라진다.

| Isolation Level  | Fanthom Read | 설명                      |
| ---------------- | ------------ | ----------------------- |
| REPEATABLE READ  | 발생하지 않는다     | 넥스트 키 락으로 범위 INSERT 차단  |
| READ COMMITTED   | 발생 가능        | 갭 락을 기본적으로 사용하지 않음      |
| READ UNCOMMITTED | 발생 가능        | Dirty Read 포함 여러 문제가 발생 |
| SERIALIZABLE     | 발생하지 않는다     | 읽기도 락으로 막는 방식           |

InnoDB의 REPEATABLE READ는 강화된 형태로 팬텀 리드를 차단한다.

---

## 3. 팬텀 리드가 실제로 발생하는 예제

아래 예제는 격리 수준이 READ COMMITTED일 때 팬텀 리드가 일어나는 시나리오이다.

### 초기 데이터

users 테이블

| id | age |
| -- | --- |
| 1  | 25  |
| 2  | 30  |

---

### 트랜잭션 예제

#### 트랜잭션 A

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;

SELECT * FROM users WHERE age >= 25;
```

결과
1 (25), 2 (30) 두 개의 행만 조회된다.

---

#### 트랜잭션 B

```sql
START TRANSACTION;

INSERT INTO users(age) VALUES(28);
COMMIT;
```

트랜잭션 B가 28세 사용자를 추가한다.

---

#### 트랜잭션 A

같은 범위를 다시 조회한다.

```sql
SELECT * FROM users WHERE age >= 25;
```

결과
1 (25), 2 (30)
추가로 insert된 28세 데이터가 포함된다.

이처럼 트랜잭션 A의 범위 조회 중간에 새로운 행이 끼어드는 현상이 팬텀 리드이다.

READ COMMITTED에서는 발생하지만
REPEATABLE READ와 SERIALIZABLE에서는 발생하지 않는다.

---

## 4. 핵심 요약

1. 레코드 락은 기존 행만 잠근다.
2. 갭 락은 행 사이 공간을 잠궈 INSERT를 막는다.
3. 넥스트 키 락은 행 + 앞 구간까지 잠그는 InnoDB 기본 방식이다.
4. InnoDB의 REPEATABLE READ는 넥스트 키 락 때문에 팬텀 리드가 발생하지 않는다.
5. READ COMMITTED에서는 팬텀 리드가 발생할 수 있다.
