
# MySQL InnoDB와 비관적 락 가이드

## 1. InnoDB와 레코드 기반 잠금

### 레코드 기반 잠금이란?

InnoDB는 트랜잭션에서 필요한 **행(row) 단위만 잠그는 방식**을 지원한다.

* 동일 테이블에서 여러 트랜잭션이 동시에 작업 가능
* 필요한 행만 잠기기 때문에 동시성 처리 효율이 높음

### 테이블 단위 잠금과 비교

* MyISAM 등은 테이블 전체를 잠금
* 한 트랜잭션이 실행되면 다른 트랜잭션이 대기해야 함
* InnoDB는 필요한 행만 잠금 → 동시성 유지

### 레코드 잠금의 장점

* 동시에 여러 사용자가 다른 행을 수정 가능
* OLTP 시스템에서 성능 우수
* 트랜잭션 + MVCC + 레코드 잠금을 지원하는 MySQL의 사실상 유일한 스토리지 엔진        
(참고로 OLTP '온라인 트랜잭션 처리(Online Transaction Processing)'의 약자로        
온라인 사용자가 데이터베이스에 실시간으로 데이터를 입력하거나 조회하는 시스템)
---

## 2. InnoDB가 기본인 이유

* 완전한 트랜잭션(ACID) 지원
* 외래키(Foreign Key) 지원
* Crash Recovery 기능 강화
* 높은 동시성 처리 가능
* MySQL 5.5 이후 기본 스토리지 엔진

---

## 3. 비관적 락(Pessimistic Lock) 사용 시 고려 사항

### 장점

* 레코드 단위 잠금 덕분에 테이블이 복잡하게 연결되어도 대부분 성능 문제 없음
* PK 기반 조회에서 단일 행만 잠그면 동시성 매우 높음

### 성능에 영향을 줄 수 있는 경우

1. 인덱스 없는 조건 + 범위 검색
2. JOIN + FOR UPDATE 남발
3. 범위 검색 시 Next-Key Lock 발생
4. 트랜잭션 내에서 오래 작업 수행

---

## 4. 안전하게 비관적 락을 쓰는 패턴

### 1) PK 또는 유니크 인덱스 기반 한 건 조회

```sql
SELECT * 
FROM account 
WHERE id = 123 
FOR UPDATE;
```

* 정확히 한 행만 잠금
* 락 경합 최소화

### 2) SELECT → UPDATE 순서 일관성

* 모든 트랜잭션이 같은 순서로 락을 잡아야 데드락 최소화

### 3) 락 유지 시간 최소화

* 락 잡은 상태에서 계산, 외부 API 호출, 파일 처리 금지
* 빠르게 UPDATE 후 COMMIT

### 4) 범위 락 필요 시 인덱스 확인

* 범위 조건에 인덱스 없으면 테이블 절반 이상 잠길 수 있음
* 반드시 적절한 인덱스 확보 후 FOR UPDATE 사용

---

## 5. 피해야 하는 패턴

1. 인덱스 없는 WHERE + FOR UPDATE

```sql
SELECT * FROM orders WHERE type = 'A' FOR UPDATE;
```

2. JOIN + FOR UPDATE 남발

```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.id = 10
FOR UPDATE;
```

3. 범위 검색 + FOR UPDATE

```sql
SELECT * FROM orders WHERE id >= 100 FOR UPDATE;
```

4. 트랜잭션 내 오래 작업

* 비즈니스 로직 처리, 외부 API 호출, 파일 업로드 등 금지

---

## 6. 핵심 요약

* InnoDB 성능은 **행 단위 잠금과 MVCC 덕분**에 높음
* 핵심 전략

  * PK 기반 한 row만 잠금
  * 적절한 인덱스 확보
  * 트랜잭션 짧게 유지
* 잘못 사용하면 레코드락이어도 성능과 동시성이 급격히 떨어짐


