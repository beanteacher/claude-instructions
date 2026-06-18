# API 규약 (Iron Rule)

> 본 문서는 **프론트엔드가 백엔드 API 를 호출할 때 따라야 할 규약**이다.
> 백엔드가 노출하는 URL 구조·응답 포맷·인증 방식은 변경하지 않는다.

- **`api/*-api.ts` 모듈만 `appAxios` 를 직접 호출한다.** 컴포넌트·훅·스토어에서 `appAxios` 또는 `axios` 를 직접 import 하지 않는다.
- **`appApi` 래퍼 함수(`api/common/app-api.ts`)를 경유한다.** `appApi.get / post / put / patch / delete` 를 사용하고, `axios.request()` 를 직접 부르지 않는다.
- **응답 타입은 `model/dto/*-dto.ts` 에 선언된 인터페이스로 받는다.** `any` 응답 타입 금지.
- **쿼리 파라미터 직렬화는 `paramsSerializer` 에 위임한다.** `qs.stringify` 가 `api/common/axios.ts` 의 `appAxios` 에 이미 등록되어 있으므로 배열 파라미터를 수동으로 join 하지 않는다.
- **React Query 키는 `[domain, 'list'|'detail', params?]` 튜플 컨벤션을 따른다.** mutation 성공 후에는 `queryClient.invalidateQueries({ queryKey: domainKeys.lists() })` 로 캐시를 무효화한다.

---

## axios 인스턴스 구조

```
api/common/axios.ts      → appAxios 인스턴스 (baseURL, paramsSerializer, showErrorPopup 기본값 정의)
api/common/app-api.ts    → appApi 래퍼 (GET/POST/PUT/PATCH/DELETE 메서드 추상화)
components/axios-interceptor.tsx
                         → 요청 인터셉터 (Authorization 헤더 주입, 토큰 갱신)
                           응답 인터셉터 (401 처리, showErrorPopup 토스트, 로그아웃)
```

### appAxios 주요 설정 (`api/common/axios.ts`)

```typescript
export const appAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
  headers: { Accept: "application/json" },
  paramsSerializer: (params) =>
    qs.stringify(params, {
      arrayFormat: 'repeat',  // roles=A&roles=B 형태
      skipNulls: true         // null/undefined 값 제외
    }),
  showErrorPopup: true,       // 기본: 에러 응답 시 toast 노출
  showLoading: false,
})
```

- `NEXT_PUBLIC_API_BASE_URL` 은 반드시 환경변수로 공급한다. 코드에 하드코딩하지 않는다.
- `showErrorPopup: false` 로 개별 요청에서 토스트 억제 가능 (예: 자동 폴링 요청).

---

## api 모듈 파일 목록

각 도메인마다 `api/<domain>-api.ts` 파일을 생성한다. 예시:

| 파일 | BASE_URL | 주요 기능 |
|------|----------|----------|
| `api/auth-api.ts` | `/auth` | 로그인(`/auth/login`), 토큰 갱신(`/auth/refresh`), 로그아웃 |
| `api/<domain-a>-api.ts` | `/<domain-a>` | 도메인 A CRUD |
| `api/<domain-b>-api.ts` | `/<domain-b>` | 도메인 B 조회/수정 |

---

## API 모듈 작성 패턴

```typescript
// api/<domain>-api.ts
import appApi from "@/api/common/app-api"
import { appAxios } from "@/api/common/axios"
import { XxxDto } from "@/model/dto/<domain>-dto"
import { PageListDto } from "@/model/page-list-dto"

export * as XxxApi from "./<domain>-api"

const BASE_URL = '/<domain>'

// 목록 조회 — GET /<domain>/search?page=1&size=20&keyword=...
export const search = (params: XxxDto.SearchParams) =>
  appApi.get<PageListDto.Response<XxxDto.Summary>>(appAxios, `${BASE_URL}/search`, { params })

// 단건 조회 — GET /<domain>/{id}
export const getById = (id: string) =>
  appApi.get<XxxDto.Detail>(appAxios, `${BASE_URL}/${id}`)

// 수정 — PUT /<domain>/{id}
export const edit = (id: string, form: XxxDto.EditForm) =>
  appApi.put<XxxDto.Detail>(appAxios, `${BASE_URL}/${id}`, { data: form })
```

- `BASE_URL` 은 파일 상단 상수로 선언. 여러 함수에서 중복 문자열 사용 금지.
- 반환 타입은 `Promise<ResponseModel<T>>`. `T` 는 `model/dto/` 의 구체적 타입으로 명시.
- `export * as XxxApi from "./<domain>-api"` 패턴으로 네임스페이스 re-export. 훅에서 `{ XxxApi }` 로 import.

---

## 응답 모델

```typescript
// model/response-model.ts
export default interface ResponseModel<T> {
  code?: string
  message?: string
  data?: T
}

// model/page-list-dto.ts (페이지네이션 응답)
export interface Response<T> {
  total?: number
  pages?: number
  page?: number
  list?: Array<T>
}
```

- 단건 응답: `ResponseModel<DocumentDto.Detail>` → `response.data` 로 접근.
- 목록 응답: `ResponseModel<PageListDto.Response<DocumentDto.Summary>>` → `response.data?.list` 로 접근.
- 단순 성공: `ResponseModel<boolean>` 또는 `ResponseModel<void>`.

---

## React Query 키 컨벤션

```typescript
// 키 팩토리 패턴 — hooks/react-query/use<Domain>Queries.ts
export const xxxKeys = {
  all: ['<domain>s'] as const,
  lists: () => [...xxxKeys.all, 'list'] as const,
  list: (params: XxxDto.SearchParams) => [...xxxKeys.lists(), params] as const,
  details: () => [...xxxKeys.all, 'detail'] as const,
  detail: (id: string) => [...xxxKeys.details(), id] as const,
}
```

| 상황 | invalidate 범위 |
|------|----------------|
| 생성/삭제 | `queryClient.invalidateQueries({ queryKey: xxxKeys.all })` |
| 수정 | `queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })` + `queryClient.invalidateQueries({ queryKey: xxxKeys.detail(id) })` |
| 특정 파라미터 캐시만 | `queryClient.invalidateQueries({ queryKey: xxxKeys.list(params) })` |

- 다른 도메인의 쿼리를 직접 invalidate 해야 할 경우, 해당 도메인의 키 팩토리를 import 해서 사용한다. 문자열 배열을 하드코딩하지 않는다.

---

## URL 구조

```
GET    /auth/login              로그인
POST   /auth/refresh            토큰 갱신 (httpOnly 쿠키 기반)
POST   /auth/logout             로그아웃

GET    /<domain-a>              목록
POST   /<domain-a>              생성
PUT    /<domain-a>/{id}         수정
DELETE /<domain-a>/{id}         삭제

GET    /<domain-b>/search       목록 (검색)
GET    /<domain-b>/{id}         단건
PUT    /<domain-b>/{id}         수정
```

- URL prefix 규칙은 프로젝트 백엔드 설정을 따른다 (예: `/api/v1/` 유무).
- 공개 경로(`/auth/login`, `/auth/refresh` 등)는 `common/constants.ts` 의 `PUBLIC_ROUTES` 에 등록되어 인증 가드와 axios 인터셉터가 인증 처리를 건너뛴다.

---

## 페이지네이션 파라미터

```typescript
// model/page-list-dto.ts
export interface Request {
  page?: number       // 1-based
  size?: number       // 기본값: DEFAULT_PAGE_SIZE (20)
  order?: string
  direction?: AppType.SortDirection  // 'ASC' | 'DESC'
}
```

- 프론트 요청은 `?page=1&size=20&sort=id,desc` 형태를 따른다.
- 기본 페이지 크기는 `common/constants.ts` 의 `DEFAULT_PAGE_SIZE = 20` 을 사용한다. 하드코딩 금지.

---

## 금지사항

```typescript
// ❌ api/ 외부에서 appAxios 직접 호출
import { appAxios } from "@/api/common/axios"
const res = await appAxios.get('/document')

// ❌ fetch / axios 기본 인스턴스 사용
const res = await fetch('/api/document')
import axios from 'axios'; axios.get(...)

// ❌ 응답 타입 any
appApi.get<any>(appAxios, '/document')

// ❌ 배열 파라미터 수동 직렬화
params.roles = roles.join(',')

// ✅ api 모듈 경유
import { XxxApi } from "@/api/<domain>-api"
const res = await XxxApi.search(params)
```
