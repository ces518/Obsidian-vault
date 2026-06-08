---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-05-10
updated: 2026-06-03
tags:
  - network
  - JWT
  - authentication
  - security
  - HTTP
---

## 개요

**모던 웹 서비스 구조(SPA + 마이크로서비스)**의 특징과,
이런 환경에서 사용자 인증/세션 관리를 위해 사용되는 **JWT(JSON Web Token)** 를 다룬다.
HTTP는 본래 Stateless인데, 이 위에 **상태(State)** 를 논리적으로 구현하는 핵심 수단이 JWT이다.

---

## 1. 모던 웹 = SPA (Single Page Application)

### 과거 (Monolithic) vs 현재 (Modern)

| 구분 | 과거 (모놀리식) | 현재 (모던) |
|------|---------------|------------|
| **프론트** | 단순 웹 서버 (HTML 송수신) | **응용 프로그램 화면** (SPA) |
| **렌더링** | 서버에서 HTML 생성 | 클라이언트에서 JS로 렌더링 (또는 SSR) |
| **로직 위치** | 백엔드(WAS)에 집중 | **프론트 + 백엔드 양쪽 분산** |
| **페이지 전환** | 새 URL로 이동 → 전체 리로드 | 같은 URL 내에서 부분 갱신 |
| **백엔드 역할** | HTML 생성 + 비즈니스 로직 | **REST API 제공자** |

### 대표 SPA 프레임워크

- **React**, **Vue**, **Angular**, AngularJS
- **Next.js** (SSR 지원 — 서버에서 HTML 미리 렌더링)

### 프론트가 "응용 프로그램화"

```
[과거]                          [모던 (SPA)]
┌──────────┐                    ┌──────────────┐
│ Browser  │                    │   Browser    │
│          │                    │              │
│ HTML 표시 │                    │ 엑셀 같은     │
│ (단순 뷰어)│                    │ 응용 프로그램  │
└────┬─────┘                    │ 화면          │
     │                          └──────┬───────┘
     │ Page request                    │
     ▼                                 │ API 호출
┌──────────┐                          ▼
│ Web Svr  │ ─────▶ HTML 통째 전송   ┌────────────┐
└──────────┘                         │ API Server │
                                     └────────────┘
```

> [!important] 핵심 변화
> 모던 웹의 프론트는 단순 송수신이 아니라 **V8 엔진을 포함하는 런타임(Node.js)** 이 동작하는 **응용 프로그램 환경**이다.

---

## 2. 마이크로서비스 아키텍처 (MSA)

### 구조

```
            사용자 (Browser SPA)
                  │
                  ▼
        ┌─────────────────────┐
        │   Frontend Server    │
        │   (Next.js 등)        │
        └──────────┬──────────┘
                   │
        ┌──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │ API #1 │ │ API #2 │ │ API #3 │ │ API #N │
   │ (회원)  │ │ (결제)  │ │ (강의)  │ │ (...)  │
   └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
   ┌────┐    ┌────┐     ┌────┐     ┌────┐
   │DB#1│    │DB#2│     │DB#3│     │DB#N│
   └────┘    └────┘     └────┘     └────┘
```

> [!note] 브라우저가 직접 API 서버 호출
> 사용자 화면은 하나지만, 내부적으로는 **여러 API 서버를 직접 들락거림**.
> 프론트를 거치지 않고 **브라우저 → API 서버 직접 호출**하는 경우가 훨씬 많음.

### 발생하는 문제 — Session ID 공유

```
[기존 세션 방식]
사용자 ──로그인──▶ Server 1
                  │ 세션 ID 발급 + 쿠키 저장
                  ▼
         (Server 1만 세션 알고 있음)

⚠ Server 2, 3, ..., N은 어떻게 이 사용자가 인증됐는지 알지?
```

→ 서버끼리 세션 정보를 공유해야 하는 부담 → 해결책 = **토큰 기반 인증** = JWT

---

## 3. JWT (JSON Web Token) 개념

> [!important] 정의
> **JWT** = 사용자가 인증된 후, **서명된(Signed) 토큰**을 발급해 주고
> 클라이언트가 매 요청마다 이 토큰을 가지고 다니며 인증을 받는 방식.

### 토큰의 본질

> "토큰" = **아주 긴 문자열** ≈ **아주 큰 숫자**
> → 랜덤 예측 가능성 ≈ 0 (수학적으로 사실상 불가능)

### 핵심 동작 원리 — "자유이용권"

```
1번 서버에서 한 번만 로그인 → 토큰 발급
                ↓
   토큰 들고 2번, 3번, N번 서버 어디든 출입 가능
   (각 서버는 토큰의 서명만 검증하면 됨)
```

> [!tip] 핵심
> 서버는 **세션을 저장하지 않는다(Stateless)**. 토큰의 **서명**만 검증한다.

---

## 4. JWT 구조 — Header.Payload.Signature

### 형태

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiIxMjM0..." . SflKxwRJSMeKKF...
└──────────── Header ────────────────┘ └─── Payload ────┘ └── Signature ──┘

         (점 `.` 두 개로 구분된 3 파트 구조)
```

### 각 파트의 역할

| 파트 | 내용 | 인코딩 |
|------|------|--------|
| **Header** | 알고리즘(`HS256`), 타입(`JWT`) | Base64URL |
| **Payload** | Claim들 — sub(주체), name, role, iat, exp 등 | Base64URL |
| **Signature** | Header + Payload + **Secret Key**로 만든 해시 | — |

### 디코딩 예시 (jwt.io 활용)

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "sub": "1234567890",
  "name": "홍길동",
  "role": "admin",
  "iat": 1715300000,
  "exp": 1715303600
}
```

> [!caution] Base64URL은 **암호화가 아니다**
> 누구나 디코딩 가능 → **Payload에 민감한 개인정보(주민번호, 비밀번호, 주소 등) 담지 말 것**.

### Base64URL 인코딩

- **Base64** + URL에서 사용 불가 문자 변환
- `+` → `-`, `/` → `_`, `=` 패딩 제거

---

## 5. JWT 검증 원리 — 서명(Signature)

### Hash와 서명의 차이

```
[일반 Hash]
원본 A ──Hash──▶ Signature_A
원본 B ──Hash──▶ Signature_B

→ A == B 인가? Signature 비교만으로 검증 가능
   (단 1비트만 달라도 Signature가 완전히 달라짐)

[서명(HMAC)]
원본 + Secret Key ──Hash──▶ Signature

→ Secret Key를 모르면 같은 Signature를 만들 수 없음
```

### JWT 검증 흐름

```
서버가 받은 토큰: Header.Payload.Signature
                          │
                          │ Header + Payload + Secret Key
                          ▼
                   서버가 직접 HMAC SHA-256 계산
                          │
                          ▼
                   계산한 결과 == 토큰의 Signature?
                          │
              ┌───────────┴───────────┐
              ▼                        ▼
         일치 ✅                    불일치 ❌
         API 처리                   요청 거부
```

> [!important] 검증의 핵심
> **Secret Key를 가진 자만이 같은 Signature를 만들 수 있다.**
> 검증 자체는 단순 해시 연산 → **매우 빠름**.

---

## 6. Secret Key 관리

### 대칭키 = PKI의 Private Key가 ❌ 아님

> [!caution] 헷갈리지 말 것
> JWT의 "Secret Key"는 **대칭키(Symmetric Key)** 다.
> PKI(공개키 기반 구조)의 **Private Key와는 다른 개념**.
> "비밀스럽게 잘 보관한다"는 의미에서 "비밀키"라 부를 뿐.

### 다중 서버 환경의 키 관리 문제

```
서버 20대가 같은 서비스를 처리
        │
        ▼
모든 서버가 동일한 Secret Key를 공유해야 함
        │
        ▼
주기적 교체 필요 (예: 1개월마다)
        │
        ▼
20대 모두 셧다운 → 키 교체 → 재시작
        │
        ▼
컨테이너 환경에서 자동 스케일링 시 더 복잡
```

### 해결책

#### ① AWS Secrets Manager

> [!tip] AWS 환경에서의 자동화
> - 키 정보를 중앙에서 관리
> - 변경 시 한 번에 모든 서버에 배포
> - 주기별 자동 교체 가능

#### ② RSA (비대칭키, PKI) 방식

```
대용량 / 초대형 서비스
      ↓
RSA 비대칭키 방식 사용
  - Private Key: 토큰 발급 서버만 보유 (서명 생성)
  - Public Key:  검증 서버들에 배포 (서명 검증만)
      ↓
검증 서버가 해킹돼도 토큰 위조 불가능 ✅
```

> [!important] 보안 강도 차이
> 대칭키: 20대 중 **1대만 해킹돼도 나머지 19대 전부 사칭 가능**
> 비대칭키: Public Key는 위조 불가능 → **노출돼도 안전**

---

## 7. JWT 인증 절차

### 로그인 (Access Token 발급)

```
Browser                              Server
  │                                    │
  │ ① POST /login                      │
  │    { email, password }             │
  │ ────────────HTTPS───────────────▶  │
  │                                    │ ② 인증 확인
  │                                    │ ③ JWT 생성 (Header + Payload + Sign)
  │ ④ Access Token 응답                │
  │ ◀────────────────────────────────  │
  │                                    │
  │ ⑤ 토큰을 메모리/Storage에 저장       │
```

### API 호출 시 — Authorization 헤더

```
GET /api/courses HTTP/1.1
Host: api.nullnull.co.kr
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
                       └── JWT Token ────┘
```

> [!tip] Authorization 헤더의 장점
> - 일반 `<form>` 태그로는 Authorization 헤더를 보낼 수 없음
> - JS fetch만 가능 → **CSRF 자동 방어** (form 기반 공격 차단)
> - 쿠키와 무관 → **Cross-Domain 환경에 적합**

---

## 8. JWT의 장점

| 장점 | 설명 |
|------|------|
| **Stateless** | 서버가 세션 저장 불필요 → **수평 확장(Scale-Out) 용이** |
| **Cross-Domain** | 쿠키와 무관 → 도메인 달라도 작동 |
| **빠른 검증** | 단순 해시 연산 → 고성능 |
| **IoT 친화적** | 브라우저/쿠키 없는 환경에서도 사용 가능 |
| **권한 정보 내장** | Payload에 role, permission 등 커스텀 클레임 포함 가능 |
| **CSRF 자연 방어** | Authorization 헤더는 form으로 못 보냄 |
| **컨테이너/K8s 적합** | 자동 스케일링 환경에서 세션 동기화 부담 ❌ |

---

## 9. JWT의 단점

### ① 강제 무효화 어려움

> [!caution] 토큰은 한번 발급되면 만료까지 유효
> 사용자를 강제 로그아웃시켜도, **클라이언트가 가진 토큰은 다른 서버에서 여전히 통과**.

#### 해결책 — 짧은 만료 + Refresh Token

```
Access Token  : 짧은 수명 (예: 15분)  → 도난당해도 영향 최소화
Refresh Token : 긴 수명 (예: 7일)    → Access Token 재발급용

흐름:
  ① 로그인 → Access(15분) + Refresh(7일) 발급
  ② API 호출 → Access Token 사용
  ③ 15분 후 만료 → Refresh Token으로 새 Access Token 받음
  ④ Worst Case: 만료된 토큰 사용 가능 시간 = 최대 15분
```

### ② Payload 평문 노출

```
Base64URL은 인코딩일 뿐 → 누구나 디코딩 가능

❌ Payload에 담으면 안 되는 정보:
   - 비밀번호
   - 주민번호
   - 신용카드 번호
   - 전화번호, 주소 (개인정보)
   
✅ 안전하게 담을 수 있는 정보:
   - 사용자 ID (sub)
   - 닉네임, 권한(role)
   - 발급/만료 시간 (iat, exp)
```

### ③ 토큰 크기 증가

- Payload가 늘어나면 토큰 크기도 증가 → 매 요청마다 전송되므로 **네트워크 효율 감소**
- 쿠키도 동일한 단점이라 큰 문제는 아니지만 인지 필요

### ④ 즉시 무효화 필요 시 — 별도 서버 운영

```
대규모 서비스에서 즉시 로그아웃이 중요한 경우:
  ↓
Token Blacklist 또는 활성 세션 관리용 별도 서버
  ↓
주로 Redis 같은 In-Memory DB 활용
```

---

## 10. 다른 세션 관리 방식 (참고)

| 방식 | 설명 |
|------|------|
| **JWT** | 토큰 기반 Stateless 인증 (이번 노트의 주제) |
| **Redis 기반 세션** | 외부 메모리 서버에 세션 저장, 모든 서버가 공유 |
| **Cookie 기반 토큰** | 쿠키에 세션 정보 저장 (전통 방식) |

---

## 11. 실무 — JWT_SECRET 관리

### EC2 환경 변수 예시

```bash
# 잘못된 예 (절대 사용 금지!)
JWT_SECRET=your-256bit-secret-key-here-min-32-chars

# 올바른 예 (랜덤 생성)
JWT_SECRET=k8N2vP9xQ4mR7tY3wJ6hL1bV5cD0fG8aZ2sE4uI...
```

### 키 생성 추천

- 온라인 랜덤 키 생성기 활용 (256bit 이상 권장)
- AWS 환경: **AWS Secrets Manager** 또는 **Parameter Store** 사용
- **절대 GitHub에 푸시하지 말 것** (`.env`, `.gitignore` 처리)

---

## 핵심 요약

> [!summary]
> 1. **모던 웹** = SPA + 마이크로서비스 — 프론트가 응용 프로그램화, 백엔드는 REST API 제공자
> 2. 다중 서버 환경에서 **세션 ID 공유 문제**를 해결하기 위해 **JWT** 등장
> 3. **JWT** = `Header.Payload.Signature` 의 3 파트 구조, 점(`.`)으로 구분, **Base64URL** 인코딩
> 4. 검증 = `Header + Payload + Secret Key`로 **HMAC SHA-256** 계산 → Signature 비교
> 5. **Secret Key는 대칭키** (PKI Private Key ❌) — 모든 서버가 사전 공유 필요
> 6. 다중 서버 키 관리 부담 → **AWS Secrets Manager** 또는 **RSA 비대칭키**(PKI) 방식 사용
> 7. **Authorization 헤더**로 토큰 전송 → form으로는 못 보냄 → **CSRF 자연 방어**
> 8. 장점: **Stateless**, Cross-Domain, 빠른 검증, IoT 친화적, 권한 정보 내장
> 9. 단점: **즉시 무효화 어려움** → Access(15분) + **Refresh Token**(7일) 조합으로 해결
> 10. **Payload는 평문 노출**되므로 민감 개인정보 절대 담지 말 것 — Base64는 암호화 ❌

---

관련: [[HTTP 와 REST API]] · [[웹 보안과 SOP CORS]]
