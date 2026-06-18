# 프로젝트 구조 (Iron Rule)

- **기존 디렉토리 경계를 존중한다.** 본 저장소는 `frontend/{api,app,common,components,hooks,lib,model,page,store,types}` 구조로 안정화되어 있다. 디렉토리 간 경계를 넘어 코드를 뒤섞지 않는다.
- 새 파일은 **가장 가까운 기능 도메인 근처**에 배치한다. 도메인별 최상위 디렉토리는 `api/`, `model/dto/`, `model/client/`, `model/schema/`, `hooks/react-query/`, `page/` 안에 이미 도메인 단위로 존재한다. 기존 도메인 파일이 있으면 새 폴더를 만들지 않는다.
- **강력한 근거 없이 최상위 디렉토리를 새로 만들지 않는다.** 현재 최상위 목록(`api / app / common / components / hooks / lib / model / page / store / types`) 은 완만하게만 확장한다. 추가가 꼭 필요하면 PR 에 사유를 남긴다.
- **아키텍처 맵과 일관된 의존성 방향을 유지한다.**
  - `page → hooks/react-query → api → model`
  - `components/` 는 `hooks/react-query`, `store`, `model` 을 참조 가능.
  - `api/*` 는 `model/dto` 와 `api/common` 만 참조한다. 컴포넌트·훅·스토어를 역참조하지 않는다.
  - `model/` 은 다른 어떤 레이어도 참조하지 않는다 (순수 타입 레이어).
- **구조가 불명확한 경우** 다음 순서로 1차 참조:
  1. `.claude/conventions/architecture.md` — 전체 디렉토리·레이어 지도
  2. `.claude/conventions/nextjs.md` — App Router·명명 규칙
  3. `.claude/conventions/library-stack.md` — 스택·설정 기준
  4. 기존 도메인의 실제 코드

---

## 디렉토리 구조 상세

```
frontend/
├── api/                            # axios 호출 함수 (도메인별 1 파일)
│   ├── common/
│   │   ├── app-api.ts              # appApi 래퍼 (GET/POST/PUT/PATCH/DELETE)
│   │   └── axios.ts                # appAxios 인스턴스 생성 + paramsSerializer
│   ├── auth-api.ts
│   ├── <domain-a>-api.ts
│   ├── <domain-b>-api.ts
│   └── ...                         # 도메인별 1 파일
│
├── app/                            # Next.js App Router (라우트 선언만)
│   ├── (<group-a>)/                # 라우트 그룹 (도메인별)
│   ├── (<group-b>)/
│   ├── login/
│   ├── layout.tsx                  # RootLayout — AppLayout 마운트
│   ├── page.tsx                    # 대시보드 진입점
│   └── not-found.tsx
│
├── common/                         # 프로젝트 전역 유틸·상수 (UI 없음)
│   ├── constants.ts                # AUTH_TYPE, PUBLIC_ROUTES, DEFAULT_PAGE_SIZE 등
│   ├── app-type.ts / app-type-name.ts / app-type-options.ts
│   ├── auth-utils.ts
│   └── utils/
│       ├── code-utils.ts           # logger 등
│       ├── date-utils.ts
│       └── search-utils.ts         # createSerializer / createDeserializer
│
├── components/                     # 재사용 UI 컴포넌트
│   ├── app-layout.tsx              # 전체 레이아웃 (Providers + AuthGuard + AxiosInterceptor)
│   ├── app-sidebar.tsx             # 사이드바 메뉴 정의
│   ├── auth-guard.tsx              # 인증 가드 (보호 라우트의 단일 진입점)
│   ├── axios-interceptor.tsx       # axios 인터셉터 등록 (Client Component)
│   ├── providers.tsx               # QueryClientProvider + ReactQueryDevtools
│   ├── shared/                     # 범용 공유 컴포넌트 (BaseTable 등)
│   ├── sheet/                      # Sheet(슬라이드 패널) 컴포넌트
│   ├── ui/                         # shadcn/ui 원본 컴포넌트
│   └── charts/                     # recharts 래퍼
│
├── hooks/                          # React 커스텀 훅
│   ├── react-query/                # TanStack Query 훅 (도메인별 1 파일)
│   │   ├── useAuthQueries.ts
│   │   ├── use<DomainA>Queries.ts
│   │   ├── use<DomainB>Queries.ts
│   │   └── ...
│   ├── use<DomainA>SearchParams.ts
│   ├── use<DomainB>SearchParams.ts
│   ├── useUrlSearchParams.ts       # URL 상태 훅 공통 기반
│   ├── use-mobile.ts
│   └── use-theme.ts
│
├── lib/
│   └── shadcn-utils.ts             # cn() — clsx + tailwind-merge
│
├── model/                          # 순수 타입 레이어 (비즈니스 로직 없음)
│   ├── dto/                        # 백엔드 응답 타입 (API 모양 그대로)
│   │   ├── auth-dto.ts
│   │   ├── <domain-a>-dto.ts
│   │   ├── <domain-b>-dto.ts
│   │   └── ...
│   ├── client/                     # UI 가 다루는 타입 (dto → client 변환 후)
│   │   ├── common-types.ts         # PaginationInfo 등 공용
│   │   ├── <domain-a>-types.ts
│   │   └── ...
│   ├── schema/                     # zod 스키마 (폼 입력 검증)
│   │   ├── auth-schema.ts
│   │   ├── <domain-a>-schema.ts
│   │   └── ...
│   ├── page-list-dto.ts            # PageListDto.Request / Response<T>
│   └── response-model.ts           # ResponseModel<T> (code / message / data)
│
├── page/                           # 페이지 단위 컴포넌트 (라우트와 1:1)
│   ├── MainPage.tsx
│   ├── login/
│   ├── <domain-a>/
│   ├── <domain-b>/
│   └── ...
│
├── store/
│   └── authStore.ts                # useAuthStore (zustand — 메모리 전용 토큰 관리)
│
└── types/
    └── api.d.ts                    # 전역 타입 선언
```

---

## 새 도메인 추가 체크리스트

새 도메인(`<domain>`)을 추가할 때 아래 순서를 따른다.

```
1. model/dto/<domain>-dto.ts          ← 백엔드 응답 타입 (Summary / Detail / SearchParams / CreateForm 등)
2. model/schema/<domain>-schema.ts    ← zod 스키마 (폼 입력 검증; 생성·수정 폼이 있는 경우)
3. model/client/<domain>-types.ts     ← UI 가 다루는 클라이언트 타입 (필터, 집계 등)
4. api/<domain>-api.ts                ← appApi 래퍼로 axios 호출 함수 선언
5. hooks/react-query/use<Domain>Queries.ts
                                      ← <domain>Keys 정의 + useQuery/useMutation 훅
6. hooks/use<Domain>SearchParams.ts   ← useUrlSearchParams 기반 URL 상태 훅 (목록 페이지)
7. page/<domain>/                     ← 페이지 컴포넌트 (kebab-case 파일명 권고)
8. app/<route>/page.tsx               ← Next.js 라우트 등록 (page/ 컴포넌트를 import해 렌더)
9. components/app-sidebar.tsx         ← items 배열에 사이드바 항목 추가
10. (필요 시) common/<domain>-utils.ts ← 도메인 전용 유틸 (변환·포매팅 등)
```

### 체크리스트 예시 (`<domain>` 도메인 구현 기준)

| 단계 | 파일 | 상태 |
|------|------|------|
| 1 | `model/dto/<domain>-dto.ts` (`Summary`, `Detail`, `SearchParams`, `EditForm` 등) | 완료 |
| 2 | `model/schema/<domain>-schema.ts` | 완료 |
| 3 | `model/client/<domain>-types.ts` (필터 타입 등) | 완료 |
| 4 | `api/<domain>-api.ts` (`search`, `edit`, `getById` 등) | 완료 |
| 5 | `hooks/react-query/use<Domain>Queries.ts` (`<domain>Keys`, `use<Domain>List`, `useEdit<Domain>`) | 완료 |
| 6 | `hooks/use<Domain>SearchParams.ts` | 완료 |
| 7 | `page/<domain>/` | 완료 |
| 8 | `app/(<group>)/<domain>/page.tsx` | 완료 |
| 9 | 사이드바 컴포넌트 items 에 메뉴 항목 추가 | 완료 |

---

## 도메인 목록

프로젝트의 실제 도메인 목록은 `api/` 디렉토리와 사이드바 컴포넌트(`items` 배열)를 확인한다.
새 도메인 추가 시 위 체크리스트를 따른다.

---

## 금지사항

- `app/` 아래 라우트 파일(`page.tsx`, `layout.tsx`)에 비즈니스 로직 직접 작성 금지. `page/` 컴포넌트를 import 해 렌더한다.
- `api/` 외부에서 `appAxios` 직접 호출 금지. 반드시 `api/*-api.ts` 를 경유한다.
- `model/` 파일에 함수·사이드이펙트 금지. 순수 타입/인터페이스/zod 스키마만 선언한다.
- 같은 레이어 간 가로지르기(`use<DomainA>Queries` 가 `use<DomainB>Queries` 를 직접 호출) 는 허용되지만, **역방향 참조**(`api/` 가 `hooks/` 를 참조) 는 금지한다.
