---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-04-18
updated: 2026-06-03
tags:
  - network
  - IP
---

# IP 헤더와 AWS ENI

## 개요

IP 헤더의 필드별 구조를 상세히 분석하고, 실제 프레임 데이터를 수작업으로 해석하는 과정을 다룬다.
이어서 AWS 환경에서 네트워크 트래픽을 수집·분석하기 위한 **ENI**와 **VPC Traffic Mirroring** 서비스를 소개한다.

---

## 1. Encapsulation 구조 — 중첩이 아니라 나열

프레임 데이터는 마트로시카처럼 "안에 들어 있는" 것이 아니라, **연속된 비트 스트림**으로 나열되어 있다.

```
[Ethernet Header][  IP Header  ][  TCP Header  ][   Payload   ]
    14 bytes        20 bytes       20 bytes          ...
       ↑                ↑              ↑
  Next Header →   Next Header →     데이터
```

- 헤더가 안에 들어가 있는 게 아니라 **옆으로 쭉 이어져 있음**
- 각 헤더는 **nested(중첩)** 가 아니라 **next(다음)** 관계
- 앞 헤더의 Type/Protocol 필드가 "다음 헤더가 무엇인지" 알려줌
    - Ethernet Type `0x0800` → 다음은 IP 헤더
    - IP Protocol `0x06` → 다음은 TCP 헤더

---

## 2. IPv4 헤더 구조

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─Version─┤──IHL──┤─────ToS─────┤───────── Total Length ─────────┤  ← 행 1
├──────── ID (Identification) ──┤─Flags─┤──Fragment Offset───────┤  ← 행 2 (단편화)
├────── TTL ─────┤── Protocol ──┤──────── Header Checksum ───────┤  ← 행 3
├─────────────────── Source IP Address ──────────────────────────┤  ← 행 4
├─────────────────── Destination IP Address ─────────────────────┤  ← 행 5
├─────────────────── Options (선택) + Payload... ────────────────┤
```

- 각 행은 **32비트(4바이트)**
- 행 1~5 = 5행 × 4바이트 = **기본 20바이트** (IHL=5일 때)

### 필드별 상세

| 필드 | 크기 | 일반적 값 | 설명 |
| --- | --- | --- | --- |
| **Version** | 4 bit | `4` | IPv4이므로 항상 4 |
| **IHL** | 4 bit | `5` | Internet Header Length. 5 × 4 = 20바이트 |
| **ToS** | 8 bit | `0x00` | Type of Service |
| **Total Length** | 16 bit | 가변 | 패킷 전체 바이트 길이 |
| **Identification** | 16 bit | 랜덤 | 단편화 시 조각 식별용 ID |
| **Flags** | 3 bit | — | DF(Don't Fragment), MF(More Fragments) |
| **Fragment Offset** | 13 bit | — | 단편화된 조각의 위치 정보 |
| **TTL** | 8 bit | `128`(Win) / `64`(Linux) | 라우터 통과 시 1씩 감소, 0이 되면 패킷 폐기 |
| **Protocol** | 8 bit | `06`(TCP) / `17`(UDP) | 상위 계층(L4) 프로토콜 식별 |
| **Header Checksum** | 16 bit | 자동 계산 | 오류 검출용 16-bit 검사합 |
| **Source IP** | 32 bit | — | 출발지 IP 주소 |
| **Destination IP** | 32 bit | — | 목적지 IP 주소 |

---

## 3. 주요 필드 심화

### TTL (Time To Live)

- 라우터(hop)를 하나 통과할 때마다 **1씩 감소**
- 0이 되면 패킷을 **폐기** → 좀비 패킷이 인터넷에 영원히 떠도는 것을 방지하는 안전장치
- OS별 기본값으로 **발신 OS를 추정**할 수 있음

| TTL 값 (16진수 → 10진수) | 추정 OS |
| --- | --- |
| `0x80` → 128 | Windows |
| `0xFF` → 255 | Linux (일부) |
| `0x40` → 64 | macOS / Linux |

### Fragmentation (단편화)

- **수신 측이 보낸 만큼을 한 번에 받을 능력이 안 될 때** 패킷을 더 잘게 쪼갬
- **ID** 값으로 같은 원본 패킷의 조각임을 식별하고, 수신 측에서 재조립
- **DF** (Don't Fragment) — 단편화 금지 플래그
- **MF** (More Fragments) — 뒤에 조각이 더 있음을 표시
- **Fragment Offset** — 원본에서의 위치, 조립 순서 결정에 사용

### Checksum (검사합)

- **16비트 단위**로 데이터를 누산하여 계산
- Checksum 위치 자체는 계산에서 제외
- 누산 중 오버플로우 발생 시 캐리는 버림
- 전송 중 비트 오류(0↔1 뒤바뀜) **검출** 목적
- 요즘은 **NIC가 하드웨어적으로 자동 계산** (CPU가 하지 않음)

### Byte Order (바이트 순서)

| 환경 | 방식 | 비고 |
| --- | --- | --- |
| 네트워크 패킷 | **Big-endian** | Network Byte Order (표준) |
| x86/x64 PC (Intel/AMD) | **Little-endian** | 값이 뒤집어져 있음 |

> [!warning] 주의
> C/C++로 패킷 분석 프로그램을 개발할 때, 바이트 순서 변환을 반드시 처리해야 한다.
> (`htons`, `ntohs`, `htonl`, `ntohl` 등의 함수 사용)

---

## 4. 프레임 수작업 분석 예시

### Raw 데이터 파싱 과정

실제 캡처된 프레임 Raw 데이터(16진수)를 순서대로 잘라서 해석한다:

```
Raw 프레임 데이터 (16진수):
XX XX XX XX XX XX  XX XX XX XX XX XX  08 00  45 00
├─ Dst MAC (6B) ─┤├─ Src MAC (6B) ─┤├Type┤├─IP─┤
                                     0800   v4 IHL=5
                                     =IPv4

00 30  XX XX  XX XX  80  06  XX XX
├TLen┤ ├─ID─┤ ├Frag┤ TTL Proto Chksum
0x30    단편화   128  TCP
=48B    관련    =Win
```

```
[Ethernet Header]
├ Dst MAC (6 bytes)
├ Src MAC (6 bytes)
└ Type    (2 bytes) → 0x0800 = IPv4

[IP Header]  ← Type이 0x0800이므로 이 이후는 IP 헤더
├ Version + IHL → 0x45 → Version=4, IHL=5 (20 bytes)
├ ToS           → 0x00
├ Total Length  → 0x0030 → 48 bytes
├ ID            → (16-bit 값)
├ Flags + Offset
├ TTL           → 0x80 → 128 → Windows로 추정
├ Protocol      → 0x06 → TCP
├ Checksum      → (16-bit 값)
├ Src IP        → 0xC0.A8.01.B6 → 192.168.1.182
└ Dst IP        → (이후 4 bytes 계산)

[TCP Header]  ← Protocol이 0x06이므로 이 이후는 TCP 헤더
└ ...
```

### IP 주소 변환 예시

16진수 → 10진수 변환으로 IP 주소를 읽어낸다:

```
C0 A8 01 B6
│  │  │  └─ 0xB6 = 182
│  │  └──── 0x01 = 1
│  └─────── 0xA8 = 168
└────────── 0xC0 = 192
→ 192.168.1.182
```

> [!tip] 핵심
> 수작업 분석이 실무에서 반드시 필요한 것은 아니지만, 헤더 구조를 이해하면 **Wireshark 분석 결과를 올바르게 해석**하는 데 큰 도움이 된다.

---

## 5. Wireshark — 프로토콜 분석기

위의 수작업 분석을 **자동으로 수행**해주는 도구.

- 네트워크 트래픽을 **캡처 + 분석**
- Raw hex 데이터를 파싱하여 필드별로 깔끔하게 표시
- 제한 사항: **100MB 이상** 캡처 파일은 버벅임 → 수집 데이터를 분할 필요

---

## 6. AWS ENI와 VPC Traffic Mirroring

### 장애 분석이 필요한 상황

3-tier 서비스에서 "느림"의 원인을 찾는 과정:

```
[사용자] → [프론트 (Next.js)] → [백엔드 (Spring Boot)] → [DB]
                "느려요!"
```

```
프론트 개발자 투입 → "모르겠는데요?"
백엔드 개발자 투입 → "모르겠는데요?"
DBA 투입           → "모르겠는데요?"
APM 로그 확인      → "모르겠는데요?"
    ↓
최후의 수단: 네트워크 트래픽 직접 캡처·분석
→ 구간별 TAP 스위치로 트래픽 수집 → Wireshark 분석
```

- APM(Application Performance Management)으로 확인 가능한 범위를 넘어설 때
- 개발팀·DBA 모두 원인을 찾지 못할 때
- **네트워크 레벨에서 패킷을 직접 분석**해야 하는 상황이 발생

### ENI (Elastic Network Interface)

- AWS EC2 인스턴스에 부착되는 **가상 네트워크 인터페이스**
- VPC Traffic Mirroring을 사용하기 위해 **ENI 설정이 필수**

### VPC Traffic Mirroring

- 물리 네트워크의 **TAP 스위치를 논리적으로 구현**한 AWS 서비스
- 운영 중인 서비스의 트래픽을 **복사(미러링)** 하여 별도로 수집·분석 가능
- 서비스에 영향 없이 패킷을 관찰할 수 있음

### 활용 사례

| 사례 | 설명 |
| --- | --- |
| **트러블 슈팅** | 대규모 서비스에서 지연 구간을 네트워크 레벨로 특정 |
| **침입 탐지 (NIDS)** | Network-based Intrusion Detection System 운영을 위한 트래픽 수집 |
| **네트워크 포렌식** | 보안 사고 발생 시 패킷 레벨 증거 확보 |

> [!note] 규모별 참고
> 소규모 서비스에서는 이 수준의 분석이 거의 필요하지 않다.
> **대규모 서비스 운영 중 장애가 발생했을 때**, 다른 방법으로 원인을 찾지 못하는 경우에 선택하는 수단이다.

---

## 한 줄 요약

> IP 헤더는 Version·TTL·Protocol·IP 주소 등의 필드로 구성되며, AWS 환경에서는 **ENI + VPC Traffic Mirroring**을 통해 네트워크 레벨의 패킷 분석 및 장애 대응이 가능하다.
