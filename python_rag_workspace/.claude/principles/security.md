# 보안 (Iron Rule)

## 시크릿 하드코딩 금지

- **절대로 시크릿이나 자격증명을 코드에 하드코딩하지 않는다.**
- JWT secret, DB 비밀번호, 외부 서비스 URL/키, API key 등 모든 민감 값은 `.env` 파일로만 주입한다.
- 운영 값은 배포 파이프라인의 환경변수/Secrets로 공급한다.
- 설정 로드는 반드시 `pydantic-settings`의 `BaseSettings`(`app/config/settings.py`)를 경유한다. `os.environ` 직접 접근이나 문자열 리터럴 하드코딩은 금지.

```python
# 금지
DATABASE_PASSWORD = "secret"
EXTERNAL_API_KEY = "sk-..."

# 올바른 방법 — pydantic-settings로만 로드
from app.config.settings import settings
password = settings.database_password
```

---

## 인증 및 인가

- 인증은 `app/routers/deps.py`의 의존성 함수를 통해 Bearer 토큰(`HTTPBearer`)으로 처리한다.
- 토큰 검증은 전용 `TokenAuth` 클래스가 담당한다. DB에서 유효성·만료·폐기 여부를 확인한다.
- 비밀번호 해싱 및 검증은 반드시 전용 유틸리티(`PasswordManager` 등, bcrypt)를 경유한다. 저수준 `bcrypt.hashpw` 직접 호출 금지.

### 공개 라우트 화이트리스트 원칙

FastAPI는 Spring Security의 `permitAll`과 달리 **`Depends(get_account_no)` 미부착 라우트가 곧 인증 없이 공개된다.** 기본은 항상 인증 요구.

```python
# 공개 라우트 — 명시적 주석 필수
# [PUBLIC] 인증 불필요 — 로그인 엔드포인트
@router.post("/auth/login", response_model=LoginResponse)
def login(...):
    ...

# 인증 필요 라우트 — Depends(get_account_no) 반드시 포함
@router.post("/ingest")
def ingest(account_no: int = Depends(get_account_no), ...):
    ...
```

- 새 엔드포인트를 공개로 만들 때는 **라우터 데코레이터 바로 위에 `# [PUBLIC]` 주석을 달고**, PR 본문에 공개 사유를 남긴다.
- 인증/인가 경로(`deps.py`, `auth.py`, `auth_service.py`, `middleware/auth.py`)를 수정할 때는 해당 변경에 대한 테스트를 반드시 추가한다.

---

## 입력 검증

- 모든 요청 스키마는 `Pydantic Field`로 제약을 선언한다(`min_length`, `max_length`, `pattern`, `ge`, `le`).
- 사용자 제어 데이터를 쿼리/프롬프트에 직접 concat 하지 않는다.

```python
from pydantic import BaseModel, Field

class LoginRequest(BaseModel):
    account_id: str = Field(min_length=1, max_length=64, pattern=r"^[a-zA-Z0-9_\-]+$")
    password: str = Field(min_length=8, max_length=128)
```

### 파일 업로드 보안

파일 업로드 엔드포인트는 아래 3가지를 모두 적용한다.

1. **매직바이트 검증**: `filetype` 또는 `puremagic`으로 실제 파일 타입 확인. 확장자만으로 신뢰 금지.
2. **경로 traversal 방지**: 파일명은 반드시 `os.path.basename`으로 정규화하고, 저장 디렉토리는 화이트리스트로 제한.
3. **사이즈 제한**: 업로드 허용 최대 크기를 명시적으로 설정.

```python
import os, filetype

def safe_save(file: UploadFile, upload_dir: str):
    # 경로 traversal 방지
    filename = os.path.basename(file.filename)

    # 매직바이트 검증
    content = await file.read(2048)
    kind = filetype.guess(content)
    if kind is None or kind.extension not in ALLOWED_EXTENSIONS:
        raise HTTPException(status_code=400, detail="허용되지 않는 파일 타입")

    # 화이트리스트 디렉토리만 허용
    dest = os.path.join(upload_dir, filename)
    if not dest.startswith(upload_dir):
        raise HTTPException(status_code=400, detail="잘못된 파일 경로")
```

---

## SQL Injection 방지

- SQLAlchemy ORM 또는 파라미터 바인딩(`cursor.execute(query, (param,))`)만 사용한다.
- 사용자 입력을 쿼리 문자열에 직접 `format`하거나 concat하는 raw SQL 금지.

```python
# 금지
query = f"SELECT * FROM tbl_account WHERE id = '{account_id}'"

# 올바른 방법 — 바인딩 파라미터
query = "SELECT no, id, password FROM tbl_account WHERE id = %s"
cursor.execute(query, (account_id,))
```

---

## 출력 인코딩

- FastAPI의 기본 JSON 직렬화(`JSONResponse`, `response_model`)가 기본 이스케이프를 처리한다.
- 사용자 입력을 그대로 HTML이나 문자열 응답에 echo 할 때는 반드시 escape 처리한다.
- LLM 프롬프트에 사용자 입력을 삽입할 때는 prompt injection 가능성을 고려해 입력을 sanitize하거나 구분자를 명확히 한다.

---

## 민감 정보 로깅 금지

- JWT 토큰, 비밀번호, OTP, `.env` 값(API key 등)을 로그에 평문으로 남기지 않는다.
- 디버그 레벨(`DEBUG`)이라도 민감 정보를 그대로 출력하지 않는다.
- 토큰을 로그에 남겨야 할 경우 앞 20자만 출력한다(`token[:20] + "..."`).

```python
# 금지
logger.info("login", f"token={access_token}")
logger.debug("password received", extra={"password": password})

# 올바른 방법
logger.info("login", "로그인 성공", extra={"token": access_token[:20] + "..."})
```

- `app/config/logging_config.py`의 포매터에서 민감 필드를 마스킹하는 필터를 추가하는 것을 권장한다.
- `debug_log_llm_messages: bool`(`settings.py`)가 `True`일 때 LLM 메시지 전체가 덤프된다 — 프롬프트에 사용자 개인정보나 시스템 시크릿이 포함되지 않도록 주의한다.

---

## 암호화

- 비밀번호 해싱/검증은 반드시 전용 유틸리티(bcrypt 기반)를 경유한다.
- 서비스 내부에서 `hashlib`, `hmac`, `cryptography.hazmat` 등 저수준 암호화 API를 직접 사용하지 않는다.
- 추후 대칭 암호화(AES 등)가 필요하면 `app/utils/crypto.py` 등 전용 유틸에 메서드를 추가하고 서비스 계층은 해당 유틸을 호출한다.

---

## 최소 권한 원칙

- DB 운영 계정에 최소 권한만 부여한다(`SELECT`/`INSERT`/`UPDATE`만, `DROP`·`CREATE` 불필요 시 제외).
- 외부 서비스(검색 엔진, LLM, 스토리지 등) API key에도 필요한 권한만 부여한다.
- `.env`에서 각 외부 서비스의 접속 자격증명을 분리 관리하고, 읽기/쓰기 계정을 분리할 수 있으면 분리한다.

---

## OWASP Top 10 체크리스트 (PR 필수 확인)

PR 작성 시 아래 4개 카테고리를 체크리스트로 항상 확인한다.

| # | 카테고리 | 확인 항목 |
|---|----------|-----------|
| 1 | **SQL Injection** | 바인딩 파라미터 사용, raw SQL concat 없음 |
| 2 | **Broken Access Control** | 모든 비공개 라우트에 `Depends(get_account_no)` 포함, `[PUBLIC]` 주석 없는 미인증 라우트 없음 |
| 3 | **Sensitive Data Exposure** | `.env` 값 하드코딩 없음, 로그에 토큰·패스워드 평문 없음, HTTPS 강제 여부 확인 |
| 4 | **Security Misconfiguration** | 디버그 로그 플래그 운영 환경에서 `False`, DB 기본 자격증명 변경, strict mode 운영에서 활성화 |
