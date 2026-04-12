# MAC 테이블이 꼬였을 때 발생하는 장애

---

## 1. Unknown Unicast Flooding 폭증 (가장 흔함)

### 무슨 일이 벌어지나?

- 스위치가 **목적지 MAC을 테이블에서 찾지 못함**
- → 정상 유니캐스트 프레임을 **모든 포트로 플러딩**

### 결과

- 모든 포트로 프레임 전파
- 네트워크 트래픽 급증
- CPU / NIC 사용률 상승
- 체감 증상:
    - "네트워크가 갑자기 느려짐"
    - "특정 시간대에만 끊김"

> [!note] 특징
> 장애가 간헐적이라 원인 파악이 매우 어렵다.

---

## 2. MAC Flapping (MAC 주소가 포트를 왔다 갔다 함)

### 원인

- 같은 MAC 주소가 여러 포트에서 번갈아 관측됨
- 대표 원인:
    - 루프 (STP 미설정)
    - VM vMotion
    - NIC Teaming 설정 오류
    - 허브/브리지 잘못 연결

### 스위치 입장

```
MACA → 포트1
MACA → 포트5
MACA → 포트1
MACA → 포트5
...
```

### 결과

- 프레임이 엉뚱한 포트로 전달
- 통신이 됐다 안 됐다 반복
- 세션 끊김, 재접속 반복

> [!warning] 로그에 자주 보이는 메시지
> `MAC address flapping between port X and Y`

---

## 3. ARP 테이블 혼선 → 통신 불능

### 상황

- MAC 테이블이 꼬이면
- ARP 응답이 엉뚱한 포트로 전달됨

### 결과

- IP는 맞지만 MAC 매핑이 틀림
- 증상:
    - 핑 안 됨
    - 특정 서버만 접근 불가
    - 같은 네트워크인데 통신 안 됨

> [!quote] 이때 사람들이 흔히 하는 말
> "IP는 맞는데 왜 안 돼요?"

---

## 4. 브로드캐스트/플러딩 폭증 → 브로드캐스트 스톰

### 흐름

1. MAC 테이블 불안정
2. Flooding 증가
3. ARP / Unknown Unicast 증가
4. 네트워크 전체가 패킷으로 가득 참

### 결과

- 스위치 CPU 100%
- 모든 장비 응답 지연
- 최악의 경우 **네트워크 전체 다운**

> [!important]
> L2 장애는 **전파 속도가 매우 빠르다.**

---

## 5. 보안 문제 (스니핑 / 세션 탈취)

### 왜 보안 문제가 되나?

- Flooding이 증가하면
    - 원래 못 봐야 할 프레임을
    - 다른 장비가 수신 가능해짐

### 대표 공격

- ARP Spoofing
- MAC Spoofing
- Session Hijacking

> [!tip] 정상 vs 비정상 상태 비교
> **정상 상태의 스위치** → "내 프레임만 본다"
> **꼬인 상태의 스위치** → "남의 프레임도 보인다"

---

## 6. 장애 증상 요약 (현업에서 이렇게 보인다)

| 증상 | 내부 원인 |
| --- | --- |
| 네트워크 느림 | Unknown Unicast Flooding |
| 통신이 됐다 안 됨 | MAC Flapping |
| 특정 서버만 안 됨 | MAC/ARP 불일치 |
| 전사 네트워크 마비 | 브로드캐스트 스톰 |
| 보안 사고 | Flooding 기반 스니핑 |

---

## 7. MAC 테이블이 왜 꼬이게 될까?

### 주요 원인 TOP 5

1. **스위치 루프 (STP 미설정)**
2. ARP Spoofing / 공격
3. VM 이동 (vMotion)
4. NIC Teaming 설정 오류
5. 허브 + 스위치 혼합 사용

---

## 8. 운영 관점 핵심 교훈

> [!danger] 핵심
> MAC 테이블 문제는 "느림"이 아니라 **"L2 신뢰 붕괴"** 문제다.

실무에서 반드시 적용해야 할 **L2 보호 장치**:

- **STP 활성화** — 루프 방지
- **Port Security** — MAC 주소 제한
- **DHCP Snooping** — 비인가 DHCP 차단
- **Dynamic ARP Inspection** — ARP 스푸핑 방지
- **VLAN 분리** — 브로드캐스트 도메인 축소

---

## 한 줄 요약

> MAC 테이블이 꼬이면 스위치는 유니캐스트를 플러딩하기 시작하고, 그 결과 **성능 저하 → 간헐적 장애 → 보안 취약점 → 브로드캐스트 스톰**까지 연쇄적으로 발생한다.

---

## 참고 이미지

- [O'Reilly - Ethernet Switching](https://www.oreilly.com/api/v2/epubs/urn%3Aorm%3Abook%3A9781449307974/files/httpatomoreillycomsourceoreillyimages840274.png)
- [Cisco - Catalyst Switch MAC Table](https://www.cisco.com/c/dam/en/us/support/docs/switches/catalyst-6000-series-switches/23563-143-00.gif)
- [IPCisco - ARP Spoofing / Dynamic ARP Inspection](https://ipcisco.com/wp-content/uploads/2020/01/arp-spoofing-dynamic-arp-inspection-ipcisco-1.jpg)
