---
tags:
  - network
  - HTTP
  - TCP
  - QUIC
created: 2026-05-05
---

## 개요

**HTTP/2와 HTTP/3**의 전송 기반 차이와 **QUIC** 프로토콜을 정리한다.
HOL(Head-of-Line) Blocking 문제가 어떻게 진화해 왔는지, QUIC이 OSI 계층에서 어디에 위치하는지,
HTTP/3가 도입된 이후 HTTPS의 처리 방식이 어떻게 바뀌었는지 다룬다.

---

## 1. HTTP 버전별 전송 기반

> [!important] 핵심
> **HTTP/2까지는 TCP 기반**, **HTTP/3부터는 UDP 기반 QUIC**이다.

| HTTP 버전 | 전송 계층 | 비고 |
|----------|----------|------|
| **HTTP/1.1** | TCP | Keep-Alive로 연결 재사용 |
| **HTTP/2** | TCP | Stream Multiplexing 도입 |
| **HTTP/3** | **UDP 기반 QUIC** | TCP를 우회한 새로운 전송 계층 |

```
HTTP/1.1  →  TCP
HTTP/2    →  TCP
HTTP/3    →  UDP 기반 QUIC
```

---

## 2. HOL (Head-of-Line) Blocking 문제

### HTTP/1.1의 HOL — Application 계층 차원

HTTP/1.1 keep-alive는 하나의 TCP 연결을 재사용할 수 있지만, **요청/응답 순서 제약**이 강하다.

```
TCP Connection
  Request A → Response A
  Request B → Response B
  Request C → Response C
```

> [!example] A 응답이 느리면 뒤가 다 막힘
> - Request A: 느린 DB 조회
> - Request B: 빠른 요청
> - Request C: 빠른 요청
> → A가 끝날 때까지 B/C 응답도 지연됨 = **HTTP 애플리케이션 계층의 HOL Blocking**

### HTTP/2의 개선 — Stream Multiplexing

HTTP/2는 하나의 TCP 연결 안에서 여러 stream을 **동시에 multiplexing**한다.

```
One TCP Connection
  ├─ Stream 1
  ├─ Stream 3
  └─ Stream 5
```

→ HTTP/1.1처럼 응답 순서 때문에 막히는 문제는 **상당 부분 완화**됨.

### HTTP/2에 남는 HOL — TCP 계층 차원

> [!caution] HTTP/2도 결국 TCP 위에서 동작
> TCP는 **순서 보장 기반 바이트 스트림** → segment 하나가 유실되면 그 뒤 segment가 도착해도 애플리케이션에 전달 불가.

```
HTTP/2
  ↓ TLS
  ↓ TCP
  ↓ IP
```

```
TCP segment #10 유실
TCP segment #11 도착 ─┐
TCP segment #12 도착 ─┴─→ #10 재전송될 때까지 대기 (앱에 전달 불가)
```

#11, #12 안에 **다른 HTTP/2 stream 데이터**가 들어 있어도 같이 막힘.

| 계층 | HTTP/1.1 | HTTP/2 |
|------|---------|--------|
| **HTTP 계층 HOL** | 발생 | 완화 |
| **TCP 계층 HOL** | 발생 | **여전히 발생** |

---

## 3. HTTP/3와 QUIC

### 구조

```
HTTP/3
  ↓ QUIC   ← 전송 신뢰성 + 보안 + 세션 통합
  ↓ UDP
  ↓ IP
```

HTTP/3가 해결하려는 핵심 문제 = **HTTP/2 over TCP에서 남아있던 TCP 계층 HOL Blocking**.

### QUIC의 Stream 단위 손실 격리

QUIC은 connection 안에 여러 stream을 두지만, **손실 영향을 stream 단위로 격리**한다.

```
QUIC Connection
  ├─ Stream A : 재전송 대기
  ├─ Stream B : 계속 전달 가능 ✅
  └─ Stream C : 계속 전달 가능 ✅
```

> [!tip] 핵심 차이
> TCP에서는 한 segment 유실이 **모든 stream을 멈추게** 했지만, QUIC은 **유실된 stream만 멈춘다**.

---

## 4. UDP 기반인데 신뢰성이 보장되는가?

> [!important] 핵심
> **UDP 자체가 신뢰성을 보장하는 게 아니라, UDP 위의 QUIC이 신뢰성을 직접 구현한다.**

### 순수 UDP가 보장하지 않는 것

- 패킷 도착 보장
- 순서 보장
- 중복 제거
- 재전송
- 흐름 제어
- 혼잡 제어
- 연결 상태 관리

### QUIC이 UDP 위에서 직접 제공하는 기능

| 분류 | 기능 |
|------|------|
| **연결 관리** | connection 관리, packet number, connection migration |
| **신뢰성** | ACK, loss detection, retransmission |
| **제어** | flow control, congestion control |
| **다중화** | stream multiplexing |
| **보안** | TLS 1.3 통합 |

> QUIC = UDP 위에서 **TCP에 준하는 신뢰 전송 계층을 새로 구현한 것**

---

## 5. HTTP/2 vs HTTP/3 — 전송 신뢰성 담당자 비교

### HTTP/2까지 — TCP가 신뢰성 담당

```
HTTP/2
  ↓ TLS
  ↓ TCP   ← 전송 신뢰성 담당 (연결, 재전송, 순서 보장, 흐름 제어, 혼잡 제어)
  ↓ IP
```

### HTTP/3 — QUIC이 신뢰성 담당

```
HTTP/3
  ↓ QUIC  ← 전송 신뢰성 담당
  ↓ UDP
  ↓ IP
```

> [!summary] 정리
> HTTP/2까지는 **TCP라는 L4 전송 계층**이 신뢰성을 제공했고,
> HTTP/3는 **UDP 위의 QUIC이라는 사용자 공간(user-space) 전송 프로토콜**이 신뢰성을 제공한다.

---

## 6. QUIC은 왜 UDP 위에 만들어졌나?

> [!note] UDP를 선택한 이유는 "UDP가 좋아서"가 아니다
> **새로운 전송 계층을 인터넷에 배포하기 위한 현실적인 운반 수단**으로 UDP를 선택한 것.

### IP 위에 새 프로토콜을 직접 만든다면?

```
IP 위에 완전히 새로운 전송 프로토콜
  → 방화벽/NAT/LB/OS 지원 문제
  → 배포 어려움
```

### UDP 위에 QUIC을 만든다면?

```
UDP 위에 QUIC
  → 기존 인터넷 인프라 통과 가능성 높음
  → 사용자 공간(user-space)에서 구현 가능
  → OS 커널 업데이트 없이 개선 가능
```

> QUIC = **TCP + TLS + HTTP/2 multiplexing의 한계를 재설계한 계층**
> UDP = **인터넷에서 통과하기 쉬운 운반 수단**

---

## 7. QUIC은 OSI 어디에 속하는가?

> [!important] 결론
> QUIC은 **L7이 아니다**. HTTP/3가 L7이고, QUIC은 그 아래 전송 계층 역할을 한다.
> 정확히는 **L4.5 ~ L5 성격의 user-space transport**이다.

### 계층 위치

```
HTTP/3      : L7   Application
QUIC        : L4.5 ~ L5 (user-space transport)
UDP         : L4   Transport
IP          : L3   Network
Ethernet    : L2   Data Link
```

### QUIC vs HTTP/3 — 담당 영역 분리

| 프로토콜 | 담당 기능 |
|---------|----------|
| **QUIC** | 연결 관리, ACK, 재전송, 손실 감지, 혼잡/흐름 제어, stream multiplexing, TLS 1.3 통합 |
| **HTTP/3** | request/response, method, status code, headers, body, **HTTP semantics** |

### L5 정도로 봐도 되는가?

실무 감각상 L5로 봐도 크게 틀리진 않지만, 더 정확히는:

> **OSI 계층에 딱 맞게 넣기 어렵다 — L4 전송 계층 기능 + L5 세션 계층 성격 + TLS 보안 기능을 함께 제공하는 프로토콜**

| L5처럼 보이는 이유 | L4(TCP)처럼 동작하는 이유 |
|------------------|----------------------|
| connection 관리 | ACK |
| connection ID | loss detection |
| connection migration | retransmission |
| stream multiplexing | congestion control |
| TLS 1.3 handshake 통합 | flow control |
| 세션 재개, 0-RTT | reliable delivery |

→ **순수 L5보다는 L4.5 / user-space transport**로 보는 게 가장 정확.

---

## 8. HTTPS와 QUIC의 관계

### 기존 HTTPS (HTTP/2까지)

```
HTTPS = HTTP + TLS + TCP

HTTP      : L7
TLS/SSL   : L5~L6 (보안 계층)
TCP       : L4
IP        : L3
```

### HTTP/3 기반 HTTPS

```
HTTPS over HTTP/3 = HTTP/3 + QUIC + UDP

HTTP/3    : L7
QUIC      : L4.5-ish (transport + security + session)
UDP       : L4
IP        : L3
```

### 핵심 차이

| 구분 | TLS | QUIC |
|------|-----|------|
| **역할** | 보안 계층 중심 | 전송 + 보안 + 세션/스트림 관리 |
| **계층** | L5~L6 | L4.5 ~ L5 |

> [!caution] QUIC ≠ TLS의 L5
> HTTPS에서 L5로 보는 대상은 주로 **TLS/SSL**이지만, QUIC은 TLS 기능을 포함하면서 **TCP급 신뢰 전송**까지 제공하므로 단순히 HTTPS의 L5와 동일하다고 보기 어렵다.

---

## 9. HTTP/3를 쓰면 HTTPS를 안 쓰는가?

> [!important] 결론
> **아니다. HTTP/3를 써도 HTTPS를 쓴다.**

### URL Scheme

사용자/애플리케이션 관점에서는 동일하게 `https://example.com`로 요청.
별도의 `https3://` 같은 scheme은 **존재하지 않는다**.

### 내부 전송 스택만 달라진다

| HTTP 버전 over HTTPS | 실제 스택 |
|---------------------|----------|
| HTTP/1.1 over HTTPS | HTTP/1.1 + **TLS** + TCP |
| HTTP/2 over HTTPS | HTTP/2 + **TLS** + TCP |
| HTTP/3 over HTTPS | HTTP/3 + **QUIC** + UDP |

→ 브라우저와 서버가 협상한 결과에 따라 내부적으로 HTTP/3 + QUIC + UDP 경로를 사용할 수 있음.

---

## 10. HTTP/3에서 HTTPS가 처리되는 방식

> URL scheme은 여전히 `https://`이며, 차이는 **QUIC 안에 TLS 1.3이 통합되어 있다**는 점.

```
HTTP/2 HTTPS              HTTP/3 HTTPS
  HTTP/2                    HTTP/3
  ↓                         ↓
  TLS                       QUIC ← TLS 1.3 기능 포함
  ↓                         ↓
  TCP                       UDP
  ↓                         ↓
  IP                        IP
```

> [!summary] 핵심 정리
> **HTTP/3를 쓴다고 HTTPS를 안 쓰는 게 아니라, HTTPS의 전송 기반이 TCP+TLS에서 QUIC+UDP로 바뀐 것이다.**

---

## 핵심 요약

> [!summary]
> 1. **HTTP/1.1, HTTP/2 → TCP** / **HTTP/3 → UDP 기반 QUIC**
> 2. HTTP/1.1의 HOL: 응답 순서 제약으로 뒷 요청이 막힘 (Application 계층 HOL)
> 3. HTTP/2: **Stream Multiplexing**으로 HTTP 계층 HOL은 완화 — 하지만 **TCP 계층 HOL은 여전**
> 4. HTTP/3: QUIC이 **stream 단위로 손실을 격리** → TCP 계층 HOL 해결
> 5. **UDP 자체는 신뢰성 없음** — UDP 위의 **QUIC**이 ACK, 재전송, 흐름/혼잡 제어를 직접 구현
> 6. QUIC이 UDP 위에 만들어진 이유 = 방화벽/NAT/OS 호환성 + user-space 배포 용이성
> 7. QUIC ≈ **L4.5 (user-space transport)** — L4 전송 + L5 세션 + TLS 보안 통합
> 8. **HTTP/3 ≠ HTTPS 안 씀** — URL은 여전히 `https://`, 내부 스택만 QUIC + UDP로 변경
> 9. QUIC에는 **TLS 1.3이 내장**되어 있어 별도의 TLS 핸드셰이크 불필요
