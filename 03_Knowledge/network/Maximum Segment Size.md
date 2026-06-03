---
type: concept
area: network
status: draft
updated: 2026-06-03
tags:
  - network
  - TCP
created: 2026-04-01
---

# 개요

MSS(Maximum Segment Size)는 **TCP 연결에서 한 번에 전송할 수 있는 데이터의 최대 크기**다. TCP 헤더와 IP 헤더를 제외한 **순수 데이터(payload)** 크기를 의미한다.

---

# 핵심 개념

| 항목 | 설명 |
| --- | --- |
| 기본값 | 536 bytes (IPv4 기준) |
| 일반적인 값 | **1460 bytes** (이더넷 환경) |
| 협상 시점 | TCP 3-way handshake (SYN / SYN-ACK) |
| 위치 | TCP 헤더의 Options 필드 |

---

# MSS 계산 방식

```
MSS = MTU - IP 헤더 - TCP 헤더
    = 1500 - 20 - 20
    = 1460 bytes  (일반 이더넷 기준)
```

| 구성 요소 | 크기 | 설명 |
| --- | --- | --- |
| MTU | 1500 bytes | 이더넷 기본값 (Maximum Transmission Unit) |
| IP 헤더 | 20 bytes | 기본 크기 |
| TCP 헤더 | 20 bytes | 기본 크기 |
| **MSS** | **1460 bytes** | 순수 데이터 영역 |

---

# 동작 방식

TCP 3-way handshake 과정에서 양쪽이 각자의 MSS를 제안하고, **더 작은 값**을 사용한다.

```
Client                          Server
  |                               |
  |--- SYN (MSS=1460) ----------->|   ← 클라이언트가 자신의 MSS 제안
  |<-- SYN-ACK (MSS=1460) --------|   ← 서버도 자신의 MSS 제안
  |--- ACK ----------------------->|
  |                               |
  |  실제 사용 MSS = min(1460, 1460) = 1460
```

---

# MSS vs MTU vs MRU

| 용어 | 계층 | 포함 항목 |
| --- | --- | --- |
| **MTU** | 네트워크 레이어 | IP 헤더 + TCP 헤더 + 데이터 |
| **MSS** | 전송 레이어 | 데이터만 (헤더 제외) |
| **MRU** | 수신 측 | 수신 가능한 최대 프레임 크기 |

```
|<-------------- MTU (1500) ------------->|
| IP 헤더(20) | TCP 헤더(20) | MSS (1460) |
```

---

# MSS가 중요한 이유

| 이유 | 설명 |
| --- | --- |
| **단편화 방지** | MSS를 MTU에 맞게 설정하면 IP 단편화를 피할 수 있다 |
| **성능 최적화** | 너무 작으면 오버헤드 증가, 너무 크면 단편화 발생 |
| **Path MTU Discovery** | 경로상 가장 작은 MTU에 맞게 MSS를 동적으로 조정 |

---

# 특수 환경에서의 MSS

| 환경 | MSS 값 | 이유 |
| --- | --- | --- |
| 이더넷 (일반) | **1460 bytes** | MTU 1500 기준 |
| VPN (IPSec 터널) | ~1380 bytes | 터널 헤더 오버헤드 |
| PPPoE (xDSL) | **1452 bytes** | PPPoE 헤더 8 bytes 추가 |
| IPv6 | 1440 bytes | IPv6 헤더 40 bytes |

---

# 정리

> MSS = **TCP가 한 번에 보낼 수 있는 순수 데이터의 최대 크기**. MTU에서 IP/TCP 헤더를 뺀 값이며, 3-way handshake 시 양쪽이 협상하여 결정한다.
