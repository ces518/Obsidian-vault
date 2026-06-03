---
type: study-note
area: network
status: raw
source: AWS 로 배우는 네트워크 (책)
updated: 2026-06-03
tags:
  - network
  - security
  - HTTP
  - CORS
  - CSRF
created: 2026-05-10
---

## 개요

대표적인 웹 보안 취약점인 **CSRF**와, 이를 막기 위해 도입된 **SOP (Same-Origin Policy)**,
그리고 SOP를 합법적으로 우회하기 위한 **CORS** 메커니즘을 다룬다.
바이브 코딩이나 백엔드 개발 시 가장 많이 부딪히는 보안 이슈이며,
**브라우저 수준의 정책**이라는 점이 핵심이다.

---

## 1. 대표적인 웹 보안 취약점

| 취약점 | 약자 | 설명 |
|--------|------|------|
| **XSS** | Cross-Site Scripting | 게시물 등에 악성 JavaScript를 심어 실행 |
| **CSRF** | Cross-Site Request Forgery | 다른 사이트에서 사용자의 세션을 도용한 요청 위조 |
| **SQL Injection** | — | 입력값에 SQL 구문을 주입해 DB 조작 |

> [!note] 이번 노트의 초점
> 위 셋 중 **CSRF**에 집중. SOP/CORS는 CSRF를 막기 위해 탄생한 정책이다.

---

## 2. CSRF (Cross-Site Request Forgery) 공격

> [!important] CSRF의 본질
> **사용자가 이미 로그인된 상태**를 악용해, **다른 사이트에서 위조된 요청**을 사용자 명의로 보내는 공격.

### 공격 시나리오 — 은행 이체 사례

```
① 사용자가 은행 사이트에 로그인 → 세션 ID가 쿠키에 저장됨
                                  (세션 유지 상태)
       │
       ▼
② 사용자가 악성 사이트에 우연히 방문
       │
       │ 악성 사이트에는 hidden form이 숨겨져 있음:
       │   <form action="bank.com/transfer" method="POST" hidden>
       │     <input name="to"     value="해커계좌">
       │     <input name="amount" value="100000">
       │   </form>
       │   <script>document.forms[0].submit()</script>
       ▼
③ 페이지 로드 즉시 form 자동 제출
       │
       ▼
④ 브라우저가 bank.com에 요청 전송 — 쿠키(세션 ID)도 함께 전송
       │
       ▼
⑤ 은행 서버: "로그인된 사용자가 이체 요청했네" → 이체 실행
```

### 핵심 문제점

> [!caution] 왜 가능한가?
> 1. **`<form>` 태그는 Cross-Origin 요청을 제한하지 않음** — 어디로든 보낼 수 있음
> 2. **쿠키는 자동으로 동봉됨** — 세션 ID 포함
> 3. **form 제출은 "사용자 의도 행동"으로 간주됨** — 브라우저가 의심하지 않음

→ 다른 사이트에서 내가 로그인한 사이트로 임의 요청을 보낼 수 있다는 것 자체가 문제의 근본.

---

## 3. SOP (Same-Origin Policy)

> [!important] 정의
> **SOP** = 출처(Origin)가 다른 리소스/요청을 **브라우저가 차단**하는 정책

### Origin(출처)의 정의

> Origin = **프로토콜 + 도메인 + 포트** 의 조합

```
https://www.nullnull.co.kr:443/path
└─┬─┘  └────────┬─────────┘ └┬┘
프로토콜    도메인           포트
└──────────── Origin ────────┘
```

### Origin 비교 예시

기준 URL: `https://www.nullnull.co.kr/page`

| 비교 URL | 같은 출처? | 이유 |
|---------|-----------|------|
| `https://www.nullnull.co.kr/other` | ✅ | 프로토콜/도메인/포트 동일 |
| `https://www.nullnull.co.kr:443/page` | ✅ | 443은 HTTPS 기본 포트 |
| `http://www.nullnull.co.kr/page` | ❌ | 프로토콜 다름 (HTTPS→HTTP, 포트도 80) |
| `https://api.nullnull.co.kr/page` | ❌ | 호스트(서브도메인) 다름 |
| `https://www.nullnull.co.kr:8080/page` | ❌ | 포트 다름 |
| `https://other.com/page` | ❌ | 도메인 다름 |

### SOP가 막는 것

```
┌────────────────────────────────┐
│ 브라우저 (현재 출처: a.com)      │
│                                │
│  화면 = a.com에서 받은 HTML     │
│                                │
│  스크립트 fetch(b.com/api) ─❌─→│ ← SOP가 차단
│                                │
└────────────────────────────────┘
```

> [!tip] 출처가 섞이는 경우는 흔함
> 예: `a.com` HTML 안에 `b.com` 이미지 포함 — 이런 정상 사용 사례를 위해 일부 예외(`<img>`, `<script src>` 등)는 허용되지만, **JavaScript의 fetch/XHR는 엄격하게 차단**.

---

## 4. CORS (Cross-Origin Resource Sharing)

> [!important] 정의
> **CORS** = SOP의 **합법적인 우회 메커니즘**
> 서버가 "이 출처는 우리 서비스니까 허용해도 된다"고 명시적으로 알려주는 설정.

### 등장 배경 — 모던 웹의 구조 변화

```
[옛날]                       [모던 웹 (SPA)]
                          
www.example.com           www.example.com    ← 프론트
  │                              │
  └─ HTML + API 한 곳            │ JS fetch
                                 ▼
                         api.example.com    ← 별도 API 서버
                         
                         ⚠ 도메인은 같지만 호스트가 달라
                            SOP에는 "다른 출처"
```

→ 한 서비스인데도 SOP가 **차단**해버림 → 우회 수단 필요 = **CORS**

### 동작 방식

API 서버가 응답에 다음 헤더를 추가:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://www.nullnull.co.kr
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

→ 브라우저가 이 헤더를 보고 "아, 이 출처는 허용되는구나" 하며 응답을 받아들임.

> [!caution] CORS 설정 주의
> - **잘못 설정하면 SOP 자체가 무력화됨** (예: `*`로 모든 출처 허용)
> - "우리 서비스의 다른 호스트만" 정확히 허용해야 보안 의미 유지

---

## 5. SOP의 한계 — 이미 늦은 차단

> [!caution] SOP는 "응답 차단"이지 "요청 차단"이 아니다
> 일반적인 Cross-Origin 요청에서 SOP는 **서버가 이미 처리한 후 응답을 받지 못하게 하는 것**. 서버 측 변경은 이미 일어남.

### 동작 시점

```
브라우저 (악성사이트.com)        서버 (bank.com)
     │                                │
     │ ① 요청 (CORS 차단 안 됨)        │
     │ ────────────────────────────▶  │
     │                                │
     │                                │ ② 서버 코드 실행됨!
     │                                │   (이체 처리됨)
     │                                │
     │ ③ 응답                          │
     │ ◀──────────────────────────── │
     │                                │
     │ ④ 브라우저가 응답 폐기           │
     │   (스크립트 후속 코드 미실행)     │
     │                                │
     │ ⚠ 화면엔 안 보이지만 이체는 완료 │
```

→ **단순 GET이나 일반 form submit은 Preflight 없이 바로 서버에 도달** → 부작용 발생 가능.

---

## 6. Preflight Request (OPTIONS)

> [!important] Preflight
> 상태 변화를 일으킬 수 있는 요청(PUT, DELETE, JSON Body가 있는 POST 등) 전에,
> 브라우저가 **OPTIONS 메서드로 사전 확인**하는 메커니즘.

### 동작 흐름

```
브라우저                              API 서버
   │                                    │
   │ ① Preflight (OPTIONS)              │
   │   "POST /transfer 해도 돼?"         │
   │   "Origin: malicious.com"          │
   │ ──────────────────────────────────▶│
   │                                    │
   │ ② 응답                              │
   │   Access-Control-Allow-Origin:     │
   │     https://www.nullnull.co.kr     │
   │ ◀──────────────────────────────── │
   │                                    │
   │ ③ 브라우저: "Origin 안 맞네"         │
   │   → 본 요청 자체를 보내지 않음 ✅   │
   │                                    │
```

> [!tip] Preflight가 있으면
> **본 요청 자체가 차단됨** → 서버 측 코드도 실행되지 않음 → 진짜로 안전.
> 단순 form submit은 Preflight 대상이 아니므로 여전히 위험.

### Preflight 발생 조건 (간략)

- 메서드가 GET/HEAD/POST가 아닌 경우 (PUT, DELETE, PATCH 등)
- Content-Type이 `application/json` 등인 경우
- 커스텀 헤더(`Authorization`, `X-CSRF-Token` 등)가 있는 경우

---

## 7. SOP/CORS가 적용되지 않는 경우

> [!important] SOP는 **브라우저 수준의 정책**이다
> 브라우저가 없으면 SOP도 없다.

| 환경 | SOP 적용? | 이유 |
|------|---------|------|
| **브라우저** | ✅ | 정책의 주체 |
| **curl, Postman** | ❌ | 브라우저가 아님 |
| **서버 ↔ 서버 호출** | ❌ | 브라우저 미개입 |
| **모바일 네이티브 앱** | ❌ | 자체 HTTP 클라이언트 사용 |
| **IoT 장치** | ❌ | 브라우저 없음 |

> [!caution] 결론
> **SOP만으로 보안이 완성되지 않는다**. 서버 측에서 별도의 CSRF 방어 로직이 반드시 필요.

---

## 8. CSRF 실질적 대응 방법

> [!important] 실무 표준
> SOP/CORS만으로는 부족 → **서버 측 다층 방어**가 필수.

### ① CSRF 토큰 검증 방식

```
① 사용자 로그인 → 서버가 CSRF Token 발급 (세션에 저장)
                  
② HTML 내 <form>에 token 삽입:
   <input type="hidden" name="csrf_token" value="abc123...">

③ 사용자가 form 제출 → token 함께 전송

④ 서버: 세션의 token == 요청의 token ?
   - 일치 ✅ → 정상 요청
   - 불일치/없음 ❌ → 차단

→ 다른 사이트에서 위조된 form에는 token이 없으므로 차단됨
```

### ② SameSite 쿠키 속성

```
Set-Cookie: session_id=xxx; SameSite=Strict; Secure; HttpOnly
```

| 속성 | 의미 |
|------|------|
| **SameSite=Strict** | 같은 사이트 요청에만 쿠키 전송 (가장 안전) |
| **SameSite=Lax** | 일부 안전한 Cross-Site 허용 (기본값) |
| **Secure** | HTTPS에서만 전송 |
| **HttpOnly** | JavaScript에서 접근 불가 (XSS 방어) |

→ Cross-Site form submit 시 **세션 쿠키가 동봉되지 않음** → CSRF 무력화

### ③ Origin / Referer 헤더 교차 검증

서버가 요청의 `Origin` 또는 `Referer` 헤더를 검사:

```
요청 헤더:
  Origin: https://malicious.com    ← 우리 도메인 아님 → 차단
  Referer: https://malicious.com/...
```

### ④ 커스텀 헤더 추가

`X-Requested-With: XMLHttpRequest` 같은 커스텀 헤더를 요구.
- 일반 form은 커스텀 헤더 못 붙임 → 차단됨
- JS fetch만 가능 → CORS 검사 통과해야 함

### 종합 권장 조합

> [!tip] 실무 권장 패턴
> **CSRF Token + SameSite Cookie + Origin/Referer 검증** = 강력한 다층 방어.

---

## 9. 바이브 코딩 시 AI 활용 팁

> [!important] AI에게 "보안 신경 써줘"는 의미 없다
> 구체적인 대책 이름을 직접 명시해야 한다.

### ❌ 안 좋은 프롬프트

```
"보안 신경 써서 만들어줘"
"CSRF 안 생기게 해줘"
```

### ✅ 좋은 프롬프트

```
"CSRF 방지를 위해 다음을 모두 적용해줘:
 - 서버에서 CSRF 토큰 발급 후 form에 hidden으로 삽입
 - 쿠키에 SameSite=Strict, Secure, HttpOnly 속성 적용
 - Origin / Referer 헤더 교차 검증
 또한 XSS 방지를 위해 게시글 등 사용자 입력에 대해 HTML Sanitize 처리,
 SQL Injection 방지를 위해 Prepared Statement 사용해줘."
```

### 추천 워크플로

```
① AI에게 "이런 서비스에 적합한 보안 대책을 모두 나열해줘" 요청
② 추천받은 대책 검토
③ "위 대책이 모두 들어간 코드 개발 프롬프트를 만들어줘" 요청
④ 그 프롬프트로 Claude Code / Gemini CLI에 코드 작성 의뢰
```

---

## 핵심 요약

> [!summary]
> 1. **CSRF** = 사용자의 로그인 세션을 도용해 다른 사이트에서 위조된 요청을 보내는 공격
> 2. CSRF 공격은 **`<form>` + 자동 쿠키 전송**의 조합으로 가능
> 3. **SOP (Same-Origin Policy)** = 출처(프로토콜 + 도메인 + 포트)가 다른 요청을 **브라우저가** 차단
> 4. 모던 웹(SPA + 분리된 API 서버)에서는 SOP가 정상 통신을 막음 → **CORS**로 우회
> 5. **CORS** = 서버가 `Access-Control-Allow-Origin` 헤더로 명시적 허용
> 6. SOP의 한계: 본 요청은 서버에 도달 → **서버 코드 이미 실행됨** (응답만 폐기)
> 7. **Preflight (OPTIONS)** = 위험한 메서드 사전 확인 → 본 요청 자체를 차단 가능
> 8. SOP는 **브라우저 정책** — curl, Postman, 서버 간 통신, 모바일 네이티브, IoT에는 적용 ❌
> 9. CSRF 실질 방어: **CSRF Token + SameSite Cookie + Origin/Referer 검증** 다층 방어
> 10. AI 활용 시 **구체적 대책 이름**을 직접 명시해야 적용됨

---

관련: [[HTTP 와 REST API]] (Section 4 — OPTIONS 메서드)
