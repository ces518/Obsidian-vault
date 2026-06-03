# Vault 상태 점검 프롬프트

## 목적

전체 LLM Wiki의 구조, 링크, 빈 문서, 오래된 TODO, MOC 누락 등을 점검한다.

## Prompt

```text
현재 Obsidian Vault 전체를 점검해줘.

확인할 항목:
1. 00_Inbox에 오래 남아 있는 문서
2. Study에서 Knowledge로 승격할 만한 문서
3. Knowledge 문서 중 YAML Properties가 없는 문서
4. 관련 문서 링크가 부족한 문서
5. MOC에 포함되지 않은 주요 Knowledge 문서
6. 중복되거나 너무 비슷한 문서
7. 오래된 TODO / 확인 필요 항목
8. Archive 후보
9. 파일명이나 폴더 위치가 애매한 문서

규칙:
- 아직 실제 파일 수정은 하지 마.
- 삭제는 제안하지 말고 Archive 후보로만 표시해.
- 각 항목마다 우선순위를 High / Medium / Low로 표시해.
- 바로 처리하면 좋은 작업 5개를 추천해줘.

결과 형식:
1. 전체 상태 요약
2. 우선순위별 개선 항목
3. 파일별 조치 제안
4. MOC 개선 제안
5. 다음 작업 프롬프트 추천
```
