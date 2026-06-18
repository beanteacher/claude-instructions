# 프론트엔드 아키텍처 가이드

> **Next.js App Router + 기능별 폴더 분리** 구조를 기준으로 작성됐다.

---

## 프로젝트 루트 구조

```
frontend/
├── app/                         # Next.js App Router 라우트 정의 (진입점만)
│   ├── layout.tsx               #   루트 레이아웃 (AppLayout 마운트)
│   ├── page.tsx                 #   / → 대시보드 리다이렉트
│   ├── not-found.tsx            #   404 글로벌 처리
│   ├── globals.scss             #   전역 스타일 (Tailwind 진입점)
│   ├── (<domain-a>)/<route-a>/page.tsx  #   Route Group 예시
│   ├── (<domain-b>)/<route-b>/page.tsx  #   Route Group 예시
│   ├── login/page.tsx
│   └── ...
│
├── page/                        # 도메인별 화면 컴포넌트 (비즈니스 로직 포함)
│   ├── MainPage.tsx             #   메인/대시보드 화면
│   ├── main-actions.tsx         #   메인 액션(다이얼로그/시트 상태 관리)
│   ├── <domain-a>/              #   도메인 A 화면
│   ├── <domain-b>/              #   도메인 B 화면
│   ├── login/                   #   로그인 화면
│   └── ...
│
├── components/                  # 재사용 컴포넌트
│   ├── app-layout.tsx           #   전체 레이아웃 (Providers + 인증 분기)
│   ├── app-sidebar.tsx          #   사이드바
│   ├── auth-guard.tsx           #   인증 가드 (토큰 갱신 + 리다이렉트)
│   ├── axios-interceptor.tsx    #   axios 요청/응답 인터셉터 (토큰 주입·갱신)
│   ├── providers.tsx            #   QueryClient + ReactQueryDevtools
│   ├── site-header.tsx          #   상단 헤더
│   ├── theme-toggle.tsx         #   다크/라이트 테마 토글
│   ├── charts/                  #   recharts 기반 차트 컴포넌트
│   ├── sheet/                   #   도메인별 Sheet (사이드 패널) 컴포넌트
│   │   └── form/                #     Sheet 내부 폼 컴포넌트
│   ├── shared/table/            #   공용 테이블 (BaseTable, BaseSearchFilters 등)
│   └── ui/                      #   shadcn/ui 기반 원자 컴포넌트
│       ├── button.tsx, input.tsx, dialog.tsx, ...
│       ├── form/                #     react-hook-form 연동 폼 필드
│       └── custom/              #     프로젝트 커스텀 UI 프리미티브
│
├── hooks/                       # 커스텀 훅
│   ├── react-query/             #   도메인별 TanStack Query 훅
│   │   ├── use<Domain>Queries.ts
│   │   └── ...
│   ├── useUrlSearchParams.ts    #   URL 쿼리스트링 ↔ 상태 동기화 훅 (공용)
│   ├── use<Domain>SearchParams.ts   # 도메인별 검색 파라미터 훅
│   ├── use-mobile.ts            #   모바일 뷰포트 감지
│   └── use-theme.ts             #   테마 훅
│
├── api/                         # 도메인별 axios API 모듈
│   ├── common/
│   │   ├── axios.ts             #   axios 인스턴스 (baseURL, paramsSerializer, 공통 옵션)
│   │   └── app-api.ts           #   API 헬퍼 (get/post/put/patch/delete 래퍼)
│   ├── <domain-a>-api.ts
│   ├── <domain-b>-api.ts
│   └── ...
│
├── model/                       # 타입 정의 (API DTO + 클라이언트 타입 + zod 스키마)
│   ├── dto/                     #   백엔드 API 응답·요청 타입
│   │   ├── <domain>-dto.ts
│   │   └── ...
│   ├── schema/                  #   react-hook-form + zod 스키마 (폼 검증)
│   │   ├── <domain>-schema.ts
│   │   └── ...
│   ├── client/                  #   프론트 전용 클라이언트 상태 타입
│   │   └── ...
│   ├── page-list-dto.ts         #   공통 페이지네이션 Request/Response 타입
│   └── response-model.ts        #   공통 API 응답 래퍼 타입
│
├── store/                       # 클라이언트 전역 상태 (zustand)
│   └── authStore.ts             #   인증 상태 (accessToken, tokenInfo — 메모리 전용)
│
├── common/                      # 프로젝트 공용 상수·유틸·타입 열거
│   ├── constants.ts             #   AUTH_HEADER, PUBLIC_ROUTES, DEFAULT_PAGE_SIZE 등
│   ├── app-type.ts              #   공용 enum/타입 (StatusCode, SortDirection 등)
│   ├── app-type-name.ts         #   enum → 표시 이름 매핑
│   ├── app-type-options.ts      #   enum → select 옵션 배열
│   ├── auth-utils.ts            #   인증 관련 유틸
│   └── utils/
│       ├── date-utils.ts
│       ├── code-utils.ts        #   logger 등 코드 유틸
│       └── search-utils.ts
│
├── lib/                         # 라이브러리 래퍼
│   └── shadcn-utils.ts          #   cn() 유틸 (clsx + tailwind-merge)
│
└── types/                       # 전역 TypeScript 선언
    └── api.d.ts                 #   axios 요청 설정 모듈 보강 (showErrorPopup 등)
```

---

## 의존성 방향 (핵심 규칙)

```
app/[route]/page.tsx
    │
    ▼
page/[domain]/XxxPage.tsx          (화면 비즈니스 로직, 훅 조합)
    │
    ├──▶  hooks/react-query/useXxxQueries.ts   (서버 상태 관리)
    │         │
    │         ▼
    │     api/<domain>-api.ts                  (axios API 호출)
    │         │
    │         ▼
    │     api/common/app-api.ts  ──▶  api/common/axios.ts (axios 인스턴스)
    │
    ├──▶  hooks/useXxxSearchParams.ts          (URL 파라미터 동기화)
    │         │
    │         ▼
    │     hooks/useUrlSearchParams.ts          (공용 URL 훅)
    │
    ├──▶  store/authStore.ts                   (클라이언트 전역 상태)
    │
    └──▶  components/[shared|sheet|ui]/        (재사용 UI 컴포넌트)
              │
              ▼
          model/dto/*.ts    model/schema/*.ts  (타입·스키마)
          common/*.ts       lib/shadcn-utils.ts
```

### 레이어별 import 규칙

| 레이어 | import 가능 | import 불가 |
|--------|------------|-------------|
| `app/` | `page/`, `components/`, `common/` | `api/`, `hooks/react-query/`, `store/` 직접 호출 금지 |
| `page/` | `hooks/`, `components/`, `model/`, `common/`, `store/`, `api/` | 다른 `page/` 도메인 직접 참조 금지 |
| `components/` | `model/`, `common/`, `lib/`, `store/`, `hooks/` | `page/`, `api/` (단, Sheet 컴포넌트는 `hooks/react-query/` 허용) |
| `hooks/react-query/` | `api/`, `model/`, `common/` | `page/`, `components/`, `store/` |
| `hooks/useXxxSearchParams` | `hooks/useUrlSearchParams`, `model/`, `common/` | `api/`, `store/` |
| `api/` | `model/`, `api/common/` | `hooks/`, `components/`, `store/`, `page/` |
| `model/` | `common/` (app-type 등) | `api/`, `hooks/`, `components/`, `store/` |
| `store/` | `model/` | `api/`, `hooks/`, `components/` |
| `common/` | 외부 라이브러리 | 내부 레이어 (`api/`, `hooks/`, `components/`, `page/`, `store/`) |

**역방향 import는 금지한다.** `api/`가 `hooks/`를 import 하거나, `model/`이 `components/`를 import 하면 순환 의존이 발생한다.

---

## App Router 라우트 구조

### Route Group 패턴

괄호로 묶인 폴더(`(<group>)`)는 URL에 노출되지 않는 **논리적 그룹**이다.
layout을 공유하지 않고 그룹화 목적으로만 사용할 수 있다.

```
app/
├── (<group-a>)/<route-a>/page.tsx   → URL: /<route-a>
├── (<group-b>)/<route-b>/page.tsx   → URL: /<route-b>
└── login/page.tsx                   → URL: /login
```

### app/[route]/page.tsx 역할

App Router의 `page.tsx`는 **라우트 진입점**이다. 비즈니스 로직 없이 `page/` 디렉토리의 Page 컴포넌트를 import해서 렌더링하는 것이 전부다.

```tsx
// app/(<group>)/<route>/page.tsx
import { XxxPage } from "@/page/<domain>/XxxPage"
export default function Page() {
  return <XxxPage />
}
```

Page 컴포넌트에 직접 비즈니스 로직을 작성하면 테스트·재사용이 불가능해지므로 금지한다.

---

## 레이어별 역할

### `app/layout.tsx` — 루트 레이아웃

```tsx
// app/layout.tsx (실제 코드)
export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <script dangerouslySetInnerHTML={{ __html: `/* 테마 플리커 방지 스크립트 */` }} />
        <AppLayout>{children}</AppLayout>
      </body>
    </html>
  )
}
```

- 폰트(`Geist`, `Geist_Mono`), 전역 스타일(`globals.scss`), `AppLayout` 마운트가 전부다.
- 비즈니스 로직 없음.

### `components/app-layout.tsx` — 앱 레이아웃

```tsx
// components/app-layout.tsx (실제 코드 요약)
export function AppLayout({ children }) {
  const isLoginPage = pathname === '/login'

  if (isLoginPage) {
    return <Providers><AxiosInterceptor /><Toaster />{children}</Providers>
  }

  return (
    <Providers>
      <AxiosInterceptor />
      <Toaster />
      <AuthGuard>
        <SidebarProvider>
          <AppSidebar />
          <SidebarInset>
            <SiteHeader />
            {children}
          </SidebarInset>
        </SidebarProvider>
      </AuthGuard>
    </Providers>
  )
}
```

- **`Providers`**: `QueryClient` + `ReactQueryDevtools` Provider 마운트.
- **`AxiosInterceptor`**: `appAxios` 에 요청/응답 인터셉터를 등록하는 effect-only 컴포넌트.
- **`Toaster`**: `components/ui/toaster.tsx` (Radix UI Toast 기반). 전역에 단 하나만 마운트.
- **`AuthGuard`**: 초기 인증 상태 확인 + `/login` 리다이렉트.
- 로그인 페이지는 사이드바 레이아웃 없이 단독 렌더.

### `page/[domain]/XxxPage.tsx` — 화면 컴포넌트

- 해당 화면의 **비즈니스 로직과 훅 조합**이 여기서 이루어진다.
- `hooks/react-query/` 훅으로 서버 상태를 관리하고, `hooks/useXxxSearchParams`로 URL 파라미터를 동기화한다.
- `components/shared/table/BaseTable`로 테이블 렌더링, `components/sheet/XxxSheet`로 상세/편집 패널 제어.

```tsx
// page/<domain>/XxxPage.tsx 패턴
export function XxxPage() {
  const { appliedValues, tempValues, ... } = useXxxSearchParams()
  const { data, isLoading } = useXxxList(appliedValues)
  const [selectedId, setSelectedId] = useState<string | null>(null)

  return (
    <>
      <XxxSearchFilters ... />
      <BaseTable columns={columns} data={data?.list} isLoading={isLoading} ... />
      <XxxSheet id={selectedId} onClose={() => setSelectedId(null)} />
    </>
  )
}
```

- `page/` 하위 `items/` 폴더에 해당 화면 전용 서브 컴포넌트를 모은다.
- `*-actions.tsx` 파일에 다이얼로그/Sheet 열기 버튼 액션 컴포넌트를 분리한다.

### `hooks/react-query/useXxxQueries.ts` — 서버 상태 훅

- `useQuery` / `useMutation` / `useInfiniteQuery` 를 도메인별로 모은 파일.
- 쿼리 키는 계층형 배열 튜플로 정의 (`documentKeys`, `promptKeys` 등).
- `useMutation` 성공 시 `queryClient.invalidateQueries()`로 관련 캐시를 무효화한다.

```ts
// hooks/react-query/useXxxQueries.ts (구조 예시)
export const xxxKeys = {
  all: ['xxx'] as const,
  lists: () => [...xxxKeys.all, 'list'] as const,
  list: (params: XxxDto.SearchParams) => [...xxxKeys.lists(), params] as const,
  detail: (id: string) => [...xxxKeys.all, 'detail', id] as const,
}

export const useXxxList = (params: XxxDto.SearchParams) =>
  useQuery({ queryKey: xxxKeys.list(params), queryFn: () => XxxApi.search(params).then(r => r.data) })
```

### `api/xxx-api.ts` — API 모듈

- `appApi`(공용 헬퍼)와 `appAxios`(인스턴스)를 조합해 엔드포인트를 정의한다.
- `export * as DocumentApi from "./document-api"` 패턴으로 네임스페이스 export.
- 모든 API 호출은 `appAxios`를 통해야 한다 — 직접 `axios.create()` 금지.

```ts
// api/<domain>-api.ts (구조 예시)
export const search = (params: XxxDto.SearchParams) =>
  appApi.get<PageListDto.Response<XxxDto.Summary>>(appAxios, `/<domain>/search`, { params })

export const edit = (id: string, form: XxxDto.EditForm) =>
  appApi.put<XxxDto.Detail>(appAxios, `/<domain>/${id}`, { data: form })
```

### `model/dto/xxx-dto.ts` — API DTO 타입

- 백엔드 API의 요청·응답 타입을 `interface`로 선언.
- `export * as DocumentDto from './document-dto'` 로 네임스페이스 export.
- 공통 페이지네이션은 `model/page-list-dto.ts` 의 `PageListDto.Request` / `PageListDto.Response<T>` 상속.

```ts
// model/dto/<domain>-dto.ts (구조 예시)
export interface SearchParams extends PageListDto.Request {
  keyword?: string
  // 도메인별 필터 파라미터
}
export interface Summary { id: string; name: string; statusCode: AppType.StatusCode; ... }
export interface Detail extends Summary { ... }
export interface EditForm { ... }
```

### `model/schema/xxx-schema.ts` — zod 스키마

- `react-hook-form` 의 `zodResolver` 와 연동하는 폼 검증 스키마.
- 도메인별로 분리. DTO 타입과 완전히 일치하지 않아도 된다 (폼 입력 vs API 요청은 다를 수 있음).

### `store/authStore.ts` — 클라이언트 전역 상태

- `zustand` 기반. **accessToken은 메모리에만 저장** (localStorage/sessionStorage 금지).
- `setToken`, `clearToken`, `logout`, `isAuthenticated`, `getAccessToken` 액션 제공.
- 인터셉터(`axios-interceptor.tsx`)와 가드(`auth-guard.tsx`)에서 접근.

### `components/shared/table/BaseTable.tsx` — 공용 테이블

- 모든 목록 테이블의 기반. `columns`, `data`, `isLoading`, `pagination` props를 받는다.
- 로딩·빈 상태·에러 상태는 `TableStates.tsx` 에 위임.
- 도메인 테이블(`DocumentsTable`, `PromptTable` 등)은 `BaseTable`을 래핑해서 만든다.

---

## 도메인 목록 (예시 패턴)

| 도메인 | 라우트 | Page 컴포넌트 | API 모듈 | React Query 훅 |
|--------|--------|---------------|----------|----------------|
| (메인) | `/` | `MainPage.tsx` | `<domain>-api.ts` | `use<Domain>Queries.ts` |
| `<domain-a>` | `/<domain-a>` | `<DomainA>Page.tsx` | `<domain-a>-api.ts` | `use<DomainA>Queries.ts` |
| `<domain-b>` | `/<domain-b>` | `<DomainB>Page.tsx` | `<domain-b>-api.ts` | `use<DomainB>Queries.ts` |
| login | `/login` | `LoginPage.tsx` | `auth-api.ts` | `useAuthQueries.ts` |

각 도메인은 위 패턴을 따라 파일명을 맞춰 추가한다.

---

## 횡단 관심사

### 인증 (`store/authStore.ts` + 인증 가드 컴포넌트 + axios 인터셉터 컴포넌트)

- `accessToken`은 zustand 메모리 상태에만 보관. 페이지 새로고침 시 `/refresh` API로 재발급.
- axios 인터셉터 컴포넌트: 모든 요청에 `Authorization: Bearer {token}` 헤더 삽입. 토큰 만료 임박 시 선제 갱신, 401 응답 시 갱신 후 재시도.
- 인증 가드 컴포넌트: 앱 초기화 시 토큰 확인 → 없으면 refresh 시도 → 실패 시 `/login` 리다이렉트.
- 공개 경로(`PUBLIC_ROUTES`)는 인증 체크에서 제외.

### 에러 처리

- axios 인터셉터의 `handleResponseError`에서 `showErrorPopup: true` 옵션이 있으면 `toast` (destructive variant) 표시.
- 401 → 토큰 갱신 시도 → 실패 시 logout + `/login` 리다이렉트.
- `useMutation`의 `onError`에서 도메인별 에러 메시지를 toast로 표시.
- 렌더 에러는 `app/error.tsx` (필요 시 추가) + React ErrorBoundary로 감싼다.

### 로딩 상태

- TanStack Query의 `isLoading` / `isFetching` 을 공용 테이블 컴포넌트와 시트 데이터 로더가 핸들링.
- 전역 로딩 인디케이터는 axios 인스턴스의 `showLoading` 옵션으로 제어.

### 테마

- `next-themes` + `components/theme-toggle.tsx`. 다크/라이트 모드 지원.
- `app/layout.tsx`의 인라인 스크립트로 초기 로드 시 테마 플리커 방지.

---

## 새 도메인 추가 체크리스트

```
1. model/dto/xxx-dto.ts        — API 응답/요청 타입 선언
2. model/schema/xxx-schema.ts  — zod 폼 검증 스키마 (폼이 있는 경우)
3. api/xxx-api.ts              — API 모듈 (appApi + appAxios 사용)
4. hooks/react-query/useXxxQueries.ts  — TanStack Query 훅
5. hooks/useXxxSearchParams.ts — URL 검색 파라미터 훅 (목록 화면인 경우)
6. page/[domain]/XxxPage.tsx   — 화면 컴포넌트 + items/ 서브 컴포넌트
7. components/sheet/XxxSheet.tsx — 상세/편집 사이드 패널 (필요 시)
8. app/[route]/page.tsx        — App Router 진입점 (XxxPage import만)
9. components/app-sidebar.tsx  — 사이드바 네비게이션 항목 추가
```

## 코드 배치 판단 기준

```
"이 코드는 어디에 넣어야 하지?"

1. Next.js 라우트 진입점인가?
   → app/[route]/page.tsx  (Page 컴포넌트 렌더링만)

2. 화면 비즈니스 로직 + 훅 조합인가?
   → page/[domain]/XxxPage.tsx

3. 여러 도메인에서 공유하는 UI 컴포넌트인가?
   → components/shared/ 또는 components/ui/

4. 특정 도메인의 사이드 패널(Sheet)인가?
   → components/sheet/XxxSheet.tsx

5. 서버 상태 관리(API 연동)인가?
   → hooks/react-query/useXxxQueries.ts

6. URL 파라미터 동기화가 필요한가?
   → hooks/useXxxSearchParams.ts (useUrlSearchParams 활용)

7. 클라이언트 전역 상태인가?
   → store/ (zustand) — 현재는 authStore만

8. axios API 호출 정의인가?
   → api/xxx-api.ts

9. API 요청/응답 타입인가?
   → model/dto/xxx-dto.ts

10. 폼 검증 스키마인가?
    → model/schema/xxx-schema.ts

11. 프로젝트 공용 상수·유틸인가?
    → common/constants.ts 또는 common/utils/

12. className 조합 유틸인가?
    → lib/shadcn-utils.ts 의 cn() 사용
```
