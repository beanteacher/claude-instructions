# 에러 처리 (Iron Rule)

- **에러 처리의 단일 진입점은 `components/axios-interceptor.tsx` 다.** 전역 HTTP 에러(401, 네트워크 오류 등)는 이 인터셉터에서 일괄 처리한다. 개별 컴포넌트에서 `try/catch` 로 HTTP 에러를 중복 처리하지 않는다.
- **모든 사용자 표시 에러는 `toast` 로 통일한다.** `alert()`, `confirm()`, `window.alert()` 사용 금지. 인라인 폼 에러는 react-hook-form `formState.errors` 로 필드 아래 표시한다.
- **에러를 조용히 삼키지 않는다.** `catch (e) { /* ignore */ }` 패턴 금지. 의도적으로 무시해야 한다면 `logger.warn(...)` 과 함께 왜 무시해도 되는지 한 줄 주석을 남긴다.
- **렌더 에러는 React Error Boundary 로 감싼다.** App Router 의 `error.tsx` 와 `not-found.tsx` 를 적극 활용하고, 크리티컬한 서브트리에는 별도 `<ErrorBoundary>` 를 씌운다.
- **프로덕션 환경에서 민감 정보를 로그에 남기지 않는다.** 개발 환경(`process.env.NODE_ENV === 'development'`) 에서만 `console.error` 를 허용한다. 로거는 `common/utils/code-utils.ts` 의 `logger` 를 사용한다.

---

## axios 인터셉터 에러 처리 (`components/axios-interceptor.tsx`)

### 401 Unauthorized 처리 흐름

```
API 응답 401
    │
    ├─ PUBLIC_ROUTES 인가? (예: /login)
    │   └─ YES → 조용히 reject (토스트/리다이렉트 없음)
    │
    ├─ refresh 요청 자체가 401 인가?
    │   └─ YES → logout() + router.push('/login')
    │
    └─ 일반 API 401
        ├─ refreshAccessToken() 시도
        │   ├─ 성공 → 원래 요청 재시도 (새 토큰 헤더 적용)
        │   └─ 실패 → toast("로그인 만료") + logout() + router.push('/login')
        └─ done
```

```typescript
// axios-interceptor.tsx 핵심 로직
if (error.response?.status === 401) {
  if (PUBLIC_ROUTES.includes(currentPath)) {
    return Promise.reject(error)          // 조용히 처리
  }
  if (isRefreshRequest(error.config?.url)) {
    logout()
    router.push("/login")
    return Promise.reject(error)
  }
  const refreshSuccess = await refreshAccessToken()
  if (refreshSuccess && error.config) {
    // 토큰 갱신 성공 → 원래 요청 재시도
    error.config.headers.Authorization = `Bearer ${getAccessToken()}`
    return appAxios.request(error.config)
  }
  logout(); router.push("/login")
}
```

### showErrorPopup 플래그

```typescript
// api/common/axios.ts — 기본값: true
showErrorPopup: true   // 에러 응답 시 toast 자동 노출

// 개별 요청에서 토스트 억제
appApi.get(appAxios, '/some/url', { showErrorPopup: false })
```

인터셉터는 `error.config?.showErrorPopup` 이 `true` 인 경우에만 toast 를 노출한다. 폴링, 백그라운드 갱신처럼 사용자에게 노출하지 않아야 하는 요청은 `showErrorPopup: false` 를 전달한다.

---

## React Query mutation 에러 처리

mutation 의 `onError` 에서 도메인별 메시지를 분기해 toast 로 표시한다.

```typescript
// hooks/react-query/use<Domain>Queries.ts
export const useEdit<Domain> = () => {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ id, form }) => XxxApi.edit(id, form),
    onSuccess: (data, { id }) => {
      toast({ title: "성공", description: "성공적으로 수정되었습니다." })
      queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
      queryClient.invalidateQueries({ queryKey: xxxKeys.detail(id) })
    },
    onError: (error: Error) => {
      toast({
        title: "오류",
        description: error.message || "수정에 실패했습니다.",
        variant: "destructive",
      })
    }
  })
}
```

### QueryClient 전역 defaultOptions

`components/providers.tsx` 의 `QueryClient` 에서 전역 기본 옵션을 설정한다. 개별 훅에서 동일한 옵션을 중복 선언하지 않는다.

```typescript
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,
      retry: 1,
      refetchOnWindowFocus: false,
    },
    // mutations 전역 onError 는 인터셉터와 중복되므로 설정하지 않음
  },
})
```

---

## React Error Boundary

### App Router error.tsx / not-found.tsx

```
app/
├── error.tsx        ← 최상위 렌더 에러 (서버·클라이언트 컴포넌트 모두)
├── not-found.tsx    ← 404 처리
└── (domain)/
    └── error.tsx    ← 도메인별 에러 (선택적)
```

```typescript
// app/error.tsx
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>문제가 발생했습니다</h2>
      <p>{error.message}</p>
      <button onClick={reset}>다시 시도</button>
    </div>
  )
}
```

### 컴포넌트 레벨 ErrorBoundary

크리티컬한 서브트리(대시보드 차트, 채팅 패널 등) 는 개별 `<ErrorBoundary>` 로 감싸 전체 페이지가 깨지지 않도록 한다.

```tsx
// react-error-boundary 패키지 사용 (설치 필요 시 권고)
import { ErrorBoundary } from 'react-error-boundary'

<ErrorBoundary fallback={<div>차트를 불러올 수 없습니다.</div>}>
  <ChartAreaInteractive />
</ErrorBoundary>
```

---

## 사용자 표시 에러 계층

| 에러 종류 | 표시 방법 | 담당 위치 |
|---------|---------|---------|
| HTTP 401 (인증 만료) | toast + 로그인 리다이렉트 | `axios-interceptor.tsx` |
| HTTP 4xx (비즈니스 에러) | toast (메시지: 서버 응답 `message` 필드) | `axios-interceptor.tsx` |
| HTTP 5xx (서버 에러) | toast ("서버 오류가 발생했습니다") | `axios-interceptor.tsx` |
| 네트워크 에러 | toast ("서버에 연결할 수 없습니다") | `axios-interceptor.tsx` / `auth-guard.tsx` |
| mutation 실패 | toast (도메인별 메시지) | 각 `useXxxQueries.ts` `onError` |
| 폼 입력 오류 | 필드 아래 인라인 메시지 | react-hook-form `formState.errors` |
| 렌더 에러 | 에러 화면 (reset 버튼) | `app/error.tsx` / ErrorBoundary |
| 404 | not-found 화면 | `app/not-found.tsx` |

---

## 로깅

```typescript
// common/utils/code-utils.ts 의 logger 사용
import { logger } from "@/common/utils/code-utils"

logger.info("토큰 갱신 성공")
logger.warn("API Error:", error.message, error.response?.status)
logger.error("Unknown error:", error)
```

- 개발 환경에서는 모든 레벨 출력. 프로덕션에서는 `logger.error` 레벨만 출력.
- `console.log` 직접 사용 금지. `logger` 를 경유한다.
- 에러 로그에 토큰·비밀번호·개인정보가 포함되지 않도록 주의한다.
- 프로덕션 에러 추적이 필요한 경우 Sentry 등 외부 모니터링 도입을 권고한다 (도입 시점은 별도 결정).

---

## 금지사항

```typescript
// ❌ 에러 조용히 삼키기
try {
  await XxxApi.edit(id, form)
} catch (e) {
  // ignore
}

// ❌ alert / confirm 사용
alert("저장되었습니다")
if (confirm("삭제하시겠습니까?")) { ... }

// ❌ 컴포넌트에서 중복 401 처리
const res = await XxxApi.search(params).catch(() => router.push('/login'))
// (axios-interceptor 가 이미 처리함)

// ❌ catch 후 아무것도 하지 않기 (throw 만 하고 사용자 표시 없음)
} catch (error) {
  throw error  // 사용자는 아무것도 볼 수 없음
}

// ✅ 올바른 패턴
onError: (error: Error) => {
  toast({
    title: "오류",
    description: error.message || "처리에 실패했습니다.",
    variant: "destructive",
  })
}
```
