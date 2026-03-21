---
tags:
  - DB
  - MySQL
  - 운영
  - 장애대응
created: 2026-03-21
---

# 🚨 1. 장애 원인

## 장애 배경

- **상황**: QueryPie를 사용 중, MySQL dump 데이터를 밀어 넣는 과정에서 **QueryPie 메타 DB의 디스크 FULL** 장애 발생
- **인프라**: 메타 DB는 이중화(DR) 구성이 되어 있었지만, **자동 승격(failover)이 되지 않았음**

## 거의 확정적인 장애 패턴

```
write 증가 → binlog 폭증 → purge 안됨 (replica lag / expire 설정 과다) → 디스크 FULL → DB write 실패 (부분 장애)
```

> [!tip]
> `binlog_expire_logs_seconds` 기본값이 **2592000(30일)**이므로, 별도 설정 없이는 30일치 binlog가 그대로 쌓인다.

## 디스크를 터뜨리는 주범

| 로그 | 영향 | 비고 |
| --- | --- | --- |
| binlog | 🔥🔥🔥 주요 원인 | 무한 증가, 직접 삭제 필요 |
| relay log | ⚠️ replica에서 증가 | lag 있으면 계속 쌓임 |
| undo log | ⚠️ long trx 시 증가 | rollback + MVCC |
| redo log | ❌ 거의 영향 없음 | 고정 크기 (순환) |

> [!important]
> **binlog가 90% 원인**이다.

---

# 🧠 2. MySQL 로그 구조 정리

## binlog

- **용도**: replication / CDC
- **파일 형태**: 파일 단위 (`mysql-bin.000001`)
- **특징**: 무한 증가 → 직접 삭제 필요

## relay log

- **용도**: replica가 가져온 binlog (실행 대기 큐)
- **특징**: lag이 있으면 계속 쌓임

## redo log

- **용도**: crash recovery
- **특징**: 고정 크기 (순환) → 디스크 원인 아님

## undo log

- **용도**: rollback + MVCC
- **특징**: long transaction 시 증가

---

# 🔄 3. Replication 구조

```
Primary
  └─ binlog 생성
         ↓
Replica (IO Thread)
  └─ relay log 저장
         ↓
Replica (SQL Thread)
  └─ 재실행
```

> [!note]
> 핵심은 **"로그 복사 → 다시 실행"**이다.

---

# ⚠️ 4. 장애 시 Replication 동작

## Primary가 죽으면

| 항목 | 상태 |
| --- | --- |
| IO Thread | ❌ 멈춤 |
| SQL Thread | ⚠️ 남은 relay log 계속 실행 |

→ 완전히 멈추는 게 아니다.

## Replica 디스크도 위험한 이유

- relay log가 쌓임
- binlog (켜져 있으면) 추가로 쌓임

> [!warning]
> Primary보다 Replica가 **먼저 디스크 FULL**이 날 수도 있다.

---

# 🚨 5. 자동 Failover가 안 되는 이유

> [!danger] 핵심 착각
> ❌ "이중화 = 자동 전환" → **아니다.**

### 이유

1. MySQL replication은 **failover 기능이 없음**
2. App이 여전히 Primary를 바라봄
3. Replica는 기본적으로 read-only
4. DR 서버도 그냥 복제본일 뿐

### Proxy가 있어도 안 되는 이유

디스크 FULL은 **부분 장애**이기 때문이다:

| 항목 | 상태 |
| --- | --- |
| TCP 연결 | ✅ 정상 |
| SELECT | ✅ 정상 |
| WRITE | ❌ 실패 |

→ 프록시가 **"정상"으로 착각**해서 자동 failover가 일어나지 않는다.

---

# 🔥 6. 디스크 FULL이 위험한 이유

> [!danger]
> 완전 장애가 아니라 **"살아있지만 쓸 수 없는 상태"**가 제일 위험하다.

```
connect → OK
select  → OK
insert  → FAIL
update  → FAIL
```

---

# 🛠️ 7. 장애 대응 전략

## 1순위: 즉시 복구 (binlog purge)

```sql
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 1 DAY);
```

## 2순위: OS 레벨 로그 정리

> [!warning]
> `*.log` 와일드카드는 MySQL이 사용 중인 redo log, error log까지 날릴 수 있으므로 **대상 파일을 반드시 특정**할 것

```bash
# 일반 시스템 로그
truncate -s 0 /var/log/syslog
truncate -s 0 /var/log/messages

# MySQL slow/general log (확인 후 정리)
truncate -s 0 /var/lib/mysql/slow.log
truncate -s 0 /var/lib/mysql/general.log
```

## 3순위: 물리적 대응

- tmp / log 디렉토리 이동
- 디스크 추가 mount

---

# 🔐 8. 권한 관련

| 작업 | 필요 권한 |
| --- | --- |
| `PURGE BINARY LOGS` | `BINLOG_ADMIN` |
| `expire` 설정 변경 | `SYSTEM_VARIABLES_ADMIN` |

---

# ⚠️ 9. 자동 삭제의 한계

```sql
SET PERSIST binlog_expire_logs_seconds = 86400;
```

> [!caution]
> 이 설정만으로는 장애 해결이 안 된다.
> - 즉시 삭제가 아님 (binlog rotation 시점에 삭제)
> - 디스크 FULL 상태에서는 새 binlog 파일 생성 자체가 실패하므로 purge도 트리거되지 않음

---

# 🧠 10. 핵심 개념 요약

| 개념 | 한 줄 정리 |
| --- | --- |
| Replication | 로그 복사해서 재실행 |
| relay log | 레플리카의 작업 대기열 |
| binlog | 디스크를 터뜨리는 로그 |
| Failover | replication이 아니라 별도 시스템 (Proxy / Orchestrator 등) |

---

# 🚀 11. 전체 흐름 한 장 요약

```
[MySQL dump 대량 write]
    ↓
[binlog 폭증]
    ↓
[replica lag → purge 안됨]
    ↓
[메타 DB 디스크 FULL]
    ↓
[write 실패 (부분 장애: connect/select OK, insert/update FAIL)]
    ↓
[프록시 감지 실패 (TCP/SELECT 정상으로 착각)]
    ↓
[DR 자동 failover 실패]
    ↓
[QueryPie 서비스 장애]
```

---

# 🛡️ 12. 재발 방지 대책

## 모니터링

- 디스크 사용률 **80% 알람** 설정 (CloudWatch, Prometheus 등)
- `Seconds_Behind_Master` 모니터링 → replica lag 조기 감지

## binlog 관리

```sql
-- expire를 1~3일로 설정
SET PERSIST binlog_expire_logs_seconds = 86400;  -- 1일

-- 파일 크기 제한 (기본 1GB → 줄이면 purge 단위가 작아짐)
SET PERSIST max_binlog_size = 536870912;  -- 512MB
```

## Failover 솔루션 도입

| 솔루션 | 특징 |
| --- | --- |
| MHA | 전통적, 검증됨 |
| Orchestrator | GitHub에서 사용, 웹 UI |
| ProxySQL | health check + 자동 라우팅 |
| InnoDB Cluster (Group Replication) | MySQL 공식, 자동 failover |

> [!important]
> replication만으로는 자동 failover가 **절대 안 된다**. 반드시 별도 솔루션 필요.

---

# ☁️ 13. AWS 환경이었다면?

## RDS Multi-AZ

- Primary/Standby가 **동일한 스토리지 설정**을 공유 → Primary가 가득 차면 Standby도 비슷한 상태
- **Storage Auto Scaling** 옵션으로 디스크 FULL 자체를 예방 가능
- 단, 디스크 FULL은 **failover 트리거 조건이 아님** (인스턴스 장애, AZ 장애, OS 패치 등이 트리거)

## Aurora

- 스토리지가 **공유 분산 스토리지 (최대 128TB 자동 확장)** → 디스크 FULL 시나리오 자체가 사실상 없음
- binlog 기본 비활성 (Aurora 자체 replication은 binlog을 사용하지 않음)
- failover는 **10~30초 내 자동 승격** (Aurora Replica가 있을 때)

## 환경별 비교

| 환경 | 디스크 FULL 발생 가능성 | 자동 failover |
| --- | --- | --- |
| IDC MySQL | 높음 | ❌ 별도 솔루션 필요 |
| RDS Multi-AZ | 낮음 (Auto Scaling) | ⚠️ 디스크 FULL로는 트리거 안됨 |
| Aurora | 거의 없음 (자동 확장) | ✅ 자동 승격 (10~30초) |

> [!tip]
> Aurora는 failover가 잘 돼서가 아니라, **스토리지 구조상 이 장애 자체가 안 생긴다**는 게 핵심이다.

---
