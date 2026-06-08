---
type: concept
area: security
status: draft
source: 개인 메모 (LDAP 운영 Q&A 정리)
created: 2026-06-08
updated: 2026-06-08
tags:
  - security
  - LDAP
  - 인증
  - 디렉터리
---

# LDAP DN/OU 구조와 인증

## 개요

LDAP에서 객체를 식별하는 **DN(Distinguished Name)** 의 구조와, **OU(Organizational Unit) 다단계 중첩** 설계를 정리한다.
- DN = 여러 **RDN(Relative Distinguished Name)** 이 쉼표로 연결된 경로
- OU는 조직 계층을 표현하기 위해 **여러 번 중첩** 가능 (표준적·일반적 설계)
- 사용자가 트리 안에서 이동하면 **DN이 바뀌고**, 인증 구현 방식(Direct Bind vs Search+Bind)에 따라 **로그인 성공/실패가 갈린다**

## 한 줄 요약

> LDAP DN은 `속성=값`(RDN)을 쉼표로 이은 경로이며 OU는 계층 표현을 위해 중첩된다. 사용자가 다른 OU로 이동해 DN이 바뀌면, **DN 직접 바인드**는 실패하기 쉽고 **검색 후 바인드**는 (base/scope 조건이 맞으면) 정상 동작한다. 인증이 통과해도 **DN으로 참조되는 그룹 멤버십(memberOf)** 은 별도로 갱신해야 한다.

---

## 1. DN과 RDN — 식별 경로의 구조

> [!important] DN = RDN의 연결
> DN(Distinguished Name)은 여러 **RDN**(`속성=값` 쌍)이 쉼표로 연결된 형태다.

```
CN=홍길동,OU=영업팀,OU=한국지사,OU=직원,DC=example,DC=com
```

| 구성 요소 | 의미 |
|-----------|------|
| **RDN** | `속성=값` 한 쌍 (예: `OU=영업팀`) |
| **CN** | Common Name — 실제 객체(사용자 등) |
| **OU** | Organizational Unit — 조직 단위 |
| **DC** | Domain Component — 보통 여러 개 연달아 (`DC=example,DC=com`) |

> [!important] 읽는 순서 — 왼쪽이 가장 구체적
> DN은 **왼쪽이 가장 하위(구체적), 오른쪽이 루트(상위)**.
> 위 예시의 트리 구조: `직원 → 한국지사 → 영업팀 → 홍길동` 순으로 내려간다.

---

## 2. OU 다단계 중첩

> [!note] OU 중첩은 표준이다
> OU는 조직 계층을 표현하기 위해 **여러 번 중첩**해서 쓰는 것이 LDAP에서 매우 일반적인 디렉터리 설계 방식.

- OU뿐 아니라 다른 속성도 중복 가능하지만, **같은 레벨(하나의 RDN 안)에서는 같은 값이 중복 불가**
- 계층이 다르면 같은 이름의 OU도 공존 가능 (예: 여러 부서 아래 각각 `OU=관리`) → 전체 DN이 다르므로 문제없음
- **multi-valued RDN**: 하나의 RDN 안에 `+`로 여러 속성을 묶는 것도 가능 (예: `CN=홍길동+UID=hong`) — OU 중첩과는 **다른 개념**

---

## 3. DN 변경(OU 이동) 시 로그인 영향

> [!caution] DN이 바뀌면 로그인이 실패할 수 있다
> 사용자가 다른 OU로 이동하면 DN이 바뀐다. 결과는 **인증을 어떻게 구현했느냐**에 따라 정반대.

예시: 사용자가 아래로 이동
```
변경 전: CN={user},OU=Employee,OU=Company,DC=example,DC=com
변경 후: CN={user},OU=Employee - Sub Division,OU=Employee,OU=Company,DC=example,DC=com
```

### 3-1. DN 직접 바인드 (Direct / Simple Bind) → 실패 가능성 큼

> [!important] 템플릿으로 DN을 직접 조합해 바인드
> 애플리케이션 설정에 사용자 DN 패턴이 박혀 있는 방식.

```
CN={username},OU=Employee,OU=Company,DC=example,DC=com
```

- 사용자가 `OU=Employee - Sub Division` 아래로 이동하면 실제 DN이 달라짐
- 앱은 여전히 **예전 위치로 바인드** 시도 → **"해당 DN의 엔트리 없음"으로 실패**
- 해결: 앱의 **DN 템플릿(user DN pattern)** 을 새 구조에 맞게 수정

### 3-2. 검색 후 바인드 (Search + Bind) → 보통 정상 동작

> [!important] 속성으로 검색해 실제 DN을 찾은 뒤 바인드
> 서비스 계정으로 먼저 바인드 → base DN 아래에서 사용자를 속성(`uid`, `sAMAccountName`, `mail`, `cn` 등)으로 검색 → 찾은 실제 DN으로 바인드.

- 사용자가 트리 안에서 이동해도 검색이 새 위치를 찾아주므로 **로그인 계속 가능**
- **단, 두 조건이 맞아야 함:**

| 조건 | 내용 |
|------|------|
| **검색 base DN** | 새 위치를 **포함**해야 함 (상위 OU가 base면 하위 추가는 여전히 포함) |
| **검색 scope** | **`subtree`** 여야 함. `onelevel`이면 한 단계 더 깊어진 사용자가 **누락되어 실패** |

---

## 4. 실무 주의 — 인증은 통과해도 권한(authorization)이 깨질 수 있다

> [!caution] memberOf / member 참조 무결성
> DN이 바뀌면 인증 자체는 통과하더라도 **그룹 소속이 끊어질 수 있다.**

- 그룹 객체의 `member` 속성, 사용자의 `memberOf`는 보통 **전체 DN으로 참조**
- 이동 시 이 참조가 갱신되지 않으면 그룹 소속이 끊김
- 같은 디렉터리 서버 내 이동(`modrdn`/move)은 서버가 **참조 무결성을 자동 처리**하는 경우가 많음
- 그러나 **애플리케이션 DB에 사용자 DN을 따로 저장**해 두었다면 그 값은 **직접 갱신** 필요

---

## 5. 면접식 설명

> LDAP의 DN은 `CN=...,OU=...,DC=...`처럼 RDN(`속성=값`)을 쉼표로 이은 식별 경로이고, 왼쪽이 가장 구체적, 오른쪽이 루트입니다. OU는 조직 계층 표현을 위해 여러 번 중첩하는 게 표준입니다. 운영에서 중요한 건 사용자가 다른 OU로 옮겨져 **DN이 바뀌었을 때**인데, 인증을 **DN 직접 바인드**로 구현했다면 앱이 박아둔 DN 템플릿과 실제 위치가 달라져 로그인이 실패하고, **검색 후 바인드**로 구현했다면 속성으로 사용자를 다시 찾아 바인드하므로 (검색 base가 새 위치를 포함하고 scope가 subtree이면) 계속 동작합니다. 마지막으로, 인증이 통과해도 `memberOf`/`member`가 전체 DN으로 참조되기 때문에 **그룹 멤버십**이 깨질 수 있고, 앱 DB에 DN을 저장해 뒀다면 그 값도 직접 갱신해야 합니다.

## 헷갈리기 쉬운 점

- DN의 **읽는 방향**: 왼쪽=하위(구체적), 오른쪽=상위(루트)
- 같은 OU 이름이라도 **계층이 다르면 전체 DN이 다르므로 공존 가능**
- `+`로 묶는 multi-valued RDN은 **OU 중첩과 무관한 별개 개념**
- 확인 필요: 디렉터리 서버별 `modrdn`/move 시 참조 무결성 자동 처리 범위 (서버 제품에 따라 다름)

## 관련 문서

- [[모던 웹과 JWT]]
