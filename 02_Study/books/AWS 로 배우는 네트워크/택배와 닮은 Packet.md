
## 개요

인터넷에서 유통되는 데이터 단위인 **Packet**의 구조를 **택배 비유**를 통해 설명한다.
Encapsulation/Decapsulation 원리와 MTU/MSS 개념까지 다룬다.

---

## 1. Packet = 택배 박스

> [!important] 정의
> **Packet** = 인터넷(L3 논리 네트워크)에서 유통되는 **데이터 단위**
> 구조가 택배와 거의 동일하다.

### 택배 ↔ 네트워크 매핑

| 택배 | 네트워크 | 설명 |
|------|---------|------|
| 선물 (책) | **Data** (Stream) | 원본 데이터 |
| 뽁뽁이 포장 | **Segmentation** | Stream을 일정 단위로 자르기 → TCP Segment |
| 박스 포장 (Boxing) | **Encapsulation** | 헤더 붙여서 Packet 완성 |
| 송장 (출발지·목적지) | **Header** | IP 주소 (집 주소), Port 번호 (사람 식별) |
| 택배 박스 | **Packet** | Header + Payload = 전송 단위 |
| 트럭 | **Frame** (L2) | Packet을 실어 나르는 L2 단위 |
| 물류사 (CJ, 한진 등) | **ISP** (KT, SK 등) | 인터넷 인프라 운영 |
| 집 현관 | **Gateway** | LAN ↔ WAN 경계 |

---

## 2. 택배 비유로 보는 전체 흐름

```
철수 (Process)                                    영희 (Process)
  │                                                  ▲
  │ ① 데이터를 Segment로 자름                          │ ⑥ Decapsulation
  │ ② Header 붙여서 Packet 완성 (Boxing)              │    (Unboxing)
  │ ③ 현관(Gateway)에 놓음                            │
  ▼                                                  │
Gateway ──── ISP(물류사) ──── 라우팅 ──── Gateway
              트럭(Frame)에 실어서 이동
```

### 식별 체계

| 택배 | 네트워크 | 역할 |
|------|---------|------|
| **집 주소** | **IP 주소** | Host 식별 |
| **집 주소 중 "OO구 OO동"** | **Network ID** | 라우팅 (네트워크 찾기) |
| **집 주소 중 "몇 번지"** | **Host ID** | 네트워크 내 호스트 찾기 |
| **받는 사람 이름 (영희)** | **Port 번호** | Host 내 Process 식별 |

> [!note] 라우팅 순서
> **Network ID로 네트워크를 먼저 찾고** → Host ID로 호스트를 찾고 → Port 번호로 프로세스를 찾는다

---

## 3. Encapsulation과 Decapsulation

### Encapsulation (캡슐화) — 보내는 쪽

> 데이터를 안쪽에서 바깥으로 **겹겹이 포장**하는 과정 (마트로시카 인형)

```
                Data
           ┌────────────┐
      TCP  │    Data    │                    → Segment
     ┌─────┼────────────┤
  IP │ TCP │    Data    │                    → Packet
  ┌──┼─────┼────────────┤
ETH│IP│ TCP │    Data    │                   → Frame
  └──┴─────┴────────────┘
  헤더  헤더   헤더   페이로드
```

### Decapsulation (역캡슐화) — 받는 쪽

> 바깥에서 안쪽으로 **헤더를 하나씩 벗겨내어** 최종 데이터를 꺼내는 과정

```
Frame 수신 → ETH 헤더 제거 → Packet
           → IP 헤더 제거  → Segment
           → TCP 헤더 제거 → Data (원본)
```

### 택배로 치면?

| 과정 | 택배 | 네트워크 |
|------|------|---------|
| **Boxing** | 선물 → 뽁뽁이 → 박스 → 송장 | Data → Segment → Packet |
| **Unboxing** | 송장 떼고 → 박스 뜯고 → 뽁뽁이 벗기고 → 선물 | Frame → Packet → Segment → Data |

---

## 4. 영희네 어머니 = 보안 장비

| 어머니의 행동 | 보안 장비 | 설명 |
|-------------|---------|------|
| 택배 열어보고 **반송시킬지 결정** | **Firewall** | 패킷을 검사하고 차단/허용 |
| 내용만 보고 **그냥 전달** | **NIDS** (Network IDS) | 트래픽을 모니터링만 하고 통과시킴 |

---

## 5. MTU와 MSS

### MTU (Maximum Transmission Unit)

> [!important] MTU = 한 번에 전송 가능한 Packet의 최대 크기
> 인터넷 환경에서 **MTU = 1,500 바이트** (반드시 암기)

### MSS (Maximum Segment Size)

> MTU에서 IP 헤더 + TCP 헤더를 뺀 **실제 데이터(페이로드)의 최대 크기**

```
┌─────────────────────── Frame (1,514 bytes) ────────────────────────┐
│ ETH Header │            MTU (1,500 bytes)                          │
│  (14 bytes)│                                                       │
│            │ IP Header │ TCP Header │         Payload              │
│            │ (20 bytes)│ (20 bytes) │      (1,460 bytes)           │
│            │           │            │      = MSS                   │
└────────────┴───────────┴────────────┴──────────────────────────────┘
```

### 핵심 수치 (암기)

| 항목 | 크기 | 설명 |
|------|------|------|
| **MTU** | **1,500 bytes** | Packet 최대 크기 (인터넷 표준) |
| IP Header | 20 bytes | — |
| TCP Header | 20 bytes | — |
| **MSS** | **1,460 bytes** | 실제 전송 가능한 데이터 최대 크기 (1500 - 40) |
| ETH Header | 14 bytes | L2 헤더 |
| Frame 크기 | 1,514 bytes | ETH Header + MTU |

### Jumbo Frame

| 구분 | 일반 Frame | Jumbo Frame |
|------|-----------|-------------|
| **크기** | ~1,514 bytes | **~9,000 bytes** (약 9KB) |
| **사용 환경** | 인터넷 (WAN) | **LAN 내부만** 가능 |
| **조건** | — | L2 스위치 + NIC 모두 Jumbo Frame 지원 필요 |

> [!caution] Jumbo Frame은 LAN 전용
> 인터넷(WAN) 환경에서는 사용 불가. 인터넷 MTU는 그냥 **1,500**이다.

---

## 6. AWS에서의 핵심

> [!tip] AWS 환경에서 중요한 것
> - **IP 주소 표기** (CIDR) — 어떤 호스트인지 식별
> - **Port 번호 설정** — 어떤 프로세스에 접근할지 결정
> - 웹 서비스(프론트엔드, 백엔드)마다 개방하는 포트가 다름
> - **Security Group**에서 IP + Port 조합으로 접근 제어

---

## 핵심 요약

> [!summary]
> 1. **Packet** = 인터넷에서 유통되는 L3 데이터 단위 (= 택배 박스)
> 2. **Header**(송장) = IP 주소(집 주소) + Port 번호(받는 사람) / **Payload** = 실제 데이터
> 3. 라우팅: **Network ID** → **Host ID** → **Port 번호** 순서로 찾아감
> 4. **Encapsulation**: Data → Segment → Packet → Frame (겹겹이 포장, 마트로시카)
> 5. **Decapsulation**: Frame → Packet → Segment → Data (헤더를 하나씩 벗겨냄)
> 6. **MTU = 1,500 bytes** (인터넷 표준) / **MSS = 1,460 bytes** (MTU - IP헤더20 - TCP헤더20)
> 7. Jumbo Frame (~9KB)은 **LAN 내부에서만** 가능
> 8. ISP = 물류사 / Gateway = 집 현관 / Firewall = 영희 어머니
