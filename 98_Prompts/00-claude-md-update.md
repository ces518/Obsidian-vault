# CLAUDE.md 관리 / 업데이트 프롬프트

## 목적

Vault 루트의 `CLAUDE.md`가 현재 Wiki 운영 규칙과 맞는지 점검하고 보강할 때 사용한다.

## Prompt

```text
현재 Obsidian Vault의 CLAUDE.md를 검토해줘.

목표:
- 개인 개발 LLM Wiki 운영 규칙이 충분히 반영되어 있는지 확인
- 폴더 구조, 문서 타입, Study/Knowledge 분리 기준, Claude Code 작업 규칙이 명확한지 점검
- 부족한 규칙이 있으면 개선안을 제안

규칙:
1. 아직 CLAUDE.md를 직접 수정하지 마.
2. 먼저 개선 제안만 보고서로 작성해줘.
3. 대량 변경이 필요한 경우 변경 전/후 비교를 보여줘.
4. 삭제, 이동, 리네이밍 관련 규칙은 반드시 보수적으로 작성해줘.
5. Obsidian 내부 링크 규칙과 YAML Properties 규칙도 함께 점검해줘.

결과 형식:
1. 현재 CLAUDE.md 요약
2. 부족한 부분
3. 추가하면 좋은 규칙
4. 수정 제안 diff 요약
5. 실제 수정 여부 확인 질문
```
