---
tags:
  - network
  - HTTP
  - DNS
  - IP
created: 2026-05-05
---

# URI · URL · DNS

## 개요

L2~L4까지의 인프라 위에 올라간 **웹(Web)** 영역의 기초 용어를 정리한다.
**URI / URL / Domain Name / Host Name**의 개념적 구분과, URL이 어떻게 **호스트 식별 → 파일 시스템 경로 → 리소스 식별**의 단계로 동작하는지를 다룬다.
바이브 코딩과 AWS 환경에서 마주하는 용어를 정확히 이해하기 위한 기반 문서.

---

## 1. 웹은 L2~L4 위에 올라간 애플리케이션 계층

```
┌─────────────────────────────────┐
│  L7 HTTP / Web                  │  ← 우리가 체감하는 인터넷 대부분
│  L5 SSL/TLS                     │
├─────────────────────────────────┤
│  L4 TCP / UDP                   │  ← OS가 알아서 처리
│  L3 IP                          │
│  L2 Ethernet                    │
└─────────────────────────────────┘
```

> [!important] 시야 전환
> - L2~L4: 운영체제·인프라가 알아서 해주는 영역
> - **웹**: 그 위에 올라간 애플리케이션 — 우리가 직접 다루는 영역
> - 따라서 **개발자는 웹 영역의 용어를 정확히 알아야** 한다

---

## 2. URI vs URL — 누가 누구를 포함하는가

> [!important] 포함 관계
> **URI ⊃ URL**
> URL은 URI의 한 종류 (위치 기반 식별).

| 약어 | Full Name | 의미 |
| --- | --- | --- |
| **URI** | **U**niform **R**esource **I**dentifier | 리소스의 **식별자** (큰 개념) |
| **URL** | **U**niform **R**esource **L**ocator | 리소스의 **위치** 지정자 |

### Resource(자원)란?

> "리소스"는 물질적 명사가 아니라 **개념적 용어**.

| 단계 | 리소스의 정체 |
| --- | --- |
| 1차 (가장 기본) | **파일** (`.html`, `.jpg`, `.pdf` 등) |
| 2차 (확장) | **데이터** (DB 레코드 등) |
| 3차 (확장) | **API** (함수 = 데이터 조작 인터페이스) |

```
Resource = File / Data / API ...
              │
              ↓ (위치 지정)
            URL
              │
              ↓ (식별)
            URI
```

---

## 3. URL의 구조 — 호스트 식별 + 파일 경로

### URL이 가리키는 대상

> "어떤 호스트의 파일 시스템 어느 위치에 있는 무엇"

```
[6.6.6.10번 클라이언트]            [7.7.7.7번 서버]
                                    │
   "나 A.html이 필요해"             │  D:\Data\A.html
        │                            │
        │  ── 다운로드 요청 ──→     │
        │                            │
        ▼                            ▼
   호스트 식별 + 파일 경로 식별 필요
```

### 단계별 식별

```
[1단계] 호스트 식별
  └─ Public IP 주소 (인터넷에서 유일)
  └─ 예: 7.7.7.7

[2단계] 파일 시스템 경로 식별
  └─ 해당 호스트의 OS 파일 시스템 상의 path
  └─ 예: D:\Data\A.html  →  /Data/A.html
       (윈도우 \ → 인터넷 표기는 / : UNIX 기원)
```

### 통신 프로토콜 명시

```
"이 리소스를 어떤 프로토콜로 가져올 건가?"
   └─ http://, https://, ftp://, ws:// ...
```

---

## 4. URL 문법 — 한 줄로 분해

```
https://www.nullnull.co.kr:443/path/to/courses?cmd=search&search_keyword=Test
└─┬─┘   └────────┬────────┘└┬┘└────┬────┘└─┬─┘└──────────┬───────────────┘
Scheme    Host (FQDN)     Port    Path  Endpoint        Query String
                                          (API name)    (parameter list)
```

### 구성 요소 의미

| 부분 | 역할 |
| --- | --- |
| **Scheme** (`https`) | 통신 프로토콜 (= L7) |
| **Host** (`www.nullnull.co.kr`) | 호스트 식별 (Domain Name + Host Name) |
| **Port** (`443`) | 프로세스 식별 (TCP/UDP 포트) |
| **Path** (`/path/to/courses`) | 리소스 위치 / API 엔드포인트 |
| **Query String** (`?cmd=search&...`) | API 매개변수(Argument) |
| **Fragment** (`#section`) | 페이지 내 앵커 (URL의 일부) |

> [!note] L7 스킴이라 L4는 신경 안 씀
> HTTP는 L7 프로토콜이므로, 그 아래 L4가 TCP인지 UDP인지는 URL 문법이 신경 쓰지 않는다.
> 다만 실무상 HTTP는 거의 **TCP 기반**.

---

## 5. Query String = 함수 호출

### REST API와 함수의 대응

```
courses(cmd="search", search_keyword="Test")
   │      │             │
   ▼      ▼             ▼
/courses?cmd=search&search_keyword=Test
└──┬───┘└─┬┘└──┬─┘└──────┬──────┘└──┬──┘
함수 이름  ?  매개변수=실인수  &     매개변수=실인수
```

| 기호 | 역할 |
| --- | --- |
| **`?`** | Query String 시작 |
| **`&`** | 매개변수 구분자 (여러 개 연결) |
| **`=`** | 매개변수 = 실인수 (Argument) |

> [!tip] API = Application Programming Interface
> 쉽게 말해 **함수**. URL은 그 함수의 호출 표현이라고 봐도 무방.

---

## 6. Domain Name vs Host Name — 헷갈리지 마라

### 분해 예시

```
www.nullnull.co.kr
└┬┘└────┬────────┘
 │      └─ Domain Name
 └──────── Host Name
```

| 부분 | 의미 | 예 |
| --- | --- | --- |
| **`.kr`** | Top Level Domain (TLD) | 국가 도메인 |
| **`.co`** | Second Level Domain | 사업자 카테고리 |
| **`nullnull`** | Third Level Domain | 사용자 등록 도메인 |
| **`nullnull.co.kr`** | **Domain Name** | 돈 주고 산 단위 |
| **`www`** | **Host Name** | 도메인에 속한 컴퓨터 이름 |

### 올바른 해석

> "**`www.nullnull.co.kr`** 는 **`nullnull.co.kr` 도메인에 속한 이름이 `www`인 컴퓨터**" 라는 뜻.

### 확장 예시

```
w2.www.nullnull.co.kr
└┬┘└┬┘└────┬─────────┘
 │  │      └─ Domain
 │  └──────── 상위 호스트
 └─────────── 하위 호스트

→ "nullnull.co.kr 도메인 안의 www 라는 그룹 안의 w2 라는 컴퓨터"
```

---

## 7. 왜 도메인 이름이 필요한가?

### 인간의 한계

```
질문) IP 주소 잘 외우세요?
답)   대부분 못 외움. 친구 전화번호도 못 외우는데...

→ 사람은 "숫자"보다 "문자열"이 익숙
→ 7.7.7.7 보다 naver.com 이 외우기 쉬움
```

### Domain Name의 등장

```
[IP 주소 기반]  7.7.7.7/test.html         ← 외우기 어려움
                   ↓
[Domain 기반]   www.naver.com/test.html  ← 사람 친화적
```

> [!important] DNS의 본질
> Domain Name → IP 주소로 **변환**하는 시스템.
> 자세한 동작 원리는 DNS 전용 문서에서 다룸.

---

## 8. URL 동작 흐름 — 한눈에 보기

```
[사용자 입력]   https://www.nullnull.co.kr/courses?cmd=search
       │
       ▼
[① DNS 변환]    www.nullnull.co.kr → 7.7.7.7
       │
       ▼
[② TCP 연결]    7.7.7.7 : 443  (3-way handshake)
       │
       ▼
[③ TLS 협상]    HTTPS 암호화 채널 수립
       │
       ▼
[④ HTTP 요청]   GET /courses?cmd=search HTTP/1.1
                Host: www.nullnull.co.kr
       │
       ▼
[⑤ 서버 처리]   파일 시스템에서 리소스 위치 확인 / API 실행
       │
       ▼
[⑥ HTTP 응답]   200 OK + 데이터(HTML/JSON 등)
       │
       ▼
[⑦ 렌더링]      브라우저가 결과 표시
```

---

## 9. 핵심 정리표

| 항목 | 내용 |
| --- | --- |
| **URI** | 리소스 식별자 (큰 개념) |
| **URL** | 리소스 위치 지정자 (URI의 부분 집합) |
| **Resource** | File / Data / API |
| **Scheme** | 통신 프로토콜 (http, https 등) |
| **Domain Name** | 사용자가 등록한 단위 (nullnull.co.kr) |
| **Host Name** | 도메인 안의 컴퓨터 (www) |
| **Port** | 프로세스 식별 (TCP/UDP) |
| **Path** | 파일 시스템 경로 / API 엔드포인트 |
| **Query String** | API 매개변수 (`?key=value&...`) |
| **DNS** | Domain Name → IP 변환 |

---

## 10. AWS 환경에서의 응용

| 항목 | 적용 |
| --- | --- |
| **Route 53** | AWS의 관리형 DNS (Domain Name → IP/리소스 매핑) |
| **Hosted Zone** | 한 도메인의 레코드 묶음 |
| **CNAME / Alias** | 호스트 이름을 다른 호스트나 AWS 리소스로 매핑 |
| **ACM (Certificate Manager)** | HTTPS 인증서 발급·연결 |
| **CloudFront** | 도메인 → 엣지 노드 → 원본(Origin) 흐름 |
| **API Gateway** | URL Path 기반 라우팅, Query String 파싱 |
| **S3 Static Website** | URL = `https://<bucket>.s3.<region>.amazonaws.com/path/file.html` |
| **ELB / ALB** | Listener Rule이 Path / Host Header로 라우팅 |

> [!tip] AWS의 모든 외부 노출은 URL로 시작
> Route 53의 도메인 → ALB/CloudFront → EC2/Lambda 흐름이 모두 **URL 분해**로 설명된다.

---

## 한 줄 요약

> **URI ⊃ URL**이며, URL은 **Scheme + Host(Domain+HostName) + Port + Path + Query**로 분해된다. Domain Name은 사용자가 산 단위(`nullnull.co.kr`), Host Name은 그 안의 컴퓨터(`www`)이며, 사람이 IP 주소를 못 외우기 때문에 **DNS가 도메인 → IP 변환**을 담당한다. AWS에서는 **Route 53 → ALB/CloudFront → EC2/Lambda** 라우팅 흐름이 모두 이 URL 분해 위에 동작한다.
