# React / Next.js 코드 컨벤션

## 프로젝트 구조

**FSD (Feature-Sliced Design) 아키텍처를 사용합니다.**
상세 구조는 `frontend-architecture.md`, 라이브러리 가이드는 `frontend-library-stack.md`를 참고하세요.

```
src/
├── app/          # Next.js App Router (라우팅 전용)
├── widgets/      # 독립적인 UI 블록 (Header, Sidebar)
├── features/     # 사용자 인터랙션 (로그인, 검색)
├── entities/     # 비즈니스 엔티티 (User, Product)
└── shared/       # 공통 모듈 (UI킷, 유틸, API 클라이언트)
```

## 네이밍 규칙

### 파일명
- 컴포넌트: `PascalCase.tsx` (예: `UserProfile.tsx`)
- 훅: `camelCase.ts` (예: `useAuth.ts`)
- 유틸리티: `camelCase.ts` (예: `formatDate.ts`)
- 타입: `camelCase.ts` (예: `user.ts`)
- 상수: `camelCase.ts` 또는 `SCREAMING_SNAKE_CASE.ts`

### 변수/함수
- 컴포넌트: `PascalCase` (예: `UserProfile`)
- 함수/변수: `camelCase` (예: `getUserData`)
- 상수: `SCREAMING_SNAKE_CASE` (예: `MAX_RETRY_COUNT`)
- 타입/인터페이스: `PascalCase` (예: `UserResponse`)
- enum: `PascalCase` (예: `UserRole`)

### 디렉토리
- 소문자 kebab-case (예: `user-profile/`, `auth-guard/`)

## 컴포넌트 작성 규칙

### 기본 패턴
```tsx
interface Props {
  title: string;
  onSubmit: (data: FormData) => void;
}

export default function ComponentName({ title, onSubmit }: Props) {
  // 1. hooks
  // 2. 상태/변수
  // 3. 핸들러
  // 4. effects
  // 5. render

  return <div>{title}</div>;
}
```

### 규칙
- 함수 컴포넌트 + `export default function` 사용
- Props 인터페이스는 컴포넌트 바로 위에 정의
- `React.FC` 사용하지 않음
- 구조 분해 할당으로 props 받기
- 조건부 렌더링 시 early return 패턴 사용

### 반복되는 UI 는 공용 컴포넌트로
- 같은 모양(태그 + 클래스)이 **두 군데 이상** 나오면 공용 컴포넌트로 빼고, 다른 부분만 props 로 받는다.
  - 화면 제목처럼 도메인과 무관하면 `shared/ui/`, 판매 구분 라벨처럼 도메인 값을 보여주면 `entities/<도메인>/ui/`.
- 새 화면을 만들기 전에 `shared/ui/index.ts`(관리자 부품을 나눈 프로젝트는 `shared/ui/admin/index.ts` 도)와 해당 `entities/*/ui` 를 먼저 확인하고, 있으면 그걸 쓴다.
- 한 번만 쓰는 모양은 미리 빼지 않는다 — 두 번째로 쓸 때 뺀다.

```tsx
// 권장
<PageTitle title="재고·중고품" />
<PageTitle title="분류 관리" variant="admin" />

// 지양 (같은 태그·클래스를 화면마다 반복)
<h1 className="text-2xl font-bold">재고·중고품</h1>
<h1 className="text-2xl font-bold">신제품</h1>
```

### 디자인 체계가 둘이면 부품을 나눈다 (관리자 화면이 있는 프로젝트)
- 프로젝트에 디자인 체계가 하나뿐이면(사용자 화면만, 또는 관리 도구만) 부품도 `shared/ui/` 한 벌로 간다. 아래는 적용하지 않는다.
- 관리자 화면과 사용자 화면이 함께 있고 시안의 크기·색 체계가 다르면, 그 체계를 따르는 부품은 나눈다.
  - 왜: 관리자 화면은 한 번에 많은 정보를 보여 주려고 촘촘하게(작은 글자·낮은 칸·좁은 여백·각진 모서리),
    사용자 화면은 깔끔하게(넉넉한 여백·큰 버튼·둥근 모서리) 만든다. 한 부품에 두 체계를 섞으면 한쪽 시안에 맞출 때 다른 쪽이 어긋난다.
  - 대상: 버튼·입력칸·표처럼 크기·색 체계를 따르는 부품. 관리자용은 `shared/ui/admin/` (예: `AdminButton`, `AdminInput`, `AdminTable`),
    사용자용은 `shared/ui/` 에 둔다.
  - 관리자 화면에서는 `admin/` 부품을, 사용자 화면에서는 사용자 부품을 쓴다. 크기·색 값은 그 화면의 시안을 따른다.
  - 관리자 화면이 있어도 두 화면의 시안이 같은 체계를 쓰면 나누지 않는다.
- 나누지 않고 하나로 두는 경우
  - 작은 차이(제목 크기, 라벨 모양 등)만 다르면 `variant` prop 하나로 고른다
    (`<PageTitle variant="admin" />`, `<SalesTypeBadge variant="admin" />`). 클래스 문자열을 props 로 넘기지 않는다.
  - 화면과 상관없는 부품(알림 창 틀 `Dialog`, 상태 문구 `StatusView` 등)은 하나로 같이 쓴다.
- 판단: 크기·여백·색·모서리 중 둘 이상이 다르면 나누고, 하나만 다르면 `variant`. 애매하면 하나로 두고, 시안에 맞추다 다른 화면이 어긋나기 시작하면 그때 나눈다.

```tsx
// 디자인 체계가 둘인 프로젝트 (예: 쇼핑몰 + 관리자)
<AdminButton size="xs" variant="outline">수정</AdminButton>   // 관리자 화면
<Button size="lg">장바구니 담기</Button>                        // 사용자 화면

// 지양 (한 버튼에 관리자·사용자 크기·색을 섞어 둔다)
<Button size="xs" variant="outline">수정</Button>
<Button size="lg">장바구니 담기</Button>
```

### Server vs Client 컴포넌트
```tsx
// 기본은 Server Component (별도 선언 불필요)
export default function ServerComponent() { ... }

// Client Component는 명시적으로 선언
'use client';
export default function ClientComponent() { ... }
```

- 기본적으로 Server Component 사용
- `useState`, `useEffect`, 이벤트 핸들러 사용 시에만 `'use client'`
- Client Component는 가능한 트리 하단에 배치

## 상태 관리

### 우선순위
1. **Server Component** — 가능하면 서버에서 데이터 처리
2. **URL 상태** — `useSearchParams`, `usePathname` (필터, 페이지네이션 등)
3. **React 기본** — `useState`, `useReducer` (로컬 상태)
4. **Context** — 테마, 인증 등 전역이 필요한 경우
5. **외부 라이브러리** — zustand, jotai 등 (복잡한 클라이언트 상태)

### 서버 상태
- `fetch` + React Server Component 우선
- 클라이언트에서 필요시 `@tanstack/react-query` 사용

## API 호출

### Server Component
```tsx
// app/users/page.tsx
export default async function UsersPage() {
  const users = await fetch(`${API_URL}/users`, {
    next: { revalidate: 60 },
  }).then(res => res.json());

  return <UserList users={users} />;
}
```

### Route Handler
```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const data = await fetchUsers();
  return NextResponse.json(data);
}
```

## 스타일링

### 우선순위
1. **Tailwind CSS** — 기본 스타일링 도구
2. **CSS Modules** — Tailwind로 해결 어려운 경우
3. **styled-components** — 레거시 프로젝트 유지보수 시

### Tailwind 규칙
- 클래스 순서: layout → sizing → spacing → typography → visual → interactive
- 긴 클래스는 `cn()` 또는 `clsx()` 유틸리티 사용
- 반복되는 스타일은 `@apply`보다 컴포넌트 추출 선호

## 에러 처리

### Error Boundary
```tsx
// app/error.tsx
'use client';

export default function Error({ error, reset }: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>문제가 발생했습니다</h2>
      <button onClick={reset}>다시 시도</button>
    </div>
  );
}
```

### Loading
```tsx
// app/loading.tsx
export default function Loading() {
  return <Skeleton />;
}
```

## Import 순서

```tsx
// 1. React/Next
import { useState } from 'react';
import Link from 'next/link';

// 2. 외부 라이브러리
import { clsx } from 'clsx';

// 3. 내부 모듈 (절대 경로)
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
import { formatDate } from '@/lib/formatDate';
import type { User } from '@/types/user';

// 4. 상대 경로
import { SubComponent } from './SubComponent';

// 5. 스타일
import styles from './Component.module.css';
```

## 금지 패키지/버전

- **axios 1.14.1** — 해킹(supply chain attack)이 확인된 버전. 절대 설치 금지
- **axios 0.30.4** — 해킹(supply chain attack)이 확인된 버전. 절대 설치 금지
- axios 사용 시 반드시 위 버전을 피하고, 공식 릴리즈 노트에서 안전한 버전을 확인할 것

## 테스트 규칙

### FIRST 원칙

| 원칙 | 핵심 |
|------|------|
| **F**ast | 외부 의존성 최소화, Mock/인메모리 활용 |
| **I**solated | 테스트 간 상태 공유 금지 |
| **R**epeatable | 환경 무관하게 동일 결과 |
| **S**elf-validating | assert로 명확한 pass/fail |
| **T**imely | 기능 구현과 동시에 작성 |

### FE 테스트 도구

| 도구 | 용도 |
|------|------|
| Vitest | zod 스키마·zustand 스토어·훅 단위 테스트 (`frontend-library-stack.md` 설정 참고) |
| Testing Library | 컴포넌트 렌더링·상호작용 테스트 |
| Playwright | 실제 브라우저로 도는 E2E — 백엔드와 실제로 붙어야 의미 있는 플로우(로그인, 결제, 핵심 폼)에 사용 |

- 유닛/컴포넌트 테스트 파일은 테스트 대상 옆에 `*.test.ts(x)`로 배치(`frontend-library-stack.md` 참고). E2E는 `e2e/*.spec.ts`.
- **테스트 없는 기능 구현 금지.** 최소 성공 케이스 1개 + 실패 케이스 1개는 유닛 테스트로 남긴다.
- API 클라이언트·에러 파싱처럼 "실제 요청/응답이 오가야 드러나는 버그"(응답 바디 재사용, 헤더 누락, 직렬화 불일치 등)는 유닛 테스트(mock)로는 못 잡는다 — 핵심 플로우는 실제 서버를 띄운 E2E로 최소 1회 검증한다.

## TypeScript 규칙

- `any` 사용 금지 — `unknown` 사용 후 타입 좁히기
- API 응답 타입은 반드시 정의
- `as` 타입 단언 최소화 — 타입 가드 사용
- `interface` 우선 사용 (`type`은 유니온/인터섹션에)
- 제네릭 남용 금지 — 필요한 경우에만

## 커밋 메시지

```
feat: 사용자 프로필 페이지 추가
fix: 로그인 토큰 만료 처리 수정
refactor: API 클라이언트 모듈화
style: 헤더 반응형 레이아웃 적용
chore: eslint 설정 업데이트
docs: API 엔드포인트 문서 추가
test: 회원가입 폼 유효성 검사 테스트 추가
```