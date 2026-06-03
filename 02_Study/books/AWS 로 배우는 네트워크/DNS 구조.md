---
type: study-note
area: network
status: raw
source: AWS 로 배우는 네트워크 (책)
updated: 2026-06-03
tags:
  - network
  - DNS
  - HTTP
  - 보안
  - IP
created: 2026-05-05
---

# DNS 구조

## 개요

**DNS(Domain Name Service)** 는 사람이 외우기 쉬운 도메인 이름을 IP 주소로 변환해주는 시스템이다.
단순한 변환을 넘어서 **분산 구조형 데이터베이스**, **계층적 트리 구조**, **캐시 DNS**, **A/CNAME 레코드** 같은 핵심 메커니즘을 담고 있으며, 결과적으로 **부하 분산**과 **국가 인터넷 인프라**의 중심이 된다.
DNS 설정 보안의 중요성과 AI 위임 시 주의점까지 함께 정리한다.

---

## 1. DNS는 왜 필요한가

### 핵심 한계

```
[Browser]                   [Naver Server]
   │                              │
   │  "naver.com 접속해야지"      │
   │                              │
   │  ❌ 도메인 이름으로는        │
   │     TCP/IP 연결 불가         │
   │  ✅ IP 주소가 있어야 연결    │
```

> [!important] DNS의 본질
> 인간은 IP 주소를 못 외우니, **도메인 이름 → IP 주소** 변환을 대신해 주는 서비스.
> 한 번에 끝이 아니라 **부하 분산**까지 확장되는 핵심 인프라.

---

## 2. DNS = 분산 구조형 데이터베이스

### 분산의 의미

```
[A 서버]   [B 서버]   [C 서버]   ...   (전 세계 다수)
   │          │          │
   ↕ 동기화   ↕         ↕
   같은 데이터 사본 보관 + 변경사항 전파
```

| 특성 | 설명 |
| --- | --- |
| **분산** | 서버가 매우 많음 (전 세계 단위) |
| **동기화** | A 변경 → B, C에 전파 |
| **보안 결합** | DNS 서버 간 강력하게 보호된 채널로 연결 |

### 동기화 시간

| 범위 | 소요 |
| --- | --- |
| 주요 캐시 (지역) | **1~2분** (안전하게 5분 이상) |
| 전 세계 완전 동기화 | **2~3일** |

> [!warning] 도메인 변경은 즉시 반영되지 않음
> 도메인 설정을 바꾸고 "왜 안 바뀌지?" → 정상.
> 적어도 5분 이상은 기다리고, 전 세계 단위는 며칠 걸린다.

---

## 3. DNS의 계층(Hierarchical) 트리 구조

### Top Level Domain (TLD)

```
                    .  (Root)
                    │
       ┌────────────┼────────────┬────────┐
       │            │            │        │
      .com         .net          .kr     .jp ...
       │                          │
   naver.com                    .co.kr
                                  │
                          nullnull.co.kr
                                  │
                               www.nullnull.co.kr
                               w2.nullnull.co.kr
                              ...
```

| 단계 | 예 |
| --- | --- |
| **Root** | `.` (생략됨, 최상단) |
| **Top Level Domain (gTLD)** | `.com`, `.net`, `.org`, ... |
| **Top Level Domain (ccTLD)** | `.kr`, `.jp`, ... (국가 코드) |
| **Second Level** | `naver.com`, `co.kr` |
| **Third Level** | `nullnull.co.kr` |
| **Host Name** | `www`, `w2`, `mail` |

### Root DNS — 전 세계 13대

> [!important] 인터넷 최상위 거점
> Root DNS는 **전 세계에 단 13대**만 존재.
> 대부분 미국 소재 (Verisign, NASA, 대학 등).

- 목록 확인: **IANA 홈페이지**의 Root server 목록
- 인터넷 종주국 = 미국이라는 사실을 단적으로 보여주는 구조

> [!example] 슈퍼 빌런 시나리오
> "전 세계 인터넷을 마비시키려면?"
> → 모든 라우터 ❌ (개수 너무 많음)
> → **Root DNS 13대 다운** → 사실상 전 세계 인터넷 정지 가능 (실제로 늘 공격 시도가 있어 방어가 매우 강력)

---

## 4. DNS 질의 흐름 — 한국은행 비유

> [!tip] 한국은행 비유
> Root DNS는 한국은행과 같다 — 개인을 직접 상대하지 않고, **다른 은행(다른 DNS)** 만 상대한다.

### 단계별 질의 (Recursive Lookup)

```
[Client]                                  [Root DNS]
   │                                          │
   │  "www.naver.com IP 알려줘"             │
   │ ─────────────────────────────────────→ │
   │                                          │
   │  ← ".com 전문가 3명 알려줄게"          │
   │                                          │
   ▼                                          ▼
   .com TLD DNS 중 하나 선택
        │
   "www.naver.com IP 알려줘"
        │
   ← "naver.com 담당 DNS 알려줄게"
        │
        ▼
   naver.com Authoritative DNS
        │
   "www인 컴퓨터 IP 뭐야?"
        │
   ← "X.X.X.X 야"
        │
        ▼
   최종 IP 획득
```

### 단계의 비효율

> 한 번 질의하려고 **3~4번 왕복** → 비효율.
> 이걸 해결하기 위해 등장한 것이 **Cache DNS**.

---

## 5. Cache DNS — 한 번에 답해주는 대행자

### 동작 원리

```
[Client PC: KT 사용자]
   │
   │  DHCP로 자동 설정 시 DNS 자동 지정
   │  → 168.126.63.1 (KT Cache DNS)
   │
   │  "www.naver.com 알려줘"
   ▼
[KT Cache DNS]
   │
   │  ① 캐시에 있나? → 있으면 즉시 응답
   │  ② 없으면 Recursive Lookup 수행 (Root → TLD → Authoritative)
   │  ③ 결과를 캐시 + Client에 응답
   ▼
[Client] IP 획득
```

### TTL — 응답에 같이 오는 유효기간

```
"X.X.X.X (TTL: 2시간)"
              │
              └─ 2시간 동안은 다시 안 물어봐도 됨
                 → 만료되면 다시 Recursive Lookup
```

### KT Cache DNS — `168.126.63.1`

| 항목 | 설명 |
| --- | --- |
| **위치** | 과거 KT 혜화동 지점 |
| **이중화** | 현재 다중화로 안정성 확보 |
| **역할** | 한국 인터넷 질의의 사실상 중심 |

> [!example] 또 다른 슈퍼 빌런 시나리오
> "한국만 인터넷 마비시키려면?"
> → KT DNS 다운 → 한국 인터넷 치명상.
> 실제로는 다중화로 보호되어 있어 쉽지 않음.

> [!note] 인터넷 강국?
> 한국은 Root DNS를 보유하지 못하고, 대부분 KT Cache DNS에 의존.
> "인터넷 강국"이라는 자칭에 약간의 그림자가 있는 부분.

---

## 6. 도메인은 웹만의 것이 아니다

### 도메인의 다양한 용도

| 서비스 | 프로토콜 | 도메인 사용 |
| --- | --- | --- |
| **Web** | HTTP/HTTPS | `www.naver.com` |
| **Email 송신** | SMTP | `cx8537@naver.com` |
| **Email 수신** | POP3, IMAP | 같음 |
| **FTP** | FTP | `ftp.example.com` |

> [!important] 도메인 ≠ 웹
> 도메인은 **이메일·FTP 등 다양한 서비스의 식별자**로도 쓰인다.
> URL의 일부가 아니라 **인터넷 서비스의 보편적 명명 체계**.

---

## 7. Authoritative DNS — 권한을 가진 서버

> [!important] Authoritative DNS
> 특정 도메인에 대한 **공식 권한(Authority)** 을 가진 DNS 서버.
> "이 도메인의 진짜 답은 나한테 물어봐"

### 응답의 권한 표시

```
[Cache DNS 응답]
   "www.naver.com → X.X.X.X 야"
   "근데 나는 권한이 있는 DB는 아니야 (Non-authoritative)"

[Authoritative DNS 응답]
   "www.naver.com → X.X.X.X 야"
   "이게 공식 답이다 (Authoritative)"
```

| 종류 | 권한 | 응답 라벨 |
| --- | --- | --- |
| **Cache DNS** | 위임받은 사본 | `Non-authoritative answer` |
| **Authoritative DNS** | 공식 권한 보유 | `Authoritative answer` |

---

## 8. DNS 레코드 종류

### A Record — 가장 보편적

```
www.example.com  →  7.7.7.7   (A 레코드)
```

- 도메인 → **IPv4 주소** 매핑
- 기본형, 가장 단순
- DNS 설정에서 가장 흔히 쓰는 레코드

### CNAME (Canonical Name) Record

```
www.example.com  →  d12345.cloudfront.net   (CNAME)
                              │
                              └─ A 레코드로 다시 IP 변환
```

- 도메인 → **다른 도메인** 매핑
- **CDN, 로드 밸런서, AWS Amplify, Vercel** 등에서 필수
- 실제 IP를 직접 노출하지 않아 **보안상 유리**

### 기타 레코드

| 레코드 | 용도 |
| --- | --- |
| **AAAA** | IPv6 주소 매핑 |
| **MX** | 메일 서버 지정 |
| **TXT** | 도메인 인증·SPF·DKIM 등 메타데이터 |
| **NS** | 네임서버 위임 |
| **SOA** | Zone의 권한 시작점 |

---

## 9. 도메인 구매와 AWS 연동 — 실전 팁

### 시나리오별 설정 횟수

| 구매처 | 운영처 | 설정 |
| --- | --- | --- |
| AWS Route 53 | AWS | **1회** (편함) |
| **Gabia** | AWS | **2회** — Gabia + AWS 양쪽 |
| Gabia | Gabia | 1회 |

> [!tip] 사업화 전 팁
> 사업/서비스 런칭 전에는 **반드시 도메인을 먼저 구매**해야 한다.
> AWS 운영 예정이라면 Route 53에서 사는 것이 가장 편하지만, Gabia 등에서 사도 한 번만 양쪽 설정해두면 끝.

### IP 노출을 피하는 패턴

```
[과거]  www.example.com  →  A 레코드  →  7.7.7.7 (직접 노출)
                                            │
                                            ↓
                                    DDoS·해킹 표적

[요즘]  www.example.com  →  CNAME  →  cdn.cloudfront.net
                                            │
                                            ↓
                                    A 레코드는 CDN이 관리
                                    원본 서버 IP 은닉
```

> [!important] 보안 베스트 프랙티스
> 가능하면 **A 레코드로 직접 IP 노출하지 말고**, CDN/Load Balancer를 통한 **CNAME 패턴** 사용.

---

## 10. nslookup으로 직접 확인하기

### 기본 사용

```bash
nslookup
> www.naver.com
```

### 출력 해석

```
서버:    kns.kornet.net          ← KT Cache DNS
Address: 168.126.63.1

권한 없는 응답:                    ← Non-authoritative (Cache 응답)
이름:    www.naver.com.nheos.com  ← CNAME으로 연결됨
Addresses:                          ← 실제 A 레코드 (4개)
   223.130.200.107
   223.130.200.104
   223.130.195.200
   223.130.195.95
Aliases: www.naver.com
```

### 해석 포인트

| 부분 | 의미 |
| --- | --- |
| `권한 없는 응답` | 이 응답은 Cache DNS에서 옴 |
| `nheos.com` | 네이버가 CNAME으로 사용하는 내부 도메인 |
| **4개의 IP** | **부하 분산** — 브라우저가 적절한 하나 선택 |

> [!tip] 부하 분산이 DNS 레벨에서 일어남
> 한 도메인에 여러 IP가 매핑 → 클라이언트가 하나 선택 → 자연스러운 트래픽 분산.
> 이것이 [[연결이라는 착각과 AWS ALB]]에서 다룬 ALB와 더불어 **DNS Round Robin** 이라 불리는 부하 분산 기법의 기본.

---

## 11. DNS 보안 — 절대 잊지 말 것

### DNS 설정 탈취 시나리오

```
[정상]
  www.example.com  →  A: 7.7.7.7  →  내 진짜 사이트
  
[Gabia 계정 탈취]
  악성코드 / 피싱으로 로그인 정보 탈취
       │
       ▼
  공격자가 A 레코드 변경
       │
       ▼
  www.example.com  →  A: 9.9.9.9  →  피싱 사이트 (외관 동일)
       │
       ▼
  💀 사용자는 진짜와 가짜 구별 못함
     계정·금융 정보 탈취
```

> [!danger] DNS 설정 탈취 = 치명상
> 한 줄의 A 레코드 변조만으로 사용자를 통째로 가짜 사이트로 보낼 수 있다.
> **2FA, 강력한 비밀번호, 관리자 IP 화이트리스트** 필수.

### AI 위임 시 추가 위험

> [!warning] Vibe Coding · Agentic AI 시대의 새 위협
> - AI 에이전트에게 도메인 관리 권한 위임 시 **AI의 모든 행동 = 사용자의 행동**
> - **Prompt Injection** — 외부 이메일·문서 등을 통해 AI에 악성 지시 주입 가능
> - 사례: 2026년 2월 현재까지 유사 피해 다수 보고됨

> [!important] 절대 규칙
> **DNS 설정같은 중요 작업은 AI 위임 금지. 반드시 수작업.**
> 공부하면 그렇게 어렵지 않다.

---

## 12. DNS 동작 한눈에 보기

```
[사용자]
   │
   │  www.example.com 입력
   ▼
[Browser]
   │
   │  DNS 질의
   ▼
[Cache DNS (KT 168.126.63.1)]
   │
   ├─ 캐시 HIT? ──→ 즉시 응답
   │
   └─ MISS → Recursive Lookup
       │
       ▼
   [Root DNS]   ".com 담당 알려줄게"
       │
       ▼
   [.com TLD DNS]   "example.com 담당 알려줄게"
       │
       ▼
   [Authoritative DNS]   "www는 7.7.7.7 야"
       │
       ▼
   [Cache DNS]  결과 캐시 + 응답 (TTL 포함)
       │
       ▼
   [Browser]  TCP/IP 연결 시도 (port 80/443)
```

---

## 13. 핵심 정리표

| 항목 | 내용 |
| --- | --- |
| **DNS 본질** | Domain → IP 변환 (분산형 DB) |
| **계층 구조** | Root → TLD → SLD → Host (Tree) |
| **Root DNS** | 전 세계 13대 (대부분 미국) |
| **Cache DNS** | Recursive Lookup 대행 + 캐시 (KT: 168.126.63.1) |
| **Authoritative DNS** | 도메인의 공식 권한 보유 |
| **A Record** | 도메인 → IPv4 |
| **CNAME** | 도메인 → 다른 도메인 (CDN/LB 패턴) |
| **TTL** | 응답 유효기간 |
| **동기화** | 5분 ~ 2-3일 |
| **부하 분산** | 한 도메인 → 여러 IP (DNS Round Robin) |
| **보안** | DNS 설정 탈취 = 치명상, AI 위임 금지 |
| **확인 명령** | `nslookup`, `dig` |

---

## 14. AWS 환경에서의 응용

| 항목 | 적용 |
| --- | --- |
| **Route 53** | AWS 관리형 DNS, 도메인 등록 + Hosted Zone |
| **Hosted Zone** | 도메인의 레코드 묶음 (Public / Private) |
| **A Record** | EC2/Public IP 직접 매핑 (지양) |
| **Alias Record** | AWS 전용, ALB/CloudFront/S3에 매핑 (CNAME 대용, 무료·유연) |
| **Routing Policy** | Simple / Weighted / Latency / Geo / Failover |
| **Health Check** | Route 53이 직접 헬스 체크 ([[연결이라는 착각과 AWS ALB]]) |
| **CloudFront + Route 53** | 글로벌 CDN + DNS 결합 |
| **ACM** | HTTPS 인증서 자동 갱신, Route 53 도메인과 연동 |

> [!tip] AWS Alias vs CNAME
> Alias는 AWS 전용 확장으로 **CNAME 대비**:
> - **무료** (CNAME은 쿼리당 과금)
> - **Zone Apex(`example.com`)** 에도 적용 가능 (CNAME은 불가)
> - AWS 리소스 변경 시 자동 추적

---

## 한 줄 요약

> **DNS는 Domain Name을 IP로 변환하는 분산형 데이터베이스**로, **Root DNS(13대) → TLD → Authoritative**의 계층 트리 구조를 가지며 **Cache DNS(KT 168.126.63.1 등)** 가 Recursive Lookup을 대행한다. **A 레코드**는 IP 직접 매핑, **CNAME**은 CDN·LB 패턴으로 IP 은닉에 활용되며, AWS에서는 **Route 53 + Alias Record**가 표준이다. DNS 설정 탈취는 치명상이므로 **AI 위임 금지·반드시 수작업**.
