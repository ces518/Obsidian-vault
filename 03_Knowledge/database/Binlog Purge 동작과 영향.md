---
type: concept
area: database
status: draft
updated: 2026-06-03
tags:
  - DB
  - MySQL
  - binlog
  - 운영
created: 2026-03-23
---

# 개요

`PURGE BINARY LOGS`는 **오래된 binlog 파일을 삭제하는 명령**이다.
binlog은 스냅샷이나 상태 정보 없이 **변경 이벤트만 나열한 파일**이므로, purge는 단순 파일 삭제로 동작한다.

## 한 줄 요약

> Binlog Purge는 이미 반영된 일기장을 찢어버리는 것이다. 현재 데이터는 멀쩡하지만, 과거로 되돌아갈 수 있는 PITR 범위가 줄고, 아직 안 읽은 binlog을 지우면 복제가 깨진다.

```sql
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 1 DAY);
```

```
Before:
  mysql-bin.000001 → 000002 → 000003 → 000004(현재)

After:
  000001 🗑️ OS에서 삭제
  000002 🗑️ OS에서 삭제
  000003 🗑️ OS에서 삭제
  000004(현재) → 유지 ✅
```

> [!tip]
> 압축, 병합, 스냅샷 생성 등은 일어나지 않는다. 파일이 OS에서 사라지는 것이 전부다.

---

# 영향 받는 것

## PITR 범위 축소

binlog은 **변경 이벤트의 나열**일 뿐이므로, 삭제된 구간으로는 복구할 수 없다.

```
Before purge:
  백업(월요일) + binlog(월~금) → 월~금 어디든 복구 가능

After purge (수요일 이전 삭제):
  백업(월요일) + binlog(수~금) → 시작점 연결 안됨 → 복구 불가 ❌
```

PITR은 **백업본(스냅샷) + Binlog(변경 이력)** 의 조합이다.
중간 binlog이 빠지면 이어붙일 수 없으므로, 백업 주기와 purge 주기를 맞춰야 한다.

## Replication 위험

Replica가 아직 읽지 않은 binlog을 purge하면 복제가 깨진다.

```
Replica가 mysql-bin.000002 읽는 중
        ↓
Primary에서 000002를 purge
        ↓
Replica: "파일 없음" 에러 → 복제 깨짐 💥
```

> [!warning]
> purge 전에 `SHOW SLAVE STATUS`로 Replica가 어디까지 읽었는지 반드시 확인해야 한다.

---

# 영향 없는 것

| 항목 | 이유 |
| --- | --- |
| 현재 DB 데이터 | 이미 반영된 일기장을 버리는 것 |
| 현재 운영 | 새 binlog는 계속 생성됨 |
| Redo / Undo Log | 완전히 별개의 로그 |

---

# 필요 권한

| 작업 | 필요 권한 |
| --- | --- |
| `PURGE BINARY LOGS` | `BINLOG_ADMIN` |
| `expire` 설정 변경 | `SYSTEM_VARIABLES_ADMIN` |

---

# 자동 삭제 설정의 한계

```sql
SET PERSIST binlog_expire_logs_seconds = 86400;  -- 1일
```

> [!caution]
> 이 설정만으로는 장애 상황에서 해결이 안 된다.
> - 즉시 삭제가 아님 (binlog rotation 시점에 삭제)
> - 디스크 FULL 상태에서는 새 binlog 파일 생성 자체가 실패하므로 purge도 트리거되지 않음
> - 디스크 FULL 시에는 반드시 **수동 `PURGE` 명령**이 필요하다. [[MySQL 운영 장애]] 참고

---

# 정리

| 항목 | 내용 |
| --- | --- |
| 동작 | 단순 파일 삭제 (스냅샷·압축 없음) |
| 현재 데이터 영향 | 없음 |
| PITR | 삭제 구간 복구 불가 |
| Replication | 미수신 binlog purge 시 복제 깨짐 |
| 자동 삭제 | rotation 시점에만 동작, 디스크 FULL 시 무용 |

> Binlog Purge는 **이미 반영된 일기장을 찢어버리는 것**이다. 현재 데이터는 멀쩡하지만, 과거로 되돌아갈 수 있는 범위가 줄어든다.

---

# 면접식 설명

> `PURGE BINARY LOGS`는 오래된 binlog 파일을 OS에서 그냥 삭제하는 명령이라, 압축이나 스냅샷 생성 없이 파일만 사라진다. 현재 DB 데이터나 운영, redo/undo log에는 영향이 없지만 두 가지를 조심해야 한다. 첫째, PITR은 백업본 + binlog 변경 이력의 조합이라 중간 binlog이 빠지면 이어붙일 수 없어 복구 범위가 줄어든다. 둘째, replica가 아직 안 읽은 binlog을 purge하면 "파일 없음" 에러로 복제가 깨지므로, purge 전에 `SHOW SLAVE STATUS`로 replica가 어디까지 읽었는지 확인해야 한다. `binlog_expire_logs_seconds` 같은 자동 삭제는 rotation 시점에만 동작하고 디스크 FULL 상태에서는 새 binlog 생성 자체가 실패해 purge도 트리거되지 않으므로, 이때는 수동 `PURGE`가 필요하다.

---

# 관련 문서

- [[MySQL 로그 시스템 (Binlog, Relay, Redo, Undo)]]
- [[MySQL 운영 장애]]
- [[binlog_format 값별 동작]]
