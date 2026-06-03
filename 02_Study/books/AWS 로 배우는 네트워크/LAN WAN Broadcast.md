---
type: study-note
area: network
status: raw
source: AWS 로 배우는 네트워크 (책)
created: 2026-04-12
updated: 2026-06-03
tags:
  - network
---

## 개요

**LAN과 WAN의 경계**를 나누는 핵심 기준인 **Broadcast**의 개념과 동작 원리를 다룬다.
L2(물리적 네트워크)에서 L3(논리적 네트워크)로 넘어가는 경계면을 이해하는 것이 핵심이다.

---

## 1. LAN vs WAN — 물리 vs 논리

| 구분 | LAN (Local Area Network) | WAN (Wide Area Network) |
|------|-------------------------|------------------------|
| **범위** | 작은 네트워크 (집, 사무실) | 대규모 네트워크 (인터넷) |
| **계층** | L1 ~ L2 (Physical) | L3 이상 (Logical) |
| **성격** | **물리적**으로 설명되는 네트워크 | **논리적(소프트웨어적)**으로 설명되는 네트워크 |
| **핵심 식별자** | MAC 주소 | IP 주소 |
| **프로토콜** | Ethernet | IP (Internet Protocol) |

> [!important] LAN과 WAN의 경계
> - **물리적 네트워크**(L2 이하) → LAN
> - **논리적 네트워크**(L3 이상, 소프트웨어 구현) → WAN
> - 이 경계를 나누는 핵심 기준 = **Broadcast 도달 범위**

### L2에서 L3로의 전환

```
L2 (Physical)                    L3 (Logical = Virtual)
┌──────────────────┐            ┌──────────────────────┐
│  LAN 구간         │            │  WAN / 인터넷 구간     │
│  MAC 주소 기반     │  ──경계──▶ │  IP 주소 기반          │
│  물리적 네트워크    │            │  논리적 네트워크        │
│  Frame 단위       │            │  Packet 단위          │
└──────────────────┘            └──────────────────────┘
```

- 작은 LAN들을 묶고 묶고 모아서 → 거대한 **인터넷(WAN)**이 탄생
- IP는 소프트웨어로 구현 → Logical = Virtual → 논리 네트워크

---

## 2. Broadcast (브로드캐스트)

### 캐스팅(Casting)의 종류

| 종류 | 설명 | 대상 |
|------|------|------|
| **Unicast** | 특정 1대에게만 전송 | 1:1 통신 |
| **Broadcast** | 같은 네트워크 내 **불특정 다수 전체**에게 전송 | 1:All |

### Broadcast의 특징

- 관리적 이유로 **필요해서 존재**하지만, 본질적으로 **비효율적**
- 보낸 쪽은 1회만 전송하면 되지만, 스위치는 연결된 모든 장치에 **복사하여 전달**해야 함
  - 예: 24포트 허브 → 발신자 제외 **23대에게 동일 데이터 복사·전달**
- Broadcast가 많아질수록 **네트워크 효율 저하**

> [!caution] Broadcast는 비효율의 원인
> 필요해서 쓰기는 하지만, 효율적인 이유가 아니라 **어쩔 수 없어서** 쓰는 것이다.

---

## 3. Broadcast 주소

Broadcast 주소는 **L2(MAC)** 와 **L3(IP)** 모두에 존재한다.

### 공통 원리

> 특정 네트워크 단위에서 목적지 주소의 **모든 비트가 1**인 경우 = Broadcast 주소

| 계층 | Broadcast 주소 | 설명 |
|------|---------------|------|
| **L2 (MAC)** | `FF:FF:FF:FF:FF:FF` | 48비트 모두 1 |
| **L3 (IP)** | 네트워크별 상이 (호스트 부분 모두 1) | 해당 서브넷 전체에 전달 |

- `FF` = 16진수, 10진수로 15, 2진수로 `1111`
- 목적지가 Broadcast 주소 → 해당 세그먼트 내 **모든 호스트에게 전달**

### Broadcast 데이터 단위

| 계층 | 단위 데이터 | Broadcast 시 |
|------|-----------|-------------|
| **L2** | Frame | Broadcast Frame |
| **L3** | Packet | Broadcast Packet |

---

## 4. Broadcast 도달 범위와 LAN/WAN 경계

### Broadcast 도달 흐름

```
                    ┌──────────┐
                    │  Router  │ ← Broadcast 차단! (바깥으로 전달 안 함)
                    │(Gateway) │
                    └────┬─────┘
                         │
            ┌────────────▼────────────┐
            │  L2 Distribution 스위치   │ ← 설정에 따라 Broadcast 범위 제어
            └──┬──────────────────┬───┘
               │                  │
     ┌─────────▼────┐     ┌──────▼────────┐
     │ L2 Access    │     │ L2 Access     │
     │ 스위치 (1번방) │     │ 스위치 (2번방)  │
     └─┬───┬───┬──┘     └─┬───┬───┬───┘
       A   B   C          H   I   J
```

#### A가 Broadcast를 전송하면?

1. **같은 Access 스위치** 내 B, C에게 전달 ✅
2. Distribution 스위치까지 올라감 → **설정에 따라** 다른 Access 스위치(H, I, J)에게도 전달 가능
3. **Router(Gateway)에서 차단** → 인터넷(외부)으로는 **절대 전달되지 않음** ❌

> [!note] Broadcast 도달 범위 = LAN 구간
> - Broadcast가 도달할 수 있는 범위 = **LAN**
> - Router(Gateway)를 넘어가는 구간 = **WAN**
> - 따라서 Broadcast 도달 범위로 LAN과 WAN의 경계를 나눌 수 있다

### Broadcast 세그먼트 / 도메인

- **Broadcast Segment(Domain)** = Broadcast 트래픽이 전달되는 단위 범위
- Router가 경계가 되어 Broadcast를 차단 → LAN 내부로 제한

---

## 5. OSI 7 Layer / DoD 모델 기준 정리

```
┌─────────────────────────────────────────────────────┐
│  DoD: Network Access 계층                            │
│  OSI: L1 (Physical) + L2 (Data Link)                │
│                                                      │
│  → 물리적 네트워크 (LAN)                              │
│  → MAC 주소 기반, Broadcast Frame                    │
├─────────────────────────────────────────────────────┤
│  DoD: Internet 계층                                  │
│  OSI: L3 (Network)                                  │
│                                                      │
│  → 논리적 네트워크 (WAN / 인터넷)                      │
│  → IP 주소 기반, Broadcast Packet                    │
│  → 소프트웨어로 구현 (Logical = Virtual)               │
└─────────────────────────────────────────────────────┘
```

---

## 6. AWS 환경에서의 Broadcast

> [!tip] AWS는 물리적 얘기를 하지 않는다
> - AWS는 **L3(IP)부터 시작** — 물리적 L2 구간은 자동화 처리
> - **IP 설정**과 **라우팅 테이블** 설정이 핵심
> - SDN(소프트웨어 정의 네트워크)으로 구현 → Broadcast 전달 범위를 **소프트웨어적으로 제어**
> - IP뿐만 아니라 도메인 기반으로도 라우팅 가능 (SDN의 유연성)

| 물리 환경 | AWS 환경 |
|----------|---------|
| Broadcast 자연 발생 | Broadcast **없음** (SDN이 제어) |
| Router에서 Broadcast 차단 | Route Table로 경로 제어 |
| LAN/WAN 물리적 경계 | **VPC + Subnet**으로 논리적 경계 |

---

## 핵심 요약

> [!summary]
> 1. **LAN** = 물리적 네트워크 (L2 이하, MAC 주소, Frame)
> 2. **WAN** = 논리적 네트워크 (L3 이상, IP 주소, Packet, 소프트웨어 구현)
> 3. LAN과 WAN의 경계 기준 = **Broadcast 도달 범위**
> 4. **Broadcast** = 네트워크 내 불특정 다수 전체에게 전송 (비효율의 원인)
> 5. Broadcast 주소 = 모든 비트가 1 (MAC: `FF:FF:FF:FF:FF:FF`)
> 6. **Router(Gateway)**에서 Broadcast 차단 → 외부(인터넷)로 전달되지 않음
> 7. Broadcast가 도달하는 범위 = **Broadcast Domain** = LAN 구간
> 8. AWS는 **L3(IP)부터 시작**, SDN으로 Broadcast를 소프트웨어적으로 제어
