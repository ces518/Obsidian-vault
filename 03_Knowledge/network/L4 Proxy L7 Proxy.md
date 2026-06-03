---
type: concept
area: network
status: draft
updated: 2026-06-03
tags:
  - network
  - TCP
  - HTTP
  - AWS
created: 2026-05-05
---

# L4 Proxy vs L7 Proxy

## 개요

L4 프록시와 L7 프록시는 모두 **User Mode 애플리케이션**이며, 클라이언트 연결을 직접 종료(terminate)하는 **Endpoint**다.
기반 구조(plumbing)는 동일하고, **"받은 스트림으로 무엇을 하느냐"** 에서 차이가 갈린다.
"L4 LB"라 불리는 IPVS·하드웨어 장비는 프록시와는 다른 패킷 포워딩 방식이라는 점도 함께 정리한다.

---

## 1. 공통점 — 의외로 중요한 출발점

> [!important] 핵심 전제
> **"프록시"인 이상 다음은 동일**하다. 차이는 그 위에서만 발생한다.

| 공통 항목 | 설명 |
| --- | --- |
| **User Mode 애플리케이션** | OS 위에서 도는 일반 프로세스 |
| **연결 종료(Terminate) Endpoint** | 클라이언트의 TCP 연결을 직접 받아 끊음 |
| **바이트 스트림 처리** | 커널이 재조립해 준 스트림을 다룸 (raw 패킷/프레임 헤더 직접 파싱 X) |
| **이중 연결 구조** | "Client ↔ Proxy", "Proxy ↔ Backend" — **두 개의 별도 TCP 연결** |

```
[Client] ──TCP 연결 A── [Proxy] ──TCP 연결 B── [Backend]
              ↑                       ↑
        둘 다 별도 연결           Proxy가 직접 양쪽 종료
```

> [!note] 기반 구조는 같다
> Socket 처리·연결 관리·OS 커널과의 상호작용 등 **plumbing 레벨은 L4·L7이 동일**.
> 그래서 같은 엔진(예: Envoy, HAProxy)이 두 모드를 모두 지원할 수 있다.

---

## 2. 비교표 — 한눈에 정리

| 항목 | **L4 Proxy** | **L7 Proxy** |
| --- | --- | --- |
| **동작 계층** | Transport (TCP/UDP) | Application (HTTP, gRPC 등) |
| **판단 기준** | IP, 포트 (소켓 메타데이터) | URL, 헤더, 쿠키, 호스트명, 메서드 |
| **스트림 파싱** | 안 함 (그대로 중계) | 함 (프로토콜 단위로 해석) |
| **결정 단위** | **연결(Connection) 단위 — 한 번** | **요청(Request) 단위 — 매번** |
| **콘텐츠 개입** | 불가 (헤더 수정·캐싱 불가) | 가능 (헤더 조작·캐싱·압축) |
| **TLS** | Passthrough (복호화 X) | Termination 가능 (복호화) |
| **프로토콜 의존성** | 무관 (어떤 TCP/UDP든 OK) | 종속적 (이해하는 프로토콜만) |
| **성능** | 빠름, 낮은 오버헤드 | 상대적으로 무거움 |
| **부하분산 정밀도** | 낮음 (연결 단위) | 높음 (요청 단위) |
| **대표 구현체** | HAProxy (TCP mode), Envoy (L4) | NGINX, AWS ALB, Envoy (L7), Traefik |

---

## 3. 핵심 차이 #1 — 결정의 단위 (Connection vs Request)

### L4 — "한 번 정하면 끝"

```
[Client]                  [L4 Proxy]                [Backends]
   │                          │                       │
   │  TCP 연결 시도           │                       │
   │ ────────────────────→   │   Backend B 선택      │
   │                          │ ────────────────→    Backend B
   │                          │                       │
   │  요청 #1 (어떤 데이터)   │ ────────────────→    Backend B
   │  요청 #2                 │ ────────────────→    Backend B
   │  요청 #3                 │ ────────────────→    Backend B
   │                          │
   │  ※ 같은 연결 동안 무조건 같은 Backend
```

- 연결 수립 시점에 백엔드를 **한 번 선택**
- 이후엔 단순한 **바이트 파이프**
- 연결 종료까지 같은 백엔드 고정

### L7 — "매 요청마다 결정"

```
[Client]                  [L7 Proxy]                [Backends]
   │                          │                       
   │  keep-alive 연결         │                       
   │ ────────────────────→   │  GET /api/users → Backend A
   │  Request #1: /api/users  │                       
   │  Request #2: /api/orders │ ────────────────→  Backend B
   │  Request #3: /static/css │ ────────────────→  Backend C (캐시)
   │                          │
   │  ※ 동일 연결 위에서도 요청별로 다른 Backend
```

- 스트림을 **파싱**해 의미 단위(Request)로 분해
- 매 요청마다 **다른 백엔드** 선택 가능
- HTTP keep-alive와 결합 시 효율 극대화

---

## 4. 핵심 차이 #2 — 콘텐츠 개입 가능 여부

### L4의 한계

| 기능 | L4 | 이유 |
| --- | --- | --- |
| 헤더 수정 | ❌ | 헤더를 보지 않음 |
| Path 기반 라우팅 | ❌ | URL을 모름 |
| 캐싱 | ❌ | 콘텐츠 식별 불가 |
| 요청 단위 재시도 | ❌ | 요청 경계를 모름 |
| 압축 / WAF / A/B 테스트 | ❌ | 콘텐츠 개입 불가 |

### L7의 능력

| 기능 | L7 | 비고 |
| --- | --- | --- |
| 헤더 조작 | ✅ | `X-Forwarded-For` 등 추가/수정 |
| Path / Host 라우팅 | ✅ | `/api/*` → A, `/static/*` → B |
| 응답 캐싱 | ✅ | URL 기반 키 |
| 요청 단위 재시도 | ✅ | 5xx 시 자동 retry |
| 압축 / WAF / A/B 테스트 | ✅ | 콘텐츠 이해 기반 |
| TLS Termination | ✅ | 복호화 후 평문 검사 |

### Trade-off

| 측면 | L4 강점 | L7 강점 |
| --- | --- | --- |
| **프로토콜 범용성** | ✅ 어떤 TCP/UDP든 OK | ❌ 아는 프로토콜만 |
| **콘텐츠 지능** | ❌ | ✅ |
| **성능 오버헤드** | 낮음 | 높음 |
| **종단 암호화 보존** | ✅ Passthrough | ❌ Terminate해야 가능 |

---

## 5. 언제 무엇을 쓰나

### L4 Proxy가 적합한 경우

- **초고속·대용량 트래픽**
- **비-HTTP 프로토콜**: DB, gRPC raw, **게임 서버** ([[TCP 와 UDP]]·[[TCP 세션과 상태 그리고 게임서버]])
- **종단 간 암호화 유지** 필수 (Compliance, Zero-Trust 등)
- 단순 부하 분산만으로 충분한 백엔드

### L7 Proxy가 적합한 경우

- **마이크로서비스 라우팅** (Path/Host 기반)
- **API Gateway**
- **A/B 테스트, Canary 배포**
- **WAF (Web Application Firewall)** 연동
- **콘텐츠 기반 캐싱**
- **요청 단위 재시도·서킷 브레이커**

### 실무 패턴 — 2단 구성

```
[Internet]
    │
    ▼
[L4 Proxy / NLB]   ← 외부 트래픽 빠르게 수용, TLS Passthrough 가능
    │
    ▼
[L7 Proxy / ALB]   ← 내부 라우팅·헤더 조작·캐싱
    │
    ┌──────┼──────┐
    ▼      ▼      ▼
  [API]  [Web]  [Auth]
```

> [!tip] 외부는 L4, 내부는 L7
> 외부 진입은 **L4의 속도**로 받아내고,
> 내부에서는 **L7의 정밀도**로 세밀 라우팅 — 가장 흔한 실무 구성.

---

## 6. 주의 — "L4 Proxy" vs "패킷 포워딩 LB"

> [!warning] 자주 헷갈리는 부분
> 다음 장비들도 "L4 LB"라 불리지만 **프록시가 아니다.**

### 패킷 포워딩 방식 LB (Non-Proxy)

| 종류 | 특징 |
| --- | --- |
| **IPVS / LVS** | Linux 커널의 IP Virtual Server |
| **하드웨어 L4 LB** | F5 BIG-IP의 일부 모드 등 |
| **eBPF / XDP 기반 LB** | Cilium, Katran 등 |

### 차이점

```
[L4 Proxy (HAProxy TCP mode)]
   Client ──TCP A── [Proxy] ──TCP B── Backend
                        ↑
              연결을 종료하고 새로 맺음 (Userspace)

[패킷 포워딩 LB (IPVS, LVS)]
   Client ──TCP────────────────────── Backend
                  ↑
        패킷 헤더만 조작해 그대로 포워딩
        (NAT / DSR / Tunneling)
        연결 종료 없음, 라우터에 가까움
```

| 구분 | L4 Proxy | 패킷 포워딩 LB |
| --- | --- | --- |
| **연결 종료** | ✅ Userspace에서 종료 | ❌ 종료 안 함, 헤더만 조작 |
| **계층** | Userspace | Kernel / 하드웨어 |
| **분류** | 프록시 | 라우터에 가까움 |
| **성능** | 빠름 | **더 빠름** (커널/하드웨어) |
| **유연성** | 더 유연 | 제한적 |

> [!important] 용어 구분의 핵심
> - **"프록시"** 라는 단어가 붙으면 → **연결 종료형**
> - 패킷 헤더를 직접 다루는 LB → **프록시 범주 밖**
> - 본 문서의 "L4 Proxy"는 **HAProxy(TCP mode)** 같은 **연결 종료 userspace 프록시**

---

## 7. AWS 환경에서의 응용

| AWS 서비스 | 계층 | 분류 |
| --- | --- | --- |
| **NLB** (Network Load Balancer) | L4 | 연결 단위, TCP/UDP/TLS Passthrough |
| **ALB** (Application Load Balancer) | L7 | 요청 단위, HTTP/HTTPS/gRPC, Path/Host 라우팅 |
| **CLB** (Classic, 레거시) | L4+L7 | 사용 지양 |
| **API Gateway** | L7 | API 관리, 인증, Throttling |
| **CloudFront** | L7 | CDN + L7 라우팅 |
| **Global Accelerator** | L4 | 글로벌 Anycast |

### 선택 가이드

```
TCP/UDP 게임 서버, 정적 IP 필요   → NLB
HTTP API + Path 라우팅 + WAF      → ALB
글로벌 가속 + Anycast             → Global Accelerator
완전 관리형 API + 인증/스로틀링   → API Gateway
글로벌 콘텐츠 캐싱                → CloudFront
```

> [!tip] [[연결이라는 착각과 AWS ALB]]와의 연결
> ALB의 Health Check / Listener Rule이 모두 **L7 콘텐츠 개입** 능력의 결과.
> NLB는 L4라 콘텐츠 무관, TCP 핸드셰이크 기반 Health Check만 가능.

---

## 8. 핵심 정리표

| 항목 | 내용 |
| --- | --- |
| **공통점** | User Mode, 연결 종료 Endpoint, 이중 TCP 연결, 바이트 스트림 |
| **L4 결정 단위** | 연결당 1회 |
| **L7 결정 단위** | 요청마다 |
| **L4 정보** | IP, Port |
| **L7 정보** | URL, Header, Cookie, Method, Host |
| **L4 TLS** | Passthrough |
| **L7 TLS** | Termination 가능 |
| **L4 프로토콜** | 무관 |
| **L7 프로토콜** | 종속적 |
| **L4 예시** | HAProxy(TCP), NLB |
| **L7 예시** | NGINX, ALB, Envoy(L7), Traefik |
| **패킷 포워딩 LB** | IPVS/LVS/하드웨어 — **프록시 아님** |
| **실무 구성** | 외부 L4 → 내부 L7 (2단) |

---

## 한 줄 요약

> L4·L7 프록시는 모두 User Mode의 **연결 종료형 Endpoint**로 기반 구조가 같다. 차이는 **L4는 연결 단위로 한 번 결정하고 단순 중계**(빠르고 범용적), **L7은 스트림을 파싱해 요청 단위로 결정하고 콘텐츠에 개입**(똑똑하지만 무겁고 프로토콜 종속적)한다는 것. 한편 **IPVS·하드웨어 LB처럼 패킷을 직접 포워딩하는 장비는 "L4 LB"라 불려도 프록시가 아니다**. AWS에서는 **NLB(L4) ↔ ALB(L7)** 가 이 구분에 그대로 대응된다.
