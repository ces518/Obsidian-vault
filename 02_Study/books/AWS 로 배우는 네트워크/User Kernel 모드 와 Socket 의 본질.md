---
type: study-note
source: AWS 로 배우는 네트워크 (책)
area: network
status: raw
created: 2026-04-01
updated: 2026-06-03
tags:
  - network
---
# User Kernel 모드 와 Socket 의 본질

## 개요

네트워크를 공부하기 위해 반드시 알아야 할 **운영체제(OS) 기초 지식**을 다룬다.
핵심은 **User 모드 / Kernel 모드**의 구분과 **Socket의 본질**이다.

---

## 1. 컴퓨터의 3계층 구조

범용 OS(Unix, Linux, Windows, iOS 등)는 공통적으로 **3개의 계층(Layer)**으로 구성된다.

```
┌─────────────────────────────────┐
│  Layer 3 · User Mode            │  ← 애플리케이션 (Excel, 게임 등)
├─────────────────────────────────┤
│  Layer 2 · Kernel Mode          │  ← 시스템 소프트웨어 (OS)
├─────────────────────────────────┤
│  Layer 1 · Hardware             │  ← CPU, RAM, NIC 등
└─────────────────────────────────┘
```

| 계층 | 구분 | 설명 | 키워드 |
|------|------|------|--------|
| **Layer 3** | User Mode | 사용자 애플리케이션이 동작하는 영역 | Application, Process |
| **Layer 2** | Kernel Mode | 운영체제가 동작하는 영역 | OS, Kernel |
| **Layer 1** | Hardware | 물리적 장치 | CPU, RAM, NIC |

> [!important] 계층의 의존 관계
> 하위 계층이 존재해야 상위 계층이 존재할 수 있다.
> Hardware → OS → Application 순으로 의존한다.
> OS(커널)가 다운되면 그 위의 모든 애플리케이션도 함께 붕괴된다.

---

## 2. 프로세스와 파일 — 주체와 대상체

### 프로세스 = 행위의 주체

- 프로그램을 실행(인스턴스화)하면 **프로세스**가 된다
- 예: Excel을 실행 → **User Mode Application Process**
- 프로세스의 행위 대부분은 **I/O(입출력)** 이다

### 파일 = 행위의 대상체

프로세스가 파일에 대해 수행하는 행위는 세 가지로 요약된다:

| 행위 | 설명 |
|------|------|
| **Read** | 파일 읽기 |
| **Write** | 파일 쓰기 |
| **Execute** | 파일 실행 |

> [!note] 권한(Permission)
> 모든 I/O에는 **권한**이 필요하며, 이 권한을 **운영체제(커널)**가 통제한다.
> 모든 User Mode 프로세스는 커널의 통제를 받는다.

---

## 3. Physical vs Logical(Virtual)

| 구분 | 의미 | 대상 |
|------|------|------|
| **Physical** | 물리적으로 실존 | Hardware (CPU, RAM, NIC) |
| **Logical = Virtual** | 소프트웨어적으로 존재 | Software, 파일, 가상화 환경 |

> [!tip] IT 용어에서 Virtual ≒ Logical ≒ Software
> `Virtual`이라는 단어가 나오면 → 소프트웨어적 구현이라고 이해하면 된다.

### 가상화와 클라우드

- **가상화(Virtualization)**: Physical한 하드웨어를 소프트웨어로 구현하는 기술
- **클라우드(Cloud)**: 가상화 기술로 구현된 인프라 환경
  - 실체가 보이지 않으므로 "구름(Cloud)"이라는 비유를 사용

> AWS 같은 클라우드 환경의 네트워크를 이해하려면, **물리적 네트워크**와 **가상화된 네트워크** 양쪽을 모두 알아야 한다.

---

## 4. 파일의 두 가지 관점

### ① 데이터 파일

- 일반적으로 알고 있는 파일 (`.xlsx`, `.mp4`, `.docx` 등)
- 데이터를 담는 논리적(Logical) 단위

### ② 디바이스 파일 (인터페이스)

- **커널 영역에 접근하기 위한 인터페이스** 역할을 하는 특수한 파일
- User Mode 프로세스는 커널 영역에 직접 접근할 수 없다
- 대신 커널이 제공하는 **디바이스 파일**을 통해 간접적으로 상호작용한다

```
┌──────────────┐
│   Process    │  User Mode
│  (Excel 등)  │
└──────┬───────┘
       │ Write / Read
  ┌────▼────┐
  │ 디바이스  │  ← 커널이 제공하는 인터페이스 (파일 형태)
  │  파일    │
  └────┬────┘
       │
┌──────▼───────┐
│    Kernel     │  Kernel Mode
│  (OS 내부)    │
└──────┬───────┘
       │
┌──────▼───────┐
│   Hardware   │  장치 구동 (NIC, 디스플레이 등)
└──────────────┘
```

> [!example] C 언어 예시
> `fopen("con", "w")`로 콘솔 디바이스 파일을 열고 `"Hello World"`를 Write하면,
> 커널을 거쳐 비디오 디바이스가 구동되어 모니터 화면에 텍스트가 출력된다.

---

## 5. Socket의 본질 = 파일

> [!important] 핵심
> **Socket은 본질적으로 파일이다.**
> TCP/IP 같은 네트워크 프로토콜을 추상화한 **디바이스 파일(인터페이스)**을 특별히 **Socket**이라고 부른다.

### TCP/IP의 위치

- TCP/IP 프로토콜은 **커널(OS) 내부에 구현**되어 있다
- User Mode 프로세스가 이 커널의 네트워크 기능에 접근할 수 있도록 제공되는 인터페이스 → **Socket**

### Socket I/O와 File I/O의 대응

| File I/O | Socket I/O | 의미 |
|----------|------------|------|
| **Write** | **Send** | 데이터 송신 |
| **Read** | **Receive** | 데이터 수신 |

### 데이터 흐름

```
Process → Socket(Write/Send) → Kernel(TCP/IP) → Device Driver → NIC → 네트워크
네트워크 → NIC → Device Driver → Kernel(TCP/IP) → Socket(Read/Receive) → Process
```

- **NIC (Network Interface Card)**: 네트워크 인터페이스 카드, 흔히 "랜카드"
- **Device Driver**: 하드웨어 장치를 구동하기 위한 소프트웨어 (모든 장치에 필요)

---

## 6. 버퍼(Buffer)와 I/O

파일(또는 소켓)에 Write할 때 실제로 일어나는 일:

```
┌─────────────────┐
│ User Mode       │
│ Buffer ①        │  ← 원본 데이터 저장
└────────┬────────┘
         │ Copy (메모리 복사)
┌────────▼────────┐
│ Kernel Mode     │
│ Buffer ②        │  ← I/O 수행용 버퍼
└────────┬────────┘
         │ 실제 I/O 수행 (OS가 주체)
     [ Hardware ]
```

| 구분 | 설명 |
|------|------|
| **Buffered I/O** | 버퍼를 채운 후 한꺼번에 입출력 (효율적) |
| **Non-buffered I/O** | 버퍼링 없이 즉시 입출력 (효율은 낮지만 즉시성) |
| **비동기(Async) I/O** | OS에게 I/O를 맡기고 프로세스는 다른 작업 수행 |

---

## 핵심 요약

> [!summary]
> 1. 컴퓨터는 **Hardware → Kernel(OS) → Application** 3계층 구조
> 2. **프로세스**(주체)는 **파일**(대상체)에 대해 Read/Write/Execute를 수행
> 3. 파일에는 **데이터 파일**과 커널 접근용 **디바이스 파일(인터페이스)** 두 종류가 있음
> 4. **Socket = 파일** — TCP/IP 프로토콜을 추상화한 특수 인터페이스
> 5. **TCP/IP는 커널에 구현**되어 있으며, Socket을 통해 User Mode에서 접근
> 6. Physical = 하드웨어 / **Virtual = Logical = 소프트웨어**
> 7. 클라우드(AWS)는 **가상화 기술**로 구현된 인프라 환경
