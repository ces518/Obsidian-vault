---
tags:
  - network
  - AWS
  - load-balancing
  - NAT
  - architecture
created: 2026-06-03
---

## 개요

[[공유기 작동 원리]] / [[Symmetric NAT]] / [[포트 포워딩과 UPnP]] 에서 본 **NAT를 거꾸로 적용**하면 **부하분산 장치(Load Balancer)** 가 된다.
- **NLB (Network LB)** = L3/L4 부하분산 — NAT 원리 그대로, 패킷 단위
- **ALB (Application LB)** = L7 부하분산 — Reverse Proxy 형태, Stream 단위 ([[Reverse Proxy 와 WAF]])
- 부하분산의 진짜 가치 = **무정지(High Availability) 시스템** + **무정지 배포**
- 실제 병목은 보통 **DB** — LB만 늘려도 한계
- 코드 레벨 해법(멀티스레딩, Non-blocking I/O)이 LB 증설보다 먼저인 경우가 많다

---

## 1. "부하"란? — 계층별로 다른 의미

> [!important] 같은 "부하"라도 보는 단위가 다르다
> L3/L4는 **패킷/세그먼트**, L7은 **요청·콘텐츠** 단위.

| 계층 | 부하의 의미 | 핵심 지표 |
|------|-------------|-----------|
| **L3 ~ L4** | 패킷/세그먼트 폭주 | PPS, 응답시간(Latency), 동시 세션 수 |
| **L7** | API 호출 빈도, 멀티미디어 전송량 | RPS, 평균 응답시간, 처리량 |

### 부하의 형태들

- **API 호출 폭주** → 연산 부하
- **멀티미디어 전송** → 송신 데이터량 부하
- **반응성 요구** → Latency 부하 (게임, 금융)
- **고가용성 요구** → 무중단 운영 부하

> [!tip] 부하분산은 가용성과 한 몸
> 부하를 분산하면 자연스럽게 **단일 장애점(SPOF)이 사라진다** → **High Availability**.

---

## 2. NLB — NAT를 거꾸로 적용

> [!important] NLB = Reverse NAT (L3/L4 LB)
> [[Symmetric NAT|NAT의 주소/포트 변환 원리]] 를 **서버 측에서 거꾸로** 적용한 것.
> 외부에서 보면 하나의 IP, 내부에서는 여러 서버.

### Forward NAT vs Reverse NAT (NLB)

```
Forward NAT (공유기)
  내부 다수 PC ──▶ 하나의 Public IP ──▶ 외부

Reverse NAT (NLB)
  외부 ──▶ 하나의 Public IP ──▶ 내부 다수 서버 (분산)
```

### 동작 흐름

```
[Client 3.3.3.3]
       │ Dst: 15.15.15.15:80 (NLB의 Public IP)
       ▼
   ┌───────────────────────────┐
   │      NLB (15.15.15.15)     │
   │   - Health Check 상시 수행 │
   │   - 가장 한가한 서버 선택   │
   └─────────┬─────────────────┘
             │ Dst 변환: 192.168.0.12:80
             ▼
       [Web Server #2 192.168.0.12]
             │
             │ 응답 (Src: 192.168.0.12)
             ▼
   ┌───────────────────────────┐
   │  NLB: Src를 15.15.15.15으로 │
   │  변환 후 클라이언트에 전송 │
   └─────────┬─────────────────┘
             ▼
       [Client 3.3.3.3]
```

### Health Check — 누구한테 보낼지의 기준

| 항목 | 내용 |
|------|------|
| **주기적 점검** | 모든 백엔드 서버의 상태/부하율 측정 |
| **선택 기준** | 가장 **널널한** 서버에 트래픽 할당 |
| **장애 감지** | 응답 없는 서버 자동 **제외** |
| **복구 감지** | 살아난 서버에 다시 트래픽 분배 |

> [!tip] 무정지의 원리
> "사용자에게는 15.15.15.15 가 매우 성능 좋아 보임" — 내부 N대 중 누가 처리했는지 모름.
> 한 대 고장 → 그 서버만 빼고 계속 운영 → **무중단**.

관련: [[Symmetric NAT]]

---

## 3. NLB 자체의 SPOF — 이중화 필수

> [!caution] LB 자체가 죽으면 끝
> 백엔드 N대가 살아 있어도 **LB 한 대가 죽으면 서비스 전체 다운**.

```
   [Client]
       │
   ┌───┴────┐
   │ LB #1   │ ──── 백엔드 1, 2, 3, 4
   │ LB #2   │ ←── Active/Standby 또는 Active/Active
   └────────┘
   둘 다 죽지 않는 한 서비스 계속
```

> [!tip] AWS의 매력
> On-premise로 이중화 LB 구성하려면 하드웨어 비용 ↑↑.
> AWS는 그냥 "서비스 선택 + 사용료 지불"로 **소프트웨어적으로 끝**.

---

## 4. AWS NLB — 언제 쓰나?

> [!important] L3/L4 부하분산이 진가를 발휘하는 곳
> 패킷 IP/Port 기반 라우팅 → 고성능, 저지연, 고가용성.

### 특징

| 항목 | 내용 |
|------|------|
| **계층** | L3 ~ L4 (Packet) |
| **분산 기준** | IP + Port |
| **성능** | 고성능, 저지연 |
| **Elastic IP** | 고정 IP 할당 지원 (외부에서 고정 주소로 접근) |

### 대표 사용처

| 사례 | 이유 |
|------|------|
| **대용량 게임 서버** | UDP/TCP 임의 포트, 저지연 필수 |
| **IoT 브로커 (MQTT)** | 수천~수만 동시 연결 |
| **금융거래 시스템** | 빠른 반응성 + 무중단 |

> [!note] 일반 웹은 보통 ALB
> "L3/L4 밸런싱"은 온프레미스에서 더 흔히 쓰임.
> AWS에서 NLB의 가장 흔한 이유는 **게임 서버**.

관련: [[TCP 세션과 상태 그리고 게임서버]] · [[TCP 와 UDP]]

---

## 5. ALB — L7 부하분산 (Reverse Proxy)

> [!important] ALB = Application Load Balancer
> [[Reverse Proxy 와 WAF|Reverse Proxy]] 형태로 동작 — Stream 단위(L7), HTTP 기반 라우팅.

### 기본 구성

```
[Client] ──HTTPS──▶ [ALB] ──▶ ┬─ [EC2 #1 API]
                              ├─ [EC2 #2 API]
                              └─ [EC2 #3 API]
                                  (동일 코드 배포)
```

### 동작 포인트

| 항목 | 내용 |
|------|------|
| **계층** | L7 (HTTP / Stream) |
| **분산 기준** | URL/Host 헤더, 경로 등 |
| **Health Check** | 백엔드 응답성 상시 점검 |
| **부하 시** | 한가한 EC2로 라우팅 |

---

## 6. ALB + NLB 혼합 구성 — 멀티미디어 분리

> [!tip] 콘텐츠 타입별 분리
> 대용량 멀티미디어는 NLB로 직접 빠지고, API 호출은 ALB로 가는 구성도 가능.

```
[Client]
   │
   ├─ 멀티미디어 다운로드 ──▶ [NLB] ──▶ 미디어 서버 N대
   │
   └─ API 호출           ──▶ [ALB] ──▶ API 서버 N대
                                          │
                                          ▼
                                       [DB Layer]
```

---

## 7. 진짜 병목은 DB — 알바생/주방장 비유

> [!danger] EC2만 늘려서 끝나지 않는다
> 알바생(API 서버) 늘려도 주방장(DB)이 하나면 결국 주방에서 막힌다.

```
[ALB] ──▶ [API #1]
      ──▶ [API #2]   ─── 모두 같은 DB로 ───▶  [DB] ⚠️ 병목!
      ──▶ [API #3]
```

### DB 부하 해소 기법

| 기법 | 설명 |
|------|------|
| **캐싱** (Redis 등) | 자주 읽는 데이터 메모리에 보관 |
| **Read Replica** | 읽기 전용 복제본 분리 |
| **Write/Read 분리** | 쓰기는 Primary, 읽기는 Replica |
| **Partitioning** | 한 테이블을 작은 단위로 분할 |
| **Sharding** | 데이터 자체를 여러 DB에 분산 |
| **Clustering** | 다중 노드 협력 운영 |

> [!important] ALB 늘리기 전에 DB부터 본다
> "응답 느림"의 진짜 원인은 대부분 DB나 동기 I/O.
> EC2 증설은 미봉책일 수 있다.

---

## 8. ALB의 진짜 가치 — 무정지 배포 (Rolling Deploy)

> [!important] EC2 1대 → ALB + 다중 EC2
> 부하분산보다 **무정지 배포(Zero-downtime Deploy)** 가 ALB의 진짜 효용일 때가 많다.

### 단일 EC2의 배포 문제

```
1대만 운영:
  배포 시 EC2 종료 → 새 코드 배포 → 재시작
  ⚠️ 약 20초 서비스 중단
```

### ALB + 다중 EC2의 Rolling Deploy

```
EC2 #1 종료 → 배포 → 재시작 (이 동안 #2, #3가 처리)
EC2 #2 종료 → 배포 → 재시작 (이 동안 #1, #3가 처리)
EC2 #3 종료 → 배포 → 재시작 (이 동안 #1, #2가 처리)
                       ↓
               ✅ 서비스 중단 0초
```

> [!tip] 회원 만 명 안 되는 서비스라면
> 20초 다운타임에 사용자가 크게 화내지 않음. ALB 도입은 트래픽이 정말 늘 때 고민하면 된다.

---

## 9. 부하 증가의 진짜 해법 — 코드 레벨 먼저

> [!important] LB 증설은 최후의 수단
> "응답이 느려서 부하분산" 이라면 **wait time이 왜 느린지** 부터 봐야 함.

### 우선 검토 순서

1. **Multi-threading** — 여러 요청 동시 처리
2. **Non-blocking I/O / 비동기 처리** — I/O 대기 시간 활용
3. **DB 최적화** — 인덱스, 쿼리, 캐싱
4. **그래도 부하라면** → LB 증설 검토

> [!caution] 함정
> "EC2를 N대로 늘려서 해결" 은 **느린 코드를 N배 비용**으로 운영하는 것일 수 있다.
> wait time이 길어지는 이유를 모르면 해결되지 않는다.

관련: [[User Kernel 모드 와 Socket 의 본질]] · [[4계층 TCP 헤더 구조와 Buffered IO]]

---

## 10. ALB vs NLB 종합

| 항목 | **ALB** | **NLB** |
|------|---------|---------|
| **계층** | L7 (Application) | L3/L4 (Network) |
| **데이터 단위** | Stream (HTTP) | Packet (IP + Port) |
| **구조 기반** | Reverse Proxy | Reverse NAT |
| **분산 기준** | URL/Host/Path | IP + Port |
| **성능** | 좋음 | **고성능, 저지연** |
| **고정 IP** | 보통 DNS 기반 | Elastic IP 지원 |
| **대표 사용처** | 웹 API, 마이크로서비스 | 게임 / MQTT / 금융 |
| **부가 기능** | WAF 연동, L7 라우팅, 쿠키 stickiness | 단순 패킷 분산 |

관련: [[Reverse Proxy 와 WAF]] · [[연결이라는 착각과 AWS ALB]] · [[AWS VPC 와 Gateway endpoint]]

---

## 핵심 요약

| 기억할 것 | 내용 |
|-----------|------|
| **부하의 의미** | L3/L4 = Packet, L7 = Request/Content |
| **부하분산의 부산물** | **무정지 시스템 (High Availability)** |
| **NLB** | **NAT 거꾸로** = L3/L4 부하분산 |
| **NLB Health Check** | 한가한 서버 선택 + 장애 서버 자동 제외 |
| **NLB 사례** | 게임 / MQTT / 금융거래 |
| **NLB Elastic IP** | 고정 IP 지원 (외부 진입점) |
| **LB 자체 SPOF** | LB도 이중화 필수 |
| **ALB** | L7 = **Reverse Proxy** 형태 |
| **ALB+NLB 혼합** | 멀티미디어는 NLB, API는 ALB |
| **진짜 병목** | 대부분 **DB** (알바 늘려도 주방장 하나) |
| **DB 해법** | 캐싱 / Replica / Partition / Sharding / Cluster |
| **ALB 진짜 가치** | **무정지 배포 (Rolling Deploy)** |
| **부하 우선 해법** | 멀티스레딩 / Non-blocking I/O → 그래도 부족하면 LB |
| **함정** | 느린 코드를 N배 비용으로 운영하지 말 것 |

---

관련: [[Symmetric NAT]] · [[공유기 작동 원리]] · [[포트 포워딩과 UPnP]] · [[Reverse Proxy 와 WAF]] · [[연결이라는 착각과 AWS ALB]] · [[AWS VPC 와 Gateway endpoint]] · [[TCP 세션과 상태 그리고 게임서버]] · [[TCP 와 UDP]] · [[User Kernel 모드 와 Socket 의 본질]] · [[모던 웹 서비스 구조]]
