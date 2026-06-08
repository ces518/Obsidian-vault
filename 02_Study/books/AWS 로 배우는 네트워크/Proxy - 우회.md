---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-05-17
updated: 2026-06-03
tags:
  - network
  - architecture
  - security
  - proxy
  - AWS
---
# Proxy - 우회

## 개요

네트워크 장치 세 가지 구조([[네트워크 장치 구조 Inline Out of Path Proxy]]) 중 마지막인 **Proxy 구조**의 첫 번째 사용 패턴 — **우회용(Forward) Proxy**를 다룬다.
- Proxy = **대리자(Application Proxy)** — User mode에서 동작, **Stream** 데이터를 다룬다.
- 구조적으로는 **Inline과 유사** (트래픽이 통과) → **필터링 가능**.
- 가장 흔한 용도 = **지역 차단/접속 제한 우회**.
- 강력한 만큼 **MITM(중간자 공격)** 위험이 항상 따라온다 — 신뢰할 수 없는 프록시는 절대 금지.

---

## 1. Proxy의 본질 — User Mode + Stream

> [!important] Inline/Out-of-Path와의 결정적 차이
> - Inline / Out-of-Path → **Kernel/Hardware 수준의 Packet** 처리
> - **Proxy → User mode(Application) 수준의 Stream** 처리

### 위치와 데이터 단위

```
┌─────────────────────────────┐
│  User Mode                   │
│  ┌──────────────┐            │
│  │  Proxy 프로세스 │ ← Stream │
│  └──────┬───────┘            │
│         │ Socket I/O          │
├─────────┼────────────────────┤
│  Kernel │  TCP / IP / Driver  │ ← Packet
├─────────┼────────────────────┤
│  HW     │  NIC                │
└─────────┴────────────────────┘
```

| 항목 | Proxy |
|------|-------|
| **계층** | Application (L7), User mode |
| **데이터 단위** | **Stream** (소켓에서 조립된 흐름) |
| **I/O 기반** | Socket I/O |
| **구조 유사성** | Inline (통과해서 지나감) |
| **필터링 가능?** | ✅ Stream 수준에서 차단·검사 가능 |

### Inline vs Proxy 정리 비교

| 구분 | **Inline** | **Proxy** |
|------|-----------|-----------|
| **데이터 단위** | Packet | Stream |
| **계층** | L3 ~ L4 | L7 |
| **동작 위치** | Kernel / Hardware | **User Mode** (Process) |
| **공통점** | 통과해서 지나감, **필터링 가능** | 동일 |
| **차이점** | Packet 헤더(IP/TCP) 볼 수 있음 | Packet 헤더 못 봄, 대신 Stream 전체 가시 |

> [!tip] Proxy는 Inline의 친척
> "통과해서 지나간다"는 점은 Inline과 동일. 단지 다루는 단위가 **Packet → Stream**으로 올라간 형태.

관련: [[네트워크 장치 구조 Inline Out of Path Proxy]] · [[Inline 구조와 라우터]] · [[User Kernel 모드 와 Socket 의 본질]]

---

## 2. Proxy = 대리자 (Proxy의 사전적 의미)

> [!note] 비유 — 변호사
> "내가 잘 모르니까, **전문가가 나 대신** 상대방을 만나서 일을 처리하고, 그 결과만 나에게 전달한다."
> 법적으로는 **변호사**. 네트워크에서는 **Proxy 서버**.

### Forward Proxy 기본 구조

```
[Browser] ────▶ [Proxy] ────▶ [Web Server]
   │  ① 요청       ② 대신 요청
   │
   │  ④ 응답 전달   ③ 응답 받음
   ◀───────── [Proxy] ◀──────
```

- 브라우저는 서버에 **직접 접속하지 않고** Proxy에 접속
- Proxy 설정은 **브라우저(또는 OS) 수준**에서 활성/비활성 가능
- 설정을 끄면 → 일반적인 직접 접속

---

## 3. 우회 시나리오 — 지역 차단 회피

> [!important] 가장 흔한 Forward Proxy 용도
> 서버가 **접속자 IP**로 지역(국가)을 판단해 차단할 때, **다른 나라에 있는 Proxy**를 거쳐서 우회.

### 동작 흐름

```
            ❌ 한국 IP 차단된 상태
[한국 PC] ─────────X────────▶ [독일 Web Server]

            ✅ 독일 Proxy 경유
[한국 PC] ──▶ [독일 Proxy] ──▶ [독일 Web Server]
                  │                  │
                  └─ 서버 입장에선  ◀┘
                     "독일에서 접속" 으로 보임
```

### 차단되면 다른 나라 Proxy로

| 시도 | 서버 입장에서 보는 접속자 IP |
|------|-----------------------------|
| 직접 접속 | 한국 → **차단** ❌ |
| 독일 Proxy | 독일 → 허용 ✅ |
| (독일 Proxy 차단당하면) 프랑스 Proxy | 프랑스 → 허용 ✅ |

### 우회 시나리오 — 단계별 흐름

| 단계 | 상황 |
|------|------|
| **1** | 독일 서버가 한국 IP 차단 |
| **2** | 브라우저에 **독일 소재 Proxy 설정** |
| **3** | TCP 연결을 서버가 아닌 **Proxy에게** 수행 |
| **4** | Proxy가 대신 접속 → 서버는 **독일 접속**으로 인식 |
| **5** | 독일 Proxy까지 차단되면 → **프랑스 Proxy**로 변경 (반복 가능) |

> [!tip] 서버는 IP만 본다
> 서버 입장에서 보이는 정보는 대부분 **IP + HTTP 헤더** 정도의 러프한 정보.
> Proxy의 IP가 곧 "접속자 IP"로 인식되므로 지역 차단을 우회할 수 있다.

---

## 4. Proxy 서버의 내부 소켓 구조

> [!important] Proxy = Server Socket + Client Socket 동시 보유
> 두 개의 독립된 TCP 연결을 **종단**하는 것이 Proxy의 본질.

```
[내 PC]                  [Proxy 서버]                  [Naver]
                    ┌──────────────────┐
TCP Client ─────▶  │ ① Server Socket   │
Socket             │   (Listen)         │
                    │                    │
                    │ ② Client Socket   │ ─────▶ TCP Server Socket
                    │   (Active connect) │
                    └──────────────────┘
   연결 ①                                            연결 ②
   (Client ↔ Proxy)                            (Proxy ↔ Naver)
```

### Proxy의 이중 역할

| 역할 | 소켓 | 동작 |
|------|------|------|
| **서버 역할** | Server Socket (`Listen`) | 클라이언트 접속 **대기** |
| **클라이언트 역할** | Client Socket | 실제 목적지 서버에 **대리 접속** |

> [!important] 한 줄 정의
> **Proxy = 서버 소켓으로 요청을 받고 + 클라이언트 소켓으로 대신 접속**
> 즉, 하나의 프로세스가 **서버 + 클라이언트 두 역할을 동시에** 수행한다.

> [!note] Proxy의 본질적 동작
> 1. 클라이언트에서 Stream 수신 → 2. 분석/판단 → 3. 새 연결로 서버에 Stream 전송
> 응답도 동일하게 두 단계로 흘러간다.

---

## 5. Proxy가 못 보는 것 — Packet 헤더

> [!caution] Stream 레벨의 한계
> Proxy는 User mode에서 **조립된 Stream**만 본다. 그 아래 **Packet/Frame 단위**의 헤더는 볼 수 없다.

| 보이는 것 | 안 보이는 것 |
|-----------|--------------|
| HTTP 헤더 / Body (L7 Stream) | TCP 헤더 |
| 쿠키, JWT, URL | IP 헤더 |
| 폼 데이터, JSON | L2 Frame 헤더 |

> [!tip] Packet 정보까지 보려면 조합
> - 패킷 검사 + **차단** 필요 → **Inline** 장치 추가 ([[Inline 구조와 라우터]])
> - 패킷 검사 + **읽기만** 필요 → **Out-of-Path** 센서 ([[Out of Path 구조와 DPI]])

---

## 6. 위험성 — MITM (중간자 공격)

> [!danger] 신뢰할 수 없는 Proxy = 정보 유출 + 조작
> Proxy를 통과하는 모든 Stream은 **Proxy 운영자에게 100% 노출**된다.
> 더 나아가 **응답을 임의로 조작**할 수도 있다 → **Man-in-the-Middle (MITM)** 상황.

### Proxy 운영자가 알 수 있는 것

```
[내 PC] ──▶ [악성 Proxy] ──▶ [서비스]
              ▲
              │ 다음을 전부 들여다보고 조작 가능:
              │  - 어떤 사이트에 접속했는지
              │  - 무엇을 보내고 받았는지
              │  - 로그인 정보 / JWT 토큰
              │  - 쿠키 (세션)
              └─ 응답 데이터 위·변조까지 가능
```

### 위험 행위 5종 분류

| 위험 | 설명 |
|------|------|
| **감시** | 모든 트래픽 내용을 **읽을 수 있음** |
| **정보 조작** | Stream 데이터를 **변조**하여 전달 가능 |
| **토큰 탈취** | JWT 등 인증 토큰을 중간에서 **Intercept** |
| **계정 유출** | 탈취한 토큰으로 **세션 하이재킹** |
| **MITM** | Man-in-the-Middle 상황이 **자동 발생** |

### 가장 위험한 시나리오 — JWT/세션 토큰 탈취

```
[내 PC] ──── 로그인 요청 ──▶ [악성 Proxy] ──▶ [Naver]
                                   │
                                   │ ◀── JWT 토큰 발급
                                   │
                                   ├─ 토큰 인터셉트 ✓
                                   │  (Proxy 운영자가 토큰 보관)
                                   ▼
                              내 PC로 토큰 전달
```

- 인터셉트된 JWT 토큰으로 **내 계정에 직접 로그인 가능**
- HTTP(평문)뿐 아니라 **HTTPS도 SSL 종단(MITM)** 으로 뚫리는 케이스가 존재
- 관련: [[모던 웹과 JWT]]

> [!danger] 절대 원칙
> **신뢰할 수 없는 Proxy 서버는 절대 사용하지 말 것.**
> 무료 VPN / 무료 Proxy 서비스는 이 위험을 항상 안고 있다.

---

## 7. 로컬 Proxy (127.0.0.1) — 자기 자신을 경유

> [!note] 특이한 형태
> 우회용이긴 한데, **내 PC 안에서** 다른 프로세스(로컬 Proxy)를 거쳐 나간다.
> Proxy 주소 = `127.0.0.1` (Loopback)

### 구조

```
[Chrome] ──▶ [Local Proxy (127.0.0.1)] ──▶ NIC ──▶ [Web Server]
              (같은 PC의 다른 프로세스)
```

### 왜 이런 짓을 할까?

| 목적 | 설명 |
|------|------|
| **트래픽 가시화** | 크롬 개발자 모드에서 안 보여주는 트래픽도 확인 |
| **요청/응답 조작** | 쿠키, 헤더, JWT를 임의로 수정해서 전송 |
| **보안 도구** | 정당한 보안 점검·취약점 분석 도구 (예: Burp Suite, OWASP ZAP) |
| **해킹 도구** | 동일한 원리가 공격 도구로도 사용됨 |

### 대표적 조작 대상

- **Cookie 값 변조** — 세션 조작
- **HTTP Header 변경** — User-Agent, Referer, Authorization 등
- **Request / Response Body 수정** — JSON 페이로드, 폼 데이터
- **인증 토큰 조작** — JWT payload 변조, 권한 우회 시도

> [!caution] 같은 구조, 다른 용도
> **보안 도구 ↔ 해킹 도구**는 사실상 **같은 구조**(로컬 Proxy)이며, **용도만 다르다.**
> Burp Suite / OWASP ZAP 으로 자기 시스템을 점검하면 보안 도구, 타인 시스템에 사용하면 공격 도구.

> [!tip] 대표 활용 — 쿠키/토큰 조작
> 로그인 유지에 쓰이는 **쿠키 값**, **JWT** 등을 가로채 임의로 바꿔 전송하면, 서버의 응답 동작을 시험해볼 수 있다.
> 보안 테스트(자기 시스템)의 정상적 용법이지만, 같은 원리가 악용도 가능.

관련: [[Inline 구조와 라우터]] · [[Out of Path 구조와 DPI]] · [[Host 자신을 가리키는  IP 주소]]

---

## 핵심 요약

| 기억할 것 | 내용 |
|-----------|------|
| **Proxy 위치** | **User Mode** (Application Proxy) |
| **데이터 단위** | **Stream** (소켓에서 조립된 흐름) |
| **구조 유사성** | **Inline**과 닮음 (통과형, 필터링 가능) |
| **사전적 의미** | **대리자** (변호사 비유) |
| **대표 용도** | 지역/IP 기반 **차단 우회** |
| **소켓 구조** | Server Socket(Listen) + Client Socket(능동 접속) **둘 다 보유** |
| **두 개의 TCP 연결** | Client↔Proxy 연결 ① / Proxy↔Server 연결 ② — 완전히 분리 |
| **못 보는 것** | TCP/IP/L2 헤더 등 **Packet 수준 정보** |
| **위험성** | 모든 트래픽을 들여다볼 수 있고 **조작도 가능** → **MITM** |
| **최악 시나리오** | **JWT/세션 토큰 인터셉트** → 계정 탈취 |
| **절대 원칙** | **신뢰 못하는 Proxy는 절대 사용 금지** |
| **로컬 Proxy** | `127.0.0.1` 경유 — 트래픽 분석/조작 (보안 도구 ↔ 해킹 도구) |

---

관련: [[네트워크 장치 구조 Inline Out of Path Proxy]] · [[Inline 구조와 라우터]] · [[Out of Path 구조와 DPI]] · [[User Kernel 모드 와 Socket 의 본질]] · [[모던 웹과 JWT]] · [[연결이라는 착각과 AWS ALB]] · [[웹 보안과 SOP CORS]]
