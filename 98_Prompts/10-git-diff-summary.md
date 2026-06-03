# Git Diff 요약 / 커밋 메시지 프롬프트

## 목적

Claude Code가 작업한 뒤 변경 내용을 점검하고 커밋 메시지를 추천받는다.

## Prompt

```text
이번 작업으로 변경된 파일을 git diff 기준으로 요약해줘.

확인할 것:
1. 새로 생성된 파일
2. 수정된 파일
3. 이동된 파일
4. 삭제된 파일이 있는지 여부
5. Obsidian 내부 링크가 깨질 가능성이 있는 부분
6. YAML Properties가 누락된 문서
7. MOC 업데이트 여부
8. 커밋 메시지 추천

중요:
- 삭제된 파일이 있으면 반드시 별도로 표시해줘.
- 대량 이동이나 리네이밍이 있으면 old path → new path 형식으로 보여줘.
- 커밋 메시지는 Conventional Commits 스타일로 추천해줘.

커밋 메시지 예시:
- docs: organize inbox notes
- docs: promote network notes to knowledge
- docs: update network MOC
- chore: archive deprecated notes
```
