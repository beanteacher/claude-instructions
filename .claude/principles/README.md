# 공통 Iron Rule 베이스 (.claude/principles/)

이 디렉터리는 `workspace_web/` 전체에 적용되는 **스택 무관 공통 Iron Rule 베이스**다.
각 Iron Rule 의 개념(원칙 진술)만 스택 중립 문장으로 추출해 한 곳에 모았다.

원칙은 **2단 구조**로 운영된다:

1. **공통 베이스 (이 디렉터리)** — 스택과 무관한 원칙 골격.
2. **스택별 구체화 레이어** — 각 하위 워크스페이스의 `.claude/principles/<같은 파일명>`.
   동일 원칙을 해당 스택의 경로·API·명령 예시로 구체화한다.
   (`java_backend_workspace`, `python_rag_workspace`, `typescript_react_frontend_workspace`)

각 공통 파일은 끝에 동일 파일명의 스택별 문서를 참조하라는 안내를 포함한다.
새 원칙을 발명하지 않으며, 공통 베이스에는 세 스택 공통 항목만 담는다.

---

## 파일 목록 및 적용 범위

| 파일 | 적용 범위 |
|------|-----------|
| `agent-docs.md` | 보편 |
| `api-conventions.md` | 보편 |
| `code-style.md` | 보편 |
| `completeness-principle.md` | 보편 |
| `error-handling.md` | 보편 |
| `project-structure.md` | 보편 |
| `search-before-build.md` | 보편 |
| `security.md` | 보편 |
| `database.md` | 백엔드(영속 계층 보유) 프로젝트 |
| `state-management.md` | 프론트엔드 프로젝트 |

요약: 보편 8 / 백엔드 전용 `database` / 프론트 전용 `state-management`.
