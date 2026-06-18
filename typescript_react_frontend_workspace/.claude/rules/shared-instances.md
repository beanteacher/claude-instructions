# 공유 인스턴스 (모듈 싱글톤 · Provider) 규칙

> 본 규칙은 `conventions.md` 의 "공유 인스턴스" 항목에서 분리된 상세 가이드다.
> 컨벤션 본문에서는 적용 대상 표와 한 줄 원칙만 두고, 세부 예시·판단 흐름은 이 파일을 참조한다.

---

## 핵심 원칙

**무거운 초기화·연결·전역 설정이 필요한 인스턴스는 절대 컴포넌트·훅 내부에서 직접 생성하지 않는다.**
반드시 모듈 싱글톤 또는 최상위 Provider 에서 1회 생성하고, import 하거나 Context 를 통해 받아 사용한다.

---

## 적용 대상 (프로젝트 실제 위치)

| 인스턴스 | 처리 방식 | 위치 |
|----------|----------|------|
| `appAxios` (axios instance) | 모듈 싱글톤 — `axios.create()` 1회 실행, `paramsSerializer` 설정 포함 | `api/common/axios.ts` |
| axios 인터셉터 등록 | `useEffect` 마운트 컴포넌트 — `appAxios` 를 import 해 등록 | `components/axios-interceptor.tsx` |
| `QueryClient` | `Providers` 컴포넌트에서 `useState(() => new QueryClient(...))` 1회 생성 | `components/providers.tsx` |
| `QueryClientProvider` | `Providers` 컴포넌트 최상위 래핑 | `components/providers.tsx` |
| `authStore` (zustand) | 모듈 싱글톤 — `create<AuthState>()(...)` 1회 실행 | `store/authStore.ts` |
| `Toaster` | `AppLayout` 에서 1회 마운트 | `components/app-layout.tsx` |
| `SidebarProvider` | `AppLayout` 에서 1회 래핑 | `components/app-layout.tsx` |
| `SmartSheetProvider` | 페이지별 1회 래핑 (sheet 컨텍스트 범위) | `page/*Page.tsx` 의 최상위 |
| `TooltipProvider` | `BaseTable` 내부에서 래핑 (테이블 단위) | `components/shared/table/BaseTable.tsx` |
| react-hook-form resolver | 폼별로 생성 OK (경량) — 단 zod 스키마는 `model/schema/` 에서 export 후 재사용 | `components/sheet/form/*.tsx` |

---

## 금지 / 권장 코드 예시

### axios instance

```ts
// ❌ 컴포넌트·훅에서 axios 인스턴스 직접 생성
function useMyData() {
  const myAxios = axios.create({ baseURL: process.env.NEXT_PUBLIC_API_BASE_URL })
  return useQuery({ queryFn: () => myAxios.get('/data') })
}

// ❌ api 파일에서 새 인스턴스 생성
const localAxios = axios.create({ baseURL: '...' })
export const search = (params) => localAxios.get('/search', { params })

// ✅ 공용 인스턴스 import
import { appAxios } from '@/api/common/axios'
import appApi from '@/api/common/app-api'
export const search = (params) => appApi.get(appAxios, '/search', { params })
```

### QueryClient

```tsx
// ❌ 컴포넌트 본문에서 직접 생성 (매 렌더마다 새로 생성됨)
function MyPage() {
  const queryClient = new QueryClient()  // ❌
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>
}

// ❌ 모듈 최상위에서 생성 (SSR 에서 인스턴스가 공유됨)
const queryClient = new QueryClient()  // ❌ 모듈 레벨

// ✅ useState 로 1회 생성 (컴포넌트 마운트당 1회 보장)
export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: { staleTime: 60 * 1000, retry: 1, refetchOnWindowFocus: false }
        }
      })
  )
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}
```

### authStore (zustand)

```ts
// ❌ 컴포넌트마다 store 생성
function LoginForm() {
  const store = create<AuthState>()(...)  // ❌
}

// ✅ 모듈 싱글톤 import
import { useAuthStore } from '@/store/authStore'
function LoginForm() {
  const { setToken } = useAuthStore()
}
```

### Toaster

```tsx
// ❌ 여러 곳에 중복 마운트
function <Domain>Page() {
  return <>
    <Toaster />   {/* ❌ 페이지마다 마운트 */}
    <Table />
  </>
}

// ✅ app-layout.tsx 에서 1회만
export function AppLayout({ children }) {
  return (
    <Providers>
      <Toaster />   {/* ✅ 전역 1회 */}
      {children}
    </Providers>
  )
}
```

### zod 스키마

```ts
// ❌ 컴포넌트 내부에서 스키마 인라인 정의 (매 렌더마다 재생성)
function <Domain>Form() {
  const schema = z.object({ name: z.string().min(1) })  // ❌
  const form = useForm({ resolver: zodResolver(schema) })
}

// ✅ model/schema/ 에서 import
import { XxxSchema } from '@/model/schema/<domain>-schema'
function <Domain>Form() {
  const form = useForm<XxxSchema.CreateFormValues>({
    resolver: zodResolver(XxxSchema.CreateSchema)  // ✅
  })
}
```

---

## 판단 흐름

```
새 인스턴스를 만들고 싶은가?

1. 여러 컴포넌트·훅에서 동일 인스턴스를 공유해야 하는가?
   - YES → 다음 단계
   - NO  → 컴포넌트별 로컬 생성 가능 (react-hook-form useForm 등)

2. 무거운 초기화·연결·설정이 있는가?
   (HTTP 클라이언트, QueryClient, zustand store, Provider 등)
   - YES → 모듈 싱글톤 또는 Provider 에서 1회 생성
   - NO  → 컴포넌트별 생성 가능

3. 전역 설정 공유가 필요한가?
   (baseURL, defaultOptions, interceptor 등)
   - YES → 반드시 모듈 싱글톤 또는 Provider
   - NO  → 상황에 따라 판단

4. SSR / Next.js 환경에서 인스턴스가 요청 간 공유되면 안 되는가?
   - YES → 컴포넌트 내 useState(() => new Instance()) 패턴 (QueryClient)
   - NO  → 모듈 최상위 싱글톤 가능 (zustand store, appAxios)
```

`appAxios`, `authStore`, `appApi` 는 위 모든 항목에 해당하므로 **항상 모듈 싱글톤으로 import 해 사용**한다.

---

## 환경별 설정값 주입

인스턴스 생성에 필요한 환경별 값은 **`process.env.NEXT_PUBLIC_*`** 로 빌드 타임에 주입한다.

```ts
// api/common/axios.ts
export const appAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,  // ✅ 빌드 타임 주입
  ...
})

// ❌ 하드코딩
export const appAxios = axios.create({
  baseURL: 'http://localhost:8080',  // ❌ 환경별로 다른 값
})
```

브라우저에서 접근해야 하는 값만 `NEXT_PUBLIC_` 접두사를 사용하고, 서버 전용 시크릿에는 붙이지 않는다.

---

## 같은 타입의 여러 인스턴스가 필요한 경우

현재 프로젝트는 단일 `appAxios` 인스턴스만 사용한다. 만약 별도 baseURL 이 필요한 외부 API 가 추가된다면:

```ts
// api/common/axios.ts
export const appAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
  ...
})

export const externalAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_EXTERNAL_API_URL,
  ...
})

// api/external-api.ts
import { externalAxios } from '@/api/common/axios'
export const fetchExternalData = () =>
  appApi.get(externalAxios, '/external/endpoint')
```

공통 `axios.ts` 파일에 named export 로 추가하고, api 모듈에서 필요한 인스턴스를 선택해 사용한다.

---

## 인스턴스 라이프사이클 요약

| 인스턴스 | 생성 시점 | 소멸 시점 | 주의사항 |
|----------|-----------|-----------|----------|
| `appAxios` | 모듈 로드 시 1회 | 앱 종료 | interceptor 는 `AxiosInterceptor` cleanup 에서 해제 |
| `QueryClient` | `Providers` 마운트 시 1회 | `Providers` 언마운트 | `useState` 초기화 함수로 생성 |
| `authStore` | 모듈 로드 시 1회 | 앱 종료 | 메모리에만 저장 (localStorage 비사용) |
| `SmartSheetProvider` | 페이지 마운트 시 | 페이지 언마운트 | 페이지별 sheet 컨텍스트 분리 |
| zod 스키마 | 모듈 로드 시 1회 | 앱 종료 | 컴포넌트 내부 인라인 정의 금지 |
