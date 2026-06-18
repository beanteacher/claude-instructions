# typescript_react_frontend_workspace

TypeScript / React / Next.js 프론트엔드 프로젝트 전용 워크스페이스.
이 폴더 하위의 모든 프론트 프로젝트는 아래 **공통 규칙**을 상속한다.
프로젝트 고유 규칙·설정은 각 프로젝트 폴더의 `.claude/` 에 따로 둔다.

## 공통 규칙 (`.claude/rules/`)

- `frontend-architecture.md` — Feature-Sliced Design(FSD) + Next.js App Router 구조·레이어 의존 규칙
- `frontend-conventions.md` — 네이밍·컴포넌트·상태관리·API 호출·스타일링·TS 규칙
- `frontend-library-stack.md` — 권장 스택(Tailwind/shadcn/react-query/zustand/RHF+zod/ky/date-fns 등)

> 금지 패키지 주의: `axios 1.14.1` / `axios 0.30.4` (공급망 공격 확인 버전) — 설치 금지.

## 강제 레이어 (이 워크스페이스 밖)

자동 포맷(prettier)·push/시크릿 차단 같은 훅은 다음 위치에서 동작한다:
- **전역 `~/.claude/`** — push/시크릿 차단 (모든 스택 공통)
- **각 프로젝트 `.claude/settings.json`** — 자동 포맷 등 경로 의존 훅

상위 워크스페이스 폴더의 `settings.json` 은 하위 프로젝트로 상속되지 않으므로,
이 워크스페이스는 규칙 문서(`.md`)만 담는다.
