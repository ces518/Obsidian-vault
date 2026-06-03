---
type: concept
area: database
status: draft
updated: 2026-06-03
tags:
  - DB
  - MySQL
  - InnoDB
  - AUTO_INCREMENT
  - 복제
created: 2026-03-21
---

`innodb_autoinc_lock_mode=2`(interleaved)는 AUTO_INCREMENT 락을 최소화해 동시성을 높이지만, **Statement-Based Replication(SBR)** 과 함께 사용하면 **비결정적(재현 불가) 실행**으로 인해 마스터와 레플리카의 AUTO_INCREMENT 값이 달라질 수 있다.

원인은 **마스터에서 AUTO_INCREMENT 값이 세션 간 interleave로 배정되는 순서가 동시성에 따라 달라지는데**, SBR은 binlog에 **SQL 문장만 기록**하고 레플리카에서 **문장을 단일 스레드로 재실행**하므로 동일 문장이라도 실행 시점의 상태에 따라 다른 값이 배정되기 때문이다.

## 한 줄 요약

> `innodb_autoinc_lock_mode=2`(interleaved)는 동시성을 높이지만 SBR과 함께 쓰면 마스터의 interleave 배정 순서를 레플리카가 단일 스레드 재실행으로 재현하지 못해 AUTO_INCREMENT 값이 어긋나므로, SBR이면 lock_mode=1을 쓰거나 `binlog_format=ROW`로 전환한다.

> [!info]
> `innodb_autoinc_lock_mode` 각 값의 동작은 [[innodb_autoinc_lock_mode 값별 동작]] 참고

---

## 발생 가능한 문제

- **multi-row INSERT / INSERT … SELECT / LOAD DATA**에서 마스터는 interleave로 인해 한 문장이 연속 범위를 고정적으로 받지 못할 수 있다.
    예) 마스터에서는 같은 INSERT가 `101,103,106,…`처럼 배정될 수 있지만, 레플리카에서는 단독 실행되어 `101,102,103,…`처럼 연속으로 배정될 수 있다.
- 애플리케이션이 **LAST_INSERT_ID()**, **id 기반 샤딩/규칙**, **외래 연동 키** 등으로 생성된 ID에 의미를 부여하면 참조 무결성이나 응용 로직이 깨질 수 있다.
- 결과적으로 레플리카에서 **Duplicate key**, 시퀀스 불일치, 또는 **마스터/레플리카 데이터 불일치**가 발생할 수 있다(예: checksum 도구로 탐지).

---

## binlog_format 옵션별 복제 동작

- **ROW-based replication(RBR)** 에서는 변경된 row 자체(=AUTO_INCREMENT 값 포함)를 복제하므로 마스터에서 배정된 값이 그대로 적용되어 안전하다.
- **MIXED**는 일부 비결정적 문장을 RBR로 기록할 수 있지만, 쿼리 패턴/버전/옵션에 따라 기대와 다르게 동작할 수 있어 안정성을 최우선하면 ROW가 가장 확실하다.

> [!tip]
> binlog_format별 상세 동작은 [[binlog_format 값별 동작]] 참고

---

## 실무 권장 조합

- **SBR을 사용한다면**: `innodb_autoinc_lock_mode=2`는 피하고 `innodb_autoinc_lock_mode=1(consecutive)`를 사용하거나 `binlog_format=ROW`로 전환한다.
- **lock_mode=2가 필요하다면**: `binlog_format=ROW`를 권장하며, MIXED를 쓸 경우 `INSERT … SELECT`, `LOAD DATA`, 대량 multi-row INSERT 등 **unsafe 가능 패턴과 경고 로그**를 반드시 모니터링한다.

---

## 면접식 설명

> `innodb_autoinc_lock_mode=2`는 AUTO_INCREMENT 락을 최소화해 동시성을 높이지만 SBR과 함께 쓰면 위험하다. 마스터에서는 세션 간 interleave로 값이 배정되는 순서가 동시성에 따라 달라지는데, SBR은 binlog에 SQL 문장만 기록하고 레플리카는 이를 단일 스레드로 재실행하기 때문에 같은 문장이라도 다른 값이 배정된다. 특히 multi-row INSERT나 `INSERT … SELECT`, `LOAD DATA`에서 마스터는 `101,103,106…`처럼 띄엄띄엄 받을 수 있지만 레플리카는 단독 실행으로 `101,102,103…`처럼 연속 배정될 수 있고, 애플리케이션이 ID에 샤딩·연동 키 같은 의미를 부여하면 Duplicate key나 마스터/레플리카 데이터 불일치로 이어진다. RBR(ROW)에서는 마스터가 배정한 값까지 포함한 row 자체를 복제하므로 안전하다. 따라서 SBR이면 lock_mode=1을 쓰거나 `binlog_format=ROW`로 전환하고, lock_mode=2가 필요하면 ROW를 권장한다.

---

## 관련 문서

- [[innodb_autoinc_lock_mode 값별 동작]]
- [[binlog_format 값별 동작]]
- [[MySQL 로그 시스템 (Binlog, Relay, Redo, Undo)]]
- [[MySQL Repeatable Read 격리 수준]]