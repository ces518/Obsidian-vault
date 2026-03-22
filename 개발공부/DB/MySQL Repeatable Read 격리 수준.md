---
tags:
  - DB
  - MySQL
  - InnoDB
  - 트랜잭션
  - 잠금
  - 격리수준
created: 2026-03-22
---

# 1. MySQL의 기본 격리 수준 = REPEATABLE READ

MySQL(InnoDB)은 **REPEATABLE READ**를 기본 격리 수준으로 사용하며, 대부분의 환경에서 이를 유지하는 것이 권장된다.

## Phantom Read가 발생하지 않는다

- SQL 표준에서는 REPEATABLE READ에서 Phantom Read(범위 쿼리 시 다른 트랜잭션이 삽입한 새 행이 보이는 현상)가 허용된다.
- 그러나 **InnoDB는 MVCC(consistent snapshot)와 gap lock을 통해 Phantom Read를 방지**한다.
  - 일반 `SELECT`: MVCC 스냅샷으로 트랜잭션 시작 시점의 데이터만 읽음
  - `SELECT ... FOR UPDATE` / `LOCK IN SHARE MODE`: next-key lock(record + gap)으로 범위를 잠금

> [!tip]
> 이것은 InnoDB 고유의 구현이다. SQL 표준에서 Phantom Read를 완전히 방지하는 수준은 SERIALIZABLE이지만, InnoDB는 REPEATABLE READ에서도 이를 달성한다.

---

# 2. REPEATABLE READ를 사용해야 하는 이유

## Replication과의 관계

- MySQL 5.7 이하에서는 `binlog_format` 기본값이 **STATEMENT**(SQL 문장 복제)
- **READ COMMITTED는 SBR(Statement-Based Replication)과 호환되지 않는다**
  - READ COMMITTED는 gap lock을 사용하지 않아 비결정적 실행이 발생
  - 마스터와 레플리카에서 같은 SQL이 다른 결과를 만들 수 있음
  - MySQL은 READ COMMITTED + SBR 조합 시 경고/에러를 발생시킨다
- READ COMMITTED를 사용하려면 `binlog_format=ROW`가 필수
  - ROW 포맷은 변경된 행 자체를 복제하므로 **replication 트래픽이 증가**
  - 문제 발생 시 binlog 분석이 어려움 (SQL문이 아닌 행 데이터이므로)

> [!info]
> binlog_format별 동작은 [[binlog_format 값별 동작]] 참고

## Undo 오버헤드

- REPEATABLE READ는 트랜잭션 시작 시점의 스냅샷을 유지하기 위해 undo log를 참조
- 이로 인한 **I/O 오버헤드는 미미한 수준**
- 오히려 **lock time**(gap lock으로 인한 대기)이 실질적인 성능 고려 대상

---

# 3. READ COMMITTED의 잠금 특성

## Record Lock만 사용한다

- `SELECT`: consistent read (잠금 없음, 단 각 문장 실행 시점의 최신 커밋 데이터를 읽음)
- `UPDATE` / `DELETE`: **record lock만 사용** (gap lock 없음)
  - 조건에 맞지 않는 행의 lock은 WHERE 평가 후 즉시 해제

## 결과

- gap lock이 없으므로 **Phantom Read가 발생**할 수 있다
- 동시성은 높지만, replication 안전성이 떨어진다

---

# 4. MySQL의 잠금은 인덱스 기반으로 동작한다

- InnoDB의 row lock은 **테이블 행이 아니라 인덱스 레코드에 대한 잠금**이다
- 인덱스가 없는 테이블에서 UPDATE/DELETE를 하면 **전체 클러스터드 인덱스를 스캔하며 모든 행에 lock**이 걸린다

```sql
-- 인덱스가 없는 컬럼으로 UPDATE하면
UPDATE t SET col = 1 WHERE non_indexed_col = 'A';
-- → 테이블 전체 행에 lock이 걸릴 수 있다
```

> [!warning]
> `UPDATE` / `DELETE` 대상 컬럼에 적절한 인덱스가 없으면 의도치 않은 광범위한 잠금이 발생한다.

---

# 5. REPEATABLE READ에서의 잠금 상세

## PK/Unique Index 동등 조건 → Record Lock

```sql
-- PK 동등 조건: record lock만 사용 (gap lock 없음)
UPDATE t SET col = 1 WHERE id = 10;
```

- WHERE에 **PK 또는 Unique Index의 동등 조건(=)**이 있으면 **record lock**으로 동작
- next-key lock(record + gap)을 사용하지 않음 → 불필요한 gap 잠금 회피

## 범위 조건 → Next-Key Lock

```sql
-- 범위 조건: next-key lock (record + gap) 사용
UPDATE t SET col = 1 WHERE id > 10 AND id < 20;
```

- 범위 스캔 시 next-key lock이 걸려 해당 범위에 INSERT가 차단됨
- 이것이 **Phantom Read를 방지**하는 메커니즘

---

# 6. REPEATABLE READ의 트레이드오프

## Undo 보유로 인한 영향

- REPEATABLE READ는 트랜잭션이 활성 상태인 동안 **시작 시점의 스냅샷을 유지**
- 해당 트랜잭션이 끝나기 전까지 undo log를 purge할 수 없음
- **장기 실행 트랜잭션**이 있으면 undo tablespace가 비대해질 수 있다

> [!caution]
> "커밋 이후에도 undo를 보유"하는 것이 아니라, **커밋하지 않고 오래 열어둔 트랜잭션**이 문제다. 커밋하면 해당 스냅샷은 해제되고 undo purge가 가능해진다.

## Deadlock 빈도

- REPEATABLE READ는 **gap lock**을 사용하므로, READ COMMITTED보다 **deadlock이 더 빈번**하게 발생할 수 있다
- gap lock 간 충돌이 원인:

```
TX1: DELETE WHERE id BETWEEN 10 AND 20  → gap lock (10, 20)
TX2: INSERT id = 15                      → gap lock 대기
TX1: INSERT id = 12                      → TX2의 gap lock과 충돌 → Deadlock
```

---

# 7. UPDATE/DELETE 서브쿼리의 S-Lock

## 현상

```sql
UPDATE t1 SET col = 1
WHERE id IN (SELECT id FROM t2 WHERE status = 'A');
```

- 이 경우 **t2의 조회 대상 행에 S-Lock(공유 잠금)**이 걸린다

## 이유: Recovery & Replication 정합성

- **Crash Recovery**: redo log 재생 시 서브쿼리의 결과가 동일해야 한다. 서브쿼리 대상이 잠금 없이 변경되면, 복구 시 다른 결과가 나올 수 있다.
- **SBR Replication**: binlog에 SQL 문장이 기록되므로, 레플리카에서 재실행할 때 서브쿼리 결과가 달라지면 마스터/레플리카 간 데이터 불일치가 발생한다.
- InnoDB는 이를 방지하기 위해 **서브쿼리가 읽는 행에 S-Lock을 설정**하여 다른 트랜잭션의 변경을 차단한다.

> [!important]
> 이 동작은 **맞다.** `UPDATE`/`DELETE` 내부의 서브쿼리 `SELECT`는 일반 `SELECT`와 달리 **locking read**로 동작하며, recovery와 replication 정합성을 위해 S-Lock을 건다. RBR 환경에서도 InnoDB 내부적으로 이 동작이 유지된다.

---

# 8. 격리 수준별 비교 요약

| 항목 | REPEATABLE READ | READ COMMITTED |
| --- | --- | --- |
| **MVCC 스냅샷** | 트랜잭션 시작 시점 고정 | 각 문장 실행 시점마다 갱신 |
| **Phantom Read** | 방지 (gap lock + MVCC) | 발생 가능 |
| **잠금 범위** | next-key lock (record + gap) | record lock만 |
| **Deadlock 빈도** | 상대적으로 높음 (gap lock 충돌) | 낮음 |
| **SBR 호환** | 안전 | 비호환 (RBR 필수) |
| **Undo 보유** | 트랜잭션 종료까지 | 문장 단위로 해제 가능 |

---

# 9. 참고 자료

- [InnoDB Lock - intomysql](http://intomysql.blogspot.com/2010/12/innodb-lock_2880.html)
- [MySQL Read Committed and Repeatable Read - minsql](http://minsql.com/mysql/MySQL-Read-committed-and-Repeatable-Read/)
- [InnoDB Transaction Isolation Modes Performance - dimitrik](http://dimitrik.free.fr/blog/archives/2015/02/mysql-performance-impact-of-innodb-transaction-isolation-modes-in-mysql-57.html)
