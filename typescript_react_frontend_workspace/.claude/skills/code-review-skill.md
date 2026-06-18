---
name: "Code Review"
description: "프론트엔드 PR 코드 리뷰 — Next.js + TypeScript 컨벤션 기반 체계적 분석, Pass 1 머지 차단 + Pass 2 개선 권고"
---

# Code Review Skill (프론트엔드)

## 목적

본 프로젝트 (Next.js / React / TypeScript / TanStack Query / Zustand / axios) 의 Pull Request 를
프로젝트 컨벤션과 원칙(`.claude/principles/`, `.claude/rules/`) 에 맞춰 일관되게 리뷰한다.

리뷰 기준 우선순위:
1. **`.claude/principles/*`** — 위반 시 머지 반려 (보안·상태관리·에러처리·API·구조·완결성·코드스타일)
2. **`.claude/rules/*`** — 일반 규칙 (컨벤션, 매핑, 공유 인스턴스, 테스팅, 디자인 패턴)

**이 스킬은 read-only 리뷰 전용이다. 코드를 수정하지 않는다.**
사용 도구: `Grep`, `Read`, `Bash(git diff ...)` 만 사용한다.

---

## 단계별 워크플로우

### Step 1. PR diff 수집

```bash
# PR 번호가 주어진 경우
gh pr view {PR_NUMBER} --json number,title,author,baseRefName,headRefName,additions,deletions,files
gh pr diff {PR_NUMBER}

# 로컬 브랜치 기반
git diff main...HEAD --name-only   # 변경 파일 목록
git diff main...HEAD --stat        # 파일별 통계
git diff main...HEAD               # 전체 diff
git log main..HEAD --oneline       # 커밋 목록
```

---

### Step 2. 변경 파일을 도메인 × 레이어 매트릭스에 매핑

변경 파일 경로를 아래 표에 대입해 영향 도메인과 레이어를 식별한다.

```
도메인 식별 규칙:
  api/*-api.ts              → api 레이어
  hooks/react-query/use*    → hook 레이어
  page/<domain>/            → page 레이어
  components/sheet/*Sheet   → component 레이어
  store/authStore.ts        → store 레이어
  model/dto/*-dto.ts        → model 레이어
  app/<route>/page.tsx      → route 레이어
  components/app-sidebar    → 전체 도메인 영향 (메뉴 노출)
  components/auth-guard     → 전체 도메인 영향 (인증 흐름)
  components/axios-interceptor → 전체 도메인 영향 (API 요청/응답)
  store/authStore.ts        → 전체 도메인 영향 (인증 상태)
```

영향 범위 표 작성 예시:
```
도메인       | api | hook | page | component | store | model | route
------------|-----|------|------|-----------|-------|-------|------
<domain-a>  |  ✅  |  ✅   |  ✅   |           |       |  ✅    |
<domain-b>  |     |      |      |           |  ✅    |       |
```

---

### Step 3. 관련 원칙 문서 로드

변경된 레이어에 따라 해당 원칙 문서를 Read 한다:

| 변경 레이어/파일 | 로드할 원칙 문서 |
|-----------------|----------------|
| `api/*`, `hooks/react-query/*` | `principles/api-conventions.md` |
| `store/authStore.ts`, `axios-interceptor.tsx`, `auth-guard.tsx` | `principles/security.md`, `principles/error-handling.md` |
| `store/*`, `hooks/react-query/*` | `principles/state-management.md` |
| `page/`, `components/sheet/*` | `principles/completeness-principle.md` |
| 모든 `.ts`, `.tsx` | `principles/code-style.md` |
| 전체 구조 변경 | `principles/project-structure.md` |

---

### Step 4. Pass 1 체크리스트 자동 점검

#### 4-1. 보안 점검 (`principles/security.md`)

```bash
# dangerouslySetInnerHTML 사용 여부
grep -r "dangerouslySetInnerHTML" --include="*.tsx" --include="*.ts" .

# localStorage/sessionStorage 에 토큰 저장 여부
grep -r "localStorage\|sessionStorage" --include="*.tsx" --include="*.ts" . | grep -i "token\|auth\|access"

# axios 직접 import (api/common/ 외부)
grep -r "import axios" --include="*.tsx" --include="*.ts" . | grep -v "api/common"

# NEXT_PUBLIC_ 환경변수 노출 위험
grep -r "NEXT_PUBLIC_" --include="*.tsx" --include="*.ts" . | grep -i "secret\|key\|password\|token"

# rel noopener 누락된 외부 링크
grep -r 'target="_blank"' --include="*.tsx" . | grep -v "noopener"
```

- [ ] `dangerouslySetInnerHTML` 에 사용자 입력·API 응답이 들어가지 않는가
- [ ] `authStore.ts` 가 메모리 전용인가 (`localStorage.setItem` 호출 없음)
- [ ] `NEXT_PUBLIC_*` 에 secret 없는가
- [ ] `api/common/` 외부에서 axios 직접 호출 없는가
- [ ] 외부 링크에 `rel="noopener noreferrer"` 있는가

#### 4-2. 상태 관리 점검 (`principles/state-management.md`)

```bash
# zustand store 에 서버 데이터 저장 여부
grep -r "useAuthStore\|create(" --include="*.ts" store/ | grep -v "authStore"

# useState 로 폼 상태 관리 (react-hook-form 우회)
grep -r "useState" --include="*.tsx" page/ | grep -i "form\|input\|field"

# invalidateQueries 누락 (mutation 파일에서 확인)
grep -r "useMutation" --include="*.ts" hooks/ -l | xargs grep -L "invalidateQueries"
```

- [ ] 서버 상태가 TanStack Query 에서만 관리되는가
- [ ] 폼 상태가 `react-hook-form` 으로 관리되는가
- [ ] mutation 성공 후 `invalidateQueries` 가 호출되는가

#### 4-3. 에러 처리 점검 (`principles/error-handling.md`)

```bash
# 빈 catch 블록 (에러 무시)
grep -r "catch" --include="*.tsx" --include="*.ts" . -A2 | grep -A2 "catch" | grep "^--$\|{$\|^  }$"

# toast 없이 catch 만 있는 경우 (hooks 파일 기준)
grep -r "onError" --include="*.ts" hooks/ -A3

# console.log 잔존
grep -r "console\.log\|console\.info" --include="*.tsx" --include="*.ts" . | grep -v "code-utils\|logger"
```

- [ ] `catch` 블록이 빈 블록이 아닌가
- [ ] API 에러 시 toast 로 사용자에게 알리는가
- [ ] `console.log` 디버그 코드가 잔존하지 않는가

#### 4-4. API 컨벤션 점검 (`principles/api-conventions.md`)

```bash
# api 모듈 외부에서 appAxios 직접 사용 여부
grep -r "appAxios" --include="*.tsx" --include="*.ts" . | grep -v "api/\|axios-interceptor"

# 쿼리 키 컨벤션 확인
grep -r "queryKey" --include="*.ts" hooks/react-query/ | head -20

# qs 사용 여부 (배열 파라미터)
grep -r "params\b" --include="*.ts" api/ | grep -v "qs\|serialize"
```

- [ ] `api/common/app-api.ts` 의 `appApi.get/post/put/patch/delete` 만 사용하는가
- [ ] queryKey 가 `[domain, 'list'|'detail', params]` 튜플 구조인가
- [ ] `ResponseModel<T>` 타입을 사용하는가 (`any` 응답 금지)

#### 4-5. 구조 점검 (`principles/project-structure.md`)

```bash
# model 타입과 api 클라이언트 혼재 여부
grep -r "AxiosInstance\|axios" --include="*.ts" model/ 2>/dev/null

# 역방향 import (api 가 page 를 import)
grep -r "from.*page/" --include="*.ts" api/ 2>/dev/null

# 'use client' 남용 여부
grep -r "'use client'" --include="*.tsx" . | grep -v "components/\|page/"
```

- [ ] `api/`, `hooks/`, `page/`, `model/` 단방향 의존이 유지되는가
- [ ] 새 도메인 파일이 올바른 폴더에 있는가
- [ ] 불필요한 `'use client'` 지시어가 없는가

#### 4-6. 완결성 점검 (`principles/completeness-principle.md`)

```bash
# BaseTable 사용 여부 (테이블 있는 page 파일 기준)
grep -r "BaseTable" --include="*.tsx" page/ -l

# isLoading prop 누락 여부
grep -r "<BaseTable" --include="*.tsx" page/ -A5 | grep -v "isLoading"

# toast 누락 (mutation 있는 hook 파일)
grep -r "useMutation" --include="*.ts" hooks/ -l | xargs grep -L "toast"

# LoadingState/ErrorState/EmptyState 직접 구현 여부 (BaseTable 우회)
grep -r "isLoading\b" --include="*.tsx" page/ | grep -v "BaseTable\|LoadingState"
```

- [ ] 테이블 컴포넌트에 로딩/에러/빈 상태가 있는가
- [ ] mutation 성공/실패 toast 가 있는가
- [ ] 새 도메인 추가 시 `app-sidebar.tsx` items 에 메뉴가 추가되었는가

#### 4-7. 코드 스타일 점검 (`principles/code-style.md`)

```bash
# any 타입 신규 도입
git diff main...HEAD | grep "^+" | grep ": any\b\|as any\b\|<any>"

# 파일명 kebab-case 위반 (page/*Page.tsx 예외)
git diff main...HEAD --name-only | grep -E "^(api|hooks|components|lib|model|store|common)/" | grep -v "[a-z0-9-]"

# cn() 미사용 className 합성
git diff main...HEAD | grep "^+" | grep 'className={`'
```

- [ ] `any` 신규 도입이 없는가
- [ ] 파일명이 kebab-case 인가 (`page/*Page.tsx` 예외)
- [ ] className 합성에 `cn()` (`lib/shadcn-utils.ts`) 를 사용하는가

---

### Step 5. 본 프로젝트 자산 재사용 강제 점검

아래 자산이 있음에도 새로 구현했다면 Pass 2 권고 또는 Pass 1 차단으로 분류한다.

| 자산 | 위치 | 강제 재사용 조건 |
|------|------|-----------------|
| axios 래퍼 (appApi 등) | `api/common/app-api.ts` | API 호출 시 항상 사용. 직접 axios 인스턴스 호출 금지 |
| axios 인스턴스 | `api/common/axios.ts` | `axios.create()` 신규 생성 금지 |
| 공용 테이블 컴포넌트 | `components/shared/table/` | 데이터 목록 테이블은 반드시 사용 |
| 로딩/에러/빈 상태 컴포넌트 | `components/shared/table/` | 공용 테이블 내부 또는 커스텀 테이블에서 재사용 |
| URL 검색 파라미터 훅 | `hooks/useUrlSearchParams.ts` | URL 기반 검색/필터/페이지 상태 관리 시 필수 |
| 도메인별 searchParams 훅 | `hooks/use<Domain>SearchParams.ts` | 동일 도메인 기존 훅 확장. 중복 생성 금지 |
| 인증 스토어 | `store/authStore.ts` | 인증 상태는 이 스토어만 사용. 별도 auth 상태 생성 금지 |
| 인증 가드 컴포넌트 | `components/auth-guard.tsx` | 보호 경로는 반드시 이 컴포넌트 통과 |
| toast | `components/ui/use-toast.ts` | toast 알림은 이 훅 사용 (또는 sonner) |
| sonner | `components/ui/sonner.tsx` | toast 알림 대안 (use-toast 와 혼용 금지) |
| `cn()` | `lib/shadcn-utils.ts` | className 합성 시 항상 사용 |
| QueryClient (Providers) | `components/providers.tsx` | QueryClient 신규 생성 금지. 기존 Provider 사용 |
| 사이드바 메뉴 items | `components/app-sidebar.tsx` | 새 도메인 메뉴 추가 시 `items` 배열에 항목 추가 |

```bash
# 자산 재사용 현황 확인
grep -r "from.*api/common/app-api" --include="*.ts" api/ | wc -l
grep -r "BaseTable" --include="*.tsx" page/ | wc -l
grep -r "useUrlSearchParams" --include="*.ts" hooks/ | wc -l
grep -r "useAuthStore" --include="*.tsx" --include="*.ts" . | grep -v "authStore.ts" | wc -l
```

---

### Step 6. 결과 출력

```markdown
# PR Review: [PR 제목 또는 변경 범위]

## 변경 요약
- X commits, Y files (+Z lines, -W lines)
- 주요 변경: [도메인/레이어 단위 요약]

## 영향 범위 (도메인 × 레이어)
| 도메인 | api | hook | page | component | store | model |
|--------|-----|------|------|-----------|-------|-------|
| ...    |  ✅  |  ✅   |      |           |       |       |

## 종합 평가
- 평가: X/5
- 머지 가능 여부: ✅ 즉시 머지 / ⏸️ 수정 후 머지 / ❌ 머지 보류

## Pass 1 (머지 차단)

### [BLOCKER] 파일:라인 — 위반 내용
근거: `.claude/principles/security.md`
문제: ...
수정 방향: ...

## Pass 2 (개선 권고)

### [SUGGESTION] 파일:라인 — 개선 제안
근거: `.claude/principles/completeness-principle.md`
제안: BaseTable 의 LoadingState 재사용 가능

## 자산 재사용 점검 결과
- appApi 사용: ✅
- BaseTable 재사용: ✅
- useUrlSearchParams 사용: ✅
- invalidateQueries 호출: ⚠️ useXxxMutation onSuccess 에서 누락

## 빌드 / 타입 검증
- `npx tsc --noEmit`: [통과/실패]
- `npm run lint`: [통과/실패]
- `npm run build`: [통과/실패]

## 머지 체크리스트
- [ ] Pass 1 항목 모두 수정
- [ ] `npx tsc --noEmit` 통과
- [ ] `npm run lint` 통과
- [ ] 수동 QA 체크리스트(`rules/testing.md` 2절) 통과
- [ ] 커밋 메시지 `type(frontend): subject` 컨벤션 준수

---
🤖 *This review was generated by [Claude Code](https://claude.ai/code)*
```

---

## 피드백 등급

| 등급 | 의미 | 머지 조건 |
|------|------|-----------|
| 🔴 **BLOCKER** | 머지 불가 (보안 위반, 인증 우회, any 신규 도입, API 인스턴스 우회) | 수정 후 재리뷰 |
| 🟡 **SUGGESTION** | 수정 권고 (자산 미재사용, invalidation 누락, 완결성 미흡) | 수정 또는 사유 설명 |
| 🟢 **CONSIDER** | 개선 제안 (네이밍, 가독성, 코드 분리 기회) | 머지 후 개선 가능 |

| 점수 | 조건 |
|------|------|
| 5/5 | BLOCKER, SUGGESTION 없음. 즉시 머지 |
| 4/5 | SUGGESTION 1~2건, BLOCKER 없음 |
| 3/5 | BLOCKER 1건 또는 SUGGESTION 3건+ |
| 2/5 | BLOCKER 다수 또는 principles 위반 |
| 1/5 | 재작성 필요 |

---

## 빠른 참조 커맨드

```bash
# PR 정보
gh pr list
gh pr view {PR_NUMBER} --json number,title,author,additions,deletions,files
gh pr diff {PR_NUMBER}

# 변경 통계
git diff main...HEAD --stat
git diff main...HEAD --name-only
git log main..HEAD --oneline

# 타입/린트 검증
npx tsc --noEmit
npm run lint
npm run build

# 자산 사용 현황 grep
grep -r "BaseTable" --include="*.tsx" page/
grep -r "useUrlSearchParams" --include="*.ts" hooks/
grep -r "invalidateQueries" --include="*.ts" hooks/react-query/
grep -r "appApi\." --include="*.ts" api/
grep -r ": any" --include="*.ts" --include="*.tsx" .

# 리뷰 포스팅
gh pr comment {PR_NUMBER} --body "$(cat review.md)"
gh pr review {PR_NUMBER} --approve --body "리뷰 내용"       # 사용자 확인 필수
gh pr review {PR_NUMBER} --request-changes --body "리뷰 내용"
```
