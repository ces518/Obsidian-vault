# 네트워크 개념 문서 리팩토링 예시 프롬프트

## 목적

`Inline / Out-of-Path / Proxy` 같은 네트워크 개념 문서를 정제 Knowledge 문서로 승격할 때 사용하는 예시다.

## Prompt

```text
다음 문서를 개인 Obsidian LLM Wiki의 concept-note 형식으로 리팩토링해줘.

대상 문서:
- [예: 02_Study/books/AWS 로 배우는 네트워크/네트워크 장치 구조 Inline Out of Path.md]

추천 생성 경로:
- 03_Knowledge/network/Inline, Out-of-Path, Proxy 구조 비교.md

요구사항:
1. 문서 상단에 YAML Properties를 추가해.
2. 제목은 "Inline, Out-of-Path, Proxy 구조 비교"로 제안해.
3. 기존 설명과 비유는 유지해.
4. "한 줄 요약" 섹션을 추가해.
5. "내 언어로 정리" 섹션을 추가해.
6. "헷갈리기 쉬운 점" 섹션을 추가해.
7. "면접식 설명" 섹션을 추가해.
8. 관련 문서를 Obsidian 내부 링크 형식으로 보강해.
9. 원문에 없는 내용을 단정하지 말고, 필요한 경우 "확인 필요:"로 표시해.
10. 기존 raw/study 문서는 삭제하지 마.
11. 변경 전후 요약을 먼저 보여주고, 내가 승인하면 파일을 수정해.

추천 YAML:
---
type: concept
area: network
topic:
  - proxy
  - packet
  - stream
  - aws-networking
status: refined
source:
created:
updated:
tags:
  - network
  - proxy
  - aws
  - alb
  - waf
  - packet
  - stream
---
```
