---
tags:
  - network
  - AWS
  - VPC
  - NAT
  - cost-optimization
created: 2026-05-27
---

## 개요

[[공유기 작동 원리]] / [[Symmetric NAT]] 의 NAT 개념을 바탕으로 AWS **VPC(Virtual Private Cloud)** 환경의 본질과, **비용 절감** 및 **보안성 강화**의 핵심 도구인 **VPC Endpoint** (Gateway / Interface) 를 다룬다.
- VPC = AWS의 **사설망** (NAT 기반) — 공유기 역할은 **IGW(Internet Gateway)**
- EC2 ↔ S3 통신이 IGW를 거치면 **인터넷 사용료 발생** (같은 리전인데도!)
- **Gateway Endpoint** = 라우팅 테이블 규칙 한 줄로 IGW 우회 → **무료** (S3 / DynamoDB)
- **ENI (Interface Endpoint)** = 별도 사설 인터페이스. 유료지만 **TAP Switch 기능(VPC Traffic Mirroring)** 으로 보안 모니터링 가능
- Vibe Coding으로 서비스 만든다면 **반드시 Gateway Endpoint 설정** 해두자

---

## 1. VPC = AWS 환경의 사설망

> [!important] VPC는 그 자체로 NAT 환경
> EC2, RDS 같은 리소스는 기본적으로 **Private Network** 안에 산다.
> 외부와 통신하려면 **공유기 = IGW (Internet Gateway)** 를 거쳐야 함.

### VPC 기본 구성도

```
                      [Internet (Public)]
                              │
                              │
                       ┌──────┴──────┐
                       │     IGW     │  ← AWS의 "공유기" (NAT Gateway)
                       │ (Router 1)  │
                       └──────┬──────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────┴────┐          ┌────┴────┐           ┌────┴────┐
   │  ALB    │          │  EC2    │           │  RDS    │
   │ (경계)   │          │ (서버)   │           │  (DB)   │
   └─────────┘          └─────────┘           └─────────┘
   ↑                        Private Network (사설망)
   퍼블릭/프라이빗 경계
```

| 리소스 | 위치 | 인터넷 접근 |
|--------|------|-------------|
| **EC2 / RDS** | Private Network | IGW(NAT) 통해서만 |
| **ALB** | 경계 (Public + Private 양쪽) | Public IP 보유 |
| **IGW** | 경계 게이트웨이 | NAT 기능 제공 |

> [!tip] 외부에서 직접 접근 안 됨
> NAT 구조상 [[Symmetric NAT|NAT Table 매핑]] 없는 Inbound는 자동 Drop → 외부에서 EC2/RDS 직접 접속 불가.
> 접속하려면 별도 설정(보안 그룹 + Public IP) 필요.

관련: [[공유기 작동 원리]] · [[Symmetric NAT]]

---

## 2. ALB의 구조 — 경계의 Reverse Proxy

> [!important] ALB = 두 개의 IP를 가진 Reverse Proxy
> Public 인터페이스 1개 + Private 인터페이스 1개 = NIC 2개.
> [[Reverse Proxy 와 WAF]] 에서 다룬 전형적인 L7 Reverse Proxy.

### 구조

```
[Client] ──Internet──▶ ALB Public IP : 8080
                          │
                    [Reverse Proxy]
                          │
                          ▼ (Private IP로 변환)
                       EC2 : 8080 (Spring Boot API)
```

| 항목 | ALB |
|------|-----|
| **데이터 단위** | Stream (L7, Socket) |
| **NIC 개수** | 2개 (Public + Private) |
| **역할** | Reverse Proxy + L7 Load Balancer |
| **추천 구성** | CloudFront + WAF 앞단에 → EC2까지 보호 |

관련: [[Reverse Proxy 와 WAF]] · [[연결이라는 착각과 AWS ALB]]

---

## 3. 외부 접속을 허용하려면 — Public IP / Elastic IP

> [!caution] EC2 관리용 SSH 접속 시
> EC2에 직접 접속(예: PuTTY로 SSH)하려면 **Public IP** 또는 **Elastic IP** 를 부여해야 함.
> → 자칫 설정 실수 시 **무방비 노출** 위험.

### Public IP vs Elastic IP

| 항목 | Public IP | Elastic IP |
|------|-----------|------------|
| **할당 방식** | EC2 시작 시 자동 부여 | 명시적으로 할당받아 부착 |
| **재시작** | IP 바뀜 | **IP 고정** |
| **용도** | 임시 외부 접속 | 고정 외부 주소 필요 시 |

### 보안 강화 필수

| 설정 | 내용 |
|------|------|
| **보안 그룹 (Security Group)** | SSH(22) 등은 **내 IP만 허용** 으로 제한 |
| **Key Pair** | 비밀번호 대신 키 기반 인증 |
| **최소 권한 원칙** | 불필요한 포트는 절대 열지 말 것 |

관련: [[Symmetric NAT]] (NAT Table ↔ 보안 그룹 규칙 동일 구조)

---

## 4. 비용 문제 — EC2 ↔ S3 가 인터넷을 탄다

> [!danger] 같은 AWS, 같은 리전인데도 돈을 낸다
> EC2에서 S3에 접근할 때 **기본 설정**은 IGW를 통해 인터넷으로 나갔다가 들어옴.
> → **인터넷 트래픽 요금 부과** (S3가 AWS 내부에 있음에도)

### 문제 흐름 (Gateway Endpoint 없을 때)

```
[EC2 (Private)]
    │
    │ S3 접근
    ▼
[IGW] ──▶ [Internet] ──▶ [S3 Public Endpoint]
    │       ⚠️ 인터넷 통과
    │       💸 트래픽 요금 발생
```

### 비용 감각

| 통신량 | 대략 비용 |
|--------|-----------|
| **250 GB** | 약 **10 USD** |
| **1 TB** | 약 **40 USD** |

> [!caution] 작아 보여도 누적
> Vibe Coding으로 서비스 운영 중이라면 모르는 사이 매달 수십 달러가 새고 있을 수 있다.

---

## 5. Gateway Endpoint — 무료 우회 경로

> [!important] 핵심 해법
> VPC의 **라우팅 테이블에 규칙 한 줄 추가** → IGW를 거치지 않고 AWS 백본으로 직접 S3 통신.
> **요금 없음 + 속도 가능성 ↑ + 보안 ↑**

### 동작 흐름

```
[EC2 (Private)]
    │
    │ S3 접근
    ▼
[Router (라우팅 테이블)]
    │
    │ "Destination = S3 → Gateway Endpoint로!"
    │
    ▼
[Gateway Endpoint] ──AWS 백본──▶ [S3]
    ✅ IGW 우회
    ✅ 인터넷 미경유
    ✅ 무료 (같은 리전)
```

### 본질

> [!tip] Gateway Endpoint = 라우팅 규칙 한 줄
> 새로운 하드웨어/소프트웨어가 아니라, **라우팅 테이블에 추가된 규칙**.
> NAT Table 의 그것과 비슷한 형태로 "이 목적지는 이쪽으로" 한 줄.

### 지원 서비스

| 서비스 | Gateway Endpoint 지원 |
|--------|----------------------|
| **S3** | ✅ |
| **DynamoDB** | ✅ |
| 그 외 | ❌ (Interface Endpoint 사용) |

관련: [[라우터에 대한 최소 이론]] (라우팅 테이블)

---

## 6. ENI / Interface Endpoint — 유료지만 강력

> [!important] ENI (Elastic Network Interface)
> 내 VPC에 **전용 가상 인터페이스(논리적 공유기)** 를 추가하는 개념.
> Gateway Endpoint와 달리 **별도 비용** 발생.

### 사용 시나리오

| 용도 | 설명 |
|------|------|
| **이중 사설망** | 데이터베이스를 한 단계 더 깊은 사설망에 격리 |
| **Gateway Endpoint 미지원 서비스** | 대부분의 AWS 서비스가 여기 해당 |
| **트래픽 미러링 / 보안 모니터링** | 아래 §7 |

> [!caution] Endpoint 생성 시 유형 주의
> AWS 콘솔에서 Endpoint 생성할 때 **반드시 `Gateway` 유형** 선택.
> 실수로 `Interface` 선택 → ENI 가 만들어져 **유료**로 전환됨.

---

## 7. ENI의 킬러 기능 — VPC Traffic Mirroring (TAP Switch)

> [!important] ENI 안에 TAP Switch 가 내장되어 있다
> [[Out of Path 구조와 DPI|TAP 스위치]] 의 미러링 기능을 ENI가 **논리적으로 제공**.
> → 실제 통신에 영향 없이 트래픽을 다른 EC2로 복사 가능.

### 구조

```
[EC2 #1] ──▶ [ENI] ──▶ Internet
                │
                │ Port Mirroring (Copy)
                ▼
           [EC2 #2 (모니터링용)]
              ├─ 패킷 수집/저장 → 포렌식
              ├─ NIDS 운영 → 침입 탐지
              ├─ 장애 분석
              └─ 평문 트래픽 분석 (EC2↔ALB 구간)
```

### 활용

| 목적 | 설명 |
|------|------|
| **NIDS** (Network IDS) | 침입 탐지 시스템 운영 |
| **포렌식** | 패킷 저장 → 사후 분석 |
| **트러블슈팅** | 장애/이상현상 추적 |
| **평문 분석** | ALB↔EC2 구간은 **평문 HTTP** → 깊이 있는 서비스 분석 가능 |

> [!tip] 보안 강화가 목적이라면 ENI
> 비용을 감수하더라도 **NIDS, 네트워크 포렌식** 을 운영하려면 ENI 활용이 정석.

관련: [[Out of Path 구조와 DPI]] · [[Reverse Proxy 와 WAF]]

---

## 8. Gateway Endpoint vs Interface (ENI) 종합

| 항목 | **Gateway Endpoint** | **Interface Endpoint (ENI)** |
|------|---------------------|------------------------------|
| **구현** | 라우팅 테이블 규칙 | 가상 NIC (실체 있는 인터페이스) |
| **지원 서비스** | S3, DynamoDB | 대부분의 AWS 서비스 |
| **비용** | **무료** (같은 리전) | **유료** (시간당 + 트래픽) |
| **속도** | AWS 백본 직결 | 동일 |
| **추가 기능** | 없음 (단순 라우팅) | **Traffic Mirroring** 등 |
| **권장 사용** | S3/DynamoDB 비용 절감 | Private Subnet 강제, 보안 모니터링 |

> [!important] 비용 vs 보안의 트레이드오프
> - **돈 안 들이고 비용 절감만** → Gateway Endpoint
> - **보안성 극대화 + 모니터링** → ENI (유료지만 트래픽 미러링 가능)
> - **Private Subnet 강제 구성 + 서비스가 GW EP 미지원** → ENI 외 선택지 없음

---

## 9. VPC Endpoint = "엔드포인트"의 의미

> [!note] 엔드포인트의 두 가지 의미
> - 일반 네트워크 용어 → **호스트** (통신 종단)
> - **VPC Endpoint** → **외부에 노출된 접점** (USB 단자 같은 인터페이스)

VPC Endpoint는 호스트가 아니라, AWS 서비스로 가는 **연결 접점**.

---

## 10. 실전 — Gateway Endpoint 설정 절차

> [!tip] AWS 콘솔 기준 단계
> 어렵지 않다. 5분이면 끝나는 비용 절감 작업.

### 단계

1. **VPC 콘솔** → 좌측 **Endpoints** 메뉴
2. **Endpoints 가 비어 있다면** → 지금 비용을 내고 있다는 뜻
3. **"엔드포인트 생성"** 클릭
4. **이름 / 리전** 확인
5. **유형: AWS 서비스**, 서비스 검색: **S3**
6. **반드시 `Gateway` 유형 선택** ⚠️ (Interface 선택하면 ENI = 유료)
7. **VPC 선택** (기본 VPC면 하나만 있음)
8. **라우팅 테이블 선택** (기본값)
9. **정책: 전체 액세스** (필요에 따라 제한)
10. **생성**

### 검증

```bash
# EC2에 SSH 접속 후
$ aws s3 ls

# 결과로 S3 버킷 이름 목록이 출력되면 ✅ 정상 연동
```

> [!caution] 가장 흔한 실수
> 유형을 `Interface` 로 선택해서 ENI가 만들어지는 것 → **무료가 아니게 됨**.
> 반드시 **Gateway** 인지 두 번 확인.

---

## 핵심 요약

| 기억할 것 | 내용 |
|-----------|------|
| **VPC** | AWS의 **사설망** (NAT 기반) |
| **IGW** | AWS의 "공유기" (Internet Gateway, NAT Gateway 역할) |
| **EC2/RDS** | 기본 Private Network — 외부 직접 접속 불가 |
| **ALB** | Public + Private IP **2개** 가진 [[Reverse Proxy 와 WAF\|Reverse Proxy]] |
| **외부 접속용** | Public IP / Elastic IP — 보안 그룹으로 IP 제한 필수 |
| **비용 함정** | EC2↔S3 가 IGW 통과 → 인터넷 트래픽 요금 (같은 리전인데도) |
| **비용 감각** | 1 TB ≈ **40 USD**, 250 GB ≈ 10 USD |
| **Gateway Endpoint** | 라우팅 규칙 한 줄로 IGW 우회 → **무료** |
| **GW EP 지원** | **S3, DynamoDB** 만 (그 외는 Interface) |
| **ENI** | 가상 NIC, **유료**, 대부분 서비스 지원 |
| **ENI 킬러 기능** | **VPC Traffic Mirroring** (TAP Switch) → NIDS, 포렌식 |
| **콘솔 함정** | 유형을 반드시 **`Gateway`** 선택 (Interface 선택 시 유료 ENI) |
| **검증 방법** | EC2에서 `aws s3 ls` 로 버킷 목록 확인 |
| **권장 전략** | 비용 절감 → Gateway / 보안 강화 → ENI |

---

관련: [[공유기 작동 원리]] · [[Symmetric NAT]] · [[Reverse Proxy 와 WAF]] · [[연결이라는 착각과 AWS ALB]] · [[Out of Path 구조와 DPI]] · [[Inline 구조와 라우터]] · [[라우터에 대한 최소 이론]] · [[IP 헤더와 AWS ENI]] · [[모던 웹 서비스 구조]]
