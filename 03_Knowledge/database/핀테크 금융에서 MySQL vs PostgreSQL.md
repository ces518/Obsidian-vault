---
type: concept
area: database
status: draft
updated: 2026-06-03
tags:
  - DB
  - MySQL
  - PostgreSQL
  - 핀테크
  - 금융
created: 2026-03-21
---

# 핀테크/금융에서 MySQL vs PostgreSQL 비교

## 개요

핀테크/금융 도메인 관점에서 MySQL과 PostgreSQL을 트랜잭션 안정성, 데이터 무결성, 확장성/기능 측면으로 비교하고, 그럼에도 MySQL을 쓰는 이유와 실제 트렌드를 정리한 문서다.

---

## 1. 트랜잭션 안정성

| 항목 | MySQL (InnoDB) | PostgreSQL |
| --- | --- | --- |
| MVCC 구현 | undo log 기반 (별도 공간) | tuple 버전 직접 관리 (Heap 내) |
| Isolation 기본값 | REPEATABLE READ | READ COMMITTED |
| Serializable | gap lock 기반 (근사치) | SSI (Serializable Snapshot Isolation) - 진짜 직렬화 보장 |

> [!important]
> 금융에서는 **정확한 직렬화 보장**이 중요하다. PostgreSQL의 SSI가 이론적으로 더 엄밀하다.

---

## 2. 데이터 무결성

| 항목 | MySQL | PostgreSQL |
| --- | --- | --- |
| 잘못된 데이터 입력 | 경고 후 **암묵적 변환** (예: 문자열 → 0) | **에러로 거부** |
| CHECK 제약조건 | 8.0.16부터 지원 (늦음) | 오래전부터 완전 지원 |
| ENUM/도메인 타입 | 제한적 | 커스텀 도메인 타입 지원 |
| 외래키 + 파티셔닝 | 제약 많음 | 자연스럽게 지원 |

> [!danger]
> 금융에서 **"잘못된 데이터가 조용히 들어가는 것"**이 가장 위험하다. PostgreSQL은 이걸 기본적으로 막아준다.

---

## 3. 확장성 & 기능

| 항목 | MySQL | PostgreSQL |
| --- | --- | --- |
| JSON 처리 | 기본 지원 | JSONB (인덱싱, 부분 업데이트) |
| 윈도우 함수 | 8.0부터 | 오래전부터 풍부 |
| CTE (WITH) | 8.0부터 | 오래전부터 지원 |
| 파티셔닝 | 기본 | 선언적 파티셔닝 + 상속 |
| GIS/지리 데이터 | 제한적 | PostGIS (업계 표준) |

---

## 4. 그럼에도 MySQL을 쓰는 이유

- **운영 성숙도**: 국내 대부분 조직이 MySQL 운영 경험이 풍부
- **읽기 성능**: 단순 CRUD 워크로드에서 빠름
- **에코시스템**: AWS RDS/Aurora 최적화, 레퍼런스 다수
- **인력 풀**: MySQL DBA/개발자가 훨씬 많음

---

## 5. 실제 트렌드

- 해외 핀테크 (Stripe, Square 등) → **PostgreSQL** 선호
- 국내 금융/핀테크 → MySQL 많지만, **신규 프로젝트에서 PostgreSQL 전환** 증가 중
- AWS Aurora도 **PostgreSQL 호환 버전** 출시 → 클라우드에서도 선택지 넓어짐

---

## 💬 한 줄 정리

> **MySQL이 "못해서"가 아니라, PostgreSQL이 "잘못된 데이터를 허용하지 않는" 철학이 금융 도메인과 맞기 때문이다.**

---

## 면접식 설명

> 핀테크/금융에서 PostgreSQL이 선호되는 이유는 크게 트랜잭션 안정성과 데이터 무결성이다. 트랜잭션 측면에서 PostgreSQL은 SSI(Serializable Snapshot Isolation)로 진짜 직렬화를 보장하는 반면, MySQL의 Serializable은 gap lock 기반 근사치다. 무결성 측면에서 MySQL은 잘못된 값을 경고 후 암묵적으로 변환(문자열을 0으로)하는 경우가 있지만 PostgreSQL은 에러로 거부하고, CHECK 제약도 PostgreSQL이 오래전부터 완전 지원한 데 비해 MySQL은 8.0.16부터 지원했다. 금융에서 가장 위험한 건 잘못된 데이터가 조용히 들어가는 것이라 이 철학 차이가 도메인과 잘 맞는다. 그럼에도 MySQL을 쓰는 이유는 국내 운영 성숙도와 인력 풀, 단순 CRUD 읽기 성능, AWS RDS/Aurora 최적화 같은 에코시스템이다. 실제 트렌드는 해외 핀테크가 PostgreSQL을 선호하고 국내도 신규 프로젝트에서 전환이 늘고 있으며, Aurora의 PostgreSQL 호환 버전으로 클라우드 선택지도 넓어졌다.

---

## 관련 문서

- [[MySQL Repeatable Read 격리 수준]]
- [[MySQL 로그 시스템 (Binlog, Relay, Redo, Undo)]]
