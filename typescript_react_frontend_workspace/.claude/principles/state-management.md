# 상태 관리 원칙 (Iron Rule)

> 본 프로젝트는 상태를 **4가지 레이어**로 명확히 분리한다.
> "어디에 상태를 두어야 하는가"에 대한 답은 아래 의사결정 트리를 먼저 참조한다.

---

## 의사결정 트리

```
이 상태는 어디에 두는가?
│
├─ 백엔드 API 에서 가져오는 데이터인가?
│  └─ YES → TanStack Query (서버 상태)
│
├─ 사용자 인증 정보·액세스 토큰인가?
│  └─ YES → zustand authStore (클라이언트 메모리 상태)
│
├─ URL 에 공유되어야 하는 필터·페이지·정렬인가?
│  └─ YES → useUrlSearchParams 계열 훅 (URL 상태)
│
├─ 폼 입력값인가?
│  └─ YES → react-hook-form (폼 상태)
│
└─ 그 외 (모달 open/close, 선택된 탭 등 UI 지역 상태)
   └─ useState / useReducer (컴포넌트 지역 상태)
```

---

## 1. 서버 상태 — TanStack Query

모든 백엔드 데이터 페칭·캐싱·동기화는 TanStack Query 를 사용한다. 컴포넌트 안에서 `useState` + `useEffect` 로 직접 fetch 하지 않는다.

### QueryClient 설정 (`components/providers.tsx`)

```typescript
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,         // 1분 — 동일 키 재요청 억제
      retry: 1,                      // 실패 시 1회 재시도
      refetchOnWindowFocus: false,   // 탭 포커스 시 자동 refetch 비활성
    },
  },
})
```

### staleTime / gcTime 기준

| 데이터 특성 | staleTime | gcTime |
|------------|-----------|--------|
| 자주 바뀌는 데이터 (문서 상태, 채팅) | 0 (기본) | 5분 |
| 보통 변경 주기 (목록 기본) | 60초 (기본) | 10분 |
| 거의 안 바뀌는 참조 데이터 (컬렉션, 프롬프트) | 30초 ~ 5분 | 10분 |
| 정적에 가까운 데이터 (RAG 서버 정보) | 5분 이상 | 30분 |

```typescript
// 예시: 도메인에서 staleTime 개별 지정
export const useXxxSearch = (keyword: string) => {
  return useInfiniteQuery({
    queryKey: xxxKeys.list({ keyword }),
    queryFn: ...,
    staleTime: 30000,   // 30초
    gcTime: 300000,     // 5분
  })
}
```

### 쿼리 키 팩토리 패턴

모든 React Query 훅 파일은 파일 상단에 키 팩토리를 선언한다. 문자열 배열을 훅 호출 시마다 인라인으로 작성하지 않는다.

```typescript
// hooks/react-query/use<Domain>Queries.ts
export const xxxKeys = {
  all: ['<domain>s'] as const,
  lists: () => [...xxxKeys.all, 'list'] as const,
  list: (params: XxxDto.SearchParams) => [...xxxKeys.lists(), params] as const,
  details: () => [...xxxKeys.all, 'detail'] as const,
  detail: (id: string) => [...xxxKeys.details(), id] as const,
}
```

### mutation 후 캐시 무효화

```typescript
// 생성/삭제: 목록 전체 무효화
queryClient.invalidateQueries({ queryKey: xxxKeys.all })

// 수정: 목록 + 해당 상세 무효화
queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
queryClient.invalidateQueries({ queryKey: xxxKeys.detail(id) })
```

- `invalidateQueries` 는 `onSuccess` 콜백에서 호출한다.
- 연관 도메인의 캐시까지 무효화해야 하면 해당 도메인의 `xxxKeys` 를 import 해서 사용한다. 문자열 하드코딩 금지.

### optimistic update 가이드

데이터 변경이 즉각적으로 UI 에 반영되어야 하는 경우에만 사용한다. 그 외 일반 CRUD 는 `invalidateQueries` 로 충분하다.

```typescript
useMutation({
  mutationFn: updateXxx,
  onMutate: async (newData) => {
    await queryClient.cancelQueries({ queryKey: xxxKeys.detail(newData.id) })
    const previous = queryClient.getQueryData(xxxKeys.detail(newData.id))
    queryClient.setQueryData(xxxKeys.detail(newData.id), newData)
    return { previous }
  },
  onError: (err, newData, context) => {
    queryClient.setQueryData(xxxKeys.detail(newData.id), context?.previous)
  },
  onSettled: (data, err, { id }) => {
    queryClient.invalidateQueries({ queryKey: xxxKeys.detail(id) })
  },
})
```

---

## 2. 클라이언트 상태 — zustand

현재 유일한 zustand store 는 `store/authStore.ts` (`useAuthStore`) 다.

**새 store 추가 기준**: 아래 조건을 모두 충족할 때만 새 store 를 만든다.
1. 여러 컴포넌트 트리에서 공유가 필요하다.
2. URL 에 넣을 수 없는 상태다 (민감 정보 또는 세션 한정 상태).
3. TanStack Query 로 표현할 수 없다 (서버 상태가 아님).

위 조건에 해당하지 않으면 `useState` 나 `useUrlSearchParams` 로 처리한다.

### authStore 설계 원칙 (`store/authStore.ts`)

```typescript
// 메모리에만 저장 — localStorage/sessionStorage 금지
export const useAuthStore = create<AuthState>()((set, get) => ({
  accessToken: null,     // 메모리 전용
  tokenInfo: null,       // 메모리 전용
  isRefreshing: false,

  setToken: (token) => { /* accessToken + tokenInfo 를 메모리에 저장 */ },
  clearToken: () => { set({ accessToken: null, tokenInfo: null }) },
  getAccessToken: () => get().accessToken || undefined,
  isAuthenticated: () => !!get().accessToken,
  logout: () => { set({ accessToken: null, tokenInfo: null, isRefreshing: false }) },
}))
```

- refresh token 은 백엔드가 httpOnly 쿠키로 관리한다. 프론트에서 직접 접근하지 않는다.
- `accessToken` 을 localStorage 에 저장하면 XSS 시 탈취된다. **반드시 메모리에만 보관**.
- 페이지 새로고침 시 `accessToken` 이 사라지므로, `auth-guard.tsx` 가 마운트 시 `/auth/refresh` 를 호출해 복구한다.

---

## 3. URL 상태 — useUrlSearchParams 계열 훅

목록 페이지의 필터·페이지·정렬은 URL 쿼리스트링에 저장한다. 이렇게 하면 새로고침·공유·뒤로가기가 자연스럽게 동작한다.

### 등록된 훅 목록 (`hooks/`)

| 훅 | 대상 페이지 |
|----|------------|
| `useUrlSearchParams.ts` | 기반 구현체 (직접 사용 가능) |
| `use<Domain>SearchParams.ts` | 도메인별 목록 페이지 |

새 목록 페이지를 만들 때는 기존 `use<Domain>SearchParams.ts` 중 하나를 참조해서 동일한 패턴으로 작성한다.

### 사용 패턴

```typescript
// hooks/use<Domain>SearchParams.ts 패턴
export function use<Domain>SearchParams() {
  const { appliedValues, tempValues, updateTempValues, updateAppliedValues,
          applySearch, reset, isInitialized }
    = useUrlSearchParams<XxxDto.SearchParams>({
        defaultValues,
        serialize,
        deserialize,
        manualSearch: true   // "검색" 버튼 클릭 시 적용
      })

  return {
    appliedFilters,  // API 요청에 사용 (URL 에 반영된 값)
    filters,         // UI 에 바인딩 (임시값)
    pagination,
    setKeyword, setPage,
    applySearch, reset, isInitialized
  }
}
```

- `appliedValues` 는 실제 API 요청에 사용하는 값이다 (URL 파라미터와 동기화).
- `tempValues` 는 사용자가 입력 중인 임시 값이다 (아직 URL 에 반영 안 됨).
- `applySearch()` 호출 시 `tempValues` 가 URL 에 반영되고 React Query 가 재요청된다.

---

## 4. 폼 상태 — react-hook-form

모든 폼 입력은 `react-hook-form` + `zod` 로 관리한다. `useState` 로 폼 필드를 하나씩 관리하지 않는다.

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { xxxSchema } from '@/model/schema/<domain>-schema'
import type { XxxSchemaType } from '@/model/schema/<domain>-schema'

const form = useForm<XxxSchemaType>({
  resolver: zodResolver(xxxSchema),
  defaultValues: { /* 초기값 */ },
})
```

- zod 스키마는 `model/schema/<domain>-schema.ts` 에 선언한다.
- 폼 에러 표시는 `formState.errors` 를 사용해 인라인으로 표시한다. 별도 토스트로 대체하지 않는다.
- 서버 에러(API 실패)는 `form.setError('root', { message: ... })` 또는 sonner toast 로 처리한다.

---

## 5. 지역 상태 — useState / useReducer

컴포넌트 내부에서만 사용되는 상태(모달 open/close, 선택된 행, 탭 인덱스 등) 는 `useState` 로 관리한다. 단일 컴포넌트 한정이면 store 나 Context 로 끌어올리지 않는다.

---

## 금지사항

```typescript
// ❌ 컴포넌트에서 직접 fetch
useEffect(() => {
  fetch('/api/<domain>').then(res => res.json()).then(setData)
}, [])

// ❌ 서버 상태를 useState 로 관리
const [items, setItems] = useState<XxxDto.Summary[]>([])

// ❌ accessToken 을 localStorage 에 저장
localStorage.setItem('accessToken', token)

// ❌ 조건 없이 새 zustand store 생성
// (authStore 외 store 추가 전 위 "새 store 추가 기준" 확인)

// ❌ 쿼리 키 인라인 문자열 사용
queryClient.invalidateQueries({ queryKey: ['<domain>s', 'list'] })

// ✅ 올바른 패턴
queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
```
