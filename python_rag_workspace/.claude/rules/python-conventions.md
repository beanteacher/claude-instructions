# Python / RAG 코드 컨벤션 (스타터)

> 첫 실제 프로젝트를 붙이면 이 문서를 그 스택에 맞춰 확정한다.
> 백엔드(`java_backend_workspace`)·프론트(`typescript_react_frontend_workspace`)와 동일한
> "규칙 전용 문서 + 예시/패턴 분리" 구성을 따른다.

---

## 프로젝트 구조 (RAG 레이어 분리)

```
src/{package}/
├── ingestion/      # 문서 적재·청킹·임베딩 (배치/파이프라인)
├── retrieval/      # 벡터/키워드 검색, 리랭킹
├── generation/     # 프롬프트 구성·LLM 호출·후처리
├── api/            # FastAPI 라우터 (요청/응답만)
├── core/           # 설정·로깅·예외·공통 의존성
└── schemas/        # Pydantic 모델 (요청/응답/도메인)
tests/              # 소스와 동일 트리 구조로 미러링
```

- **ingestion / retrieval / generation 은 서로 직접 의존하지 않는다.** 조합은 `api` 또는 서비스 레이어에서.
- 외부 LLM·벡터DB 연동은 인터페이스(`Protocol`)로 추상화해 테스트에서 교체 가능하게 한다.

---

## 명명 규칙 (PEP 8)

| 대상 | 규칙 | 예시 |
|------|------|------|
| 모듈/패키지 | snake_case | `vector_store.py` |
| 클래스 | PascalCase | `DocumentChunker` |
| 함수/변수 | snake_case | `build_prompt`, `top_k` |
| 상수 | UPPER_SNAKE_CASE | `MAX_TOKENS` |
| 비공개 | 선행 `_` | `_normalize` |

---

## 타이핑 / 데이터

- **모든 공개 함수에 타입 힌트 필수.** `Any` 금지 — 불가피하면 `object` 또는 `Protocol`.
- 요청/응답/설정은 **Pydantic v2 모델**로 정의. 원시 dict 전달 금지.
- 환경값은 `pydantic-settings` 로 묶어 주입 — 코드에 흩뿌리지 않는다.

---

## 도구 체인 (권장)

| 용도 | 도구 |
|------|------|
| 패키지/가상환경 | `uv` |
| 린트 + 포맷 | `ruff` (`ruff check` / `ruff format`) |
| 타입 체크 | `pyright` 또는 `mypy --strict` |
| 테스트 | `pytest` + `pytest-asyncio` |
| 웹 | `fastapi` + `uvicorn` |

---

## 예외 / 로깅

- 비즈니스 오류는 도메인 예외 클래스 → FastAPI `exception_handler` 에서 일괄 변환. 라우터에서 try/except 산재 금지.
- 표준 `logging`(JSON 포매터) 사용. `print` 디버깅 금지.
- LLM/검색 호출은 **입력 토큰 수·지연·결과 건수**를 구조화 로그로 남긴다.

---

## 보안 / RAG 주의

- API 키·접속정보는 절대 커밋 금지 — `.env` 는 전역 `~/.claude` deny 로 읽기 차단됨.
- 사용자 입력을 프롬프트에 넣을 때 **프롬프트 인젝션** 방어(구분자·역할 고정)를 고려한다.
- 검색 결과 출처(문서 id·score)를 응답에 함께 실어 추적 가능하게 한다.

---

## 테스트 규칙

- ingestion/retrieval/generation 각 레이어는 외부 의존(LLM·벡터DB)을 **mock/fake** 로 대체해 단위 테스트.
- 통합 테스트는 임베딩·검색을 인메모리 벡터스토어로 실행.
- 수정된 결함에는 회귀 테스트 필수.
