# 프론트엔드 Next.js 코드 컨벤션

> **App Router 규약, Server/Client 컴포넌트, TanStack Query, react-hook-form, zod** 기반 컨벤션.
> 라이브러리 상세는 `library-stack.md`, 아키텍처 의존 방향은 `architecture.md` 를 참조.

---

## 프로젝트 구조 요약

```
frontend/
├── app/          # App Router 라우트 진입점 (page.tsx / layout.tsx / …)
├── page/         # 화면 컴포넌트 (비즈니스 로직 포함)
├── components/   # 재사용 컴포넌트 (shared / sheet / ui / charts)
├── hooks/        # 커스텀 훅 (react-query / useXxxSearchParams / …)
├── api/          # axios API 모듈
├── model/        # 타입·DTO·zod 스키마
├── store/        # zustand 전역 상태
├── common/       # 상수·유틸·app-type
└── lib/          # shadcn-utils (cn 함수)
```

상세 의존 방향과 폴더 역할은 `architecture.md` 참조.

---

## App Router 규약

### 특수 파일 역할

| 파일 | 위치 | 역할 |
|------|------|------|
| `layout.tsx` | `app/layout.tsx` | 루트 레이아웃. `AppLayout` 마운트, 폰트·전역 스타일 설정. 리렌더 없이 유지. |
| `page.tsx` | `app/[route]/page.tsx` | 라우트 진입점. **`page/` 컴포넌트를 import해서 렌더하는 것이 전부.** 비즈니스 로직 금지. |
| `not-found.tsx` | `app/not-found.tsx` | 전역 404 처리. |
| `loading.tsx` | (미사용, 필요 시 추가) | 라우트 수준 Suspense 폴백. |
| `error.tsx` | (미사용, 필요 시 추가) | 라우트 수준 에러 바운더리. |

```tsx
// app/(<group>)/<route>/page.tsx — 올바른 패턴
import { XxxPage } from "@/page/<domain>/XxxPage"
export default function Page() {
  return <XxxPage />
}

// 금지 — page.tsx 에 비즈니스 로직 작성
export default function Page() {
  const { data } = useXxxList(...)   // ✕ 훅을 직접 page.tsx 에
  return <table>...</table>          // ✕ UI 를 직접 page.tsx 에
}
```

### Route Group 패턴

`(<group>)` 처럼 괄호로 묶인 폴더는 **URL에 포함되지 않는 논리 그룹**이다.

```
app/(<group-a>)/<route-a>/page.tsx  → URL: /<route-a>
app/(<group-b>)/<route-b>/page.tsx  → URL: /<route-b>
```

그룹 내 `layout.tsx`를 두지 않으면 그룹화 목적으로만 동작한다. 향후 그룹별 레이아웃이 필요하면 그룹 폴더 아래에 `layout.tsx`를 추가하면 된다.

---

## Server Component vs Client Component

### 기본 원칙

**App Router에서 모든 컴포넌트는 기본적으로 Server Component다.**
클라이언트 기능(훅, 브라우저 API, 이벤트 핸들러, Radix UI 상태)이 필요할 때만 파일 최상단에 `'use client'` 를 선언한다.

### `'use client'` 가 필요한 경우

| 상황 | 예시 |
|------|------|
| React 훅 사용 (`useState`, `useEffect`, `useRef` 등) | 검색 필터, 모달 상태 관리 |
| TanStack Query (`useQuery`, `useMutation`) | 모든 `hooks/react-query/*.ts` |
| zustand 스토어 접근 | `useAuthStore()` |
| react-hook-form | 모든 폼 컴포넌트 |
| Radix UI 컴포넌트 | Dialog, Sheet, DropdownMenu 등 |
| 브라우저 API (`window`, `localStorage`, `document`) | axios-interceptor, auth-guard |
| Next.js 클라이언트 훅 (`useRouter`, `usePathname`, `useSearchParams`) | 검색 파라미터 동기화, 라우팅 |
| 이벤트 핸들러가 있는 인터랙티브 컴포넌트 | 버튼 클릭, 폼 제출 |

```tsx
// 올바른 위치 — 파일 최상단 첫 줄
'use client'

import { useState } from 'react'
// ...
```

### 현재 프로젝트의 컴포넌트 분류

본 프로젝트는 **클라이언트 페칭 위주** 구조다. `page/` 의 모든 Page 컴포넌트와 `components/sheet/`, `components/shared/table/`, `hooks/react-query/` 는 모두 Client Component(`'use client'`)다.

| 파일/폴더 | 종류 | 이유 |
|-----------|------|------|
| `app/layout.tsx` | Server Component | 폰트·메타데이터만, 훅 없음 |
| `app/[route]/page.tsx` | Server Component | Page 컴포넌트 렌더만 |
| `components/app-layout.tsx` | Client (`'use client'`) | `usePathname()` 사용 |
| `components/auth-guard.tsx` | Client (`'use client'`) | `useEffect`, `useRouter`, zustand |
| `components/axios-interceptor.tsx` | Client (`'use client'`) | `useEffect`, `useRouter`, zustand |
| `components/providers.tsx` | Client (`'use client'`) | `useState(QueryClient)` |
| `page/**/*.tsx` | Client (`'use client'`) | 훅·상태·이벤트 |
| `hooks/**/*.ts` | Client 전용 | TanStack Query, 브라우저 API |
| `components/ui/*.tsx` | Client (`'use client'`) | Radix UI 상태 |

---

## 레이아웃 계층

```
app/layout.tsx
└── <AppLayout>                        # components/app-layout.tsx
    ├── <Providers>                    # components/providers.tsx
    │   └── QueryClientProvider        # TanStack Query
    ├── <AxiosInterceptor>             # components/axios-interceptor.tsx (effect only)
    ├── <Toaster>                      # components/ui/toaster.tsx (전역 1개)
    └── <AuthGuard>                    # components/auth-guard.tsx
        └── <SidebarProvider>          # components/ui/sidebar.tsx
            ├── <AppSidebar>           # components/app-sidebar.tsx
            └── <SidebarInset>
                ├── <SiteHeader>       # components/site-header.tsx
                └── {children}         # app/[route]/page.tsx → page/**/*.tsx
```

- 로그인 페이지(`/login`)는 `AppLayout` 내부에서 `isLoginPage` 분기로 사이드바 없이 단독 렌더.
- `<Providers>`, `<AxiosInterceptor>`, `<Toaster>` 는 로그인 페이지에서도 마운트된다.
- `<AuthGuard>` 는 인증 필요 페이지에만 마운트된다.

---

## 데이터 페칭

### 클라이언트 페칭 (기본)

본 프로젝트는 **TanStack Query 기반 클라이언트 페칭**이 기본이다.

```ts
// hooks/react-query/useXxxQueries.ts
export const useXxxList = (params: XxxDto.SearchParams) =>
  useQuery({
    queryKey: xxxKeys.list(params),
    queryFn: async () => {
      const response = await XxxApi.search(params)
      return response.data
    }
  })
```

```tsx
// page/<domain>/XxxPage.tsx
const { data, isLoading, isError } = useXxxList(appliedValues)
```

### Server Component fetch (필요 시)

Server Component에서 `fetch()`를 직접 사용하는 경우:
- `NEXT_PUBLIC_` 환경변수 대신 서버 전용 환경변수를 사용할 수 있다.
- `axios` 인터셉터가 동작하지 않는다 — 직접 Authorization 헤더를 구성해야 한다.
- 현재 프로젝트에서는 Server Component fetch를 사용하지 않는다. 도입 시 팀 합의 필요.

---

## 명명 규약

### 파일명

| 대상 | 규칙 | 예시 |
|------|------|------|
| App Router 특수 파일 | Next.js 규정 준수 | `page.tsx`, `layout.tsx`, `not-found.tsx` |
| Page 컴포넌트 | `PascalCase.tsx` (현 관행) | `XxxPage.tsx` |
| Page 내 서브 컴포넌트 | `PascalCase.tsx` | `XxxTable.tsx`, `XxxSearchFilters.tsx` |
| 액션 파일 | `[domain]-actions.tsx` (kebab-case) | `<domain>-actions.tsx` |
| 재사용 컴포넌트 | `kebab-case.tsx` | `app-layout.tsx`, `auth-guard.tsx`, `axios-interceptor.tsx` |
| shadcn/ui 컴포넌트 | `kebab-case.tsx` | `button.tsx`, `dialog.tsx`, `smart-sheet.tsx` |
| API 모듈 | `[domain]-api.ts` | `<domain>-api.ts` |
| React Query 훅 | `use[Domain]Queries.ts` (PascalCase) | `use<Domain>Queries.ts` |
| 검색 파라미터 훅 | `use[Domain]SearchParams.ts` | `use<Domain>SearchParams.ts` |
| DTO 타입 파일 | `[domain]-dto.ts` | `<domain>-dto.ts` |
| zod 스키마 파일 | `[domain]-schema.ts` | `<domain>-schema.ts` |

> **파일명 혼재 현황**: `page/` 폴더는 `DocumentsPage.tsx`(PascalCase)와 `documents-actions.tsx`(kebab-case)가 혼재한다.
> 결정(이 프로젝트 채택): `page/*Page.tsx` 는 PascalCase 기존 관행 유지. 그 외 신규 파일은 kebab-case 권고.
> 장기적으로 `documents-page.tsx` 형태로 점진 전환 예정.

### 컴포넌트 명명

```tsx
// Page 컴포넌트 — export named function
export function XxxPage() { ... }

// 재사용 컴포넌트 — export named function
export function BaseTable<T>({ ... }: BaseTableProps<T>) { ... }

// shadcn/ui 컴포넌트 — export named (shadcn CLI 생성 관행)
export const Button = React.forwardRef<...>(...)
```

- **default export 는 `app/[route]/page.tsx` 에서만** 사용한다 (Next.js App Router 요구사항).
- 나머지는 모두 named export.

### 훅 명명

```ts
// React Query 훅 — useXxx (동사 포함)
useXxxList(params)
useXxxDetail(id)
useEditXxx()      // mutation

// URL 파라미터 훅 — useXxxSearchParams
useXxxSearchParams()

// 기타 훅
useUrlSearchParams(options)   // 공용 기반 훅
useMobile()
useTheme()
```

### 이벤트 핸들러 명명

```tsx
// 이벤트 핸들러는 handle + 동사 형태
const handleSubmit = (data: FormData) => { ... }
const handleDelete = (id: string) => { ... }
const handlePageChange = (page: number) => { ... }

// props 로 전달할 때는 on + 동사 형태
<DocumentsTable onRowClick={handleRowClick} onDelete={handleDelete} />
```

---

## 폼 작성 규칙

### 기본 구조

```tsx
'use client'
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { documentEditSchema, DocumentEditForm } from "@/model/schema/document-schema"

export function DocumentForm({ defaultValues, onSubmit }: DocumentFormProps) {
  const form = useForm<DocumentEditForm>({
    resolver: zodResolver(documentEditSchema),
    defaultValues
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <CustomInputForm control={form.control} name="name" label="이름" />
        <FormActions isSubmitting={form.formState.isSubmitting} />
      </form>
    </Form>
  )
}
```

### 폼 규칙

- 스키마는 `model/schema/xxx-schema.ts` 에 선언. 컴포넌트 파일 안 인라인 금지.
- 폼 필드는 `components/ui/form/` 의 래퍼 컴포넌트 사용:
  - `CustomInputForm` — 텍스트 입력
  - `CustomSelectForm` — 단일 셀렉트
  - `CustomMultiSelectForm` — 멀티 셀렉트
  - `CustomSearchableSelectForm` — 검색 가능 셀렉트
  - `CustomTextareaForm` — 텍스트에어리어
- 제출 버튼은 `components/sheet/form/common/FormActions.tsx` 재사용.
- 폼 상태(`isDirty`, `isSubmitting`, `errors`)는 react-hook-form 에서 자동 관리 — `useState`로 별도 트래킹 금지.

---

## 뮤테이션 규칙

```ts
// hooks/react-query/useXxxQueries.ts
export const useEditXxx = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ id, form }: { id: string; form: XxxDto.EditForm }) =>
      XxxApi.edit(id, form),
    onSuccess: (_, { id }) => {
      toast({ title: "성공", description: "수정되었습니다." })
      queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
      queryClient.invalidateQueries({ queryKey: xxxKeys.detail(id) })
    },
    onError: (error: Error) => {
      toast({ title: "오류", description: error.message, variant: "destructive" })
    }
  })
}
```

- 뮤테이션 성공 시 **관련 쿼리를 반드시 invalidate** 한다.
- 성공/실패 toast 는 뮤테이션 훅의 `onSuccess`/`onError` 에서 처리 — 컴포넌트에서 중복 처리 금지.
- 낙관적 업데이트가 필요하면 `onMutate` + `onSettled` 패턴 사용.

---

## URL 파라미터 동기화

목록 화면의 검색 조건은 URL 쿼리스트링에 반영해 새로고침·뒤로가기를 지원한다.

```ts
// hooks/useXxxSearchParams.ts 패턴
export function useXxxSearchParams() {
  return useUrlSearchParams<XxxSearchState>({
    defaultValues: { keyword: '', page: 1, size: DEFAULT_PAGE_SIZE },
    serialize: (values) => ({ keyword: values.keyword, page: String(values.page) }),
    deserialize: (params) => ({ keyword: params.get('keyword') ?? '', page: Number(params.get('page') ?? 1) }),
    manualSearch: true   // 검색 버튼 클릭 시 적용
  })
}
```

- `manualSearch: true` — 검색 버튼 클릭 시 `applySearch()` 호출로 URL 갱신 + 쿼리 트리거.
- `manualSearch: false` (기본) — 값 변경 즉시 URL 갱신 (셀렉트·페이지네이션 등).
- 페이지네이션은 `updateAppliedValues({ page })` 로 즉시 적용.
- 공용 기반 훅 `hooks/useUrlSearchParams.ts` 를 직접 page 에서 쓰지 않는다. 반드시 도메인 전용 훅을 만든다.

---

## 공용 컴포넌트 사용 규칙

### BaseTable

```tsx
// components/shared/table/BaseTable.tsx 인터페이스
<BaseTable
  columns={columns}            // 컬럼 정의 배열
  data={data?.list}            // 데이터 배열
  isLoading={isLoading}        // 로딩 상태
  pagination={{ total, page, size, onPageChange }}
  onRowClick={handleRowClick}  // 행 클릭
/>
```

- 모든 목록 테이블은 `BaseTable`을 기반으로 한다. 직접 `<table>` HTML 을 page 에 작성 금지.
- 로딩 / 빈 상태 / 에러 상태 UI는 `components/shared/table/TableStates.tsx` 에 위임.

### Sheet (사이드 패널)

- 도메인별 상세/편집 패널은 `components/sheet/XxxSheet.tsx` 패턴.
- 데이터 로딩은 `components/sheet/SheetDataLoader.tsx` 를 사용.
- 폼은 `components/sheet/form/XxxForm.tsx` 로 분리.

### 토스트

```ts
// 현 프로젝트 관행 — @/components/ui/use-toast 의 toast()
import { toast } from "@/components/ui/use-toast"

toast({ title: "성공", description: "저장되었습니다." })
toast({ variant: "destructive", title: "오류", description: error.message })
```

- 전역 `<Toaster>`는 `components/app-layout.tsx`에 단 하나만 마운트한다. 추가 마운트 금지.

---

## 환경변수

### 사용 원칙

```bash
# .env.local (로컬 개발 — Git 커밋 금지)
NEXT_PUBLIC_API_BASE_URL=http://localhost:7072

# .env.production (프로덕션)
NEXT_PUBLIC_API_BASE_URL=https://api.example.com
```

| 규칙 | 설명 |
|------|------|
| `NEXT_PUBLIC_` prefix | 클라이언트 번들에 포함됨. 브라우저에서 접근 가능. |
| prefix 없음 | 서버 전용. 브라우저에서 `undefined`. |
| 시크릿 금지 | `NEXT_PUBLIC_*` 에 JWT 서명키, DB 비밀번호, API 시크릿 절대 불가 |
| `process.env.NEXT_PUBLIC_API_BASE_URL` | `api/common/axios.ts` 에서만 참조. 다른 곳에서 직접 읽기 금지 |

---

## 빌드 및 개발

### 커맨드

```bash
npm run dev     # next dev --webpack (개발 서버, watchOptions 커스텀 적용)
npm run build   # next build --webpack (프로덕션 빌드)
npm run start   # next start (프로덕션 서버)
npm run lint    # eslint
```

### `--webpack` 사용 이유 (선택적)

`next.config.ts` 에서 webpack의 `watchOptions.ignored` 를 커스터마이즈해 특정 아티팩트 변경이 개발 서버를 불필요하게 재시작하지 않도록 설정할 수 있다.

```ts
// next.config.ts (예시)
const nextConfig: NextConfig = {
  webpack: (config, { dev }) => {
    if (dev) {
      config.watchOptions = {
        ...config.watchOptions,
        ignored: ["**/node_modules/**", "**/.next/**", "**/.git/**",
                  "**/*.png", "**/*.jpg", "**/*.jpeg", "**/*.log"]
      }
    }
    return config
  }
}
```

이 설정이 필요한 경우 `next dev` 대신 `next dev --webpack` 을 사용한다. Turbopack은 `watchOptions` 커스텀을 지원하지 않는다.

### `tsconfig.json` path alias

```json
{ "compilerOptions": { "paths": { "@/*": ["./*"] } } }
```

모든 import는 `@/` alias 를 사용한다. 상대경로(`../../`)는 금지.

---

## TypeScript 스타일

### 기본 원칙

- **`any` 타입 사용 금지** — `unknown` + 타입 가드 또는 명시적 타입으로 대체.
- **`as any` 캐스팅 금지** — 타입 에러를 회피하지 않는다.
- 함수형 컴포넌트만 사용 — 클래스 컴포넌트 신규 작성 금지.
- `interface` vs `type`: 컴포넌트 props 는 `interface`, union/intersection 은 `type`.

```ts
// 권장 — interface for props
interface XxxTableProps {
  data: XxxDto.Summary[]
  isLoading: boolean
  onRowClick: (id: string) => void
}

// 권장 — type for union
type StatusVariant = 'success' | 'warning' | 'error'
```

- `React.FC<Props>` 사용 금지 — props 타입을 직접 매개변수에 선언한다.

```tsx
// 권장
export function XxxTable({ data, isLoading, onRowClick }: XxxTableProps) { ... }

// 금지
export const XxxTable: React.FC<XxxTableProps> = ({ data, ... }) => { ... }
```

- `export default` 는 `app/[route]/page.tsx` 에서만 허용.

### import 순서

```ts
// 1. React / Next.js
import { useState, useEffect } from 'react'
import { useRouter } from 'next/navigation'

// 2. 외부 라이브러리
import { useQuery } from '@tanstack/react-query'
import { z } from 'zod'

// 3. 내부 — @/ alias 사용
import { useXxxList } from '@/hooks/react-query/useXxxQueries'
import { XxxApi } from '@/api/<domain>-api'
import { XxxDto } from '@/model/dto/<domain>-dto'
import { cn } from '@/lib/shadcn-utils'
```

- 와일드카드 import (`import * from`) 금지. 단, DTO namespace 패턴(`import { DocumentDto } from ...`)은 허용.

---

## 에러 처리

- **API 에러 기본 처리**: `axios-interceptor.tsx` 의 `handleResponseError` 가 `showErrorPopup: true` 옵션 기준으로 toast 표시.
- **뮤테이션 에러**: `useMutation` 의 `onError` 에서 도메인별 메시지 toast.
- **401 Unauthorized**: 인터셉터에서 자동 토큰 갱신 → 실패 시 `logout()` + `/login` 리다이렉트.
- **렌더 에러**: `app/error.tsx` 를 추가해 React ErrorBoundary로 감싼다 (현재 미구현, 필요 시 추가).
- **컴포넌트에서 try/catch 로 toast 직접 표시 금지** — 뮤테이션 훅의 `onError` 에서 처리한다.

---

## 보안 주의사항

- `dangerouslySetInnerHTML` 사용 금지 — `app/layout.tsx` 의 테마 플리커 방지 스크립트가 유일한 예외이며 신규 추가는 금지.
- `accessToken` 을 `localStorage` / `sessionStorage` / 쿠키에 저장 금지 — zustand 메모리만 사용.
- `NEXT_PUBLIC_*` 변수에 시크릿 저장 금지 — 클라이언트 번들에 노출된다.
- 외부 링크는 `target="_blank" rel="noopener noreferrer"` 추가.

---

## 코드 포매팅 규칙

- 들여쓰기: 2 spaces (프로젝트 기존 스타일).
- 세미콜론: 없음 (ESLint `eslint-config-next` 기본).
- 따옴표: 쌍따옴표 (`"`) — JSX attribute 및 import 문.
- 줄 길이: 120자 초과 시 줄바꿈 권고.
- JSX 다중 props: 한 줄에 2개 이상이면 세로로 정렬.

```tsx
// 권장
<BaseTable
  columns={columns}
  data={data?.list}
  isLoading={isLoading}
  pagination={pagination}
/>

// 지양 (props 가 많을 때 한 줄로)
<BaseTable columns={columns} data={data?.list} isLoading={isLoading} pagination={pagination} />
```
