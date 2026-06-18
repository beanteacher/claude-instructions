# 만들기 전에 검색하기 (Iron Rule)

- 익숙하지 않은 UI/데이터/인증 패턴을 도입하기 전에 **기존 솔루션을 먼저 확인**한다. 본 저장소는 이미 다음을 갖추고 있으니 중복 구현을 금지한다.
  - **HTTP 클라이언트** → `api/common/axios.ts` 의 `appAxios` 인스턴스 (`qs` 파라미터 직렬화, `showErrorPopup` / `showLoading` 옵션 내장)
  - **인증 인터셉터** → `components/axios-interceptor.tsx` 의 `AxiosInterceptor` (401 → refresh → 재시도 → 로그아웃 흐름)
  - **인증 상태** → `store/authStore.ts` 의 `useAuthStore` (zustand, 메모리 전용 토큰)
  - **라우트 가드** → `components/auth-guard.tsx` 의 `AuthGuard` (`PUBLIC_ROUTES` 기반 인증/리다이렉트)
  - **페이지 레이아웃** → `components/app-layout.tsx` 의 `AppLayout` (`Providers` + `AxiosInterceptor` + `Toaster` + `AuthGuard` + `SidebarProvider` 조합)
  - **사이드바** → `components/app-sidebar.tsx` 의 `AppSidebar` + `items` 배열 (메뉴 추가 시 여기에만 항목 추가)
  - **헤더** → `components/site-header.tsx` 의 `SiteHeader`
  - **테이블** → `components/shared/table/BaseTable.tsx` 의 `BaseTable<T>` (로딩·에러·빈 상태 자동 처리, 페이지네이션 내장)
  - **테이블 상태 컴포넌트** → `components/shared/table/TableStates.tsx` (`LoadingState`, `ErrorState`, `EmptyState`)
  - **서버 상태 (React Query)** → `components/providers.tsx` 의 `Providers` 내 `QueryClient` (`staleTime: 60s`, `retry: 1`, `refetchOnWindowFocus: false`)
  - **URL 검색 파라미터** → `hooks/useUrlSearchParams.ts` 의 `useUrlSearchParams` (수동/자동 검색 모드, 직렬화·역직렬화 주입)
  - **도메인별 URL 파라미터 훅** (useUrlSearchParams 래퍼):
    - `hooks/use<Domain>SearchParams.ts` — 도메인별 목록 필터 + 페이지네이션
    - (프로젝트 내 기존 훅 목록은 `hooks/` 디렉토리를 grep 으로 확인)
  - **Tailwind 클래스 병합** → `lib/shadcn-utils.ts` 의 `cn()` (`clsx` + `tailwind-merge` 조합)
  - **토스트** → `components/ui/use-toast.ts` 의 `toast()` (성공: `toast({ title, description })`, 실패: `toast({ variant: "destructive", ... })`)
    - 참고: `components/ui/sonner.tsx` 의 `Toaster` 는 sonner 기반 toaster 컴포넌트 (현재 `app-layout.tsx` 에는 shadcn `Toaster` 사용 중 — 혼용 주의)
  - **페이지 응답 타입** → `model/page-list-dto.ts` (`PageListDto.Request` / `PageListDto.Response<T>`)
  - **공통 응답 타입** → `model/response-model.ts` (`ResponseModel<T>` — `code?`, `message?`, `data?`)
  - **앱 타입 / 상수** → `common/app-type.ts`, `common/app-type-name.ts`, `common/app-type-options.ts`, `common/constants.ts`
  - **인증 유틸** → `common/auth-utils.ts`
  - **공통 유틸리티** → `common/utils/code-utils.ts`, `common/utils/date-utils.ts`, `common/utils/search-utils.ts`
  - **shadcn UI 컴포넌트** (`components/ui/` — 새 UI 요소 만들기 전 여기를 먼저 확인):

    | 컴포넌트 | 파일 |
    |----------|------|
    | AlertDialog | `components/ui/alert-dialog.tsx` |
    | Badge | `components/ui/badge.tsx` |
    | Button | `components/ui/button.tsx` |
    | Card / CardHeader / CardContent | `components/ui/card.tsx` |
    | Chart | `components/ui/chart.tsx` |
    | Checkbox | `components/ui/checkbox.tsx` |
    | Command | `components/ui/command.tsx` |
    | CustomInput | `components/ui/custom/custom-input.tsx` |
    | CustomSelect | `components/ui/custom/custom-select.tsx` |
    | DataPagination | `components/ui/data-pagination.tsx` |
    | Dialog | `components/ui/dialog.tsx` |
    | DropdownMenu | `components/ui/dropdown-menu.tsx` |
    | CustomInputForm | `components/ui/form/custom-input-form.tsx` |
    | CustomMultiSelectForm | `components/ui/form/custom-multi-select-form.tsx` |
    | CustomSearchableSelectForm | `components/ui/form/custom-searchable-select-form.tsx` |
    | CustomSelectForm | `components/ui/form/custom-select-form.tsx` |
    | CustomTextareaForm | `components/ui/form/custom-textarea-form.tsx` |
    | Input | `components/ui/input.tsx` |
    | Label | `components/ui/label.tsx` |
    | MultiSelect | `components/ui/multi-select.tsx` |
    | Pagination | `components/ui/pagination.tsx` |
    | Popover | `components/ui/popover.tsx` |
    | SearchableSelect | `components/ui/searchable-select.tsx` |
    | Select | `components/ui/select.tsx` |
    | Separator | `components/ui/separator.tsx` |
    | Sheet / SmartSheet | `components/ui/sheet.tsx`, `components/ui/smart-sheet.tsx` |
    | Sidebar (+ SidebarProvider 등) | `components/ui/sidebar.tsx` |
    | Skeleton | `components/ui/skeleton.tsx` |
    | Sonner Toaster | `components/ui/sonner.tsx` |
    | Switch | `components/ui/switch.tsx` |
    | Table / TableHeader / TableBody 등 | `components/ui/table.tsx` |
    | Textarea | `components/ui/textarea.tsx` |
    | Toast / Toaster | `components/ui/toast.tsx`, `components/ui/toaster.tsx` |
    | Tooltip / TooltipProvider | `components/ui/tooltip.tsx` |

- 제약 조건을 충족하는 경우 **검증된 로컬 패턴을 재사용**한다. `.claude/rules/react-patterns.md` 와 `.claude/conventions/*.md` 의 예시가 1차 참조 대상이다.
- 비표준 접근 방식(새로운 HTTP 라이브러리, 전역 상태 관리 대안, 커스텀 라우터 등) 을 선택하는 경우 **원칙 기반 근거를 문서화**하고 PR 본문에 기록한다.
- 다음을 구분해서 판단한다:
  - **검증된 패턴** — 이미 본 저장소나 Next.js/React 생태계에서 일반화된 구성 (예: `BaseTable<T>` 재사용, `useUrlSearchParams` 파생 훅 패턴, `appAxios` 인스턴스 사용)
  - **검토가 필요한 새로운/인기 있는 패턴** — 도입 시 팀 리뷰 필요 (예: Server Component 전환, React 19 `use()` 훅 대량 적용, Zustand slice 패턴)
  - **특정 제약 조건으로 정당화되는 독자적 접근 방식** — `axios-interceptor.tsx` 처럼 본 프로젝트의 토큰 갱신·재시도 요건 때문에 만들어진 구조
- 외부 라이브러리 추가 판단 흐름:
  1. shadcn UI / Radix UI / TanStack 생태계에 이미 있는가? → 있으면 그걸 쓴다.
  2. 기존 의존성으로 20줄 안에 해결되는가? → 그러면 라이브러리를 추가하지 않는다.
  3. 추가해야 한다면 `package.json` 의 기존 의존성과 충돌하지 않는지, 번들 사이즈 영향이 허용 범위인지 먼저 확인한다.

---

## 새 컴포넌트/훅 만들기 전 체크리스트

```
(1) components/shared/ 와 components/ui/ grep:
    이미 동일한 역할을 하는 컴포넌트가 있는가?

(2) hooks/ grep:
    동일한 로직의 커스텀 훅이 이미 있는가?
    (useUrlSearchParams, useIsMobile, use-theme.ts 등)

(3) lib/ 과 common/ grep:
    같은 유틸리티 함수가 이미 있는가?
    (cn(), code-utils, date-utils, search-utils, auth-utils 등)

(4) 비슷한 도메인의 page/<domain>/ 살펴보기:
    이미 같은 패턴으로 구현된 페이지가 있는가?
    (예: page/document/documents/ → page/prompt/ 구조 참고)
```
