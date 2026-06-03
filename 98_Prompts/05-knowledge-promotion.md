# Study / Inbox → Knowledge 승격 프롬프트

## 목적

raw 메모나 Study 메모 중 재사용 가치가 높은 내용을 `03_Knowledge`의 concept-note로 승격한다.

## Prompt

```text
다음 문서를 기반으로 03_Knowledge에 concept-note를 만들어줘.

대상 문서:
- [여기에 대상 파일 경로 입력]

추천 생성 경로:
- [예: 03_Knowledge/network/Inline, Out-of-Path, Proxy 구조 비교.md]

요구사항:
1. concept-note 형식으로 작성해.
2. 기존 raw 메모의 표현과 비유는 최대한 유지해.
3. 아래 섹션을 포함해:
   - 개요
   - 한 줄 요약
   - 왜 필요한가
   - 핵심 개념
   - 예시
   - 실무에서 주의할 점
   - 헷갈리기 쉬운 점
   - 면접식 설명
   - 관련 문서
4. YAML Properties를 추가해.
5. 관련 Obsidian 링크를 추가해.
6. 원문에 없는 내용은 단정하지 말고 "확인 필요:"로 표시해.
7. 기존 raw 문서는 삭제하지 마.
8. 기존 유사 문서가 있으면 새로 만들지 말고 업데이트 후보로 제안해.
9. 작업 전 생성/수정 계획을 먼저 보여줘.

YAML 예시:
---
type: concept
area:
status: draft
source:
created:
updated:
tags:
---
```
