# java_backend_workspace

Java / Spring Boot 백엔드 프로젝트 전용 워크스페이스.
이 폴더 하위의 모든 백엔드 프로젝트는 아래 **공통 규칙**을 상속한다.
프로젝트 고유 규칙·설정은 각 프로젝트 폴더의 `.claude/` 에 따로 둔다.

## 공통 규칙 (`.claude/rules/`)

- `backend-conventions.md` — 레이어 책임·명명·DTO record·API 설계·Spring·예외·테스트 규칙
- `backend-java-patterns.md` — 레이어별 코드 예시 (product 도메인 기준)
- `backend-design-patterns.md` — 레이어별 디자인 패턴 가이드·판단 기준
- `backend-options/` — 프로젝트 초기화/기능/테스트 선택 모듈
  - `init-single` / `init-multi-module` / `init-msa` — 구조 선택
  - `init-redis` / `init-kafka` / `init-security` — 인프라 옵션
  - `feature-*` / `test-*` — 기능·테스트 템플릿

> 규칙 세트는 `ApiResponse<T>` envelope + `/api/v1/{resource}` 경로 컨벤션 기반(제네릭 템플릿)이다.

## 강제 레이어 (이 워크스페이스 밖)

push/시크릿 차단 같은 **강제 훅은 전역 `~/.claude/`** 에서 동작한다.
이 워크스페이스는 규칙 문서(`.md`)만 담는다 — 훅은 두지 않는다
(상위 워크스페이스 폴더의 `settings.json` 은 하위 프로젝트로 상속되지 않기 때문).
