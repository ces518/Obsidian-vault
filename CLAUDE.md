# CLAUDE.md

이 저장소는 개인 Obsidian 기반 LLM Wiki이다.
Claude Code는 이 Vault를 정리하고, 문서를 리팩토링하고, 내부 링크를 보강하고, 학습 메모를 재사용 가능한 지식으로 승격하는 보조 도구로 사용한다.

---

## 1. Vault 목적

이 Vault의 목적은 다음과 같다.

1. 개발 관련 지식 정리
2. 책, 강의, 스터디 메모 관리
3. 백엔드, 네트워크, DB, MQ, 시스템 설계 지식 축적
4. 직접 실험한 코드와 학습 결과 기록
5. 면접, 이력서, 커리어 자료로 재활용 가능한 지식 자산화

이 Vault는 단순 메모장이 아니라, 시간이 지날수록 검색 가능하고 연결 가능한 개인 개발 지식베이스로 발전시키는 것을 목표로 한다.

---

## 2. 기본 운영 원칙

* 모든 문서는 Markdown으로 작성한다.
* Obsidian 내부 링크는 `[[문서명]]` 형식을 사용한다.
* 문서 삭제는 사용자 확인 없이 하지 않는다.
* 대량 이동, 대량 수정, 대량 리네이밍 전에는 반드시 계획을 먼저 제안한다.
* 원문에 없는 내용을 사실처럼 단정하지 않는다.
* 불확실한 내용은 `확인 필요:` 또는 `TODO:`로 표시한다.
* Study 문서와 Knowledge 문서를 구분한다.
* Study 문서는 원천 학습 흐름을 보존한다.
* Knowledge 문서는 나중에 독립적으로 읽을 수 있게 정제한다.
* 문서 제목은 검색 가능한 핵심 키워드를 포함한다.
* 가능하면 문서 하단에 관련 문서 링크를 추가한다.
* 기존 문서의 의미를 임의로 바꾸지 않는다.
* 사용자의 개인적인 이해 방식, 비유, 정리 스타일은 최대한 유지한다.

---

## 3. 폴더 구조

Vault는 다음 구조를 기준으로 관리한다.

```text
LLM-Wiki/
├─ 00_Inbox/
├─ 01_Daily/
├─ 02_Study/
├─ 03_Knowledge/
├─ 04_Projects/
├─ 05_Career/
├─ 06_Archive/
├─ 90_Templates/
├─ 91_MOC/
├─ 98_Prompts/
├─ 99_Attachments/
└─ CLAUDE.md
```

---

## 4. 폴더별 역할

### 00_Inbox

아직 분류되지 않은 임시 메모를 저장한다.

사용 예시:

* 갑자기 떠오른 아이디어
* 빠르게 복사해둔 Claude 답변
* 아직 어느 카테고리인지 애매한 메모
* 추후 정리가 필요한 초안
* 미완성 질문 또는 조사거리

하위 구조 예시:

```text
00_Inbox/
├─ quick-notes/
├─ ideas/
└─ to-refine/
```

Inbox 문서는 가능하면 주기적으로 `02_Study`, `03_Knowledge`, `04_Projects`, `05_Career` 중 하나로 이동하거나 정리한다.

---

### 01_Daily

날짜별 학습 기록, 작업 로그, 회고를 저장한다.

사용 예시:

* 오늘 공부한 내용
* 새로 이해한 개념
* 헷갈리는 점
* Knowledge로 승격할 후보
* 다음에 볼 주제

하위 구조 예시:

```text
01_Daily/
└─ 2026/
   ├─ 2026-06-03.md
   └─ 2026-06-04.md
```

Daily 문서는 완성도보다 기록성을 우선한다.

---

### 02_Study

책, 강의, 스터디, 영상 등을 보며 작성한 원천 학습 메모를 저장한다.

사용 예시:

* 책 챕터별 정리
* 강의 수강 메모
* 스터디 발표 준비
* 원문 흐름을 따라 정리한 학습 기록

하위 구조 예시:

```text
02_Study/
├─ books/
│  ├─ AWS 로 배우는 네트워크/
│  └─ 가상 면접 사례로 배우는 대규모 시스템 설계/
├─ lectures/
└─ seminars/
```

Study 문서는 원천 흐름을 보존한다.
단, 재사용 가치가 높은 내용은 `03_Knowledge`로 승격할 수 있다.

---

### 03_Knowledge

재사용 가능한 정제 지식을 저장한다.

사용 예시:

* 개념 정리
* 기술 비교
* 아키텍처 패턴
* 장애 원인과 대응 방식
* 면접에도 활용 가능한 핵심 개념
* 실무에서 다시 찾아볼 설명

하위 구조 예시:

```text
03_Knowledge/
├─ architecture/
├─ backend/
├─ database/
├─ java/
├─ network/
├─ mq/
├─ performance/
├─ security/
├─ testing/
├─ web/
├─ cs/
├─ finance/
└─ ai/
```

Knowledge 문서는 책이나 강의 흐름에 종속되지 않고, 독립적인 개념 문서로 작성한다.

예시:

```text
03_Knowledge/network/Inline, Out-of-Path, Proxy 구조 비교.md
03_Knowledge/database/인덱스와 클러스터링 팩터.md
03_Knowledge/mq/Outbox Pattern.md
03_Knowledge/java/트랜잭션 전파 속성.md
```

---

### 04_Projects

직접 실험한 코드, 토이 프로젝트, 개인 프로젝트, 테스트 결과를 저장한다.

사용 예시:

* Spring Batch 실험
* Redis 멱등키 테스트
* Kafka 리밸런싱 실험
* Docker Compose 샘플
* 성능 테스트 결과
* 코드 스니펫

하위 구조 예시:

```text
04_Projects/
├─ toy-projects/
├─ experiments/
├─ labs/
└─ snippets/
```

Projects 문서는 “내가 직접 해본 것”을 중심으로 작성한다.

---

### 05_Career

면접, 이력서, 포트폴리오, 커리어 회고 관련 문서를 저장한다.

사용 예시:

* 기술 면접 질문
* 시스템 설계 면접 답변
* 이력서 문장
* 프로젝트 경험 정리
* 커리어 회고
* 자기소개 자료

하위 구조 예시:

```text
05_Career/
├─ interview/
├─ resume/
├─ portfolio/
└─ retrospectives/
```

`03_Knowledge`의 내용을 기반으로 면접 답변이나 이력서 문장으로 재가공할 수 있다.

---

### 06_Archive

더 이상 자주 보지 않지만 삭제하기는 애매한 문서를 보관한다.

사용 예시:

* 오래된 스터디 문서
* 더 이상 사용하지 않는 초안
* 중복 정리 전 원본
* Deprecated된 기술 문서

하위 구조 예시:

```text
06_Archive/
├─ old-study/
├─ deprecated-notes/
└─ old-projects/
```

문서를 삭제하기보다는 우선 Archive로 이동한다.

---

### 90_Templates

Obsidian 템플릿을 저장한다.

추천 템플릿:

```text
90_Templates/
├─ daily-note.md
├─ study-note.md
├─ concept-note.md
├─ troubleshooting-note.md
├─ project-note.md
├─ interview-note.md
└─ moc.md
```

---

### 91_MOC

MOC는 Map of Content의 약자로, 주제별 인덱스 문서를 저장한다.

사용 예시:

```text
91_MOC/
├─ Backend MOC.md
├─ Java MOC.md
├─ Database MOC.md
├─ Network MOC.md
├─ MQ MOC.md
├─ System Design MOC.md
├─ Security MOC.md
└─ Career MOC.md
```

MOC는 단순 목차가 아니라 관련 문서를 주제별로 연결하는 허브 문서다.

---

### 98_Prompts

Vault 운영에 반복적으로 사용하는 재사용 프롬프트를 저장한다.

사용 예시:

* Inbox 분석, 리팩토링 계획
* Study → Knowledge 승격
* MOC 업데이트, 중복 문서 탐지
* 면접 자료 생성, Git diff 요약
* CLAUDE.md 점검/업데이트

프롬프트는 번호 접두사로 정렬하고, `Prompts MOC.md`로 묶어 관리한다.

---

### 99_Attachments

이미지, PDF, 캡처, 첨부파일을 저장한다.

하위 구조 예시:

```text
99_Attachments/
├─ images/
├─ pdf/
└─ screenshots/
```

Obsidian의 첨부파일 기본 저장 위치는 이 폴더로 설정한다.

---

## 5. 문서 타입

문서 상단에는 가능한 경우 YAML Properties를 작성한다.

기본 형식:

```yaml
---
type:
area:
status:
source:
created:
updated:
tags:
---
```

### type 값

사용 가능한 type 예시:

```text
daily-note
study-note
concept
troubleshooting
project-note
experiment
snippet
interview-note
moc
retrospective
```

### area 값

사용 가능한 area 예시:

```text
backend
architecture
database
java
network
mq
performance
security
testing
web
cs
finance
ai
career
```

### status 값

사용 가능한 status 예시:

```text
raw
draft
refined
verified
archived
```

상태 의미:

* `raw`: 원천 메모, 아직 정리 전
* `draft`: 초안
* `refined`: 어느 정도 정제됨
* `verified`: 검증되었거나 신뢰 가능한 상태
* `archived`: 더 이상 적극적으로 관리하지 않음

---

## 6. 문서 작성 규칙

### 제목 규칙

문서 제목은 검색 가능한 핵심 키워드를 포함한다.

좋은 예:

```text
Inline, Out-of-Path, Proxy 구조 비교
L4 Proxy와 L7 Proxy 차이
Security Group과 NACL 차이
MySQL 인덱스와 클러스터링 팩터
Outbox Pattern과 멱등성
```

나쁜 예:

```text
정리
개념
네트워크
오늘 배운 것
스터디 1
```

---

### 링크 규칙

Obsidian 내부 링크는 다음 형식을 사용한다.

```markdown
[[문서명]]
```

관련 문서 섹션은 가능하면 문서 하단에 둔다.

```markdown
## 관련 문서

- [[Packet과 Stream의 차이]]
- [[TCP 연결과 소켓 Stream]]
- [[L4 Proxy와 L7 Proxy 차이]]
```

---

### 불확실한 내용 처리

확실하지 않은 내용은 단정하지 않는다.

```markdown
확인 필요: 이 내용은 아직 공식 문서나 실습으로 검증하지 않음.
TODO: 실제 테스트 후 결과 추가 필요.
```

---

### 삭제 금지

Claude Code는 사용자 승인 없이 문서를 삭제하지 않는다.

정리 대상 문서는 우선 다음 위치로 이동한다.

```text
06_Archive/
```

또는 애매한 문서는 다음 위치로 이동한다.

```text
00_Inbox/to-refine/
```

---

## 7. Study 문서 규칙

Study 문서는 책, 강의, 스터디 흐름을 보존한다.

권장 구조:

```markdown
---
type: study-note
source:
area:
status: raw
created:
updated:
tags:
---

# 제목

## 출처

## 핵심 내용

## 이해한 내용

## 질문 / 헷갈리는 점

## 예시 / 비유

## Knowledge로 승격할 후보

- 

## 관련 문서

- 
```

Study 문서는 원천 메모이므로 과도하게 정제하지 않아도 된다.
원천 노트가 이미 충분히 정리되어 있다면 위 섹션을 강제하지 않는다.
`개요` + 자유로운 번호 섹션 + `핵심 요약` + `관련 문서` 형태의 구조도 study-note로 허용한다.
단, YAML Properties(`type/source/area/status/created/updated/tags`)와 검색 가능한 제목은 갖춘다.

---

## 8. Knowledge 문서 규칙

Knowledge 문서는 독립적으로 읽을 수 있는 개념 문서로 작성한다.

권장 구조:

```markdown
---
type: concept
area:
status: draft
source:
created:
updated:
tags:
---

# 개념명

## 개요

## 한 줄 요약

## 왜 필요한가

## 핵심 개념

## 예시

## 실무에서 주의할 점

## 헷갈리기 쉬운 점

## 면접식 설명

## 관련 문서

- 
```

Knowledge 문서는 단순 요약이 아니라, 나중에 다시 읽었을 때 바로 이해할 수 있어야 한다.

---

## 9. Troubleshooting 문서 규칙

장애, 오류, 문제 해결 기록은 troubleshooting 문서로 작성한다.

권장 구조:

```markdown
---
type: troubleshooting
area:
status: draft
created:
updated:
tags:
---

# 문제명

## 증상

## 발생 상황

## 원인 후보

## 확인 방법

## 해결 방법

## 재발 방지

## 관련 문서

- 
```

---

## 10. Project / Experiment 문서 규칙

직접 실험하거나 구현한 내용은 project-note 또는 experiment 문서로 작성한다.

권장 구조:

```markdown
---
type: experiment
area:
status: draft
created:
updated:
tags:
---

# 실험명

## 목적

## 가설

## 실험 환경

## 코드 / 설정

## 결과

## 알게 된 점

## 한계

## 관련 문서

- 
```

---

## 11. MOC 문서 규칙

MOC 문서는 주제별 인덱스 역할을 한다.

권장 구조:

```markdown
---
type: moc
area:
status: maintained
---

# 주제 MOC

## 핵심 개념

- 

## 심화 개념

- 

## 실무 / 트러블슈팅

- 

## 면접 대비

- 

## 관련 스터디

- 
```

MOC는 정기적으로 업데이트한다.

---

## 12. Claude Code 작업 규칙

Claude Code는 다음 작업을 수행할 수 있다.

### 가능 작업

* 문서 구조 개선
* YAML Properties 추가
* 오타 수정
* 중복 문서 후보 탐지
* 내부 링크 보강
* MOC 업데이트
* Study 문서를 Knowledge 문서로 승격
* Daily 문서에서 지식 후보 추출
* 면접 질문/답변 초안 생성
* Archive 후보 제안
* Git diff 기반 변경 사항 요약

### 주의 작업

다음 작업은 반드시 사용자 승인 후 수행한다.

* 대량 파일 이동
* 대량 파일명 변경
* 문서 병합
* 문서 삭제
* Archive 이동
* 기존 문서의 큰 의미 변경
* 폴더 구조 변경

---

## 13. Claude Code에게 요청할 때의 기본 프롬프트

### Vault 구조 분석

```text
현재 Obsidian Vault 구조를 분석해서 개인 개발 LLM Wiki 관점에서 개선안을 제안해줘.

확인할 항목:
1. 중복되는 카테고리
2. 너무 넓은 카테고리
3. Study에 남겨야 할 문서
4. Knowledge로 승격할 문서
5. MOC로 묶으면 좋은 주제

아직 파일 이동이나 수정은 하지 말고 report만 작성해줘.
```

### Study → Knowledge 승격

```text
02_Study 아래 최근 수정된 문서를 검토해서,
반복해서 다시 볼 만한 개념을 03_Knowledge에 concept-note 형식으로 승격해줘.

규칙:
1. 원문에 없는 내용은 추측하지 말 것
2. 불확실한 내용은 "확인 필요"로 표시할 것
3. 기존 관련 문서가 있으면 새로 만들지 말고 링크할 것
4. 변경 전 요약을 먼저 보여줄 것
5. 내가 승인하기 전까지 실제 파일 수정은 하지 말 것
```

### MOC 업데이트

```text
03_Knowledge/network 아래 문서를 기준으로
91_MOC/Network MOC.md를 업데이트해줘.

분류 기준:
1. 기본 개념
2. TCP/IP
3. HTTP
4. Proxy / Load Balancer
5. AWS Networking
6. Security
7. Troubleshooting

문서에 없는 내용은 만들지 말고, 기존 문서 링크 중심으로 구성해줘.
```

### 중복 문서 정리

```text
03_Knowledge 아래에서 주제가 중복되는 문서를 찾아줘.

결과는 다음 형식으로 정리해줘:
1. 중복 후보
2. 유지할 문서
3. 병합 후보
4. 이름 변경 후보
5. 삭제하지 말고 Archive로 보낼 후보

아직 실제 수정은 하지 마.
```

### 면접 자료 생성

```text
03_Knowledge 문서들을 기반으로
05_Career/interview에 백엔드 개발자 면접 질문과 답변 초안을 만들어줘.

규칙:
1. 답변은 내 문서에 있는 내용을 우선으로 작성
2. 각 답변 아래에 참고한 Obsidian 링크 추가
3. 모르는 내용은 추측하지 말고 "보강 필요"로 표시
4. Java, DB, Network, MQ, System Design으로 분류
```

---

## 14. Git 운영 규칙

이 Vault는 개인 Git 저장소로 관리한다.

추천 브랜치 전략:

```text
main
└─ working/wiki-refactor-YYYYMMDD
```

일상적인 문서 작성은 `main`에서 해도 된다.
단, Claude Code를 이용해 대량 정리, 파일 이동, 이름 변경을 할 때는 별도 브랜치를 사용한다.

예시:

```bash
git checkout -b working/wiki-refactor-20260603
```

작업 후 반드시 확인한다.

```bash
git status
git diff
```

커밋 메시지 예시:

```text
docs: add network concept notes
docs: reorganize study notes
docs: update network MOC
docs: promote study notes to knowledge
chore: archive deprecated notes
```

추천 `.gitignore`:

```gitignore
.DS_Store
.trash/
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
```

Obsidian 설정을 공유하고 싶다면 `.obsidian` 전체를 무시하지 않는다.
단, workspace 관련 파일은 개인 화면 상태이므로 Git에서 제외한다.

---

## 15. 문서 승격 기준

Study 문서를 Knowledge로 승격할 때는 다음 기준을 사용한다.

### Study에 남길 문서

* 책이나 강의의 흐름을 따라가는 문서
* 아직 내 언어로 정리되지 않은 문서
* 원천 메모 성격이 강한 문서
* 출처 맥락이 중요한 문서

### Knowledge로 승격할 문서

* 독립적으로 다시 읽어도 이해되는 문서
* 면접, 업무, 실습에 재사용 가능한 문서
* 여러 문서에서 반복적으로 참조될 개념
* 직접 이해한 비유나 실무 관점이 포함된 문서
* 특정 책에만 묶이지 않고 일반 개념으로 사용할 수 있는 문서

예시:

```text
02_Study/books/AWS 로 배우는 네트워크/네트워크 장치 구조 Inline Out of Path.md

→ 03_Knowledge/network/Inline, Out-of-Path, Proxy 구조 비교.md
```

---

## 16. 좋은 문서의 기준

좋은 문서는 다음 조건을 만족한다.

1. 제목만 봐도 주제를 알 수 있다.
2. 한 줄 요약이 있다.
3. 핵심 개념이 명확하다.
4. 예시나 비유가 있다.
5. 헷갈리기 쉬운 점이 정리되어 있다.
6. 관련 문서 링크가 있다.
7. 나중에 면접 답변이나 실무 설명으로 재사용할 수 있다.
8. 출처가 필요한 경우 source가 명시되어 있다.
9. 불확실한 내용은 확인 필요로 표시되어 있다.

---

## 17. 이 Vault에서 중요하게 다룰 주제

현재 주요 관심 주제는 다음과 같다.

```text
- Java
- Spring Boot
- JPA / QueryDSL
- MySQL
- Redis
- Kafka / MQ
- Outbox / Inbox Pattern
- 멱등성
- 네트워크
- HTTP
- Proxy / Load Balancer
- AWS
- Docker
- 성능
- 테스트
- 시스템 설계
- 백엔드 면접
```

문서 정리 시 위 주제들을 중심으로 MOC와 내부 링크를 보강한다.

---

## 18. 최종 원칙

이 Vault는 완벽한 분류보다 지속적인 축적을 우선한다.

* 일단 적는다.
* Inbox에 모은다.
* Study에 원천 메모를 남긴다.
* 중요한 것은 Knowledge로 승격한다.
* MOC로 연결한다.
* Career 자료로 재활용한다.
* Claude Code는 정리자 역할을 한다.

가장 중요한 원칙은 다음이다.

> Obsidian은 사람이 읽고 쌓는 지식 저장소이고, Claude Code는 그 지식 저장소를 정리하고 연결하는 보조 운영자다.

