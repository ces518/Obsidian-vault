---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-05-09
updated: 2026-06-03
tags:
  - network
  - DNS
  - AWS
  - Route53
---
# DNS 캐싱과 Route53

## 개요

**DNS 캐싱이 일어나는 여러 지점**과 **AWS Route 53**을 활용한 도메인 설정을 다룬다.
DNS 응답은 IP 주소를 알아내기 위한 것이지만, 매번 질의하면 서버 부담이 크기 때문에
브라우저 → OS → hosts 파일 → 공유기 → 캐시 DNS 등 **여러 계층에서 캐싱**한다.
또한 실제 AWS 서비스(Amplify + CloudFront + ALB + EC2 + S3) 배포 시
Route 53에 어떤 레코드를 어떻게 설정해야 하는지 정리한다.

---

## 1. DNS 캐싱 포인트 — 5단계

> [!important] 캐싱 위치
> DNS 응답은 다음 **5개 지점**에서 단계적으로 캐싱된다.

```
사용자 요청
    │
    ▼
① 브라우저 캐시          (예: 크롬 DNS Cache)
    │
    ▼
② OS 캐시               (Windows DNS Resolver)
    │
    ▼
③ hosts 파일            (정적 매핑 파일)
    │
    ▼
④ 공유기 (DNS Forwarder) ← LAN 게이트웨이
    │
    ▼
⑤ 캐시 DNS 서버          (KT, 8.8.8.8 등)
    │
    ▼
   (Authoritative DNS 까지)
```

| 단계 | 위치 | 캐시 확인/제거 명령 (Windows) |
|------|------|----------------------------|
| ① | 브라우저 (Chrome) | `chrome://net-internals/#dns` |
| ② | OS | `ipconfig /displaydns` / `ipconfig /flushdns` |
| ③ | hosts 파일 | `C:\Windows\System32\drivers\etc\hosts` |
| ④ | 공유기 (Gateway) | 공유기 관리자 페이지 |
| ⑤ | ISP 캐시 DNS / Public DNS | — |

---

## 2. 공유기와 DNS Forwarding

### 공유기에 포함된 기능들

| 기능 | 설명 |
|------|------|
| **DHCP 서버** | 사설 IP 자동 할당 (`192.168.x.x`) |
| **NAT** | Private IP ↔ Public IP 변환 |
| **DNS Forwarding** | DNS 질의를 외부 DNS로 대신 전달 + 응답 캐싱 |
| **L2 스위치 기능** | 내부 단말 간 통신 |
| **무선 AP** | Wi-Fi 제공 |

### DNS Forwarding 동작

```
PC (192.168.0.10)
  │
  │ ① 네이버 IP 알려줘 → DNS = 192.168.0.1 (= 게이트웨이)
  ▼
공유기 (192.168.0.1)
  │
  │ ② 자기 캐시 확인 → 없으면
  │ ③ 외부 DNS(예: 168.126.63.1)에 질의
  ▼
ISP의 캐시 DNS 서버
  │
  │ ④ 응답 ← 캐싱
  ▼
공유기 캐싱 ← 그 다음 같은 질의는 공유기가 직접 응답
```

> [!note] PC의 DNS 설정이 게이트웨이로 되어 있는 이유
> ipconfig 결과를 보면 DNS가 `192.168.0.1`로 되어 있는데, 이는 공유기가 **DNS 자체는 아니지만** DNS Forwarder 역할을 하기 때문.

---

## 3. 보안 위협 — 캐싱 지점이 곧 공격 지점

> [!caution] 공유기가 해킹되면?
> 모든 내부 단말의 DNS 응답을 공격자가 조작 가능 → **모든 트래픽이 위조된 사이트로 유도**될 수 있음.
> → **공유기 펌웨어 업데이트 필수**.

### hosts 파일 공격

```
악성코드가 PC의 hosts 파일에:
  www.naver.com    악성서버IP
이렇게 추가하면 → 네이버 접속 시 악성 사이트로 이동
```

> [!note] Windows의 대응
> 너무 많은 보안 사고로 인해 MS가 hosts 파일에 **OS 레벨 락**을 걸어버림.
> 관리자 권한 가져도 쉽게 못 바꾸도록 보호.

### hosts 파일의 정상 용도

- `localhost` ↔ `127.0.0.1` 매핑이 hosts 파일에 정의되어 있음
- 로컬 개발 환경에서 `http://localhost:3000`로 접속 가능한 근거

### 보안 권장사항

> [!tip] 강사 권장
> DHCP로 IP를 받더라도 **DNS만큼은 직접 지정**하는 게 안전.
> 예: **`8.8.8.8`** (Google Public DNS)
> → 공유기가 해킹돼도 DNS 조작 영향 회피 가능.

---

## 4. AWS Route 53이란?

> [!important] Route 53
> AWS의 **DNS 서비스**. 도메인 등록부터 라우팅 정책, 헬스체크까지 통합 제공.

### 역할

- 도메인 구매/이전
- DNS 레코드 관리 (A, CNAME, NS, SOA, MX, TXT 등)
- AWS 다른 서비스(Amplify, CloudFront, ALB, S3 등)와 자동 연동

---

## 5. 주요 DNS 레코드 종류

| 레코드 | 의미 | 예시 |
|--------|------|------|
| **A** | Address — 도메인 → **IPv4 주소** 매핑 | `nullnull.co.kr → 13.124.x.x` |
| **AAAA** | 도메인 → IPv6 주소 매핑 | — |
| **CNAME** | Canonical Name — 도메인 → **다른 도메인** alias | `www.x.com → x.cloudfront.net` |
| **NS** | Name Server — 이 도메인을 관리하는 **네임서버 목록** | Route 53 NS 4개 |
| **SOA** | Start of Authority — 도메인의 **권한 정보** (관리자, 갱신 주기 등) | `ns-xxx.awsdns.com` |
| **MX** | Mail Exchange — 메일 서버 지정 | — |
| **TXT** | 텍스트 — 도메인 검증, SPF 등 | ACM 인증 등 |

---

## 6. 실제 Route 53 설정 사례 (강의 예시)

### 서비스 아키텍처

```
                    사용자
                      │
                      ▼ www.nullnull.co.kr
                ┌─────────────┐
                │  Route 53   │
                └──────┬──────┘
                       │
      ┌────────────────┼────────────────┐
      │                │                │
      ▼ A (Alias)      ▼ A (Alias)     ▼ A (Alias)
┌───────────┐   ┌────────────┐   ┌──────────┐
│ Amplify   │   │ CloudFront │   │CloudFront│
│ (프론트)   │   │ (API 보호) │   │ (S3 CDN) │
│ +CloudFront│   │            │   │          │
└─────┬─────┘   └──────┬─────┘   └─────┬────┘
      │                │                │
      ▼                ▼                ▼
   Next.js 빌드      ALB (LB)         S3 (이미지/
   GitHub CI/CD     │                  비디오)
                    ▼
                  EC2 (API 서버)
```

### 레코드별 의미

| 레코드 | 도메인 | 값 | 역할 |
|--------|-------|-----|------|
| **A** (Alias) | `www.nullnull.co.kr` | `d2h9...amplifyapp.com` | 프론트엔드 → Amplify |
| **A** (Alias) | `nullnull.co.kr` | (동일 Amplify 주소) | apex 도메인 |
| **A** (Alias) | `api.nullnull.co.kr` | `xxx.cloudfront.net` | API → CloudFront → ALB → EC2 |
| **CNAME** | `_xxx.nullnull.co.kr` | ACM Validation 값 | **SSL 인증서 검증용** |
| **NS** | `nullnull.co.kr` | AWS 네임서버 4개 | 이 도메인은 Route 53이 관리 |
| **SOA** | `nullnull.co.kr` | `ns-xxx.awsdns.com ...` | 도메인 권한 정보 |

> [!note] A 레코드인데 IP가 아니라 도메인이 보이는 이유
> AWS의 **Alias Record** 기능 — 내부적으로 IP를 동적으로 매핑하되, **AWS 리소스(CloudFront, ALB, S3 등)를 직접 가리킬 수 있게** 한 확장 기능.
> 일반 DNS의 A 레코드는 IP만 가리키지만, Alias는 AWS 서비스 endpoint를 가리킴.

---

## 7. 핵심 구성 요소 정리

### Amplify

- **프론트엔드 배포 서비스** (Next.js, React 등)
- GitHub push → CI/CD 자동 빌드 → 배포
- 내부적으로 **CloudFront(CDN)** 사용 → 전 세계 빠른 응답
- 발급되는 URL: `xxx.amplifyapp.com`

### CloudFront

- AWS의 **CDN(Content Delivery Network)** 서비스
- **Edge Location** 캐싱으로 응답 속도 향상
- **방화벽(WAF) 부착 가능** → 직접 EC2 노출 위험 차단
- API Gateway 역할도 수행 가능

### ALB (Application Load Balancer)

- **L7 로드밸런서** (HTTP/HTTPS 기반)
- CloudFront ↔ EC2 사이의 중계
- 헬스체크 + 트래픽 분산

### EC2 (API 서버)

- 직접 노출 ❌ → CloudFront + ALB **뒤에 숨김**
- 보안상 매우 중요

### S3 (Simple Storage Service)

- **이미지/비디오 등 정적 자원** 저장
- DB에 저장하면 I/O 과부하 → S3 + CloudFront로 분리
- 비디오 스트리밍 + 접근 제어도 가능

---

## 8. CNAME = SSL 인증서 검증 (ACM)

> [!important] ACM Validation의 의미
> AWS Certificate Manager가 SSL 인증서를 발급하기 전,
> "정말 이 도메인이 네 것이냐?"를 검증하기 위해 **CNAME 값을 도메인에 추가하라**고 요구한다.

### 동작 흐름

```
① ACM: "_validation.x.com → _xxx.acm-validations.aws"
       이 CNAME을 추가해주세요

② 사용자: Route 53에 해당 CNAME 등록

③ ACM: 등록된 CNAME 확인 → 도메인 소유 검증 완료

④ SSL 인증서 발급
```

→ Route 53에 `Validation`이라는 단어가 들어간 CNAME들이 잔뜩 보이는 이유.

---

## 9. NS / SOA 레코드의 의미

### NS (Name Server)

- 이 도메인을 **누가 관리하는가**를 나타냄
- 도메인 등록업체(가비아 등)에서 NS 레코드를 **AWS Route 53의 4개 네임서버**로 변경해야 Route 53이 작동
- "이 도메인은 Route 53에 물어봐라" = NS 설정의 본질

### SOA (Start of Authority)

- 도메인의 **권한(Authority) 정보**
- 주 네임서버, 관리자 이메일, 갱신 주기, TTL 등 메타데이터 포함
- AWS에서는 자동으로 Amazon 값으로 세팅됨

---

## 10. 도메인 + 서비스 배포 체크리스트

> [!tip] 바이브 코딩으로 서비스 배포 시
> 1. 도메인 구매 (가비아, Route 53 직접 등)
> 2. NS 레코드를 Route 53 네임서버로 변경
> 3. ACM에서 SSL 인증서 발급 → CNAME 검증
> 4. Amplify에 도메인 연결 → A(Alias) 레코드 자동 생성
> 5. API 서버는 **CloudFront + ALB 뒤로 숨김** → EC2 직접 노출 ❌
> 6. 이미지/비디오는 S3 + CloudFront 조합

---

## 핵심 요약

> [!summary]
> 1. **DNS 캐싱은 5단계**: 브라우저 → OS → hosts 파일 → 공유기 → 캐시 DNS 서버
> 2. 공유기는 **DNS Forwarder** 역할 + 응답 캐싱 → 해킹되면 모든 단말 트래픽이 위협
> 3. **hosts 파일 조작**으로 단일 PC에서도 DNS 위조 가능 → Windows는 OS 레벨 락 적용
> 4. 보안 권장: DHCP 사용해도 **DNS는 8.8.8.8 같은 Public DNS로 직접 지정**
> 5. **Route 53** = AWS의 DNS 서비스 — 도메인 등록 + 레코드 관리 + AWS 서비스 자동 연동
> 6. 핵심 레코드: **A**(IP 매핑) / **CNAME**(도메인 alias, ACM 검증) / **NS**(네임서버) / **SOA**(권한 정보)
> 7. AWS Alias 레코드 = A 레코드인데 **IP 대신 AWS 리소스(CloudFront, ALB 등)를 직접 가리킴**
> 8. 일반 서비스 구조: **Route 53 → Amplify(프론트) + CloudFront(API/S3 보호) + ALB → EC2**
> 9. EC2는 **직접 노출하지 말고** CloudFront + ALB 뒤에 숨길 것 (보안)
> 10. CNAME에 `Validation`이 보이면 = **ACM SSL 인증서 검증용**
