# MOC 업데이트 프롬프트

## 목적

새로 추가/수정된 Knowledge 문서를 기준으로 관련 MOC를 갱신한다.

## Prompt

```text
이번에 추가/수정된 Knowledge 문서를 기준으로 관련 MOC를 업데이트해줘.

대상 MOC:
- [예: 91_MOC/Network MOC.md]

요구사항:
1. 기존 MOC 내용은 최대한 유지해.
2. 새로 추가된 문서를 적절한 섹션에 배치해.
3. 존재하지 않는 문서 링크는 만들지 마.
4. 중복 링크는 제거해.
5. 섹션이 부족하면 필요한 섹션을 제안해.
6. 작업 전 변경 계획을 먼저 보여줘.
7. 내가 승인하기 전까지 파일을 수정하지 마.

추천 섹션 예시:
- 기본 개념
- TCP/IP
- HTTP
- Proxy / Load Balancer
- AWS Networking
- Security
- Troubleshooting
- 면접 대비
- 관련 Study
```
