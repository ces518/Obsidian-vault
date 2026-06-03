# LLM Wiki Prompts

개인 Obsidian + Claude Code 기반 LLM Wiki 관리를 위한 재사용 프롬프트 모음입니다.

## 사용 흐름

```text
Raw 입력
→ 00_Inbox
→ Claude Code 분석
→ 정리 계획 확인
→ Study 또는 Knowledge로 이동/생성
→ MOC 갱신
→ git diff 확인
→ commit
```

## 추천 사용 순서

1. `01-inbox-analysis.md`
2. `02-refactor-plan.md`
3. `03-approved-organize.md`
4. `08-moc-update.md`
5. `10-git-diff-summary.md`

## 핵심 원칙

- 바로 수정하지 말고 먼저 분석/계획부터 요청한다.
- 삭제는 금지한다.
- 대량 이동/리네이밍 전에는 반드시 승인 단계를 둔다.
- Study 문서와 Knowledge 문서를 분리한다.
- 원문에 없는 내용은 단정하지 않는다.
- 불확실한 내용은 `확인 필요:` 또는 `TODO:`로 표시한다.
