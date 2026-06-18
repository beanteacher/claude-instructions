# React 코드 패턴 참고 자료 (코드 예시 전용)

> **이 파일은 코드 예시만 포함합니다. 규칙·컨벤션은 `conventions.md`를 참고하세요.**
> 디자인 패턴 가이드: `design-patterns.md`. 매핑 규칙: `mapping.md`. 공유 인스턴스: `shared-instances.md`.
> 아래 예시는 특정 도메인 기반이며 패턴 학습 목적으로 사용한다. 실제 도메인명은 프로젝트에 맞게 치환한다.

---

## 레이어별 코드 구조

### 1. `model/dto/<domain>-dto.ts` — 백엔드 응답 타입

```ts
// model/dto/<domain>-dto.ts
import { XxxRelatedDto } from "./<related>-dto"

export * as XxxDto from './<domain>-dto'

export interface Summary {
  id?: number
  name?: string
  statusCode?: string        // 프로젝트 enum 타입으로 교체
  createdAt?: string
  modifiedAt?: string
}

export interface Detail extends Summary {
  relatedItem?: XxxRelatedDto.Summary
}

export interface SearchParams {
  keyword?: string
  page?: number
  size?: number
}

export interface CreateForm {
  name: string
  statusCode?: string
}

export interface EditForm {
  name?: string
  statusCode?: string
}
```

핵심 포인트:
- `export * as XxxDto from './<domain>-dto'` 로 namespace 를 만들어 `XxxDto.Summary` 형태로 사용.
- 인터페이스만 선언한다 — 함수, 클래스, 로직 금지.
- `extends` 로 공통 필드를 상속한다 (`Detail extends Summary`).

금지 패턴:
```ts
// ❌ 타입 파일에 변환 함수 추가
export function toDetail(raw: any): XxxDto.Detail { ... }

// ❌ 모든 필드를 필수로 선언 (백엔드 응답이 optional 일 수 있음)
export interface Summary { id: number; name: string }
```

---

### 2. `model/schema/<domain>-schema.ts` — zod 스키마 + 폼 타입

```ts
// model/schema/<domain>-schema.ts
import { z } from 'zod'

export * as XxxSchema from './<domain>-schema'

const nameField = z.string()
  .min(1, "이름은 필수입니다")
  .max(200, "최대 200자까지 입력 가능합니다")

export const CreateSchema = z.object({
  name: nameField,
  // 프로젝트 필드에 맞게 추가
})

export const EditSchema = z.object({
  name: nameField.optional(),
  // 프로젝트 필드에 맞게 추가
})

// 타입은 스키마에서 추출 — 별도 interface 선언 금지
export type CreateFormValues = z.infer<typeof CreateSchema>
export type EditFormValues = z.infer<typeof EditSchema>

export const getFormSchema = (isEdit?: boolean) => {
  return isEdit ? EditSchema : CreateSchema
}
```

핵심 포인트:
- `z.infer<typeof Schema>` 로 타입 추출 — TypeScript `interface` 와 zod 스키마를 이중 선언하지 않는다.
- 재사용 필드(`nameField` 등)를 변수로 분리해 스키마 간 공유.
- `getFormSchema(isEdit?)` 헬퍼로 create/edit 분기 처리.

---

### 3. `api/<domain>-api.ts` — HTTP 호출

```ts
// api/<domain>-api.ts
import appApi from "@/api/common/app-api"
import { appAxios } from "@/api/common/axios"
import { XxxDto } from "@/model/dto/<domain>-dto"
import { PageListDto } from "@/model/page-list-dto"

export * as XxxApi from "./<domain>-api"

const BASE_URL = '/<domain>'

export const search = (params: XxxDto.SearchParams) =>
  appApi.get<PageListDto.Response<XxxDto.Summary>>(appAxios, `${BASE_URL}/search`, { params })

export const detail = (id: number) =>
  appApi.get<XxxDto.Detail>(appAxios, `${BASE_URL}/${id}`)

export const create = (data: XxxDto.CreateForm) =>
  appApi.post<XxxDto.Detail>(appAxios, BASE_URL, { data })

export const edit = (id: number, data: XxxDto.EditForm) =>
  appApi.put<XxxDto.Detail>(appAxios, `${BASE_URL}/${id}`, { data })
```

핵심 포인트:
- `export * as XxxApi from "./<domain>-api"` 로 namespace export — hook 에서 `XxxApi.search(...)` 형태로 사용.
- `appApi.get/post/put/patch/delete` + `appAxios` 조합 필수.
- `BASE_URL` 상수를 파일 상단에 선언한다.
- 반환 타입은 `Promise<ResponseModel<T>>` — `appApi` 내부에서 처리하므로 명시 불필요.

금지 패턴:
```ts
// ❌ fetch 직접 사용
const response = await fetch(`/<domain>/${id}`)

// ❌ axios 인스턴스 직접 생성
const myAxios = axios.create({ baseURL: process.env.NEXT_PUBLIC_API_BASE_URL })

// ❌ 응답 파싱 로직을 api 모듈에 추가
export const search = async (params) => {
  const response = await appApi.get(...)
  return response.data.list.map(item => ({ ...item, label: item.name }))
}
```

---

### 4. `hooks/react-query/use<Domain>Queries.ts` — TanStack Query 훅

```ts
// hooks/react-query/use<Domain>Queries.ts
import { useQuery, useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { toast } from "@/components/ui/use-toast"
import { XxxApi } from '@/api/<domain>-api'
import { XxxDto } from '@/model/dto/<domain>-dto'

// 쿼리 키 팩토리 — 파일 상단에 필수 선언
export const xxxKeys = {
  all: ['<domain>'] as const,
  lists: () => [...xxxKeys.all, 'list'] as const,
  list: (params: XxxDto.SearchParams) => [...xxxKeys.lists(), params] as const,
  details: () => [...xxxKeys.all, 'detail'] as const,
  detail: (id: number) => [...xxxKeys.details(), id] as const,
}

// 목록 조회
export const useXxxList = (params: XxxDto.SearchParams) => {
  return useQuery({
    queryKey: xxxKeys.list(params),
    queryFn: async () => {
      const response = await XxxApi.search(params)
      return response.data
    },
  })
}

// 단건 조회
export const useXxxDetail = (id: number) => {
  return useQuery({
    queryKey: xxxKeys.detail(id),
    queryFn: async () => {
      const response = await XxxApi.detail(id)
      return response.data
    },
    enabled: !!id,  // id 가 falsy 이면 쿼리 비활성화
  })
}

// 무한 스크롤 검색
export const useXxxSearch = (keyword: string) => {
  return useInfiniteQuery({
    queryKey: xxxKeys.list({ keyword: keyword.trim() || undefined }),
    queryFn: async ({ pageParam = 1 }) => {
      const params: XxxDto.SearchParams = { page: pageParam as number, size: 50 }
      if (keyword.trim()) params.keyword = keyword.trim()
      const response = await XxxApi.search(params)
      return response.data
    },
    getNextPageParam: (lastPage, allPages) => {
      if (lastPage?.list && lastPage.list.length === 50) {
        return allPages.length + 1
      }
      return undefined
    },
    initialPageParam: 1,
    staleTime: 30000,
    gcTime: 300000,
  })
}

// 생성 mutation
export const useCreateXxx = () => {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (data: XxxDto.CreateForm) => XxxApi.create(data),
    onSuccess: () => {
      toast({ title: "성공", description: "성공적으로 등록되었습니다." })
      queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
    },
    onError: () => {
      toast({ variant: "destructive", title: "오류", description: "등록에 실패했습니다." })
    }
  })
}

// 수정 mutation
export const useEditXxx = () => {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: XxxDto.EditForm }) =>
      XxxApi.edit(id, data),
    onSuccess: () => {
      toast({ title: "성공", description: "성공적으로 수정되었습니다." })
      queryClient.invalidateQueries({ queryKey: xxxKeys.lists() })
      queryClient.invalidateQueries({ queryKey: xxxKeys.details() })
    },
    onError: () => {
      toast({ variant: "destructive", title: "오류", description: "수정에 실패했습니다." })
    }
  })
}
```

핵심 포인트:
- `*Keys` 팩토리 객체를 파일 상단에 선언해 모든 쿼리 키를 한 곳에서 관리.
- `enabled: !!id` 처럼 조건부 활성화로 불필요한 요청을 방지.
- `onSuccess` 에서 toast + `invalidateQueries` 를 항상 함께 처리.
- `useInfiniteQuery` 사용 시 `initialPageParam` 을 명시한다.

금지 패턴:
```ts
// ❌ queryKey 를 inline 문자열로 작성
queryKey: ['<domain>', 'list', params]

// ❌ hook 내부에서 직접 axios 호출
queryFn: async () => {
  const response = await appAxios.get('/<domain>/search')
  return response.data
}

// ❌ onSuccess 에서 invalidateQueries 없이 UI 만 업데이트
onSuccess: () => { setIsOpen(false) }
```

---

### 5. `hooks/use<Domain>SearchParams.ts` — URL 파라미터 상태

```ts
// hooks/use<Domain>SearchParams.ts
import { DEFAULT_PAGE_SIZE } from '@/common/constants'
import { useUrlSearchParams } from './useUrlSearchParams'
import { XxxDto } from '@/model/dto/<domain>-dto'
import type { XxxFilters } from '@/model/client/<domain>-types'
import type { PaginationInfo } from '@/model/client/common-types'
import { SearchUtils } from "@/common/utils/search-utils"

const defaultValues: XxxDto.SearchParams = {
  keyword: '',
  page: 1,
  size: DEFAULT_PAGE_SIZE
}

const serialize = SearchUtils.createSerializer<XxxDto.SearchParams>(['size'])
const deserialize = SearchUtils.createDeserializer<XxxDto.SearchParams>(
  defaultValues,
  []  // 배열 필드가 있으면 여기에 키 추가
)

export function use<Domain>SearchParams() {
  const {
    appliedValues,
    tempValues,
    updateTempValues,
    updateAppliedValues,
    applySearch: originalApplySearch,
    reset,
    isInitialized
  } = useUrlSearchParams<XxxDto.SearchParams>({
    defaultValues,
    serialize,
    deserialize,
    manualSearch: true as const
  })

  const appliedFilters: XxxFilters = {
    keyword: appliedValues.keyword || '',
  }

  const filters: XxxFilters = {
    keyword: tempValues.keyword || '',
  }

  const pagination: PaginationInfo = {
    page: appliedValues.page || 1,
    size: appliedValues.size || DEFAULT_PAGE_SIZE
  }

  return {
    appliedFilters,
    filters,
    pagination,
    setKeyword: (keyword: string) => updateTempValues({ keyword }),
    setPage: (page: number) => updateAppliedValues({ page }),
    applySearch: () => originalApplySearch({ page: 1 }),
    reset,
    isInitialized
  }
}
```

핵심 포인트:
- `useUrlSearchParams` 를 기반으로 확장한다 — `useSearchParams` 를 각 컴포넌트에서 직접 사용하지 않는다.
- `appliedFilters` (실제 적용) 와 `filters` (임시 UI 상태) 를 분리해 "검색" 버튼 누르기 전까지는 쿼리가 바뀌지 않도록 한다.
- `applySearch` 시 `page: 1` 로 초기화한다.

---

### 6. `page/<domain>/<Domain>Page.tsx` — 화면 조립

```tsx
// page/<domain>/<Domain>Page.tsx
"use client"

import { SmartSheetProvider } from "@/components/ui/smart-sheet"
import { use<Domain>Actions } from "./<domain>-actions"
import { <Domain>SearchFilters } from "./items/<Domain>SearchFilters"
import { <Domain>Table } from "./items/<Domain>Table"

function <Domain>PageContent() {
  const {
    filters,
    currentPage,
    items,
    totalCount,
    totalPages,
    isLoading,
    error,
    handleSearch,
    handleReset,
    handleRowClick,
    handlePageChange,
    handleFilterChange
  } = use<Domain>Actions()

  return (
    <div className="flex flex-col gap-4 py-4 md:gap-6 md:py-6">
      <<Domain>SearchFilters
        filters={filters}
        isLoading={isLoading}
        onSearch={handleSearch}
        onReset={handleReset}
        onFilterChange={handleFilterChange}
      />
      <<Domain>Table
        data={items}
        totalCount={totalCount}
        totalPages={totalPages}
        currentPage={currentPage}
        isLoading={isLoading}
        error={error}
        onRowClick={handleRowClick}
        onPageChange={handlePageChange}
      />
    </div>
  )
}

export default function <Domain>Page() {
  return (
    <SmartSheetProvider>
      <<Domain>PageContent />
    </SmartSheetProvider>
  )
}
```

핵심 포인트:
- `SmartSheetProvider` 로 페이지 전체를 감싸 sheet 컨텍스트를 제공한다.
- 내부 `*PageContent` 컴포넌트가 실제 로직을 담당하고, export default 는 Provider 래핑만 한다.
- `use<Domain>Actions()` 훅 하나에서 모든 상태·핸들러를 가져온다 — 페이지 컴포넌트에 로직이 없다.

---

### 7. `page/<domain>/<domain>-actions.tsx` — 액션 Hook (Container Logic)

```tsx
// page/<domain>/<domain>-actions.tsx
import { useSmartSheet } from "@/components/ui/smart-sheet"
import { XxxDto } from "@/model/dto/<domain>-dto"
import { useXxxList } from "@/hooks/react-query/use<Domain>Queries"
import <Domain>Sheet from "@/components/sheet/<Domain>Sheet"
import { use<Domain>SearchParams } from "@/hooks/use<Domain>SearchParams"
import type { <Domain>PageReturn } from "@/model/client/<domain>-types"

export function use<Domain>Actions(): <Domain>PageReturn {
  const { openSheet } = useSmartSheet()
  const {
    appliedFilters,
    filters,
    pagination,
    setKeyword,
    setPage,
    applySearch,
    reset,
    isInitialized
  } = use<Domain>SearchParams()

  const searchParams: XxxDto.SearchParams = {
    page: pagination.page,
    size: pagination.size,
    keyword: appliedFilters.keyword || undefined,
  }

  const { data, isLoading, error } = useXxxList(searchParams)

  const handleRowClick = (item: XxxDto.Summary) => {
    openSheet({
      id: `<domain>-detail-${item.id}`,
      title: `상세 정보 - ${item.name}`,
      content: <<Domain>Sheet itemId={item.id} />
    })
  }

  const handleCreate = () => {
    openSheet({
      id: "<domain>-create",
      title: "등록",
      content: <<Domain>Sheet />
    })
  }

  return {
    filters,
    currentPage: pagination.page,
    items: data?.list || [],
    totalCount: data?.total || 0,
    totalPages: data?.pages || 1,
    isLoading,
    error,
    isInitialized,
    handleSearch: applySearch,
    handleReset: reset,
    handleRowClick,
    handleCreate,
    handlePageChange: setPage,
    handleFilterChange: {
      keyword: setKeyword,
    }
  }
}
```

핵심 포인트:
- `.tsx` 확장자 — JSX 를 포함하므로 (sheet content 에 컴포넌트를 인라인 전달).
- 반환 타입은 `model/client/` 에 선언된 `*PageReturn` 인터페이스로 명시한다.
- sheet 열기(`openSheet`) 가 이 레이어에서 이루어진다 — page 컴포넌트는 단순 전달만.

---

### 8. `app/<route>/page.tsx` — Next.js 라우트 진입점

```tsx
// app/(<group>)/<domain>/page.tsx
import <Domain>Page from "@/page/<domain>/<Domain>Page"

export default function <Domain>App() {
  return <<Domain>Page />
}
```

핵심 포인트:
- 로직이 없다. page/ 컴포넌트 호출만.
- `'use client'` 없음 — 서버 컴포넌트로 유지.
- 메타데이터가 필요하면 `export const metadata = { title: '...' }` 를 추가한다.

---

### 9. `components/sheet/<Domain>Sheet.tsx` — Sheet 컴포넌트

```tsx
// components/sheet/<Domain>Sheet.tsx
"use client"

import { useXxxDetail } from "@/hooks/react-query/use<Domain>Queries"
import { <Domain>Form } from "@/components/sheet/form/<Domain>Form"
import { SheetDataLoader } from "./SheetDataLoader"

interface <Domain>SheetProps {
  itemId?: number
}

export default function <Domain>Sheet({ itemId }: <Domain>SheetProps) {
  if (!itemId) {
    return <<Domain>Form />
  }

  const { data: item, isLoading, error } = useXxxDetail(itemId)

  return (
    <SheetDataLoader
      isLoading={isLoading}
      error={error}
      data={item}
      resourceName="항목"
    >
      {(item) => <<Domain>Form item={item} />}
    </SheetDataLoader>
  )
}
```

핵심 포인트:
- `itemId` 유무로 등록/수정 모드를 분기한다.
- 로딩·에러 상태는 `SheetDataLoader` 에 위임한다.
- `<Domain>Form` 은 props 로 데이터를 받아 렌더하며 직접 API 를 호출하지 않는다.

---

### 10. `components/sheet/form/<Domain>Form.tsx` — 폼 컴포넌트

```tsx
// components/sheet/form/<Domain>Form.tsx (구조 예시)
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { XxxSchema } from "@/model/schema/<domain>-schema"
import { XxxDto } from "@/model/dto/<domain>-dto"
import { useCreateXxx, useEditXxx } from "@/hooks/react-query/use<Domain>Queries"

interface <Domain>FormProps {
  item?: XxxDto.Detail  // 없으면 등록 모드
}

export function <Domain>Form({ item }: <Domain>FormProps) {
  const isEdit = !!item
  const schema = XxxSchema.getFormSchema(isEdit)

  const form = useForm<XxxSchema.CreateFormValues>({
    resolver: zodResolver(schema),
    defaultValues: {
      name: item?.name || '',
      // 프로젝트 필드에 맞게 추가
    }
  })

  const { mutate: createItem, isPending: isCreating } = useCreateXxx()
  const { mutate: editItem, isPending: isEditing } = useEditXxx()

  const onSubmit = (values: XxxSchema.CreateFormValues) => {
    if (isEdit && item?.id) {
      editItem({ id: item.id, data: values })
    } else {
      createItem(values)
    }
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* FormField 컴포넌트들 */}
    </form>
  )
}
```

핵심 포인트:
- `useForm` 의 `resolver` 에 `zodResolver(schema)` 를 연결한다.
- `defaultValues` 에 기존 데이터를 넣어 수정 모드를 처리한다.
- `isPending` 으로 로딩 상태를 버튼에 반영한다.
- 폼 컴포넌트는 mutation hook 을 가져오되, API 를 직접 호출하지 않는다.

---

### 11. `components/shared/table/BaseTable.tsx` — 공용 테이블

```tsx
// 사용 예시 (<Domain>Table.tsx 에서)
import { BaseTable } from "@/components/shared/table/BaseTable"
import { XxxDto } from "@/model/dto/<domain>-dto"

export function <Domain>Table({ data, isLoading, error, onRowClick, ...rest }) {
  return (
    <BaseTable<XxxDto.Summary>
      title="목록"
      data={data}
      isLoading={isLoading}
      error={error}
      columnsCount={4}
      tableHeaders={
        <>
          <TableHead>ID</TableHead>
          <TableHead>이름</TableHead>
          {/* ... */}
        </>
      }
      renderRow={(item) => (
        <<Domain>TableRow
          key={item.id}
          item={item}
          onClick={() => onRowClick?.(item)}
        />
      )}
      {...rest}
    />
  )
}
```

핵심 포인트:
- `BaseTable<T>` 는 제네릭 컴포넌트 — 데이터 타입을 타입 파라미터로 전달.
- `renderRow` prop 으로 행 렌더를 주입한다 (Render Prop 패턴).
- 로딩·에러·빈 상태는 `BaseTable` 내부의 `TableStates` 가 처리한다.
- 페이지네이션은 `showPagination` prop 으로 활성화한다.

---

## 인증 플로우 요약

```
app/layout.tsx
  └── AppLayout (components/app-layout.tsx)
        ├── Providers (QueryClient)
        ├── AxiosInterceptor (토큰 첨부·갱신·401 처리)
        ├── Toaster
        └── AuthGuard
              └── {children}  ← 인증된 경우만 렌더
```

- `authStore` (zustand) 가 메모리에만 `accessToken` 을 보관.
- `AxiosInterceptor` 가 모든 요청에 `Authorization: Bearer {token}` 을 붙인다.
- 401 응답 → 토큰 갱신 → 재시도 → 실패 시 `/login` 리다이렉트.
- `AuthGuard` 는 초기 마운트 시 refresh API 를 호출해 토큰 유효성을 확인한다.
