`innodb_autoinc_lock_mode`는 **AUTO_INCREMENT 값을 할당할 때 InnoDB가 어떤 잠금 전략을 사용할지** 결정한다.

---

## 0 — Traditional(전통 모드)

### 동작

- 모든 INSERT 계열 문에서 **AUTO-INC 테이블 락(AUTO_INCREMENT lock)**을 획득한다.
    
- 락은 **statement가 끝날 때까지 유지**된다.
    

`INSERT INTO t VALUES (NULL, ...);`

→ 다른 세션은 AUTO_INCREMENT 할당이 끝날 때까지 대기한다.

### 특징

- AUTO_INCREMENT 값이 **항상 연속적**이다(조건이 같다면).
    
- **statement 단위로 완전 직렬화**되어 동시성이 매우 낮다.
    
- **SBR(Statement-Based Replication)에서 안전**하다.
    

### 적합한 경우

- 레거시 환경
    
- **연속 ID가 매우 중요한** 워크로드
    
- 동시성이 낮은 시스템
    

---

## 1 — Consecutive(중간 모드)

### 동작

모든 INSERT가 동일하게 동작하지 않고, **문 종류에 따라** 전략이 달라진다.

#### (1) 단일 row INSERT

- AUTO-INC 테이블 락을 사용하지 않는다.
    
- 내부적으로 **가벼운 mutex**로 AUTO_INCREMENT를 할당한다.
    

`INSERT INTO t VALUES (NULL, ...);`

#### (2) 다중 row / Bulk INSERT 계열

- AUTO-INC 테이블 락을 사용해 **해당 statement를 직렬화**한다.
    

`INSERT INTO t SELECT ...; LOAD DATA INSERT INTO t ...; INSERT INTO t VALUES (...), (...), (...);`

### 특징

- 대부분의 일반 INSERT에서 **성능과 동시성이 개선**된다.
    
- Bulk 계열에서는 여전히 직렬화된다.
    
- AUTO_INCREMENT 값은 **statement 단위로는 연속성**이 유지된다.
    
- **SBR에서 안전**하다.
    

### 기본값(과거)

|버전|기본값|
|---|---|
|MySQL 5.1 ~ 5.7|1|

---

## 2 — Interleaved(고동시성 모드)

### 동작

- AUTO-INC 테이블 락을 **사용하지 않는다**.
    
- 항상 **lightweight mutex 기반**으로 여러 트랜잭션이 동시에 AUTO_INCREMENT를 할당한다.
    

### 특징

- **동시성이 가장 높다.**
    
- ID는 **단조 증가 + 유니크**만 보장한다.
    
- **statement 단위 연속성은 보장되지 않으며**, gap이 생길 수 있다.
    

### SBR 주의

- master/slave에서 **동시 실행 순서가 달라지면** AUTO_INCREMENT 값이 달라질 수 있다.
    
- 따라서 **SBR에서는 충돌 가능성이 있어 주의가 필요**하다.
    

### 기본값

|버전|기본값|
|---|---|
|MySQL 8.0|2|

---

## 요약

|값|이름|잠금 전략|동시성|연속성 보장|SBR 안전성|
|---|---|---|---|---|---|
|0|Traditional|모든 INSERT에서 AUTO-INC 테이블 락|낮음|높음|안전|
|1|Consecutive|단일 row는 mutex, Bulk 계열은 테이블 락|보통|statement 단위 보장|안전|
|2|Interleaved|테이블 락 없음(mutex만 사용)|높음|보장 안 됨(gap 가능)|주의|

**핵심 정리**

- `=2`는 **최대 동시성**을 얻는 대신, 동시 INSERT에서 ID가 **interleave**될 수 있어 **연속성은 보장되지 않는다**.
    
- `=0`은 **연속성/재현성**이 강하지만, 동시성이 가장 낮다.
    
- `=1`은 일반적인 환경에서 **균형점**이 된다.