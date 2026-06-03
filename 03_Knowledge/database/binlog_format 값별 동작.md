---
type: concept
area: database
status: draft
updated: 2026-06-03
tags:
  - DB
  - MySQL
  - binlog
  - 복제
created: 2026-03-21
---

# 개요

`binlog_format`이 STATEMENT(SBR), ROW(RBR), MIXED일 때 각각 binlog에 무엇을 기록하고 어떤 특징·제약을 가지는지 정리한 문서다.

## 한 줄 요약

> STATEMENT는 SQL 문장을 기록해 크기는 작지만 비결정적 쿼리에서 재현이 깨지고, ROW는 변경된 row 이미지를 기록해 안전하지만 크기가 커지며, MIXED는 둘의 절충안이다.

---

## STATEMENT (SBR)

### 동작

- 실행한 **SQL 문장**을 그대로 binlog에 기록
- replica에서 **동일 SQL을 재실행**해 변경 사항을 재현

```sql
UPDATE t SET col = 1 WHERE id = 10;
```

### 특징

- binlog 크기가 작다
- **쿼리가 결정적으로 동작한다는 가정**에 의존한다
- `NOW()`, `UUID()`, `AUTO_INCREMENT`(동시 실행) 등 **비결정적 동작**이 섞이면 재현 결과가 달라질 수 있다

### 기본값

| 버전 | 기본값 |
| --- | --- |
| MySQL 5.0 ~ 5.7 | **STATEMENT** |

---

## ROW (RBR)

### 동작

- 변경된 row의 **변경 전/후 이미지**를 binlog에 기록

```
id=10, col: 0 → 1
```

### 특징

- SQL 재실행이 아니라 **결과(row 변화)**를 기록하므로 비결정성 문제가 없다
- 병렬 replication에서 동작이 안정적이다
- 변경량이 크면 binlog 크기가 커질 수 있다

### 기본값

| 버전 | 기본값 |
| --- | --- |
| MySQL 8.0 | **ROW** |

---

## MIXED

### 동작

- 기본은 SBR로 기록
- 비결정적이거나 안전하지 않은 쿼리는 RBR로 기록

### 특징

- SBR과 RBR의 절충안이다
- 쿼리별로 기록 방식이 달라져 운영/분석 복잡도가 올라간다
- 최근에는 RBR을 주로 사용해 선택 빈도가 낮다

---

## 면접식 설명

> `binlog_format`은 binlog에 무엇을 기록할지를 정한다. STATEMENT(SBR)는 실행한 SQL 문장을 그대로 기록하고 replica에서 재실행하는 방식이라 binlog 크기는 작지만, `NOW()`, `UUID()`, 동시 실행되는 AUTO_INCREMENT처럼 비결정적인 동작이 섞이면 마스터와 replica 결과가 달라질 수 있다. ROW(RBR)는 변경된 row의 전/후 이미지를 기록하므로 SQL 재실행이 아니라 결과를 그대로 적용해 비결정성 문제가 없고 병렬 복제도 안정적이지만 변경량이 크면 binlog이 커진다. MIXED는 평소엔 SBR로 기록하다 안전하지 않은 쿼리만 RBR로 전환하는 절충안인데, 운영·분석 복잡도가 올라가 최근엔 RBR을 주로 쓴다. 기본값도 5.0~5.7은 STATEMENT, 8.0은 ROW로 바뀌었다.

---

## 관련 문서

- [[MySQL 로그 시스템 (Binlog, Relay, Redo, Undo)]]
- [[innodb_autoinc_lock_mode 와 replication 문제]]
- [[innodb_autoinc_lock_mode 값별 동작]]
- [[MySQL Repeatable Read 격리 수준]]
- [[Binlog Purge 동작과 영향]]
