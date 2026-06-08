---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-05-01
updated: 2026-06-03
tags:
  - network
  - TCP
  - IP
---

# TCP 세션과 상태 그리고 게임서버

## 개요

TCP의 핵심 개념인 **연결(Connection / Virtual Circuit) → 세션 → 상태(State)** 의 흐름과,
**3-way handshake / 4-way handshake**를 통한 상태 전이 절차를 다룬다.
실전 응용으로 **게임 서버 개발 시의 핵심 규칙** — "클라이언트가 항상 Active, 서버는 절대 먼저 끊지 마라" 와 **TIME_WAIT 장애 사례**를 함께 정리한다.

---

## 1. 연결(Connection) = Virtual Circuit

> [!important] 연결의 본질
> TCP의 연결은 **물리적 회로**가 아니라 **논리적 가상 회로(Virtual Circuit)** 이다.
> 전선이 실제로 연결되는 것이 아니라, **상태(State)** 라는 형태로 추상화된 개념.

### 연결이 만드는 부산물

```
연결(Connection)
   ↓
세션(Session)        ← 추상화된 의미
   ↓
상태(State)          ← 절차의 적정성을 판단하는 기준
   ↓
상태 전이(Transition) ← Stateful Inspection의 근거
```

- **Stateful Inspection** = 상태 전이의 적정성을 검사하는 방화벽 기능 ([[TCP 와 UDP]] 참조)

### 전체 상태 전이 흐름 (요약)

```
CLOSED
   │
   ├─ [서버] LISTEN          ← 미리 대기
   │
   └─ [클라이언트 시작]
         │
         ▼
    3-Way Handshake          ← 연결 수립 (SYN → SYN+ACK → ACK)
         │
         ▼
    ESTABLISHED              ← 연결 완료
         │
         ▼
    데이터 송수신
         │
         ▼
    4-Way Handshake          ← 연결 종료 (FIN → ACK → FIN+ACK → ACK)
         │
         ▼
       CLOSED
```

---

## 2. Active vs Passive — 누가 주도하는가

| 구분 | 역할 | Socket 종류 | 주도성 |
| --- | --- | --- | --- |
| **서버** | 연결을 기다림 (LISTEN) | **Passive Socket** | 수동적 |
| **클라이언트** | 연결을 시도 (Connect) | **Active Socket** | 능동적 |

> [!important] 핵심 원칙
> **연결을 처음 시작한 쪽이 클라이언트, 끊을 때도 클라이언트가 먼저** 끊는다.
> 서버는 항상 **Passive**.

---

## 3. 3-way Handshake — 연결 수립

### 절차

```
[Client]                                [Server]
  CLOSED                                  LISTEN  ① 미리 대기 중
                                            │
  ① SYN(seq=1000) ────────────────────→ │
                                            │   "어, 누가 연결하래?"
  ② ←──── SYN(seq=4000) + ACK(1001) ──── │
            "내 번호는 4000, 네가 보낸 1000 잘 받았어 → ACK=1001"
            
  ③ ACK(seq=1001, ack=4001) ────────→ │
            "네 번호 4000도 잘 받았어 → ACK=4001"
            
  ESTABLISHED                          ESTABLISHED
```

### 시퀀스 번호 교환의 본질

> [!important] 3-way handshake의 정체
> "연결한다"의 본질 = **서로의 32비트 시퀀스 번호를 교환·동기화**하는 행위.
> SYN(32bit) + ACK(32bit) = **64비트의 정보를 정확히 sync** 해야 함.
> 어긋나면 통신 자체가 성립하지 못함.

### handshake 시 함께 교환되는 정보

| 정보 | 의미 |
| --- | --- |
| **MSS (Maximum Segment Size)** | 양쪽이 1460 등을 교환해 단편화 회피 |
| **SACK (Selective Acknowledgment)** | 혼잡 제어 시 부분 재전송 지원 여부 |
| **Window Size** | 수신 버퍼 크기 (Flow Control 기준값) |
| Window Scale, Timestamp 등 | 추가 옵션 |

> [!note] MSS 협상
> MTU = 1500 → MSS = **1460** ([[택배와 닮은 Packet]] 참조)
> "우리 단편화 안 나도록 1460 단위로 자르자"라는 합의를 이때 한다.

### 연결 인지 시점

- **클라이언트가 RTT만큼 먼저** 연결됨을 인지
- 서버는 마지막 ACK를 받고 나서야 ESTABLISHED 인식

---

## 4. 4-way Handshake — 연결 종료

### 절차 (정상 종료)

```
[Client]                              [Server]
  ESTABLISHED                          ESTABLISHED
        │                                  │
  ① FIN ────────────────────────────→ │  "이제 끊자"
        │                                  │
        │ ←──────── ACK ───────────────── │  "OK"
        │                                  │
        │ ←──────── FIN ───────────────── │  "나도 끊을게"
        │                                  │
  ② ACK ──────────────────────────→  │  "OK"
        │                                  │
  TIME_WAIT  (잠시 대기 후) → CLOSED    CLOSED
```

> [!tip] 실제로는 2-way로 보일 수도 있음
> Wireshark로 보면 4-way 절차가 **합쳐져서 2-way**처럼 끝나기도 한다.
> 이는 잘못된 것이 아니라 **시간 절약을 위해 ACK+FIN을 묶어 보낸** 결과.

### 핵심 규칙 — 누가 먼저 끊는가?

> [!danger] 절대 규칙
> **연결 종료의 시작도 항상 클라이언트가 한다.**
> 처음 연결을 시도한 쪽(Active)이 끊는 것도 시도해야 함.

---

## 5. TCP 상태 다이어그램

### 서버 측 상태 전이

```
CLOSED → LISTEN → SYN_RCVD → ESTABLISHED 
                                  │
                                  ↓
                              CLOSE_WAIT → LAST_ACK → CLOSED
```

### 클라이언트 측 상태 전이

```
CLOSED → SYN_SENT → ESTABLISHED 
                         │
                         ↓
                     FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

### 주요 상태 의미

| 상태 | 위치 | 의미 |
| --- | --- | --- |
| **CLOSED** | 양쪽 | 닫힘 (초기 상태) |
| **LISTEN** | 서버 | 접속 대기 중 |
| **SYN_SENT / SYN_RCVD** | C / S | handshake 중 |
| **ESTABLISHED** | 양쪽 | 연결 완료, 송수신 가능 |
| **FIN_WAIT** | 종료 시작 측 | FIN 보낸 후 응답 대기 |
| **CLOSE_WAIT** | 종료 받은 측 | FIN 수신, 닫을 준비 |
| **TIME_WAIT** | 종료 시작 측 | 마지막 정리 시간 (2MSL 대기) |

---

## 6. 게임 서버 개발 — 절대 규칙

> [!danger] 서버는 먼저 끊으면 안 된다
> "클라이언트의 행위는 항상 Active"
> **서버가 먼저 "야 우리 연결 끊자"라고 FIN을 보내면 안 된다.**

### 시나리오 — 클라이언트가 이상 행동을 한다면?

```
[클라이언트]    이상 행동 감지
                       │
                       ↓
[서버]           튕겨야겠다!
                       │
                       │  ❌ FIN 보내기 (잘못된 방식)
                       │  ✅ RST 유도하기 (올바른 방식)
                       ↓
              소켓 옵션 변경 → 강제 회수 → RST 발생
```

### 올바른 처리 방법

> [!important] 강제 종료는 RST로
> 서버가 강제로 클라이언트를 끊어야 한다면 **정상 종료(FIN)가 아니라 비정상 종료(RST)** 를 유도해야 한다.
> 소켓 옵션을 변경해 **소켓이 강제 회수**되도록 → RST 패킷이 자동 발생.

---

## 7. TIME_WAIT 장애 — 서버 개발자가 모르면 큰일 나는 함정

### TIME_WAIT은 왜 생기나?

- **연결을 먼저 끊는 쪽**에 발생하는 상태
- 마지막 ACK 분실에 대비해 **2MSL(Maximum Segment Lifetime)** 만큼 대기
- 보통 **수십 초 ~ 수 분**

### 서버에서 TIME_WAIT이 발생한다면? = 개발 실수

> [!warning] 서버에 TIME_WAIT이 보인다면
> "아, 서버가 먼저 연결을 끊는 잘못된 코드가 있구나"
> 누가 개발했는지 찾아서 코드를 검토해야 한다.

### 진단 방법

```bash
netstat -ano | findstr TIME_WAIT     # Windows
netstat -an | grep TIME_WAIT          # Linux/Mac
```

**서버 측에서 TIME_WAIT이 다수 출력 → 코드 결함 의심**

### 소켓 자원 고갈 — 시각화

```
[서버 소켓 풀 상태]
┌─────────────────────────────────────────┐
│  소켓 #1 :  ESTABLISHED  (Client A)     │  ✅ 정상 사용
│  소켓 #2 :  ESTABLISHED  (Client B)     │  ✅ 정상 사용
│  소켓 #3 :  TIME_WAIT                   │  ⚠️ 묶여 있음 (수 분)
│  소켓 #4 :  TIME_WAIT                   │  ⚠️ 묶여 있음
│  소켓 #5 :  TIME_WAIT                   │  ⚠️ 묶여 있음
│  소켓 #6 :  TIME_WAIT                   │  ⚠️ 묶여 있음
│  소켓 #7 :  TIME_WAIT                   │  ⚠️ 묶여 있음
│  ...                                    │
│  소켓 #N :  TIME_WAIT                   │
└─────────────────────────────────────────┘
                    │
                    ▼
        가용 소켓 자원 고갈
                    │
                    ▼
        새 연결 시도 → 실패 → 서버 장애 💀
```

### 실제 장애 시나리오 — 머리 빠지는 사례

```
[서버 실행]
    Day 1:  잘 돌아감 ✅
    Day 2:  잘 돌아감 ✅
    Day 3 새벽 3시: 갑자기 소켓 못 열고 서버 죽음 💀

원인: TIME_WAIT 누적
   → 가용 소켓 자원 고갈
   → 새 연결 거부
   → 서버 다운
```

> [!tip] "왜 꼭 2~3일째 새벽에만 죽어요?"
> TIME_WAIT 소켓이 누적되면서 자원이 고갈되는 시점이 며칠 걸리고,
> 트래픽 패턴에 따라 새벽에 임계점에 도달하기 때문.
> **개발자에게 원형 탈모를 부르는 대표적 버그.**

### 해결책

| 방법 | 설명 |
| --- | --- |
| **클라이언트가 끊게 설계** | 프로토콜 설계 시 종료는 항상 클라이언트가 주도 |
| **RST으로 강제 종료** | 서버가 끊어야 할 때는 소켓 옵션 변경 → RST 전송 |
| **`SO_LINGER` 옵션** | 소켓 옵션을 `linger=0`으로 설정하면 close() 시 **즉시 RST** 전송 → TIME_WAIT 회피 |

> [!example] SO_LINGER 활용 — 강제 RST 종료
> ```c
> struct linger so_linger;
> so_linger.l_onoff  = 1;
> so_linger.l_linger = 0;          // 0초 → 즉시 RST
> setsockopt(sock, SOL_SOCKET, SO_LINGER, &so_linger, sizeof(so_linger));
> close(sock);                      // FIN 대신 RST가 발송됨
> ```

> [!important] 정리
> 1. 서버는 절대 **FIN을 먼저 보내지 않는다**
> 2. 클라이언트가 이상 행동 시 → **RST 유도** (`SO_LINGER` 등 소켓 옵션으로 즉시 리셋)
> 3. 정상 종료는 **반드시 클라이언트가 시작**하도록 프로토콜 설계

---

## 8. 운영자 관점 — netstat로 진단

| 발견 위치 | 의미 |
| --- | --- |
| **클라이언트에서 TIME_WAIT** | 정상 (클라이언트가 먼저 끊은 결과) |
| **서버에서 TIME_WAIT** | ⚠️ 비정상 → 코드 검토 필요 |
| **서버에서 CLOSE_WAIT 누적** | ⚠️ 클라이언트가 끊었는데 서버가 close()를 안 함 |

```bash
# 서버 소켓 상태 확인
netstat -ano                  # Windows
ss -tan state time-wait       # Linux (modern)
ss -tan state close-wait      # CLOSE_WAIT 누적 점검
```

---

## 9. 정리 — 한 그림

```
                    [Server: LISTEN]
                          │
        ① SYN ────────→  │  
                          │   ┌─── 3-way handshake ───┐
        ② ←─ SYN+ACK ──── │   │  Sequence 번호 교환    │
                          │   │  MSS / SACK 옵션 합의  │
        ③ ACK ────────→  │   └────────────────────────┘
                          │
              [ESTABLISHED 양쪽]
                          │
                  ─── 데이터 송수신 ───
                          │
        ④ FIN ────────→  │   ← Active(클라이언트)가 시작!
        ⑤ ←──── ACK ───── │
        ⑥ ←──── FIN ───── │
        ⑦ ACK ────────→  │
                          │
        [TIME_WAIT]    [CLOSED]
              │
        2MSL 대기
              │
          [CLOSED]
```

---

## 10. 핵심 정리표

| 항목 | 내용 |
| --- | --- |
| **연결 수립** | **3-Way Handshake** (SYN → SYN+ACK → ACK) |
| **연결 종료** | **4-Way Handshake** (FIN → ACK → FIN+ACK → ACK) |
| **연결 시도 주체** | **클라이언트 (Active)** |
| **종료 시도 주체** | **반드시 클라이언트** |
| **서버가 끊어야 할 때** | **RST(Reset)** 으로 강제 종료 (`SO_LINGER` 옵션 활용) |
| **서버에서 TIME_WAIT** | ⚠️ 잘못된 개발 — 코드 수정 필요 |
| **확인 명령** | `netstat`, `ss` |
| **Stateful Inspection** | 상태 전이의 적정성 검사 (방화벽) |
| **handshake 시 교환 정보** | MSS, SACK, Window Size 등 |

---

## 11. AWS 환경에서의 응용

| 항목 | 적용 |
| --- | --- |
| **NLB Health Check** | TCP 핸드셰이크(SYN/ACK)로 헬스 판정 |
| **Security Group** | Stateful — 핸드셰이크 추적해 자동 응답 허용 |
| **EC2 게임 서버** | TIME_WAIT 누적 시 EC2 재시작으로는 근본 해결 X → 코드 수정 필요 |
| **CloudWatch Metrics** | `EstablishedConnections`, `RST` 카운트 모니터링 |
| **Connection Draining** | ELB가 정상 종료 절차를 보장하도록 grace 기간 부여 |

---

## 한 줄 요약

> TCP의 연결은 **Virtual Circuit**으로 시작·송수신·종료의 **상태 전이 절차**가 있으며, **3-way handshake**로 시퀀스 번호를 교환해 연결하고 **4-way handshake**로 종료한다. **클라이언트는 항상 Active** — 연결도, 종료도 클라이언트가 먼저 시작해야 하며, **서버가 먼저 끊으면 TIME_WAIT 누적으로 며칠 뒤 새벽에 서버가 죽는 함정**에 빠진다. 강제 종료가 필요하면 **FIN이 아닌 RST**를 유도하는 것이 정답이다.
