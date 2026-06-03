---
type: concept
area: network
status: draft
updated: 2026-06-03
tags:
  - network
  - IP
created: 2026-04-17
---

# IPv4 와 IPv6 실무

---

## 현재 실무 구조: 엣지에서 IPv6 수용, 내부는 IPv4 유지

- 서버 환경은 대부분 **IPv4 기반**
- IPv6 는 모바일 클라이언트 대응을 위해 **네트워크 엣지(CDN/LB)** 에서 수용
- 중간 홉에서 IPv4 로 변환하여 백엔드 서버로 전달

```
[모바일 - IPv6] ──→ Route53 (AAAA + A 레코드)
                          │
                    ┌─────▼──────┐
                    │ CloudFront │  ← Dual-Stack (IPv4/IPv6 모두 수신)
                    │ or ALB     │
                    └─────┬──────┘
                          │ IPv4 only
                    ┌─────▼──────┐
                    │ EC2 / ECS  │  ← 내부는 IPv4 사설 대역
                    │ 10.0.x.x   │
                    └────────────┘
```

---

## 서버 측에서 IPv6 를 안 쓰는 현실적 이유

- 사설 네트워크(VPC)가 **IPv4 기반으로 이미 구축**되어 있음
- 내부 통신은 `10.x.x.x` 대역으로 충분하고, NAT 도 문제없음
- IPv6 도입 시 **보안그룹, ACL, 모니터링, 로깅 전부 이중 관리** 필요
- 운영 복잡도 대비 실익이 거의 없음

---

## 클라이언트 측에서 IPv6 가 필요한 이유

- **Apple ATS 정책** — iOS 앱은 IPv6 네트워크에서 동작해야 App Store 심사 통과
- **모바일 통신사** — IPv4 주소 고갈로 많은 통신사가 NAT64/DNS64 환경으로 전환
- **한국 LTE/5G** — IPv6 only 네트워크가 늘어나는 추세

---

## 변환 방식

| 방식 | 위치 | 설명 |
| --- | --- | --- |
| **Dual-Stack** | Edge / LB | IPv4, IPv6 둘 다 리스닝하고, 백엔드는 IPv4 로 전달 |
| **NAT64 / DNS64** | 통신사 / 네트워크 장비 | IPv6 only 클라이언트가 IPv4 서버에 접근 가능하게 변환 |
| **CloudFront, ALB** | AWS Edge | IPv6 요청 수신 → Origin 에는 IPv4 로 전달 |

---

## 정리

> [!summary]
> 서버 인프라를 IPv6 로 전면 전환하는 것이 아니라, **엣지(CDN/LB)에서 IPv6 를 수용하고 내부는 IPv4 를 유지**하는 것이 현재 업계의 주류 패턴이다.
> 서버 내부까지 IPv6 를 쓰는 경우는 대규모 클라우드 사업자나 IPv6-native 로 새로 설계하는 경우 정도이다.
