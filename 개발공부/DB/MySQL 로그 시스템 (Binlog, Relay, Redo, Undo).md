---
tags:
  - DB
  - MySQL
  - InnoDB
  - 로그
  - 복제
created: 2026-03-22
---

# 1. Binary Log (Binlog)

## 개요

- **레벨**: MySQL Server 레벨 (스토리지 엔진과 무관)
- **목적**: 복제(Replication), Point-in-Time Recovery (PITR), CDC(Change Data Capture)
- **기록 내용**: 데이터를 **변경**하는 모든 SQL문 또는 행 변경사항 (SELECT 제외)
- **파일 형태**: 파일 단위로 순차 생성 (`mysql-bin.000001`, `mysql-bin.000002`, …)
- **디스크 영향**: 무한 증가 → expire 설정 또는 수동 purge로 직접 삭제 필요

## 기록 포맷 (`binlog_format`)

![[binlog_format 값별 동작]]

## 동작 방식

1. 트랜잭션 실행 중 변경사항을 **binlog cache**(메모리)에 버퍼링
2. 트랜잭션 COMMIT 시 binlog 파일에 기록
3. `sync_binlog=1`이면 매 커밋마다 디스크 fsync (안전하지만 느림)

## 핵심 설정

| 설정 | 설명 |
| --- | --- |
| `log_bin` | 활성화 여부 |
| `binlog_expire_logs_seconds` | 자동 삭제 주기 (기본 30일) |
| `max_binlog_size` | 파일 크기 제한 (기본 1GB, 초과 시 rotate) |
| `sync_binlog` | fsync 빈도 (1 = 매 커밋) |

> [!warning]
> `binlog_expire_logs_seconds` 기본값이 **2592000(30일)**이므로 별도 설정 없이는 30일치 binlog가 쌓여 디스크 FULL의 원인이 된다. [[MySQL 운영 장애]] 참고

---

# 2. Relay Log (릴레이 로그)

## 개요

- **레벨**: MySQL Server 레벨 (Replica/Slave에만 존재)
- **목적**: Master의 binlog를 수신하여 **임시 저장**하는 중간 버퍼
- **디스크 영향**: replica lag이 있으면 계속 쌓임 → Primary보다 Replica가 먼저 디스크 FULL이 날 수 있음

## 동작 방식

```
Master                          Replica (Slave)
┌──────────┐    네트워크     ┌──────────────┐     ┌──────────────┐
│  Binlog  │ ──────────────→│  Relay Log   │────→│  데이터 적용   │
└──────────┘   IO Thread     └──────────────┘     └──────────────┘
                              (디스크 저장)        SQL Thread
```

1. **I/O Thread**: Master에 접속하여 binlog 이벤트를 읽어와 relay log에 기록
2. **SQL Thread**: relay log를 읽어서 실제 SQL/행 변경을 Slave DB에 적용
3. 적용 완료된 relay log는 자동 삭제

## 핵심 특징

- 포맷은 binlog과 동일 (사실상 binlog의 복사본)
- Master-Slave 간 네트워크 속도 차이를 흡수하는 **버퍼 역할**
- `relay_log_recovery=ON`이면 Slave 재시작 시 relay log를 버리고 Master에서 다시 수신

---

# 3. Redo Log (InnoDB)

## 개요

- **레벨**: InnoDB 스토리지 엔진 레벨
- **목적**: **Crash Recovery** — 비정상 종료 후 커밋된 트랜잭션의 데이터를 복구
- **로그 타입**: 물리적(Physical) — 페이지 단위의 변경 기록
- **디스크 영향**: 고정 크기(순환) → 디스크 FULL의 원인이 되지 않음

## WAL (Write-Ahead Logging) 원리

```
UPDATE 실행
    │
    ▼
Buffer Pool(메모리)의 페이지 수정 → "Dirty Page" 생성
    │
    ▼
Redo Log에 변경 내용 기록 (순차 I/O, 빠름)
    │
    ▼
COMMIT 성공 응답 반환
    │
    ... (나중에 비동기로)
    ▼
Dirty Page를 디스크의 데이터 파일에 Flush (랜덤 I/O, 느림)
```

> [!tip]
> 디스크에 바로 쓰면 **랜덤 I/O**라 느리지만, Redo Log는 **순차 I/O**라 빠르다. 커밋 시 redo log만 디스크에 기록하고, 실제 데이터 파일 flush는 비동기로 처리하여 성능을 확보한다.

## 순환 버퍼 구조

```
┌─────────────┐    ┌─────────────┐
│ ib_logfile0 │───→│ ib_logfile1 │──┐
└─────────────┘    └─────────────┘  │
       ▲                            │
       └────────────────────────────┘
            순환(Circular) 방식
```

- 고정 크기 파일을 **순환** 방식으로 재사용
- `LSN (Log Sequence Number)`으로 위치 추적
- **Checkpoint**: dirty page가 디스크에 flush되면 해당 redo log 공간을 재사용 가능으로 표시

## 핵심 설정

| 설정 | 설명 |
| --- | --- |
| `innodb_log_file_size` | 각 redo log 파일 크기 |
| `innodb_log_files_in_group` | 파일 수 (기본 2) |
| `innodb_flush_log_at_trx_commit` | flush 정책 (아래 표 참고) |

### `innodb_flush_log_at_trx_commit` 값별 동작

| 값 | 동작 | 특성 |
| --- | --- | --- |
| **1** | 매 커밋마다 flush + fsync | 가장 안전, 가장 느림 **(기본값)** |
| **0** | 1초마다 flush | 최대 1초 데이터 손실 가능 |
| **2** | 매 커밋마다 OS 버퍼까지만 flush | OS 크래시 시 손실 가능 |

---

# 4. Undo Log (InnoDB)

## 개요

- **레벨**: InnoDB 스토리지 엔진 레벨
- **목적**: 트랜잭션 **롤백** + **MVCC** (Multi-Version Concurrency Control)
- **로그 타입**: 논리적(Logical) — 변경 전 데이터 기록
- **저장 위치**: System Tablespace 또는 별도 Undo Tablespace (`innodb_undo_tablespaces`)
- **디스크 영향**: long transaction 시 purge 불가로 증가 → undo tablespace 비대화

## 동작 방식

```
UPDATE users SET name='B' WHERE id=1;  (기존 값: name='A')

┌─────────────────────────────────────────────┐
│  Undo Log에 기록: id=1, name='A' (이전 값)   │
│  Buffer Pool에 반영: id=1, name='B' (새 값)  │
│  Redo Log에 기록: 변경 내역                   │
└─────────────────────────────────────────────┘
         │
         ├── ROLLBACK → Undo Log로 name='A' 복원
         │
         └── COMMIT → Undo Log는 즉시 삭제되지 않음
                       (다른 트랜잭션의 MVCC 읽기에 필요)
                       → Purge Thread가 불필요해지면 정리
```

## MVCC에서의 역할

```
TX1: BEGIN; UPDATE name='B';  (아직 COMMIT 안 함)
TX2: SELECT name FROM users WHERE id=1;
     → Undo Log의 이전 버전 'A'를 읽음 (Consistent Read)
```

- 각 행에는 숨겨진 컬럼 `DB_ROLL_PTR`이 있어 undo log 체인을 가리킴
- 여러 버전의 데이터가 체인으로 연결 → **버전 체인(Version Chain)**
- 이를 통해 Lock 없이 일관된 읽기(Non-Locking Consistent Read)가 가능

## 주의 사항

> [!warning]
> 장기 실행 트랜잭션이 있으면 undo log를 purge할 수 없어 **undo tablespace가 비대해진다.** `innodb_undo_tablespaces`를 설정하여 별도 undo tablespace를 사용하면 truncate로 관리할 수 있다.

---

# 5. 전체 비교

| | **Binlog** | **Relay Log** | **Redo Log** | **Undo Log** |
| --- | --- | --- | --- | --- |
| **레벨** | Server | Server (Slave) | InnoDB Engine | InnoDB Engine |
| **목적** | 복제, PITR | 복제 중간 버퍼 | Crash Recovery | Rollback, MVCC |
| **기록 내용** | 변경 SQL/Row | Binlog 복사본 | 물리적 페이지 변경 | 변경 전 데이터 |
| **로그 타입** | 논리적 | 논리적 | 물리적 | 논리적 |
| **쓰기 시점** | COMMIT 시 | IO Thread 수신 시 | 변경 발생 시 (WAL) | 변경 발생 시 |
| **삭제 시점** | expire 설정 | SQL Thread 적용 후 | Checkpoint 이후 | Purge Thread |
| **디스크 영향** | 무한 증가 (주요 원인) | lag 시 증가 | 고정 크기 (영향 없음) | long trx 시 증가 |

---

# 6. 트랜잭션 커밋 시 전체 흐름

```
1.  UPDATE 실행
2.  Undo Log에 이전 값 기록
3.  Buffer Pool의 페이지 수정 (Dirty Page)
4.  Redo Log에 변경 기록 (WAL)
5.  COMMIT 요청
6.  Redo Log flush (innodb_flush_log_at_trx_commit에 따라)
7.  Binlog에 기록 + flush (sync_binlog에 따라)
8.  커밋 완료 응답
9.  (비동기) Dirty Page → 디스크 데이터 파일 flush
10. (비동기) Undo Log purge
```

> [!important] 2-Phase Commit
> InnoDB의 redo log와 MySQL의 binlog 간 일관성을 보장하기 위해 내부적으로 **prepare → binlog write → commit** 순서로 진행한다. Crash 시 이 순서를 기반으로 커밋 여부를 판단한다.
