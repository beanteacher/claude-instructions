---
name: review
description: "프론트엔드 PR/변경 코드 리뷰. Pass 1 머지 차단 + Pass 2 개선 권고 2단계 워크플로우."
argument-hint: "[optional: PR 번호, 파일 경로, 도메인, 또는 변경 범위]"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
본 저장소(Next.js + React + TypeScript + TanStack Query + Zustand + axios)의 변경 사항을
**스타일 체크리스트가 아니라 리스크 탐지 워크플로우**로 리뷰한다.
`.claude/principles/*` 위반은 머지 반려 근거가 된다.
</objective>

<context>
Scope: $ARGUMENTS

References:
- @.claude/principles/security.md
- @.claude/principles/state-management.md
- @.claude/principles/error-handling.md
- @.claude/principles/api-conventions.md
- @.claude/principles/project-structure.md
- @.claude/principles/code-style.md
- @.claude/principles/completeness-principle.md
- @.claude/principles/search-before-build.md
- @.claude/rules/conventions.md
- @.claude/rules/testing.md
- @.claude/rules/git-workflow.md
</context>

<process>

## 1. 리뷰 범위 결정

- `$ARGUMENTS` 가 PR 번호이면 `gh pr view {N}` / `gh pr diff {N}` 으로 변경 사항 수집
- 파일 경로 / 도메인이 주어지면 해당 영역만 대상
- 인자 없으면 staged + unstaged 변경 (`git diff --stat`, `git diff main...HEAD`)

```bash
# PR diff
gh pr diff {PR_NUMBER}

# 로컬 변경
git diff main...HEAD --name-only
git diff main...HEAD --stat
git log main..HEAD --oneline
```

---

## 2. 영향 범위 분류 — 도메인 × 레이어 매트릭스

변경된 파일을 아래 표에 매핑한다. 변경된 셀(도메인+레이어 교차점)에 따라 점검 우선순위를 자동 도출한다.

| 도메인 | api | hook | page | component | store | model | route |
|--------|-----|------|------|-----------|-------|-------|-------|
| \<domain-a\> | `api/<domain-a>-api.ts` | `use<DomainA>Queries.ts` | `page/<domain-a>/` | `sheet/<DomainA>Sheet.tsx` | — | `model/dto/<domain-a>-dto.ts` | `app/(<domain-a>)/` |
| \<domain-b\> | `api/<domain-b>-api.ts` | `use<DomainB>Queries.ts` | `page/<domain-b>/` | `sheet/<DomainB>Sheet.tsx` | — | — | `app/<domain-b>/` |
| auth | `api/auth-api.ts` | `useAuthQueries.ts` | `page/login/` | `components/axios-interceptor.tsx` | `store/authStore.ts` | — | `app/login/` |

**도출 규칙:**
- 동일 도메인에서 `api` + `hook` + `page` 가 함께 변경되면 → 일관성 점검 필수
- `store/authStore.ts` 또는 `components/axios-interceptor.tsx` 변경 → Pass 1 보안 항목 최우선 점검
- `components/auth-guard.tsx` 변경 → 라우트 가드 우회 가능성 점검 필수

---

## 3. Pass 1 — 머지 차단 (principles 위반)

아래 항목 중 하나라도 위반하면 **머지 차단**. 수정 후 재리뷰 필요.

### 3-1. 보안 (`principles/security.md`)

- [ ] `dangerouslySetInnerHTML` 을 사용자 입력·API 응답으로 렌더하고 있지 않은가
- [ ] `localStorage` / `sessionStorage` 에 accessToken 을 저장하고 있지 않은가 (`authStore.ts` 는 메모리 전용)
- [ ] `NEXT_PUBLIC_*` 환경변수에 secret 값(API 키, JWT 시크릿 등)이 노출되지 않는가
- [ ] axios 를 `api/common/` 인스턴스 외부에서 직접 `axios.create()` 하거나 `import axios from 'axios'` 로 직접 호출하고 있지 않은가
- [ ] 외부 링크에 `rel="noopener noreferrer"` 가 있는가
- [ ] 로그에 토큰·비밀번호·개인정보가 출력되지 않는가

### 3-2. 상태 관리 (`principles/state-management.md`)

- [ ] 서버 데이터(API 응답)를 zustand (`authStore` 등) 에 저장하고 있지 않은가 — 서버 상태는 TanStack Query 전담
- [ ] `useState` 로 폼 상태를 관리하면서 `react-hook-form` 을 우회하고 있지 않은가
- [ ] mutation 성공 후 `queryClient.invalidateQueries` 가 적절히 호출되고 있는가 (캐시 무효화 누락 금지)
- [ ] URL 상태(검색어, 필터, 페이지)를 `useState` 로만 관리하면서 `useUrlSearchParams.ts` 계열을 우회하고 있지 않은가

### 3-3. 에러 처리 (`principles/error-handling.md`)

- [ ] `catch` 블록에서 에러를 무시(빈 블록, `console.log` 만)하고 사용자에게 표시하지 않는 코드가 없는가
- [ ] API 에러 시 `toast` (sonner 또는 `components/ui/use-toast`) 로 사용자에게 알리고 있는가
- [ ] `axios-interceptor.tsx` 의 인터셉터를 우회해 직접 try/catch 로만 처리하고 있지 않은가
- [ ] 컴포넌트 렌더 에러에 `<ErrorBoundary>` 또는 Next.js `error.tsx` 가 있는가

### 3-4. API 컨벤션 (`principles/api-conventions.md`)

- [ ] `api/` 모듈 외부에서 axios 를 직접 호출하고 있지 않은가 (`api/common/app-api.ts` 의 `appApi.get/post/put/patch/delete` 사용 필수)
- [ ] 쿼리 키가 `[domain, 'list'|'detail', params]` 튜플 컨벤션을 따르는가 (예: `documentKeys.list(params)`)
- [ ] 배열·객체 쿼리 파라미터를 문자열 결합으로 직렬화하지 않고 `qs` 라이브러리를 사용하는가
- [ ] `ResponseModel<T>` 타입(`model/response-model.ts`)을 사용하지 않고 `any` 로 응답을 받고 있지 않은가

### 3-5. 구조 (`principles/project-structure.md`)

- [ ] 새 도메인 코드가 `api/`, `hooks/react-query/`, `page/`, `model/dto/` 의 올바른 폴더에 배치되었는가
- [ ] `model/dto/` 타입 파일과 `api/` 클라이언트가 혼재(같은 파일에 타입 + axios 호출)되어 있지 않은가
- [ ] 역방향 import 가 없는가 — `api/` → `hooks/` → `page/` 단방향 의존. `api/` 가 `page/` 를 import 하지 않는가
- [ ] `'use client'` 지시어가 필요한 컴포넌트에만 붙었는가 (불필요한 client 컴포넌트 확대 금지)

### 3-6. 완결성 (`principles/completeness-principle.md`)

- [ ] 로딩 상태 UI 가 있는가 (`BaseTable` 의 `LoadingState`, 또는 커스텀 스켈레톤)
- [ ] 에러 상태 UI 가 있는가 (`ErrorState` 또는 toast)
- [ ] 빈 상태 UI 가 있는가 (`EmptyState`)
- [ ] mutation 성공/실패 toast 가 누락되지 않았는가
- [ ] 모바일 뷰(375px)를 확인했는가
- [ ] 다크모드를 확인했는가

### 3-7. 코드 스타일 (`principles/code-style.md`)

- [ ] `any` 타입이 새로 도입되지 않았는가
- [ ] 파일명이 kebab-case 인가 (`page/*Page.tsx` 형태의 PascalCase 는 예외)
- [ ] className 합성에 `cn()` (`lib/shadcn-utils.ts`) 를 사용했는가 — 문자열 템플릿 리터럴로 className 을 결합하지 않는다

---

## 4. Pass 2 — 개선 권고 (rules 위반, 설계 개선)

아래 항목은 머지는 가능하나 **수정 권고**.

- [ ] 기존 자산을 재사용하지 않고 새로 만든 코드가 있는가
  - 테이블 → `components/shared/table/BaseTable.tsx` 재사용 가능
  - 로딩/에러/빈 상태 → `components/shared/table/TableStates.tsx` (`LoadingState`, `ErrorState`, `EmptyState`)
  - toast → `components/ui/use-toast.ts` 또는 `sonner` (`components/ui/sonner.tsx`)
  - URL 상태 → `hooks/useUrlSearchParams.ts` 및 도메인별 훅(`useDocumentsSearchParams.ts` 등)
  - 권한 가드 → `components/auth-guard.tsx` 통과 구조 유지
  - 사이드바 메뉴 노출 → `components/app-sidebar.tsx` 의 `items` 배열 확장
  - axios 인스턴스 → `api/common/app-api.ts` 의 `appApi` 메서드 사용
- [ ] react-hook-form + zod 없이 폼을 직접 구현하고 있지 않은가
- [ ] TanStack Query 없이 `useEffect` + `useState` 로 API 를 직접 호출하고 있지 않은가
- [ ] `use<Domain>Queries.ts` 등 기존 훅 패턴을 따라 작성되었는가 (queryKey 구조, toast 위치, invalidateQueries 범위)
- [ ] 컴포넌트가 너무 커서 분리가 가능한가 (200줄 이상 단일 파일이면 분리 검토)
- [ ] 자동화 테스트 추가 가능 영역인가 (도입 후 `rules/testing.md` 5절 로드맵 참조)
- [ ] 커밋 메시지가 `type(scope): subject` 컨벤션을 따르는가 (`rules/git-workflow.md`)

---

## 5. 리뷰 출력 포맷

```markdown
## Pass 1 (머지 차단)

- [BLOCKER] `파일경로:라인` — 위반 내용
  근거: `.claude/principles/security.md` — XSS 위험
  수정 방향: ...

## Pass 2 (개선 권고)

- [SUGGESTION] `파일경로:라인` — 개선 제안
  근거: `.claude/principles/completeness-principle.md`
  제안: BaseTable 의 LoadingState 재사용

## 요약

- 머지 가능 여부: ✅ 즉시 머지 / ⏸️ 수정 후 머지 / ❌ 머지 보류
- 영향 도메인 × 레이어: document(api, hook, page), user(store)
- Pass 1 차단 항목: N건
- Pass 2 권고 항목: N건
```

---

## 6. fix-first 휴리스틱

- 단순/저위험 발견 (오타, import 순서, 포매팅, 미사용 변수) → `AUTO-FIXED` 표기 후 수정
- 모호/고위험 발견 (인증 흐름, authStore 변경, axios 인터셉터, 보안 관련) → `NEEDS INPUT` 표기 후 수정하지 않음

---

## 7. 마무리 점검 (사용자에게 묻기 전 셀프체크)

- 인증 관련 파일(`authStore.ts` / `axios-interceptor.tsx` / `auth-guard.tsx`) 변경이 있으면 인증 흐름 전체 재점검
- 사이드바(`app-sidebar.tsx`) 변경이 있으면 권한별 메뉴 노출 점검
- 새 도메인 페이지 추가 시 `app/` 라우트 등록 + 사이드바 items 추가 여부 확인
- 빌드 검증 명령(`npx tsc --noEmit`, `npm run lint`, `npm run build`) 이 PR 본문/머지 체크리스트에 표기되었는가
- PR 승인(`gh pr review --approve`) 은 **반드시 사용자 확인 후** 실행
</process>
