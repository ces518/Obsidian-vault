---
type: concept
area: network
status: draft
source: AWS 로 배우는 네트워크 (책)
created: 2026-07-19
updated: 2026-07-19
tags:
  - network
  - mac
  - ethernet
  - frame
  - l2
---

# MAC 주소와 Ethernet Frame의 역할

## 개요

MAC 주소와 Ethernet Frame은 L2(데이터 링크 계층)에서 데이터를 **현재 링크의 다음 장치**로 전달하기 위한 핵심 요소다.
IP 주소가 네트워크 간 목적지를 식별한다면, MAC 주소는 각 L2 구간에서 실제 전달 대상을 식별한다.

## 한 줄 요약

> IP 주소는 목적지 네트워크까지의 경로를 찾는 데 쓰이고, MAC 주소는 각 L2 링크에서 Frame을 다음 장치로 전달하는 데 쓰인다. 따라서 라우터를 지날 때마다 Frame과 MAC 주소는 바뀔 수 있다.

---

## 왜 필요한가

웹 서버에 요청을 보낼 때 애플리케이션은 서버의 IP 주소를 목적지로 삼는다. 하지만 첫 번째 Ethernet 링크에서 NIC가 실제로 Frame을 보낼 대상은 보통 웹 서버가 아니라 **기본 게이트웨이**다.

이 차이를 이해하면 다음을 설명할 수 있다.

- 같은 LAN 안의 통신과 다른 네트워크로 나가는 통신의 차이
- L2 스위치가 MAC 주소 테이블을 사용하는 이유
- 라우터를 지날 때 MAC 주소는 바뀌고 IP 주소는 보통 유지되는 이유
- Wi-Fi MAC 랜덤화와 가상 NIC가 가능한 이유

---

## 핵심 개념

### 1. MAC 주소는 L2 링크의 인터페이스 주소다

일반적인 Ethernet MAC 주소는 48비트다. 제조사가 할당한 기본 MAC(BIA)은 보통 전역적으로 고유하도록 관리되지만, 운영체제나 드라이버는 현재 사용 MAC을 덮어쓸 수 있다.

| 구분 | 의미 |
| --- | --- |
| 제조사 할당 MAC | NIC에 저장된 기본 주소(BIA) |
| 현재 사용 MAC | 운영체제나 드라이버가 실제 전송에 사용하는 주소 |
| 로컬 관리 MAC | 관리자가 지정하거나 운영체제가 랜덤화한 주소 |

> [!note] OUI
> 전역 관리 MAC 주소의 앞 24비트는 IEEE가 조직에 할당한 OUI일 수 있다. 하지만 로컬 관리 주소와 일부 가상 환경의 MAC에는 이 해석을 그대로 적용할 수 없다.

### 2. Ethernet Frame은 L2 전달 단위다

Ethernet II Frame의 핵심 필드는 다음과 같다.

| 필드 | 크기 | 역할 |
| --- | --- | --- |
| Destination MAC | 48비트 | 현재 링크에서 Frame을 받을 다음 장치 |
| Source MAC | 48비트 | 현재 링크에서 Frame을 보낸 인터페이스 |
| EtherType | 16비트 | Payload의 상위 프로토콜 식별 |
| Payload | 가변 | 일반적으로 IP Packet 등 |

`0x0800` EtherType은 IPv4 Packet이 뒤따름을 뜻한다.

> [!note] 범위
> 위 표는 Ethernet II의 핵심 필드만 다룬다. Preamble/SFD, FCS, VLAN 태그는 생략했으며 IEEE 802.3 형식에서는 같은 16비트 위치를 Length로 해석할 수 있다.

### 3. IP와 MAC은 경쟁 관계가 아니라 역할 분담이다

| 구분 | IP 주소 | MAC 주소 |
| --- | --- | --- |
| 계층 | L3 | L2 |
| 주된 역할 | 네트워크 간 목적지 식별과 라우팅 | 현재 L2 링크의 다음 장치 전달 |
| 적용 범위 | 여러 네트워크를 지나는 Packet | 하나의 L2 링크에서의 Frame |
| 라우터 통과 시 | 일반적으로 유지 | 다음 링크에 맞게 교체 |

IP Packet은 목적지 IP를 바탕으로 라우팅되고, 그 Packet을 담은 Ethernet Frame은 각 링크에서 새로 만들어진다.

---

## 예시

### 다른 네트워크의 서버로 통신할 때

```
PC (192.168.1.10)                Router                 Server (8.8.8.8)
        │                           │                           │
Frame 1 │ Dst=Router MAC            │                           │
Packet  │ Src=192.168.1.10          │                           │
        │ Dst=8.8.8.8               │                           │
        ├──────────────────────────>│                           │
                                    │ Frame 2: 새 MAC 주소 사용 │
                                    ├──────────────────────────>│
                                    │ Packet: 같은 목적지 IP로 전달
```

PC는 서버 IP인 `8.8.8.8`을 Packet의 목적지로 유지한다. 그러나 현재 LAN에서 보내야 할 다음 장치는 기본 게이트웨이이므로, 첫 Frame의 Destination MAC은 Router의 MAC이다.

Router는 Packet을 다음 링크로 전달할 때 새 Frame을 만들고, 그 링크에 맞는 Source MAC과 Destination MAC을 사용한다.

> [!caution] NAT 예외
> 일반적인 라우팅은 IP 주소를 유지하지만, NAT 장비를 통과하면 Source 또는 Destination IP 주소와 포트가 변할 수 있다.

### 같은 LAN 안에서 통신할 때

같은 L2 네트워크에 있는 두 장치는 목적지 장치의 MAC 주소를 Destination MAC으로 사용한다. L2 스위치는 MAC 주소 테이블을 보고 해당 MAC이 연결된 포트로 Frame을 전달한다.

---

## 실무에서 주의할 점

- MAC 주소는 제조사 할당 주소와 현재 사용 주소를 구분해서 봐야 한다. Wi-Fi MAC 랜덤화, 가상 NIC, MAC spoofing은 모두 현재 사용 주소에 영향을 준다.
- MAC 주소는 인터넷 전체에서 직접 라우팅되지 않는다. 각 L2 링크에서 다음 장치로 전달하는 데 사용된다.
- Packet 분석 시 Ethernet II인지 IEEE 802.3인지, VLAN 태그가 있는지에 따라 Frame 필드 해석이 달라질 수 있다.
- AWS VPC에서는 물리 L2 구성과 스위치의 MAC 테이블을 직접 설정하지 않는다. ENI와 라우팅 테이블 같은 추상화된 인터페이스를 사용한다.

---

## 헷갈리기 쉬운 점

### MAC 주소는 항상 전 세계에서 유일한가?

제조사가 할당한 전역 관리 MAC은 일반적으로 고유하도록 관리된다. 하지만 로컬 관리 주소, 가상 NIC, 랜덤화된 MAC은 이 전제를 따르지 않을 수 있다.

### MAC 주소로는 인터넷을 사용할 수 없는가?

IP 통신도 각 링크에서는 MAC 주소를 사용한다. 다만 인터넷 규모의 네트워크 간 목적지 식별과 라우팅은 IP 주소가 담당하며, MAC 주소는 각 링크에서만 의미가 있다.

### Destination MAC은 최종 서버의 MAC인가?

같은 L2 네트워크가 아니라면 보통 아니다. 목적지가 다른 네트워크에 있을 때 첫 Frame의 Destination MAC은 기본 게이트웨이의 MAC이다.

---

## 면접식 설명

> MAC 주소는 L2에서 네트워크 인터페이스를 식별하는 주소이고, Ethernet Frame은 MAC 주소를 이용해 현재 링크의 다음 장치로 데이터를 전달하는 단위입니다. 다른 네트워크의 서버에 통신할 때 Packet의 목적지 IP는 서버를 가리키지만, 첫 Ethernet Frame의 목적지 MAC은 보통 기본 게이트웨이입니다. 라우터는 다음 링크로 Packet을 전달할 때 새 Frame을 만들기 때문에 MAC 주소는 홉마다 바뀌고, IP 주소는 NAT 같은 예외가 없으면 유지됩니다.

---

## 관련 문서

- [[NIC  LAN 카드  MAC 주소  Frame]]
- [[계층별 데이터 단위]]
- [[L2 스위치]]
- [[LAN WAN Broadcast]]
- [[라우터에 대한 최소 이론]]
- [[스위칭과 스위치의 판단 근거]]
- [[MAC 테이블이 꼬였을때 발생하는 장애]]
