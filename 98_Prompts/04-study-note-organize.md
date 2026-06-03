# Study 원천 메모 정리 프롬프트

## 목적

책, 강의, 스터디 흐름을 따라가는 raw 메모를 `02_Study`에 정리한다.

## Prompt

```text
00_Inbox 아래의 raw 메모 중 책/강의/스터디 성격의 문서를
02_Study 아래의 study-note 형식으로 정리해줘.

요구사항:
1. 책/강의/스터디의 원래 흐름을 유지해.
2. 원천 메모 성격을 보존해.
3. 과도하게 일반화하거나 Knowledge 문서처럼 바꾸지 마.
4. 문서 상단에 YAML Properties를 추가해.
5. 아래 섹션을 사용해:
   - 출처
   - 핵심 내용
   - 이해한 내용
   - 질문 / 헷갈리는 점
   - Knowledge로 승격할 후보
   - 관련 문서
6. 원문에 없는 내용은 추가하지 말고, 필요한 경우 "확인 필요:"로 표시해.
7. 원본 raw 문서는 삭제하지 마.
8. 작업 전 이동/수정 계획을 먼저 보여줘.

YAML 예시:
---
type: study-note
source:
area:
status: raw
created:
updated:
tags:
---
```
