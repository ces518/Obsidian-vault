---
tags:
  - network
  - architecture
  - security
  - proxy
  - AWS
  - load-balancing
created: 2026-05-18
---

## 개요

네트워크 장치 구조 시리즈의 **마지막** — **Reverse Proxy**.
앞서 다룬 [[Proxy - 우회]] / [[Proxy 보호와 감시]] 가 **클라이언트 측에 붙는 Forward Proxy**였다면, **Reverse Proxy는 서버 측에 붙는 Proxy**.
- 대표 구현: **AWS ALB** (Application Load Balancer), **Nginx**, **WAF**
- 핵심 용도: **보안(WAF) / 부하분산(LB) / SSL Termination / 캐싱·압축**
- 핵심 트릭: 사용자가 접속하는 IP가 **실제 웹서버가 아니라 Reverse Proxy**
- 핵심 부작용: 백엔드는 **진짜 클라이언트 IP**를 모름 → **X-Forwarded-For (XFF)** 로 전달

---

## 1. Reverse Proxy = 서버에 붙는 Proxy

> [!important] Forward vs Reverse
> Proxy의 **구조는 동일**(Stream, L7, User Mode)이지만 **누구 옆에 서느냐**가 다르다.

| 구분 | **Forward Proxy** | **Reverse Proxy** |
|------|------------------|-------------------|
| **붙는 위치** | 클라이언트 측 | **서버 측** |
| **클라이언트 입장** | "내가 Proxy를 쓰고 있다" 인지 | **모름** — 그냥 웹서버로 착각 |
| **대표 용도** | 우회 / 사내 보호·감시 | **WAF / 로드밸런서 / SSL 종단** |
| **설정 위치** | 브라우저 / OS | **DNS** (서버 IP 자리에 RP IP 등록) |
| **대표 장치** | 사내 Proxy | **AWS ALB**, Nginx, CloudFront |

> [!tip] Inline/Proxy 둘 다 "통과해서 지나감"
> Reverse Proxy도 결국 Proxy → 데이터가 **통과**하므로 **필터링 가능**.
> 단지 단위가 Packet이 아닌 **Stream**일 뿐. ([[Inline 구조와 라우터]] 와 본질적 동일)

관련: [[Proxy - 우회]] · [[Proxy 보호와 감시]] · [[네트워크 장치 구조 Inline Out of Path Proxy]]

---

## 2. 동작 트릭 — DNS가 알려주는 IP는 RP의 IP

> [!important] 클라이언트는 진짜 서버 IP를 모른다
> DNS가 응답하는 IP는 **Reverse Proxy의 IP**.
> 클라이언트는 RP에 접속하면서 "이게 웹서버구나" 하고 착각한다.

### 접속 흐름

```
[Client]
   │ ① "www.a.com 의 IP 알려줘"
   ▼
[DNS] ────▶ ② 10.10.10.20 (← RP의 IP!)
   │
   │ ③ TCP 연결 시도
   ▼
[Reverse Proxy 10.10.10.20]   ← 클라이언트는 여기를 웹서버로 인식
   │
   │ ④ 내부 망에서 실제 서버에 대신 접속
   ▼
[실제 Web Server 10.0.0.5]
```

### 결과적으로

| 관점 | 보는 것 |
|------|---------|
| **클라이언트** | RP를 웹서버로 인식 (진짜 서버 존재 자체를 모름) |
| **실제 서버** | 모든 트래픽이 RP에서 온다고 인식 (진짜 클라이언트 IP 모름) |

관련: [[DNS 구조]] · [[DNS 캐싱과 Route53]]

---

## 3. WAF — Web Application Firewall

> [!important] WAF는 본질적으로 Reverse Proxy
> 웹 트래픽(HTTP) = L7 = Stream → **Proxy 구조 외 선택지 없음**.
> 그것도 서버 보호용 → **Reverse Proxy 형태**가 자연스러움.

### 동작 예시 — 악성 첨부파일 차단

```
[악성 사용자]
   │ 게시판에 악성코드 첨부 파일 업로드 시도
   ▼
[WAF (Reverse Proxy)] ── Stream 검사
   │   "어, 실행파일 + 악성코드 시그니처 매치"
   │
   ├─ 정상 입력 → ✅ Bypass → [실제 Web Server]
   └─ 위협 감지 → ❌ Drop  → 사용자에게 차단 응답
                            (실제 서버까지 도달조차 못함)
```

### WAF가 차단하는 대표 위협

| 공격 유형 | 검사 대상 |
|-----------|-----------|
| **SQL Injection** | URL 파라미터, 폼 데이터 |
| **XSS** | HTML/JS 페이로드 |
| **악성 파일 업로드** | 첨부 파일의 바이너리 시그니처 |
| **CSRF / Path Traversal** | HTTP 헤더, URL 패턴 |

> [!tip] Stream 단위라서 가능한 검사
> SQL Injection, XSS 같은 공격은 **HTTP 요청 본문 전체**를 봐야 판단 가능 → Stream 단위 = Proxy 영역.
> Packet 단편으로는 절대 못 잡는다.

관련: [[웹 보안과 SOP CORS]] · [[HTTP 와 REST API]]

---

## 4. 부하분산 (Load Balancing) — ALB

> [!important] AWS ALB = 전형적인 Reverse Proxy + L7 로드밸런서
> 하나의 IP 뒤에 여러 백엔드 서버를 두고, **요청을 분산**시킨다.

### 동작 구조

```
[Client] ──▶ www.a.com (10.10.10.10) ──▶ [ALB / Reverse Proxy]
                                              │
                                       ┌──────┼──────┐
                                       ▼      ▼      ▼
                                  [Server1] [Server2] [Server3]
                                   (Round-robin / Least-conn / ...)
```

### 사용자/운영자 관점

| 관점 | 인식 |
|------|------|
| **사용자** | 그냥 IP 한 개에 접속한 것 — 백엔드가 1대인지 100대인지 모름 |
| **운영자** | 백엔드 서버를 **자유롭게 추가/제거 가능** (Auto Scaling) |

> [!note] ALB 자체의 부하
> ALB는 **트래픽 토스**만 하지 직접 비즈니스 로직을 처리하지 않음 → 실제 웹서비스 부하는 백엔드가 짊어짐.
> 그래도 트래픽이 한 곳에 모이므로 ALB도 적절한 스펙·이중화 필요.

관련: [[연결이라는 착각과 AWS ALB]] · [[모던 웹 서비스 구조]]

---

## 5. SSL Termination — RP에 인증서를 올리는 이유

> [!important] SSL 종단을 RP에서 처리
> SSL 인증서를 **Reverse Proxy에 설치** → RP가 복호화 담당.
> RP ↔ 백엔드 구간은 **평문 통신** → 내부 분석/캐싱/검사 모두 용이.

### 흐름

```
[Client]                [Reverse Proxy]              [Web Server]
   │                          │                          │
   ├──HTTPS (TLS)─────────────▶│                          │
   │                          │── SSL 종단 (복호화)        │
   │                          │                          │
   │                          ├──HTTP (평문)─────────────▶│
   │                          │                          │
   │                          │◀──HTTP (평문)─────────────┤
   │                          │── 재암호화 (TLS)           │
   │◀──HTTPS (TLS)────────────┤                          │
```

### 평문 구간의 활용

| 활용 | 설명 |
|------|------|
| **Port Mirroring + 분석** | RP↔서버 구간을 [[Out of Path 구조와 DPI\|Tapping]] 해서 평문 분석 가능 |
| **L7 라우팅** | URL/Host 헤더 보고 다른 백엔드로 라우팅 |
| **캐싱** | 응답 콘텐츠 캐싱 (동일 요청은 백엔드 우회) |
| **압축 / 최적화** | gzip, brotli 등 |

> [!tip] 보안 vs 가시성의 트레이드오프
> RP에서 SSL 종단 = 검사·분석 편리. 단 **암호화 평문 구간이 생기므로** 내부망 보안을 별도로 챙겨야 함.

관련: [[Out of Path 구조와 DPI]] · [[HTTP 와 REST API]]

---

## 6. 백엔드의 고민 — "진짜 클라이언트 IP가 뭐죠?"

> [!caution] RP를 두는 순간 출발지 IP는 항상 RP
> 백엔드 서버 입장에서 모든 패킷의 Src IP = RP의 IP.
> **진짜 사용자 IP를 알 수 없음** → 로깅·통계·차단 정책 무력화.

### 문제 상황

```
[Real Client 1.2.3.4] ──▶ [RP 10.10.10.20] ──▶ [Web Server]
                                                      │
                                                      ▼
                                              "방금 누가 접속함?"
                                              → 10.10.10.20 ← RP만 보임
                                              (1.2.3.4 영영 모름)
```

### 해결책 — X-Forwarded-For (XFF)

> [!important] XFF = 원본 IP를 HTTP 헤더에 담아 전달
> Proxy를 거칠 때마다 **append** 되는 표준 헤더.

```
Client(1.2.3.4) ──▶ Proxy1(5.5.5.5) ──▶ Proxy2(7.7.7.7) ──▶ Server

HTTP 헤더:
   X-Forwarded-For: 1.2.3.4, 5.5.5.5, 7.7.7.7
                    └─ 진짜 클라이언트 ─┘  └ 경유지 ─┘
```

### XFF의 한계

| 항목 | 내용 |
|------|------|
| **표준성** | 사실상 업계 표준 (RFC 7239의 `Forwarded:` 가 정식, XFF가 더 흔함) |
| **보안성** | ❌ **없음** — 클라이언트가 임의 조작 가능 |
| **신뢰 조건** | 가장 가까운(자기 RP가 추가한) IP만 신뢰 가능 |
| **활용** | 로깅, 통계 등 비결정적 용도 권장. 차단 정책엔 신중 |

> [!caution] XFF만 믿고 차단하지 마라
> 공격자가 `X-Forwarded-For: 8.8.8.8` 같은 식으로 조작하면 IP 기반 화이트리스트가 뚫린다.
> 신뢰할 수 있는 RP가 **덮어쓰는(override) 정책**으로 운용해야 한다.

---

## 7. 그 외 Reverse Proxy의 부가 기능

| 기능 | 설명 |
|------|------|
| **압축** | gzip / brotli 로 응답 크기 축소 |
| **캐싱** | 동일 요청은 백엔드 거치지 않고 RP가 응답 |
| **콘텐츠 최적화** | 이미지 변환, 헤더 정리 등 |
| **L7 라우팅** | URL/Host 기반 백엔드 선택 |
| **Rate Limiting** | 요청 빈도 제한 |
| **SSL 종단** | 인증서 통합 관리 (§5) |

### 대표 구현체

| 구현 | 특징 |
|------|------|
| **AWS ALB** | Managed L7 Load Balancer, WAF 연동 가능 |
| **Nginx** | OSS, 웹서버 겸 Reverse Proxy — **소스 공개**라 WAF 자작도 가능 |
| **HAProxy** | 고성능 LB |
| **CloudFront** | CDN + Reverse Proxy 통합 |
| **Envoy / Traefik** | 마이크로서비스 친화 |

> [!tip] Nginx 소스 뜯어보기
> Reverse Proxy의 동작 원리를 깊게 이해하고 싶다면 **Nginx 소스 코드 분석**이 정석.
> Nginx 위에 직접 WAF 모듈을 얹는 것도 충분히 가능한 학습 코스.

---

## 8. 시리즈 종합 — 4가지 네트워크 장치 구조

> [!summary] 어떤 장치를 만나든 "이건 어느 구조?" 부터 물어라

| 구조 | 데이터 단위 | 계층 | 위치 | 차단 | 대표 |
|------|-----------|------|------|------|------|
| **Inline** | Packet | L3~L4 | 경로 위 | ✅ | Router, Firewall, IPS |
| **Out-of-Path** | Packet | L3~L4 | 경로 밖 (Mirror) | ❌ | NIDS, Wireshark/Npcap |
| **Forward Proxy** | Stream | L7 | 클라이언트 측 | ✅ | 사내 Proxy, 우회용 |
| **Reverse Proxy** | Stream | L7 | 서버 측 | ✅ | **ALB, WAF, Nginx** |

### 의사결정 트리

```
어떤 데이터를 봐야 하나?
   │
   ├─ Packet (L3~L4)
   │     │
   │     ├─ 차단 필요? → Inline ([[Inline 구조와 라우터]])
   │     └─ 관찰만 OK? → Out-of-Path ([[Out of Path 구조와 DPI]])
   │
   └─ Stream (L7)
         │
         ├─ 클라이언트 측 → Forward Proxy ([[Proxy - 우회]] · [[Proxy 보호와 감시]])
         └─ 서버 측      → Reverse Proxy (본 문서)
```

> [!important] 학습 팁
> 새로운 네트워크 장비/서비스를 만나면 **"이 4가지 중 무엇인가?"** 를 가장 먼저 분류해보자.
> 분류만 되면 → 다루는 데이터, 가능한 동작, 한계가 자동으로 도출된다.

---

## 핵심 요약

| 기억할 것 | 내용 |
|-----------|------|
| **Reverse Proxy** | **서버 측**에 붙는 Proxy (Forward와 정반대) |
| **클라이언트 인식** | RP를 웹서버로 **착각** (DNS가 RP IP를 응답) |
| **대표 용도** | **보안(WAF) / 부하분산(ALB) / SSL 종단 / 캐싱** |
| **WAF** | HTTP Stream 검사로 SQLi, XSS, 악성 업로드 차단 |
| **ALB** | AWS의 L7 로드밸런서 — 전형적 Reverse Proxy |
| **SSL Termination** | RP에서 복호화 → 백엔드는 평문 → 검사/분석 용이 |
| **백엔드의 한계** | 출발지 IP가 항상 RP → 진짜 클라이언트 IP 모름 |
| **해결책 XFF** | `X-Forwarded-For` 헤더로 원본 IP 전달 (단 보안성 ❌) |
| **부가 기능** | 압축, 캐싱, L7 라우팅, Rate Limiting |
| **대표 구현** | **AWS ALB / Nginx / HAProxy / CloudFront** |
| **시리즈 종합** | Inline / Out-of-Path / Forward Proxy / **Reverse Proxy** 4구조 |
| **분류 능력** | 새 장비 만나면 "어느 구조?" 부터 묻기 — 데이터 단위/계층/한계 자동 도출 |

---

관련: [[Proxy - 우회]] · [[Proxy 보호와 감시]] · [[Inline 구조와 라우터]] · [[Out of Path 구조와 DPI]] · [[네트워크 장치 구조 Inline Out of Path Proxy]] · [[연결이라는 착각과 AWS ALB]] · [[모던 웹 서비스 구조]] · [[웹 보안과 SOP CORS]] · [[DNS 구조]]
