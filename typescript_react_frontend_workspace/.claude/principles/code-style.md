# 코드 스타일 (Iron Rule)

- 프로젝트 포매터/린터 설정을 최우선으로 따른다. ESLint (`eslint-config-next` 기반), Prettier 설정이 있으면 그 결과가 우선이다.
- **영리한 추상화보다 명확한 이름의 작은 함수를 선호한다.** 중첩 3단 이상의 체인/조건보다 이름이 설명해 주는 helper 2~3개로 나누는 쪽이 좋다.
- **브라운필드 파일에서는 diff 표면을 최소화한다.** 기존 파일의 포매팅을 대량으로 바꾸지 않는다. 로직 변경과 포매팅이 같은 PR 에 섞이면 리뷰 반려 대상이다. 포매팅만 정리하려면 별도 `style: ...` 커밋으로 분리한다.
- **승인된 패턴 우선 재사용.** 아래 문서에 기재된 패턴·예시 외의 구조를 도입하려면 PR 본문에 근거를 남긴다.
  - `.claude/conventions/architecture.md` / `library-stack.md` / `nextjs.md`
  - `.claude/rules/conventions.md` / `react-patterns.md` / `design-patterns.md` / `mapping.md`
- **주석은 WHY 를 남긴다.** WHAT 은 잘 지어진 이름과 타입이 대신한다. 불필요한 `// TODO` / `// FIXME` 를 남기려면 이슈 링크·복구 계획을 같이 적는다. 한글 주석 허용.

---

## TypeScript

### `any` 금지

```typescript
// ❌ 금지
const data: any = response.data
function process(input: any) { ... }

// ✅ unknown + 타입 가드
const data: unknown = response.data
if (isDocumentSummary(data)) { ... }

// ✅ 구체 타입 명시
const data: DocumentDto.Summary = response.data
```

- 불가피하게 `any` 가 필요한 경우 `// eslint-disable-next-line @typescript-eslint/no-explicit-any` 와 함께 이유를 주석으로 명시한다.
- 함수 파라미터, 반환 타입, 상태 타입에 `any` 를 쓰면 리뷰 반려 대상이다.

### `interface` vs `type`

| 상황 | 선택 | 이유 |
|------|------|------|
| 컴포넌트 props | `interface XxxProps` | 확장 가능, 선언 병합 지원 |
| 백엔드 응답 타입 (`model/dto/`) | `interface` | 확장 가능, 상속 구조 표현 가능 |
| 유니온·인터섹션·유틸리티 타입 조합 | `type` | `type A = B \| C` 는 `interface` 로 불가 |
| zod 스키마에서 추론된 타입 | `type` | `type T = z.infer<typeof schema>` |
| 함수 시그니처 별칭 | `type` | `type Handler = (e: Event) => void` |

```typescript
// ✅ props — interface
interface DocumentCardProps {
  document: DocumentDto.Summary
  onEdit: (id: string) => void
}

// ✅ 유니온 — type
type StatusBadgeVariant = 'success' | 'warning' | 'error'

// ✅ zod 추론 — type
import { z } from 'zod'
import { documentSchema } from '@/model/schema/document-schema'
type DocumentFormValues = z.infer<typeof documentSchema>
```

### `var` 금지 / `const` 우선

```typescript
// ❌ var
var count = 0

// ✅ const (재할당 없으면 항상 const)
const QUERY_KEYS = { DOCUMENTS: 'documents' } as const

// ✅ let (루프·재할당 필요한 경우만)
let loadingCount = 0
```

---

## 컴포넌트

### 함수형 컴포넌트 only

```typescript
// ✅ 함수형 컴포넌트
export function DocumentCard({ document, onEdit }: DocumentCardProps) {
  return <div>...</div>
}

// ✅ 화살표 함수 (기본 export 시)
const DocumentCard = ({ document, onEdit }: DocumentCardProps) => {
  return <div>...</div>
}
export default DocumentCard
```

클래스형 컴포넌트는 사용하지 않는다.

### props 타입 선언

```typescript
// ✅ interface XxxProps 명시
interface CollectionSelectProps {
  value?: number
  onChange: (id: number | undefined) => void
  disabled?: boolean
}

export function CollectionSelect({ value, onChange, disabled }: CollectionSelectProps) {
  ...
}
```

- props 인라인 타입(`function Foo({ x }: { x: string })`)은 props 가 1~2개일 때만 허용. 3개 이상이면 `interface` 로 분리한다.

### forwardRef

`ref` 를 전달해야 하는 UI 컴포넌트(input, button 등)는 `forwardRef` 를 사용한다.

```typescript
import { forwardRef } from 'react'

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, ...props }, ref) => (
    <div>
      {label && <label>{label}</label>}
      <input ref={ref} {...props} />
    </div>
  )
)
Input.displayName = 'Input'
```

---

## 이벤트 핸들러 명명

| 위치 | 네이밍 | 설명 |
|------|--------|------|
| props (외부에서 받는 콜백) | `on` + 동작 (`onSubmit`, `onClick`, `onChange`) | "이 이벤트가 발생했을 때 호출" |
| 컴포넌트 내부 함수 | `handle` + 동작 (`handleSubmit`, `handleClick`, `handleChange`) | "이 이벤트를 처리하는 함수" |

```typescript
interface FormProps {
  onSubmit: (data: FormValues) => void   // props: on + 동작
}

function MyForm({ onSubmit }: FormProps) {
  const handleSubmit = (data: FormValues) => {  // 내부: handle + 동작
    // 전처리 후 props 콜백 호출
    onSubmit(data)
  }

  return <form onSubmit={form.handleSubmit(handleSubmit)}>...</form>
}
```

---

## 파일명 컨벤션

### 기본 규칙: kebab-case

```
components/app-layout.tsx       ✅
components/auth-guard.tsx       ✅
components/axios-interceptor.tsx ✅
hooks/useDocumentsSearchParams.ts  ✅ (훅은 camelCase 시작)
api/document-api.ts             ✅
model/dto/document-dto.ts       ✅
```

### 예외: `page/` 디렉토리

기존 코드베이스의 `page/` 는 PascalCase 파일명 관행이 있다 (`MainPage.tsx`). 이를 인정하되, **신규 페이지 파일은 kebab-case 를 권고**한다.

```
page/MainPage.tsx           기존 — PascalCase (유지)
page/main-actions.tsx       기존 — kebab-case
page/document/              신규 디렉토리 — kebab-case ✅
page/document/document-list-page.tsx   신규 — kebab-case 권고
```

### 디렉토리명: kebab-case 통일

```
hooks/react-query/           ✅
app/document-history/        ✅
app/(user)/                  ✅ (라우트 그룹)
page/document-history/       ✅
```

---

## className 합성

`lib/shadcn-utils.ts` 의 `cn()` 을 사용한다. `clsx()` 나 `twMerge()` 를 컴포넌트에서 직접 호출하지 않는다.

```typescript
import { cn } from "@/lib/shadcn-utils"

// ✅
<div className={cn("base-class", isActive && "active", className)} />

// ❌ clsx 직접 호출
import { clsx } from 'clsx'
<div className={clsx("base-class", isActive && "active")} />

// ❌ 문자열 템플릿 직접 조합
<div className={`base-class ${isActive ? 'active' : ''}`} />
```

---

## import 순서

```typescript
// 1. 외부 라이브러리
import { useQuery } from '@tanstack/react-query'
import { useForm } from 'react-hook-form'

// 2. 절대 경로 (@/ 별칭)
import { DocumentApi } from "@/api/document-api"
import { documentKeys } from "@/hooks/react-query/useDocumentQueries"
import { DocumentDto } from "@/model/dto/document-dto"
import { cn } from "@/lib/shadcn-utils"

// 3. 상대 경로
import { DocumentCard } from "./document-card"
import type { LocalConfig } from "./types"
```

- 그룹 사이에는 빈 줄을 한 줄 둔다.
- 와일드카드 import (`import * as ...`) 는 `DocumentApi` / `DocumentDto` 같이 네임스페이스 re-export 패턴에서만 허용한다. 무분별한 `*` import 는 금지.

---

## 들여쓰기 / 포매팅

- **2 spaces 들여쓰기** (`eslint-config-next` 기본).
- JSX 와 TypeScript 모두 2 spaces.
- 함수 시그니처는 120자 미만이면 한 줄 유지. 초과 시 props multi-line.

```typescript
// ✅ 한 줄 (120자 미만)
export function DocumentList({ params, onSelect }: DocumentListProps) {

// ✅ 120자 초과 시 multi-line props
export function DocumentList({
  params,
  onSelect,
  className,
}: DocumentListProps) {
```

- JSX 속성이 3개 이상이거나 줄이 길어지면 multi-line 으로 전환.

```tsx
// ✅ multi-line JSX
<Button
  variant="destructive"
  size="sm"
  disabled={isLoading}
  onClick={handleDelete}
>
  삭제
</Button>
```

---

## JSDoc

- `components/shared/`, `hooks/react-query/` 의 공용 API 에만 JSDoc 을 작성한다.
- 페이지 컴포넌트, 로컬 유틸 함수에는 JSDoc 이 없어도 된다.
- 복잡한 로직(토큰 갱신 플로우, URL 직렬화 등)에는 인라인 한글 주석으로 WHY 를 설명한다.

---

## 안티패턴 금지 목록

| 금지 | 대안 |
|------|------|
| `any` 타입 | `unknown` + 타입 가드, 구체 타입 명시 |
| `localStorage` 에 토큰 저장 | `authStore` (메모리) |
| `appAxios` 직접 호출 (api/ 외부) | `api/*-api.ts` 경유 |
| `alert()` / `confirm()` | `toast` / 확인 Dialog |
| `clsx()` / `twMerge()` 직접 호출 | `cn()` from `lib/shadcn-utils` |
| 서버 상태를 `useState` 로 관리 | TanStack Query |
| `useEffect` + `fetch` 직접 페칭 | `useQuery` |
| 쿼리 키 문자열 하드코딩 | `xxxKeys` 팩토리 |
| 폼 필드를 `useState` 로 하나씩 | `react-hook-form` + `zod` |
| `dangerouslySetInnerHTML` (외부 입력) | sanitizer 적용 후 사용 |
| 클래스형 컴포넌트 | 함수형 컴포넌트 |
| 와일드카드 import 남용 | 명시적 named import |
