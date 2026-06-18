# API 규약 (Iron Rule)

- **경계에서 요청 입력을 검증한다. 클라이언트 페이로드를 절대 신뢰하지 않는다.**
  - 모든 요청 DTO 는 `Pydantic BaseModel` + `Field(...)` 로 선언. `...`(필수) / `default=` / `ge=` / `le=` / `min_length=` / `max_length=` / `pattern=` 를 활용한다.
  - 라우터 파라미터에 `Pydantic` 모델을 직접 바인딩하고, 실패는 `install_exception_handlers` 가 `validation_error` 코드로 변환(HTTP 422)한다.
  - 경로 파라미터는 라우터에서 받지만 **서비스 레벨에서 소유권·권한을 재검증**한다.

- **명시적 에러 코드가 포함된 안정적인 응답 형태를 반환한다.**
  - 성공 응답은 `response_model=XxxResponse` (Pydantic 모델)로 선언하거나 `dict` 를 직접 반환.
  - 실패 응답은 `app/exception/api_error_handler.py` 의 `install_exception_handlers` 가 일괄 변환. 에러 envelope 형태: `{"error": {"code": "...", "message": "...", "detail": {...}}}`.
  - 새 오류는 `app/exception/errors.py` 에 `AppError` 또는 그 하위 클래스를 추가한 뒤 `raise XxxError("message", detail={...})` 로 던진다. 라우터에서 직접 `JSONResponse(status_code=400, ...)` 를 만들지 않는다.

- **핸들러는 가볍게 유지한다. 비즈니스 로직은 서비스 레이어로 이동한다.**
  - 라우터 메서드는 기본적으로 3~5줄. 서비스 `await service.method(...)` 호출과 return 외의 로직을 두지 않는다.
  - 분기가 3개 이상 생기면 서비스 내부 private helper 로 분리.

- **API 구현 작업 노트에 요구사항 ID 추적을 포함한다.**
  - 커밋 / PR body 에 식별자를 표기한다.
  - 영향받는 프론트 경로·알림 채널·연동 서비스 여부도 함께 기록.

---

## URL 규칙

- **`/api/vN/...` prefix 사용 여부는 프로젝트 관행을 따른다.** 버전 분리가 필요 없으면 도메인 경로로 직접 시작한다.
  ```
  /auth/login
  /auth/logout
  /items
  /items/{id}
  /items/{id}/status
  ```
- 외부 공개 API 가 새로 생기면서 **버전 분리가 필요할 때만** `/v1/` 세그먼트 도입을 논의한다.
- **공개 경로는 반드시 미들웨어의 allowlist 에 명시**한다. 기본은 인증 요구.
- HTTP method 시맨틱을 지킨다.
  ```
  GET    조회
  POST   생성 또는 복잡한 질의(body 필요 시)
  PUT    전체 수정
  PATCH  부분 수정
  DELETE 삭제
  ```
- 상태 변경을 `POST /xxx/do-something` 같은 동사형 경로로 만들지 않는다. 리소스 중심으로 `PUT /xxx/{id}/state` 또는 전용 하위 리소스로 모델링.
- **레거시 호환 경로**는 새 경로로 대체하되, 클라이언트 마이그레이션 완료 전까지 하위 호환 라우터를 유지한다.

---

## 인증 (Bearer JWT via deps.py)

- 모든 보호 엔드포인트는 `Authorization: Bearer <token>` 헤더를 요구한다.
- `app/routers/deps.py` 의 의존성 함수가 토큰을 검증하고 계정 식별자를 반환한다.
- 서비스 의존성 함수는 `Depends(get_current_user)` 형태로 계정 컨텍스트를 전달받는다.

```python
# deps.py 핵심 패턴
security = HTTPBearer()

def get_current_user(
    request: Request,
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> int:
    return token_auth.verify_token(credentials.credentials)

def get_item_service(
    request: Request,
    user_id: int = Depends(get_current_user)
) -> ItemService:
    ...
    return ItemService(..., user_id=user_id)
```

- 공개 엔드포인트는 `Depends(security)` 없이 선언.
- logout 엔드포인트는 `Optional[str] = Header(None)` 으로 토큰을 받고, 없으면 HTTP 401 을 직접 raise.

---

## 응답 형태

- **단건**: Pydantic Response 모델 (예: `ItemResponse`, `DetailResponse`).
- **목록(페이지 없음)**: Pydantic 목록 래퍼 모델 (예: `ItemListResponse(items=[], total=N)`).
- **목록(페이지)**: `limit` / `offset` query 파라미터를 받아 `{"items": [...], "total": N}` 형태로 반환. Spring `Pageable` 의 `page/size/sort` 대신 `limit`/`offset` 를 사용한다.
- **단순 성공 확인**: `{"success": True, ...}` dict 또는 `True`.
- **실패**: `install_exception_handlers` 가 `{"error": {"code": "...", "message": "...", "detail": {...}}}` 로 일괄 변환. 라우터에서 직접 에러 body 를 구성하지 않는다.

```python
# 에러 envelope 예시 (api_error_handler.py 동작 결과)
{
  "error": {
    "code": "not_found",
    "message": "해당 리소스를 찾을 수 없습니다.",
    "detail": {"resource": "prompt", "id": 99}
  }
}

# 검증 오류 envelope (HTTP 422)
{
  "error": {
    "code": "validation_error",
    "message": "요청 형식이 올바르지 않습니다.",
    "detail": [{"loc": ["body", "question"], "msg": "field required", "type": "value_error.missing"}]
  }
}
```

---

## Pagination / Sorting

- Spring `Pageable` (`page`, `size`, `sort`) 대신 **`limit` / `offset` query 파라미터**를 사용한다.
- 기본값은 라우터 `Query(...)` 에 명시:

  ```python
  @router.get("/prompts", response_model=PromptListResponse)
  async def list_prompts(
      agent_code: str = Query(None, description="에이전트 코드 필터"),
      status_code: str = Query(None, description="상태 코드 필터 (A/D)"),
      limit: int = Query(100, ge=1, le=1000, description="최대 조회 개수"),
      offset: int = Query(0, ge=0, description="오프셋"),
      service: PromptService = Depends(get_prompt_service)
  ):
      return service.list_prompts(agent_code=agent_code, status_code=status_code, limit=limit, offset=offset)
  ```

- 대용량 스캔이 필요한 경우 `limit` 범위를 넓게 허용할 수 있다.

---

## 에러 계층 (errors.py)

| 클래스 | code | HTTP |
|---|---|---|
| `AppError` | `app_error` | 500 |
| `BadRequestError` | `bad_request` | 400 |
| `NotFoundError` | `not_found` | 404 |
| `ExternalServiceError` | `external_service_error` | 502 |
| `TimeoutError` | `timeout_error` | 504 |

새 오류를 추가할 때: `errors.py` 에 하위 클래스를 선언하고 `code` / `status_code` 클래스 변수를 지정한다. 핸들러 수정은 불필요.

```python
# errors.py 추가 예시
class DuplicateItemError(BadRequestError):
    code = "duplicate_item"
    message = "이미 존재하는 항목입니다."

# 서비스에서 사용
raise DuplicateItemError(detail={"item_id": item_id})
```

---

## 예시

### 인증 라우터 (`/auth`)

```python
# app/routers/auth.py
router = APIRouter(prefix="/auth", tags=["Authentication"])

@router.post("/login", response_model=LoginResponse)
def login(
    request_body: LoginRequest,
    request: Request,
    auth_service: AuthService = Depends(get_auth_service)
):
    result = auth_service.login(request_body.account_id, request_body.password)
    return LoginResponse(access_token=result["access_token"], ...)

@router.post("/logout")
def logout(
    authorization: Optional[str] = Header(None),
    auth_service: AuthService = Depends(get_auth_service)
):
    if not authorization:
        raise HTTPException(status_code=401, detail="토큰이 제공되지 않았습니다.")
    auth_service.logout(authorization)
    return {"message": "로그아웃 성공"}
```

### 일반 라우터 예시

```python
# app/routers/items.py
router = APIRouter()

@router.post("/items", response_model=ItemResponse)
async def create_item(
    request: ItemCreateRequest,
    service: ItemService = Depends(get_item_service),
):
    return await service.create(request)

@router.get("/items/{item_id}", response_model=ItemResponse)
async def get_item(
    item_id: str,
    service: ItemService = Depends(get_item_service),
):
    return await service.get(item_id)

@router.get("/items")
async def list_items(
    limit: int = Query(default=20, ge=1, le=1000),
    offset: int = Query(default=0, ge=0),
    service: ItemService = Depends(get_item_service),
):
    return service.list(limit=limit, offset=offset)
```

### Pydantic 요청/응답 스키마 (`schema.py`)

```python
# app/routers/schema.py
class ItemCreateRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, description="항목 이름")
    top_k: Optional[int] = Field(default=5, ge=1, le=50)
    tags: Optional[List[str]] = Field(None)
    session_id: Optional[str] = Field(None, max_length=64)

class ItemResponse(BaseModel):
    id: str
    name: str
    status: str
    metrics: Optional[Dict[str, float]] = None
```

---

## Swagger UI 활용

- FastAPI 는 기본으로 `/docs` (Swagger UI) 와 `/redoc` (ReDoc) 을 제공한다.
- 라우터 함수의 docstring 이 Swagger UI 의 description 으로 노출되므로 **Args / Returns 를 명시**한다.
- `response_model=XxxResponse` 선언 시 Swagger UI 에서 응답 스키마가 자동 생성된다.
- `tags=[...]` 를 `APIRouter` 또는 개별 라우터에 지정하여 Swagger UI 그룹을 명확히 분류한다.
- 개발 중 새 엔드포인트 추가 후 `/docs` 에서 직접 테스트하는 것을 권장한다.
