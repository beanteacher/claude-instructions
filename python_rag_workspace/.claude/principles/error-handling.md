# 에러 처리 (Iron Rule)

## 핵심 원칙

- **경계에서는 빠르게 실패하고, 외부 호출에 대해서는 안전하게 실패한다.**
  - 서비스 진입부에서 선행 조건을 즉시 검증하고 적절한 `AppError` 서브클래스로 던진다.
  - 외부 시스템(LLM, 벡터스토어, 검색 엔진 등) 호출은 `tenacity` 재시도/백오프로 감싸고, 부분 실패는 격리한다.

---

## 1. 예외 계층 구조

모든 비즈니스 예외는 `app/exception/errors.py`의 `AppError`를 상속한다.

```
AppError (base, 500)
├── BadRequestError (400)
│   ├── ValidationError
│   └── UnsupportedTypeError
├── NotFoundError (404)
├── ExternalServiceError (502)
│   └── QueryEngineError (502)
├── ProcessingError (500)
└── TimeoutError (504)
```

**새 오류 케이스가 필요하면** `errors.py`에 서브클래스를 먼저 추가하고, `code`·`status_code`·`message`를 클래스 속성으로 한 곳에서 관리한다.  
도메인별로 `XxxNotFoundException`을 새로 만들지 않는다 — `NotFoundError(message="...", detail={...})`로 통일한다.

---

## 2. 예외 핸들러 위임 (install_exception_handlers)

`app/exception/api_error_handler.py`의 `install_exception_handlers`가 모든 예외를 일괄 변환한다.

| 예외 타입 | HTTP 응답 |
|---|---|
| `AppError` (및 서브클래스) | `{"error": {"code": ..., "message": ..., "detail": ...}}` |
| `StarletteHTTPException` | `{"error": {"code": "http_error", "message": ...}}` |
| `RequestValidationError` | `{"error": {"code": "validation_error", "message": ..., "detail": [...]}}` |
| `Exception` (미처리) | `{"error": {"code": "internal_error", "message": "서버 오류가 발생했습니다."}}` |

**컨트롤러/서비스에서 try/except로 응답을 직접 만들지 않는다.**  
Infrastructure 경계(예: 외부 SDK 호출 실패)에서만 예외를 래핑 후 재던진다.

---

## 3. HTTPException vs 도메인 Exception 분리 기준

| 상황 | 사용할 예외 |
|---|---|
| FastAPI 라우팅·미들웨어 레벨 오류 | `StarletteHTTPException` (직접 사용 최소화) |
| 비즈니스 로직 위반, 도메인 규칙 실패 | `AppError` 서브클래스 |
| 요청 파라미터 검증 | Pydantic 자동 처리 → `RequestValidationError` |
| 외부 서비스 장애 | `ExternalServiceError`, `QueryEngineError` 등 |

규칙: **비즈니스 코드에서 `HTTPException`을 직접 raise하지 않는다.** 도메인 예외를 던지면 핸들러가 HTTP로 변환한다.

---

## 4. 외부 호출 안전 실패

외부 시스템(LLM, 벡터스토어, 검색 엔진 등) 호출은 다음 패턴을 따른다.

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    retry=retry_if_exception_type((ConnectionError, TimeoutError)),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    reraise=True,
)
async def call_external_service(...):
    ...
```

- **부분 실패 격리**: 배치 처리 시 개별 항목 실패가 전체를 중단시키지 않도록 격리한다. 실패 항목은 로그에 기록하고 나머지를 계속 처리한다.
- **멱등성**: 재시도 가능한 작업에는 `document_id`, `chunk_id` 등 식별자를 키로 사용해 중복 처리를 방지한다.
- 재시도 소진 후에도 실패하면 적절한 `AppError` 서브클래스로 래핑해 던진다.

---

## 5. 상관 식별자 로깅

로거에 상관 식별자를 반드시 포함한다.

```python
import logging

logger = logging.getLogger(__name__)

# 처리 시작 시
logger.info("처리 완료", extra={
    "item_id": item_id,
    "result_count": len(results),
})

# 오류 발생 시
logger.error("처리 실패", extra={
    "request_id": request_id,
    "item_id": item_id,
}, exc_info=e)

# 부분 실패 시
logger.warning("일부 항목 처리 실패", extra={
    "item_id": item_id,
    "failed_count": failed,
})
```

**필수 상관 식별자**:
- `request_id`: API 요청 추적 (가능하면 미들웨어에서 주입)
- `item_id` / `document_id` 등 도메인 식별자: 처리 파이프라인 전 단계
- `job_id` / `query_id` 등 작업 식별자: 비동기 작업·쿼리 실행 단계

민감 정보(API 키, 토큰)는 로그에 포함하지 않는다.

---

## 6. 에러를 조용히 삼키지 않는다

```python
# 절대 금지
try:
    do_something()
except Exception:
    pass

# 의도적 무시 시 — WHY 한 줄 + log.warning 필수
try:
    update_cache()
except Exception as e:
    # 캐시 업데이트 실패는 응답에 영향 없음; 다음 요청에서 재시도됨
    logger.warning("cache", "캐시 업데이트 실패 (무시)", exc=e)
```

---

## 7. 실패 경로 테스트 의무화

변경된 비즈니스 로직에 대해 **최소 하나의 실패 경로 pytest**를 포함한다.

```python
import pytest
from app.exception.errors import NotFoundError, ExternalServiceError

def test_item_not_found(service):
    with pytest.raises(NotFoundError):
        service.get_item("nonexistent-id")

def test_external_service_failure(service, mock_client):
    mock_client.query.side_effect = ConnectionError("연결 실패")
    with pytest.raises(ExternalServiceError):
        service.search("query")
```

성공 경로만 테스트하는 PR은 반려 대상이다.

---

## 8. 체크리스트

- [ ] 새 오류는 `errors.py`에 `AppError` 서브클래스로 추가했는가?
- [ ] 컨트롤러/서비스에서 직접 `JSONResponse`/`HTTPException`으로 응답을 만들지 않았는가?
- [ ] 외부 호출에 `tenacity` 재시도 또는 명시적 예외 처리가 있는가?
- [ ] 로그에 `request_id`, `item_id` 등 상관 식별자가 포함되는가?
- [ ] `except Exception: pass` 패턴이 없는가? (있다면 WHY 주석 + log.warning)
- [ ] 실패 경로 테스트가 최소 하나 이상 있는가?
