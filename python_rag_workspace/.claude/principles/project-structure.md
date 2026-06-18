# 프로젝트 구조 (Iron Rule)

## 패키지 트리

```
{project-name}/
├── app/
│   ├── main.py                        # FastAPI 앱 엔트리포인트
│   ├── config/                        # 설정·로깅·공통 스키마
│   │   ├── settings.py                # 환경변수 기반 전역 설정
│   │   ├── logging_config.py          # 로깅 포맷·핸들러
│   │   ├── log_helper.py              # 로그 유틸
│   │   └── schemas.py                 # 공통 Pydantic 스키마
│   ├── core/                          # 도메인 핵심 인프라 로직
│   │   ├── initializer.py             # 코어 컴포넌트 초기화
│   │   ├── db/                        # DB 연결·클라이언트
│   │   ├── middleware/                # FastAPI 미들웨어
│   │   │   ├── auth.py
│   │   │   └── timeout_middleware.py
│   │   └── search/                    # 검색·병합·리랭킹 (필요 시)
│   ├── exception/                     # 예외 정의·핸들러
│   │   ├── errors.py
│   │   └── api_error_handler.py
│   ├── routers/                       # FastAPI 라우터 (HTTP 경계)
│   │   ├── deps.py                    # 의존성 주입
│   │   └── schema.py                  # 라우터 공개 Pydantic 스키마
│   ├── service/                       # 비즈니스 로직
│   └── utils/                         # 범용 유틸리티
├── conf/                              # 외부 설정 파일 (YAML, .env 등)
├── docker/                            # Docker 관련 파일
├── scripts/                           # 운영·배포 스크립트
├── templates/                         # 프롬프트·문서 템플릿 (필요 시)
├── outputs/                           # 런타임 출력 디렉토리
├── requirements.txt
└── README.md
```

---

## 의존 방향 (Iron Rule)

```
routers → service → core → 외부 인프라 클라이언트
```

- **routers**: HTTP 요청 수신, 응답 직렬화. `service`만 호출한다.
- **service**: 비즈니스 로직. `core`의 DB·외부 클라이언트·검색을 조합한다.
- **core**: 인프라 추상화 계층. 외부 클라이언트(DB, 검색 엔진, LLM 등)를 직접 다룬다.
- **exception**: 모든 레이어가 참조 가능한 예외 정의.
- **config**: 모든 레이어가 참조 가능한 설정·로깅.

### 금지 방향
- `core`가 `service`를 호출하는 것은 **금지**한다.
- `service`가 `routers`를 직접 참조하는 것은 **금지**한다.
- `routers`가 `core`를 직접 호출하는 것은 **금지**한다. (반드시 `service` 경유)

---

## Schema 경계 원칙

- **Pydantic 스키마(Request/Response 모델)는 `routers/` 경계에서만 노출**한다.
  - `routers/schema.py` — 공개 API 계약
  - `config/schemas.py` — 설정·공통 타입 (앱 전역 참조 가능)
- **내부 도메인 객체(dataclass, TypedDict 등)는 `core/`, `service/` 안에서만 유통**된다. 라우터 경계를 넘을 때는 반드시 Pydantic 스키마로 변환한다.

---

## 도메인별 서브패키지 표준

| 패키지 | 허용 서브패키지 / 파일 |
|--------|----------------------|
| `core/db/` | DB·외부 저장소 클라이언트 |
| `core/middleware/` | FastAPI 미들웨어 |
| `core/search/` | 검색 알고리즘·병합·리랭킹 (필요 시) |

새 서브패키지가 필요할 때는 위 집합으로 소화 가능한지 먼저 확인한다. 불가능한 경우에만 PR에 사유를 남기고 신설한다.

---

## 새 모듈 배치 원칙

1. **기능 도메인을 먼저 파악한다.** DB 접근 → `core/db/`, 검색 관련 → `core/search/`, 비즈니스 조합 → `service/`.
2. **기존 패키지 안에 파일을 추가**한다. 새 서브패키지·최상위 패키지 신설은 강력한 근거가 없으면 금지한다.
3. **최상위 `app/` 직속 패키지 신설은 금지**한다. `app/{config,core,exception,routers,service,utils}` 외 패키지를 새로 만들려면 아키텍처 검토가 필요하다.
4. 어댑터·플러그인 계열 확장은 해당 도메인의 `adapters/` 서브디렉토리 패턴을 따른다.

---

## 구조가 불명확할 때 1차 참조 순서

1. `README.md` — 프로젝트 개요·실행 방법
2. `app/main.py` — 라우터 등록·미들웨어 등록 순서
3. `app/config/settings.py` — 환경변수·설정 전체 목록
4. `.claude/principles/` — 아키텍처·코드 규칙 문서
5. 기존 도메인 코드 (`service/`, `core/` 등)

---

## 외부 디렉토리 역할

| 디렉토리 | 용도 |
|----------|------|
| `conf/` | 환경별 설정 파일 (YAML, .env) |
| `docker/` | Dockerfile, compose 파일 |
| `scripts/` | 운영·배포·마이그레이션 스크립트 |
| `templates/` | 프롬프트 템플릿, 문서 템플릿 (필요 시) |
| `outputs/` | 런타임 생성 파일 (커밋 금지) |
