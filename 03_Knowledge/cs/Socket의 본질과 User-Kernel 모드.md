---
type: concept
area: cs
status: draft
source: AWS 로 배우는 네트워크 (강의·책) — [[User Kernel 모드 와 Socket 의 본질]]에서 승격
created: 2026-06-15
updated: 2026-06-15
tags:
  - cs
  - OS
  - kernel
  - socket
  - IO
---

# Socket의 본질과 User-Kernel 모드

## 개요

네트워크 프로그래밍과 OS를 이해하는 토대 — **User 모드/Kernel 모드의 구분**, **파일을 통한 I/O 추상화**, 그리고 그 위에서 **Socket이 곧 (디바이스) 파일**이라는 사실을 정리한다.

## 한 줄 요약

> 컴퓨터는 Hardware → Kernel(OS) → Application 3계층이며, User 모드 프로세스는 커널에 직접 접근하지 못하고 **디바이스 파일**이라는 인터페이스로 간접 접근한다. **Socket은 TCP/IP(커널 구현)를 추상화한 디바이스 파일**이라서, 파일의 Read/Write가 Socket에서는 Receive/Send로 대응된다.

## 왜 필요한가

- 네트워크는 결국 **커널에 구현된 TCP/IP**를 다루는 일 → User 모드 앱이 어떻게 거기에 닿는지 알아야 함
- "Socket = 파일"이라는 관점을 잡으면 **File I/O 지식이 그대로 네트워크 I/O로 전이**됨
- 클라우드(AWS)는 **가상화(=논리/소프트웨어)** 인프라라, 물리/가상 양쪽 이해의 출발점이 OS 계층 개념

## 핵심 개념

### 1) 컴퓨터의 3계층

```
Layer 3 · User Mode   ← 애플리케이션 (Excel, 게임)
Layer 2 · Kernel Mode ← 운영체제(OS)
Layer 1 · Hardware    ← CPU, RAM, NIC
```

- 하위가 있어야 상위가 존재. **커널이 죽으면 그 위 앱 전부 붕괴**.

### 2) 프로세스(주체) ↔ 파일(대상체)

- 프로그램 실행 → **프로세스**(User Mode), 행위 대부분이 **I/O**
- 파일에 대한 행위 = **Read / Write / Execute**, 모든 I/O 권한은 **커널이 통제**

### 3) 파일의 두 종류

| 종류 | 설명 |
|------|------|
| **데이터 파일** | `.xlsx`, `.mp4` 등 데이터를 담는 논리 단위 |
| **디바이스 파일(인터페이스)** | 커널 영역 접근용 특수 파일. User 모드는 커널에 직접 접근 불가 → 디바이스 파일로 간접 상호작용 |

### 4) Socket = 파일

- TCP/IP는 **커널 내부에 구현** → User 모드가 접근하도록 제공되는 인터페이스가 **Socket**(디바이스 파일의 일종)

| File I/O | Socket I/O |
|----------|------------|
| Write | **Send** (송신) |
| Read | **Receive** (수신) |

```
Process → Socket(Send) → Kernel(TCP/IP) → Device Driver → NIC → 네트워크
네트워크 → NIC → Device Driver → Kernel(TCP/IP) → Socket(Receive) → Process
```

### 5) 버퍼와 I/O 모델

- Write 시 **User 버퍼 → (복사) → Kernel 버퍼 → 실제 I/O(OS가 주체)**

| 모델 | 특징 |
|------|------|
| **Buffered I/O** | 버퍼를 채워 한꺼번에 — 효율적 |
| **Non-buffered I/O** | 즉시 입출력 — 즉시성, 효율↓ |
| **비동기(Async) I/O** | OS에 I/O 위임, 프로세스는 다른 작업 |

> 성능 관점의 fsync/page cache flush 부작용은 [[Sequential write throughput이 낮아지는 문제]] 참고.

## 예시

> [!example] C 언어 — 콘솔 디바이스 파일
> `fopen("con", "w")`로 콘솔 디바이스 파일을 열고 `"Hello World"`를 Write하면, 커널을 거쳐 비디오 디바이스가 구동되어 모니터에 출력된다.
> Socket도 동일한 원리 — 파일을 다루듯 열고 Send/Receive 한다.

## 실무에서 주의할 점

- Physical = 하드웨어 / **Virtual = Logical = 소프트웨어** — `Virtual`이 보이면 소프트웨어적 구현으로 해석
- User ↔ Kernel 버퍼 **복사 비용**이 존재 → 고성능 I/O에서 zero-copy 등을 고려 (확인 필요: 구체 기법은 별도 학습)
- Buffered I/O는 효율적이지만 **flush 시점**에 따라 지연/유실 특성이 달라짐

## 헷갈리기 쉬운 점

- "Socket은 통신 객체"라기보다 **본질은 파일(디바이스 파일)** — 그래서 File I/O와 대응됨
- TCP/IP는 앱이 아니라 **커널에 구현**되어 있음
- Buffered/Non-buffered는 "버퍼 유무", 비동기는 "I/O 주체를 OS에 위임" — **다른 축**
- 확인 필요: OS·언어 런타임별 기본 버퍼링 정책 차이

## 면접식 설명

> 컴퓨터는 Hardware, Kernel(OS), Application 3계층이고, User 모드 프로세스는 커널에 직접 접근할 수 없어서 **디바이스 파일**이라는 인터페이스로 간접 접근합니다. 네트워크의 TCP/IP는 커널에 구현돼 있는데, 여기에 접근하는 인터페이스가 바로 **Socket이고 본질은 파일**입니다. 그래서 파일의 Read/Write가 Socket에서는 Receive/Send로 대응되죠. 실제 Write는 User 버퍼에서 Kernel 버퍼로 복사된 뒤 OS가 I/O를 수행하며, 버퍼를 모았다 처리하면 Buffered I/O, 즉시 처리하면 Non-buffered, OS에 위임하면 비동기 I/O가 됩니다.

## 관련 문서

- [[User Kernel 모드 와 Socket 의 본질]] (원천 study 노트)
- [[4계층 TCP 헤더 구조와 Buffered IO]]
- [[TCP 와 UDP]]
- [[Sequential write throughput이 낮아지는 문제]]
