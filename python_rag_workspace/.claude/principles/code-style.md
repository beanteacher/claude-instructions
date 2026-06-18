# 코드 스타일 (Iron Rule)

## 포매터 / 린터 설정 최우선

프로젝트의 포매터·린터 설정을 항상 최우선으로 따른다.

- **black**: 코드 포매터 (line-length 100)
- **ruff**: 린터 (pyflakes, pycodestyle, isort 규칙 포함)
- **isort**: import 정렬 (black 호환 프로파일 사용)

설정은 `pyproject.toml` 또는 `setup.cfg`에 명시하고, CI에서 강제한다.  
IDE 기본 포매팅 대신 위 도구를 사용한다.

---

## from __future__ import annotations 필수

모든 Python 소스 파일 최상단(docstring 직후, 일반 import 직전)에 반드시 선언한다.

```python
from __future__ import annotations
```

이 선언이 파서의 어노테이션 평가 방식을 바꾸므로 반드시 파일 최상단에 위치해야 한다.  
(`main.py` 참조: 주석에 "파서가 파일 전체 동작 방식을 바꾸기 때문에 반드시 맨 위에 둬야 함"이라고 명시되어 있음)

---

## 타입 힌트 필수

모든 함수·메서드의 파라미터와 반환값에 타입 힌트를 붙인다.

```python
# 올바른 예
async def ask(self, req: AskRequest) -> AskResponse:
    ...

def get_extension_or_raise(file: UploadFile) -> str:
    ...
```

- `Any`는 외부 의존성(LLM, DB 매니저 등)처럼 타입이 확정되지 않은 경우에 한해 허용한다.
- `Optional[X]` 대신 `X | None` (Python 3.10+) 또는 `from __future__ import annotations` 사용 후 `X | None` 표기를 권장한다.

---

## Pydantic BaseModel로 DTO / 스키마 표현

요청·응답 스키마는 반드시 `pydantic.BaseSettings` 또는 `pydantic.BaseModel`로 정의한다.  
설정값은 `pydantic_settings.BaseSettings`를 사용하고 `.env` 파일로 관리한다.

```python
class ItemRequest(BaseModel):
    name: str
    top_k: int | None = None
    session_id: str | None = None
    category: str
    page: int | None = None

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    database_url: str = Field(default="http://127.0.0.1:5432")
```

- `dict`, `TypedDict`로 DTO를 표현하지 않는다.
- 스키마 필드에 `description`을 명시하면 Swagger 문서 품질이 높아진다.

---

## 작은 함수 분리 (단계별 헬퍼 패턴)

한 함수가 여러 단계를 수행할 때는 단계별 private 헬퍼로 분리한다.  
`ingestion_service.py`의 `ingest` 메서드는 이 패턴의 좋은 예시다.

```
process()
  └─ _validate_input(data)          # 입력 검증 (모듈 수준 헬퍼로 분리)
  └─ _read_content(source)          # 데이터 읽기
  └─ _check_duplicate(hash_value)   # 중복 체크
  └─ _build_initial_state(...)      # 상태 초기화
```

- **함수 하나 = 한 가지 책임**. 여러 관심사를 합치지 않는다.
- 제네릭 3중 nesting 한 줄 표현보다, 이름이 설명해주는 헬퍼 2~3개가 낫다.
- 관련 책임은 작은 헬퍼 모듈로 추출한다.

---

## import 순서 (isort 기준)

```python
# 1. __future__
from __future__ import annotations

# 2. 표준 라이브러리
import hashlib
import logging
import uuid
from pathlib import Path

# 3. 서드파티
from fastapi import FastAPI, UploadFile
from pydantic import BaseModel, Field

# 4. 로컬 (app.*)
from app.config.settings import settings
from app.exception.errors import BadRequestError
```

- 와일드카드 import 절대 금지: `from module import *`
- 함수 내부에서 `import`를 선언하지 않는다 (불가피한 경우 주석으로 이유 명시).

---

## 주석은 WHY

WHAT은 잘 지어진 이름과 타입이 대신한다.  
주석은 **왜 이 결정을 했는지** 비자명한 제약·배경을 남긴다.

```python
# 파서가 파일 전체 동작 방식을 바꾸기 때문에 반드시 맨 위에 둬야 함
from __future__ import annotations

# 중복 체크는 해시만 사용 — 파싱 없이 즉시 → 성능 최적화
```

불필요한 `# TODO` / `# FIXME`를 남기려면 이슈 링크·복구 계획을 함께 적는다.

---

## docstring 스타일: Google Style

프로젝트 전체에서 **Google Style** 하나만 사용한다.

```python
def get_extension_or_raise(file: UploadFile) -> str:
    """파일 확장자 검증 유틸리티.

    지원하지 않는 확장자인 경우 UnsupportedFileTypeError를 발생시킨다.

    Args:
        file: FastAPI UploadFile 객체

    Returns:
        소문자 확장자 문자열 (예: ".pdf")

    Raises:
        BadRequestError: 파일명이 비어 있거나 파싱 실패 시
        UnsupportedFileTypeError: 지원하지 않는 확장자
    """
```

- 클래스·공개 메서드에는 docstring을 작성한다.
- 내부 헬퍼(`_`로 시작)는 한 줄 요약 docstring으로 충분하다.
- 모듈 수준 docstring은 파일 최상단에 작성한다 (`from __future__ import annotations` 직전).

---

## 브라운필드 파일: diff 표면 최소화

기존 파일의 포매팅을 대량으로 바꾸지 않는다.  
로직 변경과 포매팅 정리를 같은 커밋에 섞으면 리뷰 반려.  
포매팅만 정리하려면 별도 `style: ...` 커밋으로 분리한다.

---

## 안티패턴 금지 목록 (Python)

| 안티패턴 | 설명 | 대안 |
|---|---|---|
| **mutable default args** | `def f(items=[])` → 호출 간 공유됨 | `def f(items=None): items = items or []` |
| **broad except** | `except Exception: pass` → 에러 은폐 | 구체적 예외 타입 명시, 또는 커스텀 에러(`BadRequestError` 등) |
| **와일드카드 import** | `from module import *` → 네임스페이스 오염 | 명시적 이름 import |
| **글로벌 가변 상태** | 모듈 수준 `_cache = {}` 직접 변경 | 싱글톤 패턴 또는 의존성 주입 |
| **httpx.AsyncClient 매번 새로 생성** | 요청마다 `AsyncClient()` 인스턴스화 → 커넥션풀 낭비 | 앱 시작 시 생성, DI로 주입 |
| **RuntimeError 그대로 던지기** | `raise RuntimeError(msg)` | 커스텀 예외 계층(`BadRequestError` 등) 사용 |
| **Optional 필드 남용** | 모든 필드를 `Optional`로 선언 | 필수값은 필수 필드로, 선택값만 `X | None` |
| **var 대신 명시적 타입 생략** | 중간 변수에 타입 힌트 없이 복잡한 표현식 대입 | 중간 변수에 타입 어노테이션 추가 |
| **함수 내부 import** | 지연 import 남용 | 불가피한 경우(순환 참조 방지 등)에만 허용, 주석으로 이유 명시 |
| **중첩 try-except 남용** | try 블록 안에 또 try → 제어 흐름 불명확 | 단계를 함수로 분리하고 예외를 한 곳에서 처리 |

---

## 승인된 패턴 우선 재사용

아래 파일에 기재된 패턴·예시 외의 구조를 도입하려면 PR 본문에 근거를 남긴다.

- `.claude/principles/api-conventions.md`
- `.claude/principles/error-handling.md`
- `.claude/principles/project-structure.md`
- `.claude/principles/security.md`
