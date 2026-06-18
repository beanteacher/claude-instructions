# 에이전트 우선 문서화 (Iron Rule)

- 에이전트를 위한 문서를 먼저 작성한다: **명시적 입력, 출력, 제약조건, 게이트.**
  - 입력: 어떤 파일·도메인·컴포넌트·훅이 영향을 받는가?
  - 출력: 어떤 산출물이 만들어져야 하는가? (코드, 타입, 테스트, `next.config.ts` 변경 등)
  - 제약: App Router 전용, `'use client'` 경계, 메모리 전용 토큰(authStore), 공개 경로(`PUBLIC_ROUTES`) 정의, axios 인스턴스 사용 의무, TanStack Query `invalidateQueries` 기반 캐시 무효화 등 본 저장소의 불변조건.
  - 게이트: "완료" 판단 기준 — `npm run build`, `npm run dev` 실행 후 정상 렌더, 로딩·에러·빈 상태 UI 점검, 토스트 메시지 표시 확인 등.
- **서술형 산문보다 체크리스트, 스키마, 예시를 선호한다.**
  - "~를 고려하여 잘 처리한다" 같은 모호한 문장 대신:
    ```
    [ ] model/dto/ 에 DTO 타입 선언
    [ ] model/schema/ 에 zod 스키마 작성
    [ ] api/<domain>-api.ts 에 axios 래퍼 호출 함수 추가
    [ ] hooks/react-query/use<Domain>Queries.ts 에 TanStack Query 훅 작성
    [ ] 사이드바 컴포넌트의 items 배열에 메뉴 항목 추가
    [ ] 인증 가드(auth-guard) PUBLIC_ROUTES 추가/제거 여부 확인
    ```
- **파일 경로와 명령어 예시를 구체적으로 유지한다.**
  - `components/shared/table/BaseTable.tsx:36` 처럼 경로·라인을 명시.
  - 명령은 복붙 가능한 형태로: `npm run build`, `npm run dev`.
- **근거는 짧은 글머리표로, 실행 단계는 결정적(deterministic) 순서**로 작성한다.
  - 근거: "왜 이 규칙이 필요한가" → 한두 줄 bullet.
  - 실행 단계: 1 → 2 → 3 숫자 순서. 사이클이 있으면 "X 가 실패하면 Y 로 돌아간다" 로 명시.
- 에이전트가 **사람에게 설명한다고 가정한다.** 그 반대가 아니다. 즉, 문서는 사람이 에이전트를 가르치는 목적이 아니라, 에이전트가 새 컨텍스트에서도 정확하게 재현할 수 있도록 **자기 완결적**으로 기술한다.

---

## 본 저장소의 에이전트 참조 맵 (우선순위 순)

1. `CLAUDE.md` — 프로젝트 요약, 빌드·실행 방법, 아키텍처 개요, 보안 정책.
2. `.claude/conventions/architecture.md` — 디렉토리·레이어 지도 (`api/`, `app/`, `page/`, `components/`, `hooks/`, `store/`, `model/`, `lib/`, `common/`).
3. `.claude/conventions/nextjs.md` — App Router 규약, `'use client'` 배치, Server vs Client 컴포넌트 결정.
4. `.claude/conventions/library-stack.md` — 스택·의존성·설정 (TanStack Query, Zustand, axios, zod, shadcn 등).
5. `.claude/rules/conventions.md` — 규칙 전용 (레이어별 책임, 명명 규칙, 폼·쿼리·스토어 패턴).
6. `.claude/rules/react-patterns.md` — 실제 코드 템플릿 (도메인별 패턴 예시).
7. `.claude/rules/design-patterns.md` / `mapping.md` / `shared-instances.md` — 패턴 상세.
8. `.claude/rules/testing.md` — Vitest 도입 권고 및 테스트 계층 순서.
9. `.claude/principles/*.md` — 엄격 규칙 (보안·에러·상태관리·완전성·코드스타일·프로젝트구조·search-before-build·agent-docs·api-conventions).

---

## PR / 작업 노트 템플릿 (권장)

```
## 변경 요약
- (한두 줄, 왜 바꾸는가)

## 영향 범위
- 코드: <변경된 파일 경로 (TS/TSX)>
- 타입: <model/dto, model/client, model/schema 변경 여부>
- 라우트: <app/**/page.tsx 추가/이동/삭제 여부>
- 번들 사이즈: <큰 라이브러리 추가 시 측정 — 없으면 "해당 없음">
- 접근성: <a11y — Radix 가 기본 보장하지만 추가 컴포넌트는 직접 점검>
- 디자인 토큰: <Tailwind config / globals.scss 변경 여부>
- 환경변수: <NEXT_PUBLIC_* 신규/변경 여부>
- 보안: <axios-interceptor / auth-guard 우회 여부>
- i18n: <다국어 키 — 도입 시점 결정 필요로 명시>

## 검증
- [ ] npm run build (빌드 성공)
- [ ] npm run dev 실행 후 해당 화면 정상 렌더
- [ ] 로딩 상태 UI 확인 (skeleton / spinner)
- [ ] 에러 상태 UI 확인 (토스트 또는 fallback)
- [ ] 빈 상태 UI 확인 (empty state)
- [ ] 모바일 뷰 점검 (hooks/use-mobile.ts 활용)
- [ ] 다크모드 점검 (next-themes + theme-toggle.tsx)

## 관련 요구사항
- <이슈/티켓 ID>
```
