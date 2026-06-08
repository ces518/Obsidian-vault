---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-05-09
updated: 2026-06-03
tags:
  - network
  - HTTP
  - REST
  - API
---
# HTTP 와 REST API

## 개요

**웹의 탄생 배경**부터 **HTTP 프로토콜의 구조**, 그리고 오늘날 표준이 된 **REST API**까지 다룬다.
HTTP는 단순한 문서 전달 프로토콜로 시작했지만, 리소스(파일/데이터)에 대한
CRUD 행위를 표현하는 프로토콜로 진화했다.

---

## 1. 웹의 탄생 — HTML + HTTP

### 배경 — CERN과 Tim Berners-Lee

- 1980년대 후반, CERN(유럽입자물리연구소)의 컨설턴트 **Tim Berners-Lee**가 고안
- 연구자들이 논문을 읽다가 참고문헌을 따라 또 다른 논문을 찾는 **반복 작업의 비효율**을 해결하려는 시도
- TCP/IP가 갓 등장한 시기 → **클릭 한 번으로 문서가 열리는 시스템**을 구상
- 2017년 **튜링상** 수상 (IT 분야의 노벨상)

### 두 개의 핵심 기술

| 기술 | 역할 |
|------|------|
| **HTML** | 문서 형식 (하이퍼링크 + 태그 + 텍스트) |
| **HTTP** | 문서를 실어 나르는 통신 프로토콜 |

```
문서 A ──참고문헌──▶ 문서 B
                      │
                      └─참고문헌──▶ 문서 C
                                     │
                                     └─참고문헌──▶ 문서 D
                                     
거미줄(Web) 형태로 연결 → "Web"이라 명명
```

> [!note] 역사적 흥미
> - `http://`의 슬래시 두 개(`//`)는 **Tim Berners-Lee가 실수**라고 인정함 (불필요)
> - HTTP 헤더의 `Referer`도 **오타**(원래 Referrer)인데 그대로 표준이 됨

---

## 2. 초기 웹 서비스의 구조

> [!important] 본질
> 초기 웹 서비스 = **문서 전달 시스템**
> 브라우저 = **원격 문서 뷰어** (로컬 파일 시스템 대신 인터넷의 원격 컴퓨터 파일을 봄)

### 동작 흐름

```
사용자 (브라우저)                    서버
   │                                  │
   │ ① TCP/IP 연결                     │
   │ ────────────────────────────────▶│
   │                                  │
   │ ② "HTML 문서 좀 주세요" (요청)      │
   │ ────────────────────────────────▶│
   │                                  │
   │ ③ HTML 파일 송신                  │
   │ ◀────────────────────────────────│
   │                                  │
   │ ④ 연결 종료                       │
   │                                  │
   │ ⑤ 화면에 렌더링                    │
```

### 핵심 개념 — Resource

- 문서가 어디에 있는지 = **컴퓨터 식별(IP)** + **파일 시스템 식별(경로)**
- 이를 통칭하여 **리소스(Resource)** 라고 함

---

## 3. HTTP 프로토콜의 특징

### 텍스트 기반 (ASCII)

> [!important] 설계 철학
> 당시 클라이언트-서버 환경에서 **바이너리 프로토콜이 효율적**이었지만, Tim Berners-Lee는 **일반 텍스트(ASCII)** 로 설계 → 단순함과 가독성 우선.

### Request/Response 구조

```
[Request]
  GET / HTTP/1.1
  Host: www.example.com
  User-Agent: Mozilla/5.0 ...
  Accept-Language: en
  Accept-Encoding: gzip, deflate
  
  (개행 두 번 = \r\n\r\n)

[Response]
  HTTP/1.1 200 OK
  Server: nginx
  Date: Mon, 09 May 2026 ...
  Content-Type: text/html
  Content-Length: 1234
  
  (개행 두 번 = \r\n\r\n)
  
  <html>
    <body>Hello, World!</body>
  </html>
```

### 헤더와 바디 구분 — `\r\n\r\n`

> [!tip] 개행 두 번의 의미
> - **개행 = `\r\n`** (16진수: `0x0D 0x0A`)
> - **개행이 두 번** 연속 등장하면 그 위는 **헤더**, 그 아래는 **바디**
> - HTTP 응답 파싱의 핵심 규칙

---

## 4. HTTP Method — 리소스에 대한 행위

### 등장 배경

초기에는 GET만 있었지만, 리소스에 **읽기 외 다양한 행위**(생성/수정/삭제 등)가 필요해지면서 메서드가 확장됨.

### 주요 메서드

| Method | 의미 | CRUD | 예시 |
|--------|------|------|------|
| **GET** | 리소스 **읽기** | Read | 강의 목록 조회 |
| **POST** | 리소스 **생성** (또는 일반 데이터 전송) | Create | 새 강의 등록 |
| **PUT** | 리소스 **전체 수정** (덮어쓰기) | Update | 레코드 통째로 교체 |
| **PATCH** | 리소스 **부분 수정** | Update | 주소 필드만 변경 |
| **DELETE** | 리소스 **삭제** | Delete | 강의 삭제 |
| **HEAD** | 헤더만 반환 (바디 X) | — | 존재 여부/메타 확인 |
| **OPTIONS** | 지원하는 메서드 목록 확인 | — | **CORS Preflight** |

### PUT vs PATCH

> [!example] 호성이의 주소 변경
> - 기존: `{ name: "최호성", address: "서울시" }`
> - **PATCH** → 주소만 부분 수정: `{ address: "경기도" }` → 결과: `{ name: "최호성", address: "경기도" }`
> - **PUT** → 전체 교체: `{ name: "김호성", address: "경기도" }` → 결과: 그대로 통째로 교체
> 
> 일반적으로 수정은 **PATCH**가 더 자주 쓰임.

### OPTIONS와 CORS

> [!caution] 바이브 코딩의 단골 골칫거리 — CORS
> - **CORS** (Cross-Origin Resource Sharing) = 다른 출처의 리소스 접근 제어
> - 브라우저가 본 요청 전에 **Preflight Request**를 보냄 → 이때 사용되는 메서드가 **OPTIONS**
> - 서버가 "이 메서드는 허용한다"고 응답해야 본 요청 진행 가능

---

## 5. 리소스의 확장 — 파일 + 데이터

> [!important] 리소스의 두 가지 형태
> 1. **파일** (HTML, 이미지 등)
> 2. **데이터** (DB의 레코드 등)

### RDB 레코드도 리소스

```
강의 테이블 (courses)
┌────┬──────────────┬────────┐
│ id │ title        │ author │
├────┼──────────────┼────────┤
│  1 │ AWS 기초      │ 홍길동  │  ← 한 row = 한 record = 한 resource
│  2 │ 네트워크 입문  │ 김철수  │
└────┴──────────────┴────────┘
```

- 한 레코드(row) = **단위 데이터** = **하나의 리소스**
- 따라서 HTTP는 **DB 레코드에도 CRUD 가능**한 프로토콜로 진화

---

## 6. HTTP Response Code

| 범위 | 의미 | 대표 예시 |
|------|------|----------|
| **2xx** | **성공** | 200 OK, **201 Created** (생성 성공) |
| **3xx** | 리다이렉션 | 301 Moved Permanently |
| **4xx** | **클라이언트 오류** | **404 Not Found**, **403 Forbidden** |
| **5xx** | **서버 오류** | 500 Internal Server Error, 502 Bad Gateway |

### 400대 — 자주 마주치는 오류

| 코드 | 의미 | 흔한 발생 원인 |
|------|------|--------------|
| **404** | Not Found | 요청한 리소스가 서버에 존재하지 않음 |
| **403** | Forbidden | 권한 없음, **CORS 문제**, **Mixed Content**(HTTP/HTTPS 혼용) |
| **401** | Unauthorized | 로그인 필요 |
| **400** | Bad Request | 요청 형식 오류 |

### 500대 — 서버 사이드 오류

> [!caution] 골치 아픈 오류
> 보통 웹 서비스는 **3-Tier**로 구성:
> ```
> 웹 서버 ──▶ 애플리케이션 서버 ──▶ 데이터베이스
> ```
> 이 사이의 **백엔드 로직/통신 문제** 발생 시 500번대 오류 발생.

---

## 7. REST API

### 정의

> [!important] REST = Representational State Transfer
> **"리소스의 상태를 표현(Representation)으로 변환하여 HTTP로 전달하는 API"**

### 핵심 원리 — 자원 중심 설계

| 요소 | 의미 |
|------|------|
| **Resource** | 서버에 존재하는 데이터/객체 (파일, DB 레코드 등) |
| **State** | 리소스가 가진 현재 상태 |
| **Representation** | 그 상태를 표준 형식(JSON 등)으로 변환한 표현 |
| **Transfer** | HTTP 메서드를 통해 전달 |

### 왜 JSON으로 변환하는가?

- 서버의 리소스가 `.docx`, `.bin` 등 **브라우저가 못 읽는 형식**일 수도 있음
- 누구나 이해 가능한 **표준 형식 = JSON** (또는 XML)으로 변환해서 전달
- 클라이언트 입장에서 어떤 언어/플랫폼이든 파싱 가능

```
서버 리소스 (DB Record)
         │
         │ JSON으로 변환 (Representation)
         ▼
{ "id": 1, "title": "AWS 기초", "author": "홍길동" }
         │
         │ HTTP Response로 전달 (Transfer)
         ▼
       클라이언트
```

---

## 8. CRUD 매핑

> [!important] HTTP Method ↔ DB CRUD 대응
> REST API는 결국 DB의 CRUD를 HTTP로 표현하는 것.

| 작업 | HTTP Method | URI 예시 | 설명 |
|------|------------|---------|------|
| **C**reate | POST | `POST /courses` | 새 강의 등록 |
| **R**ead (전체) | GET | `GET /courses` | 강의 목록 조회 |
| **R**ead (단건) | GET | `GET /courses/1` | 1번 강의 조회 |
| **U**pdate (부분) | **PATCH** | `PATCH /courses/1` | 1번 강의 일부 수정 |
| **U**pdate (전체) | PUT | `PUT /courses/1` | 1번 강의 통째 교체 |
| **D**elete | DELETE | `DELETE /courses/1` | 1번 강의 삭제 |

> [!tip] URI는 같아도 Method가 다르면 다른 엔드포인트
> `GET /courses` 와 `POST /courses` 는 **URI는 같지만 Method가 다른 별도의 API**다.
> REST API는 **URI + Method**의 조합으로 식별된다.

### 실제 응답 예시

```http
GET /courses HTTP/1.1

→ 응답:
HTTP/1.1 200 OK
Content-Type: application/json

{
  "courses": [
    { "id": 1, "title": "AWS 기초", "author": "홍길동" },
    { "id": 2, "title": "네트워크 입문", "author": "김철수" }
  ]
}
```

```http
POST /courses HTTP/1.1
Content-Type: application/json

{ "title": "Spring Boot", "author": "이영희" }

→ 응답:
HTTP/1.1 201 Created
Location: /courses/3
```

---

## 9. API = 함수 — f(x) = y 모델

> [!note] API의 본질
> API ≈ 함수 — 매개변수(x)를 받아 결과(y)를 반환

| 요소 | 의미 | 전달 방법 |
|------|------|----------|
| **함수명** | URI + Method | `GET /courses/1` |
| **매개변수 x** | Query Parameter / Path Variable / Body | `?id=1`, `/courses/1`, JSON body |
| **결과값 y** | Response Body | 거의 대부분 **JSON** (XML도 가능) |

---

## 10. 바이브 코딩과 HTTP — 알아둬야 할 것들

> [!tip] 웹 서비스 개발자가 반드시 알아야 할 핵심
> 1. **HTTP는 Request ↔ Response 구조** — 항상 쌍으로 동작
> 2. **Request는 Method**(GET/POST/...)를 가지고, **Response는 상태 코드**(200/404/...)를 가짐
> 3. **데이터는 거의 JSON** — 표준 표현 방식
> 4. **REST API는 자원 중심** — URI는 자원의 위치, Method는 행위
> 5. **CRUD ↔ POST/GET/PATCH/DELETE** 매핑 (PUT은 거의 안 씀)
> 6. **CORS와 Mixed Content** 오류 자주 마주침 — OPTIONS, 403 에러로 나타남

---

## 핵심 요약

> [!summary]
> 1. 웹 = **HTML(문서 형식) + HTTP(전달 프로토콜)** — Tim Berners-Lee가 CERN에서 고안
> 2. 초기 웹 = **문서 전달 시스템**, 브라우저 = **원격 문서 뷰어**
> 3. HTTP는 **텍스트(ASCII) 기반** — 단순함 우선, **헤더/바디는 `\r\n\r\n`로 구분**
> 4. **Request/Response** 구조 — Request는 Method, Response는 상태 코드를 가짐
> 5. **HTTP Method**: GET(읽기), POST(생성), PATCH(부분수정), PUT(전체수정), DELETE(삭제), OPTIONS(CORS Preflight)
> 6. **상태 코드**: 200(성공) / 201(생성) / 404(없음) / 403(권한/CORS/Mixed) / 500(서버 오류)
> 7. **리소스** = 파일 + 데이터(DB 레코드 포함) — 모두 HTTP로 CRUD 가능
> 8. **REST API** = 리소스 상태를 **JSON으로 표현(Representation)** 하여 HTTP로 전달
> 9. **CRUD 매핑**: Create(POST) / Read(GET) / Update(PATCH) / Delete(DELETE)
> 10. URI는 같아도 **Method가 다르면 다른 API** — 식별자는 **URI + Method**
