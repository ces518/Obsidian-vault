---
tags:
  - DB
  - MySQL
  - PostgreSQL
  - 핀테크
  - 금융
created: 2026-03-21
---

# 핀테크/금융에서 MySQL vs PostgreSQL 비교

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
