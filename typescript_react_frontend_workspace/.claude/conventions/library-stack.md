# 프론트엔드 라이브러리 스택 가이드

> Next.js App Router + React 생태계 기준으로 작성됐다.
> `package.json` 의 실제 의존성을 기준으로 각 라이브러리의 선택 이유와 사용 규칙을 정리한다.

---

## 스택 요약

| 카테고리 | 라이브러리 | 선택 이유 |
|----------|-----------|-----------|
| 프레임워크 | `next` | App Router, Server/Client Component, 파일 기반 라우팅 |
| UI 런타임 | `react`, `react-dom` | Concurrent Features, Actions API |
| 언어 | `typescript` ^5 | 타입 안전성, IDE 지원 |
| 서버 상태 | `@tanstack/react-query` ^5 | 선언적 캐시·무효화·낙관적 업데이트, SSR 지원 |
| HTTP 클라이언트 | `axios` | 인터셉터, 타입 선언 병합(`showErrorPopup`), 취소 토큰 |
| URL 직렬화 | `qs` | 배열 파라미터 직렬화(`arrayFormat: 'repeat'`) |
| 클라이언트 상태 | `zustand` | 경량 전역 상태, 메모리 토큰 관리 |
| 폼 관리 | `react-hook-form` | 비제어 컴포넌트 기반, 성능 우수 |
| 폼 검증 | `zod` + `@hookform/resolvers` | 런타임 타입 검증, TypeScript 타입 추론 |
| UI 프리미티브 | Radix UI + `radix-ui` | 접근성(WAI-ARIA), 헤드리스, shadcn 패턴과 결합 |
| 스타일링 | `tailwindcss` ^4, `@tailwindcss/postcss` ^4 | Utility-first, JIT, 디자인 토큰 |
| 스타일 유틸 | `class-variance-authority`, `clsx`, `tailwind-merge` | cva로 변형(variant) 관리, cn()으로 클래스 충돌 해결 |
| 애니메이션 | `tw-animate-css` | Tailwind 기반 CSS 애니메이션 유틸 |
| 아이콘 | `lucide-react` | SVG 아이콘 컴포넌트, tree-shaking 지원 |
| 명령 팔레트 | `cmdk` | Command 컴포넌트 (keyboard-first UI) |
| 토스트 | `sonner` | 가볍고 스타일 커스터마이즈 가능한 toast |
| 테마 | `next-themes` | 다크/라이트 모드, SSR 플리커 방지 |
| 차트 | `recharts` | React 네이티브 차트, `components/charts/` 에서 래핑 |
| 셀렉트 | `react-select` | 멀티셀렉트, 검색 가능 셀렉트 |
| CSS 전처리 | `sass` | `app/globals.scss` 전역 스타일 |
| 쿼리 개발도구 | `@tanstack/react-query-devtools` | 개발 환경 캐시 시각화 |

---

## 카테고리별 상세 가이드

### 1. 프레임워크 — Next.js + React

#### 왜 이 조합인가
- App Router로 레이아웃 공유(`app/layout.tsx`), Route Group(`(<group>)`), 파일 기반 라우팅을 활용.
- React Concurrent Features로 UX 개선 가능성 확보.

#### 빌드 & 개발 커맨드
```bash
npm run dev    # next dev (개발 서버)
npm run build  # next build (프로덕션 빌드)
npm run start  # next start
npm run lint   # eslint
```

> `next.config.ts`에서 webpack의 `watchOptions.ignored`를 커스터마이즈해 특정 아티팩트 변경이 Hot Reload를 트리거하지 않도록 설정할 수 있다. 이 경우 `--webpack` 플래그를 스크립트에 추가해 Turbopack 대신 webpack을 명시적으로 사용한다.

---

### 2. 서버 상태 — TanStack Query 5 (`@tanstack/react-query`)

#### 왜 TanStack Query인가
- 서버 상태(API 응답)와 클라이언트 상태(zustand)를 명확히 분리.
- `staleTime`, `retry`, `refetchOnWindowFocus`를 전역 기본값으로 설정 (`components/providers.tsx`).
- `invalidateQueries`로 뮤테이션 성공 후 캐시 자동 무효화.

#### 기본 설정 (`components/providers.tsx`)
```ts
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,      // 1분간 fresh 유지
      retry: 1,                   // 실패 시 1회 재시도
      refetchOnWindowFocus: false // 포커스 복귀 시 자동 재요청 비활성화
    }
  }
})
```

#### 쿼리 키 규칙
```ts
// hooks/react-query/useXxxQueries.ts 패턴
export const xxxKeys = {
  all: ['<domain>'] as const,
  lists: () => [...xxxKeys.all, 'list'] as const,
  list: (params: XxxDto.SearchParams) => [...xxxKeys.lists(), params] as const,
  detail: (id: string) => [...xxxKeys.all, 'detail', id] as const,
}
```
- 최상위 키는 도메인 이름 문자열 (`'documents'`, `'prompts'` 등).
- 하위 키를 배열로 계층화해 `invalidateQueries({ queryKey: documentKeys.lists() })`로 목록 전체 무효화.

#### 사용 규칙
- `useQuery`·`useMutation`·`useInfiniteQuery`는 반드시 `hooks/react-query/useXxxQueries.ts` 안에서만 선언한다. `page/`나 `components/`에서 직접 선언 금지.
- 뮤테이션 성공 콜백(`onSuccess`)에서 `invalidateQueries`로 관련 목록과 상세 캐시를 무효화.
- 에러 콜백(`onError`)에서 `toast` (destructive)로 사용자에게 알림.

**대신 쓰지 말 것**: `useEffect` + `useState`로 직접 API 호출 — TanStack Query가 이미 처리한다. SWR — 이 프로젝트는 TanStack Query로 통일.

---

### 3. HTTP 클라이언트 — axios + qs

#### 왜 axios인가
- TypeScript 선언 병합(declaration merging)으로 `AxiosRequestConfig`에 `showErrorPopup`, `showLoading` 커스텀 옵션 추가 가능.
- 인터셉터로 토큰 주입·갱신·에러 처리를 한 곳(`components/axios-interceptor.tsx`)에 집중.

#### 단일 인스턴스 (`api/common/axios.ts`)
```ts
export const appAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
  headers: { Accept: "application/json" },
  paramsSerializer: (params) => qs.stringify(params, { arrayFormat: 'repeat', skipNulls: true }),
  showErrorPopup: true,
  showLoading: false
})
```
- **모든 API 호출은 공용 axios 인스턴스를 경유해야 한다.** `axios.create()`를 새로 호출하거나, 날 `fetch()`를 직접 사용 금지.
- `baseURL`은 `NEXT_PUBLIC_API_BASE_URL` 환경변수로만 주입.

#### `qs` 직렬화 규칙
- 배열 파라미터는 `arrayFormat: 'repeat'` — `roles=VALUE_A&roles=VALUE_B` 형태.
- `skipNulls: true` — `undefined`/`null` 파라미터는 자동 제외.

**대신 쓰지 말 것**: `fetch()` 직접 호출 — 인터셉터·에러 처리가 누락된다. `URLSearchParams`로 배열 직렬화 — `qs`로 통일.

---

### 4. 클라이언트 상태 — zustand 5

#### 왜 zustand인가
- React Context 대비 리렌더 최적화 우수. 간결한 API.
- 인증 토큰처럼 **메모리에만 유지해야 하는 상태**에 적합.

#### 스토어 구조
```
store/
└── authStore.ts   — accessToken, tokenInfo, isRefreshing
                     setToken / clearToken / logout / isAuthenticated / getAccessToken
                     (추가 전역 상태가 필요하면 store/ 아래에 별도 파일로 추가)
```

#### 사용 규칙
- **`accessToken`은 메모리에만 저장** — `localStorage.setItem`, `sessionStorage.setItem`, 쿠키 직접 쓰기 금지.
- 스토어는 `store/` 폴더에만 둔다.
- 새 전역 상태가 필요하면 스토어 추가 전에 "서버 상태(TanStack Query)인가? URL 상태인가? 폼 상태인가?"를 먼저 판단 — zustand는 마지막 수단.

**대신 쓰지 말 것**: Redux — 이 프로젝트는 zustand로 통일. `useState`를 props drilling으로 내리는 패턴 — 전역 상태가 필요하면 zustand.

---

### 5. 폼 관리 — react-hook-form 7 + zod 4 + @hookform/resolvers

#### 왜 이 조합인가
- `react-hook-form`: 비제어 컴포넌트 기반으로 불필요한 리렌더 최소화. 복잡한 폼 상태(`isDirty`, `isSubmitting`, `errors`) 자동 관리.
- `zod`: 런타임 검증 + TypeScript 타입 추론. `z.infer<typeof schema>`로 폼 타입 자동 생성.
- `@hookform/resolvers`: `zodResolver(schema)` 한 줄로 zod와 연동.

#### 패턴
```ts
// model/schema/<domain>-schema.ts
import { z } from "zod"
export const xxxEditSchema = z.object({
  fieldName: z.string().min(1, "값을 입력해주세요")
})
export type XxxEditForm = z.infer<typeof xxxEditSchema>
```

```tsx
// components/sheet/form/XxxForm.tsx
const form = useForm<XxxEditForm>({
  resolver: zodResolver(xxxEditSchema),
  defaultValues: { fieldName: "" }
})
```

#### 사용 규칙
- 스키마는 반드시 `model/schema/xxx-schema.ts`에 선언한다. 컴포넌트 파일 안에 인라인으로 작성 금지.
- `z.infer<typeof schema>` 로 추론된 타입을 폼 타입으로 사용하고, 별도 interface 중복 선언하지 않는다.
- 폼 필드는 `components/ui/form/` 의 `CustomInputForm`, `CustomSelectForm`, `CustomTextareaForm` 등 래퍼를 사용.

**대신 쓰지 말 것**: Formik — 이 프로젝트는 react-hook-form으로 통일. yup — zod로 통일. 수동 `useState`로 폼 값 관리 — react-hook-form이 이미 처리한다.

---

### 6. UI 프리미티브 — Radix UI + shadcn 패턴

#### 왜 Radix UI인가
- 접근성(WAI-ARIA), 키보드 내비게이션, 포커스 트랩이 기본 내장.
- 헤드리스(unstyled) — Tailwind로 완전히 커스터마이즈 가능.

#### 사용 중인 Radix UI 패키지
```
@radix-ui/react-checkbox        — components/ui/checkbox.tsx
@radix-ui/react-dialog          — components/ui/dialog.tsx
@radix-ui/react-dropdown-menu   — components/ui/dropdown-menu.tsx
@radix-ui/react-label           — components/ui/label.tsx
@radix-ui/react-popover         — components/ui/popover.tsx
@radix-ui/react-select          — components/ui/select.tsx
@radix-ui/react-separator       — components/ui/separator.tsx
@radix-ui/react-slot            — components/ui/button.tsx 등 (asChild 패턴)
@radix-ui/react-toast           — components/ui/toast.tsx, toaster.tsx
@radix-ui/react-tooltip         — components/ui/tooltip.tsx
radix-ui (메타 패키지)           — 추가 프리미티브 일괄 접근
```

#### shadcn 패턴
- `components/ui/` 의 모든 파일은 shadcn 패턴으로 생성된 컴포넌트다 (`components.json` 설정 기반).
- 직접 수정 가능한 소스로 관리 — `npx shadcn add` 로 추가, 이후 프로젝트 규칙에 맞게 수정.
- 새 UI 컴포넌트가 필요하면 shadcn CLI로 추가 후 `components/ui/`에 놓는다. Radix UI 프리미티브를 직접 조합한 파일을 `page/`나 `components/sheet/`에 흩뿌리지 않는다.

**대신 쓰지 말 것**: `@mui/material`, `antd`, `chakra-ui` 같은 다른 컴포넌트 라이브러리 — 이 프로젝트는 Radix UI + shadcn + Tailwind로 통일.

---

### 7. 스타일링 — Tailwind v4 + 유틸리티

#### 설치 구성
```
tailwindcss ^4          — 핵심 (PostCSS 플러그인 기반)
@tailwindcss/postcss ^4 — postcss.config.mjs 에서 플러그인으로 등록
tw-animate-css ^1.4.0   — CSS 애니메이션 유틸 (globals.scss 에서 @import)
```

#### className 조합 규칙 — `cn()` 필수
```ts
// lib/shadcn-utils.ts (실제 코드)
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

- **모든 조건부 className 조합은 `cn()`을 사용한다.**
- `class-variance-authority`(`cva`)는 variant 기반 컴포넌트(`Button`, `Badge` 등)에서만 사용.
- 문자열 템플릿 리터럴로 className 조합 금지 — 클래스 충돌이 발생한다.

```tsx
// 권장
className={cn("base-class", isActive && "active-class", variant === "primary" && "text-primary")}

// 금지
className={`base-class ${isActive ? "active-class" : ""}`}
```

**대신 쓰지 말 것**: styled-components, CSS Modules(globals.scss 외부), emotion — Tailwind로 통일. `clsx()` 또는 `twMerge()` 단독 사용 — `cn()`으로 통일.

---

### 8. 토스트 알림 — sonner

#### 왜 sonner인가
- 경량, 스타일 커스터마이즈 용이, Next.js와 잘 통합.

#### 사용 규칙
- `components/ui/sonner.tsx` 에 `<Toaster />` 래퍼 존재. `components/app-layout.tsx`에서 단 한 번만 마운트.
- 현재 프로젝트는 `components/ui/use-toast.ts` (Radix Toast 기반 `toast()`)도 병용 중.

> **주의**: `toast()` 호출 시 `@/components/ui/use-toast`(현 프로젝트 관행)와 sonner의 `toast`를 혼용하지 않는다. 새 코드는 팀 내 합의된 하나로 통일한다.

**대신 쓰지 말 것**: `react-toastify`, `react-hot-toast` 등 다른 toast 라이브러리 — 이 프로젝트는 sonner / Radix Toast 중 하나로 통일.

---

### 9. 아이콘 — lucide-react

#### 사용 규칙
```tsx
import { Search, Plus, Trash2, ChevronDown } from "lucide-react"
// <Search className="h-4 w-4" />
```
- 이름 있는 import만 사용 — tree-shaking 보장.
- 크기는 Tailwind `h-4 w-4` (16px) 기본, 상황에 따라 `h-5 w-5`, `h-6 w-6`.

**대신 쓰지 말 것**: `react-icons`, `@heroicons/react`, Font Awesome — lucide-react로 통일.

---

### 10. 차트 — recharts

#### 사용 규칙
- 직접 recharts API를 `page/`에서 쓰지 않는다. `components/charts/` 아래에 도메인 차트 컴포넌트를 만든다.
```
components/charts/
├── QuestionTrendChart.tsx
├── SimpleStatusChart.tsx
├── StatusBadgeChart.tsx
├── StatusDistributionChart.tsx
└── StatusDistributionContent.tsx
```

---

### 11. 셀렉트 — react-select

#### 사용 규칙
- 검색 가능 셀렉트(`components/ui/searchable-select.tsx`), 멀티셀렉트(`components/ui/multi-select.tsx`)의 구현에 사용.
- 직접 `react-select`를 `page/`나 `components/sheet/`에서 import 하지 않는다. 래퍼 컴포넌트를 경유.
- 폼 연동은 `components/ui/form/custom-searchable-select-form.tsx`, `components/ui/form/custom-multi-select-form.tsx` 사용.

---

### 12. 테마 — next-themes

#### 사용 규칙
- `components/theme-toggle.tsx` 에서 `useTheme()` 으로 토글.
- `app/layout.tsx` 의 인라인 스크립트가 SSR 플리커를 방지한다.

---

### 13. CSS 전처리 — sass

#### 사용 규칙
- `app/globals.scss` 전역 스타일에만 사용.
- 컴포넌트별 `.scss` 파일 신규 생성 금지 — Tailwind utility class로 처리한다.

---

## 환경변수

| 변수 | 필수 | 설명 |
|------|------|------|
| `NEXT_PUBLIC_API_BASE_URL` | 필수 | 백엔드 API 기본 URL (`appAxios.baseURL`) |

- `NEXT_PUBLIC_` prefix 가 없는 변수는 서버에서만 접근 가능하며 브라우저 번들에 포함되지 않는다.
- `NEXT_PUBLIC_` 변수는 **클라이언트 번들에 노출**된다 — 시크릿(JWT 서명키, DB 비밀번호 등)을 절대 넣지 않는다.
- `.env.local` (로컬 개발), `.env.production` (프로덕션). `.env.local`은 Git에 커밋하지 않는다.

---

## 새 라이브러리 도입 판단 기준

```
"이 라이브러리를 꼭 추가해야 하나?"

1. 이미 설치된 라이브러리로 해결 가능한가?
   → 기존 라이브러리 우선.
     (예: 단순 날짜 포맷 → date-utils.ts 에 직접 구현, dayjs 추가 불필요)

2. shadcn CLI로 추가 가능한가?
   → npx shadcn add [component] 로 Radix 기반 컴포넌트를 먼저 확인.

3. 번들 크기 영향이 큰가?
   → tree-shaking 지원 여부 확인.
   → 무거운 라이브러리는 dynamic import 로 분리.

4. 유사 라이브러리가 이미 있는가?
   → 토스트 → sonner / Radix Toast 중 프로젝트 관행 따를 것.
   → 셀렉트 → react-select 래퍼 컴포넌트 먼저 확인.
   → 아이콘 → lucide-react 먼저 확인.

5. `NEXT_PUBLIC_` 환경변수나 브라우저 API에 의존하는가?
   → Server Component 에서는 사용 불가 — 'use client' 경계 설계 필요.
```
