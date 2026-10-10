# 백엔드 코드 컨벤션 (규칙 전용)

> **이 파일은 Spring Boot 백엔드의 범용 규칙·컨벤션만 포함합니다.**
> 코드 예시는 `backend-java-patterns.md`, 디자인 패턴은 `backend-design-patterns.md`,
> 엔티티·Repository·쿼리·트랜잭션은 `backend-jpa.md`를 참고하세요.

---

## 레이어별 책임 원칙

| 레이어 | 책임 | 금지 사항 |
|--------|------|-----------|
| **Controller** | 요청 수신 + 응답 반환 | 비즈니스 로직 금지 |
| **Service** | 비즈니스 로직 | DB 직접 접근 금지 (Repository 경유) |
| **Repository** | 쿼리 실행 | 비즈니스 로직 금지 |
| **Domain (Entity)** | 상태 보유 + 도메인 메서드 캡슐화 | 요청/응답 어노테이션 금지 |

- Controller → Service → Repository 단방향 의존. 순환 의존 금지.
- Entity를 API 응답으로 직접 반환 금지. 반드시 DTO로 변환.

---

## 명명 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| 클래스명 | PascalCase | `ProductService` |
| 메서드/변수명 | camelCase | `getById`, `productList` |
| 상수 | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE` |
| 패키지명 | 소문자, 점 구분 | `com.company.service.controller` |

---

## DTO Record 규칙

```
규칙 1. dto/request/, dto/response/ 하위 폴더 구조 사용 금지.

규칙 2. 기능 패키지 내 dto/{Domain}Dto.java 파일 하나에 inner record를 모두 선언.
        → product/dto/ProductDto.java    (CreateRequest, Response 등)
        → order/dto/OrderDto.java        (CreateRequest, Response 등)

규칙 3. Controller / Service에 직접 record 선언 금지.
        항상 dto/{Domain}Dto.java를 참조한다.

규칙 4. record 유효성 검증은 파라미터에 직접 어노테이션 적용.
        public record CreateRequest(@NotBlank String name, @Positive BigDecimal price) {}

규칙 5. Entity → Record 변환은 record 내부 static 팩토리 메서드로 처리.
        public static Response from(Product product) { return new Response(...); }

규칙 6. record는 불변(immutable) — Setter, Lombok @Builder 사용 금지.

규칙 7. 계층별 DTO 분리 — 누구에게 넘기는지에 따라 이름 끝을 나눈다.
        Request / Response  프론트와 주고받는 API 형식 (JSON 으로 나가는 것만)
        Result              서비스가 컨트롤러에 돌려주는 값 (쿠키·헤더로 나갈 값 등 응답 본문에 없는 것 포함)
        → Response 안에 Result 를, Result 안에 Response 를 넣지 않는다.
        → 컨트롤러가 Response.from(result) 로 바꿔서 내보낸다.
        → 서비스 결과가 Response 와 똑같으면 Result 를 만들지 않고 Response 를 바로 돌려준다.
        → 파일 안에서 "API 요청·응답" 묶음과 "서비스 → 컨트롤러" 묶음을 주석으로 나눈다.

규칙 8. 서비스 입력(Command)은 필요할 때만 만든다. 기본은 서비스가 Request 를 그대로 받는다.
        아래 중 하나라도 해당하면 → {동작}Command record 를 만든다
        1. 같은 서비스를 HTTP 말고 다른 곳(배치·메시지·다른 서비스)에서도 부른다
        2. 서비스에 넘길 값이 요청 본문과 다르고, 따로 넘길 매개변수가 3개를 넘는다
           (로그인 정보·경로 값 1~2개는 매개변수로 따로 넘긴다: changePassword(accountId, request))
        3. 요청 본문에 서비스가 믿으면 안 되는 값(화면 표시용 값 등)이 섞여 있다
        → 모두 아니면 Request 와 필드가 똑같은 Command 를 미리 만들지 않는다.

규칙 9. 계층 사이(반환·매개변수)와 API 요청·응답에 Map 을 쓰지 않는다. 명시적 record 로 주고받는다.
        키·값이 무엇인지 타입에 안 보이기 때문이다.
        → 메서드 안에서 찾기용으로 잠깐 쓰는 지역 변수는 괜찮다.
```

```java
// 권장
public record LoginResponse(Long id, String userId, Role role) {
    public static LoginResponse from(LoginResult result) { ... }
}
public record LoginResult(Long id, String userId, Role role, String accessToken, String refreshToken) {}

// 컨트롤러
LoginResult result = authService.login(request);

return ResponseEntity.ok()
    .header(HttpHeaders.SET_COOKIE, authCookies.accessToken(result.accessToken()).toString())
    .body(ApiResponse.success(LoginResponse.from(result)));

// 지양 (Result 가 Response 를 품고 있어 어디까지가 API 형식인지 흐려진다)
public record LoginResult(LoginResponse account, String accessToken, String refreshToken) {}
```

---

## API 설계 컨벤션

### REST API 규칙

```
GET    /api/v1/{resource}          목록 조회
GET    /api/v1/{resource}/{id}     단건 조회
POST   /api/v1/{resource}          생성
PUT    /api/v1/{resource}/{id}     전체 수정
PATCH  /api/v1/{resource}/{id}     부분 수정
DELETE /api/v1/{resource}/{id}     삭제
```

### 응답 포맷

- 성공: `{ success, data, message, timestamp }`
- 실패: `{ success: false, message, timestamp }`
- 모든 응답은 `ApiResponse<T>`로 래핑

---

## Service 구조 판단 기준

```
아래 조건 중 하나라도 해당하면 → 인터페이스 + 구현체
모두 해당하지 않으면         → 구체 클래스 (기본)

[인터페이스 + 구현체 조건]
1. 동일 역할의 구현이 2개 이상 존재하거나 예정된 경우
2. 런타임에 구현체를 교체하거나 선택해야 하는 경우
3. 외부 시스템 연동을 추상화하여 테스트 대체(Mock)가 필요한 경우

[구체 클래스 조건 — 위 조건에 해당하지 않는 모든 경우]
  → 일반 CRUD, 단일 비즈니스 로직 등
  → "혹시 나중에 바뀔 수도 있으니까" 같은 추측성 분리 금지
```

---

## Spring 컨벤션

- 생성자 주입 — `@RequiredArgsConstructor` 사용. `@Autowired` 필드 주입 금지.
  - 생성자를 직접 쓰지 않는다. 설정값(`@ConfigurationProperties`)에서 꺼낸 값은 필드로 옮기지 말고 properties 를 주입받아 쓸 때 읽는다. 고정값은 `static final` 상수로 둔다.
  - 예외: 켜질 때 한 번 만들고 잘못되면 바로 실패해야 하는 값(예: JWT 서명 키), `super(...)` 를 불러야 하는 예외 클래스.

```java
// 권장
@Service
@RequiredArgsConstructor
public class AuthService {
    private final SecurityProperties properties;
    ...
    LocalDateTime.now().plus(properties.jwt().refreshTokenTtl());

// 지양 (주입만 하는 긴 생성자)
public AuthService(AccountRepository accountRepository, SecurityProperties properties, ...) {
    this.accountRepository = accountRepository;
    this.refreshTokenTtl = properties.jwt().refreshTokenTtl();
    ...
}
```
- `Optional` — 반환 타입에만 사용. 필드, 파라미터에 사용 금지.

---

## 코드 포매팅 규칙

- **메서드 시그니처 개행 금지** — 파라미터가 과도하게 긴 경우(라인 120자 초과 등)가 아니면 시그니처를 한 줄로 유지한다. 파라미터가 2~3개인 평범한 메서드를 굳이 여러 줄로 쪼개지 않는다.
- 메서드 호출도 동일 — 특별히 길거나 가독성이 떨어지는 경우에만 개행한다.

```java
// 권장
public ResponseEntity<ApiResponse<UsageDto.Response>> getHistory(int page, int size) { ... }

// 지양 (불필요한 개행)
public ResponseEntity<ApiResponse<UsageDto.Response>> getHistory(
    int page,
    int size
) { ... }
```

- **줄바꿈한 시그니처는 8칸 들여쓰기** — 120자를 넘어 매개변수·`throws` 를 줄바꿈할 때는 선언보다 8칸(두 단계) 들여쓴다. 본문(4칸)과 같은 깊이면 어디서 본문이 시작하는지 안 보인다. 생성자·record 선언도 같다. 긴 상수 이름은 static import 로 줄인다.

```java
// 권장 (매개변수 12칸, 본문 8칸)
    public ResponseEntity<ApiResponse<Void>> logout(
            @AuthenticationPrincipal Long accountId,
            @CookieValue(name = REFRESH_TOKEN, required = false) String refreshToken) {
        authService.logout(accountId, refreshToken);

// 지양 (매개변수와 본문이 같은 깊이)
    public ResponseEntity<ApiResponse<Void>> logout(
        @AuthenticationPrincipal Long accountId,
        @CookieValue(name = AuthCookies.REFRESH_TOKEN, required = false) String refreshToken) {
        authService.logout(accountId, refreshToken);
```

- **패키지 경로를 코드에 쓰지 않는다** — 클래스는 `import` 하고 본문에는 클래스 이름만 쓴다. 테스트 코드도 같다. 예외는 이름이 같은 클래스 두 개를 한 파일에서 써야 할 때뿐이며, 그때도 덜 쓰는 쪽 하나만 전체 경로로 쓴다.

```java
// 권장
import org.springframework.test.web.servlet.ResultActions;
...
private ResultActions login(String userId, String password) throws Exception { ... }

// 지양
private org.springframework.test.web.servlet.ResultActions login(String userId, String password) throws Exception { ... }
```

- **서비스 결과는 변수로 받고 빈 줄** — 컨트롤러는 서비스(다른 Bean) 호출 결과를 변수에 받은 뒤 `return` 한다. `return` 안에 서비스 호출을 넣지 않는다 (호출과 응답 감싸기가 한 줄에 겹쳐 읽기 어렵다). 호출과 `return` 사이에는 빈 줄을 한 줄 둔다. 반환값이 없는 호출(`void`)도 호출 뒤 빈 줄을 둔다.

```java
// 권장
AuthService.LoginResult result = authService.login(request);

return ResponseEntity.ok()
    .header(HttpHeaders.SET_COOKIE, ...)
    .body(ApiResponse.success(result.account()));

// 지양 (호출과 return 이 붙어 있음)
AuthService.LoginResult result = authService.login(request);
return ResponseEntity.ok()...;

// 지양 (return 안에서 바로 호출)
return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.success(categoryService.create(request)));
```

---

## 예외 처리 규칙

- 엔티티마다 별도 예외 클래스 생성 금지. 공통 `EntityNotFoundException` 하나를 두고 엔티티명을 파라미터로 전달.
- 모든 예외는 `GlobalExceptionHandler`에서 일괄 처리. 각 Controller에 예외 처리 산재 금지.

### 입력 검증 위치 — `@Valid` 냐 서비스냐

```
요청 본문만 보고 판단할 수 있다 → @Valid (DTO 어노테이션)
  빈 값·길이·바이트 수·숫자 범위·정규식 형식
  → VALIDATION_ERROR + errors[{ field, message }] 가 자동으로 붙는다
  → 표준 어노테이션으로 안 되면 common/validation 에 검사 어노테이션을 만든다 (예: @MaxBytes)
  → 서비스에서 형식 검사를 하고 VALIDATION_ERROR 를 던지지 않는다 (칸 정보가 빠진다)

DB·현재 상태를 봐야 안다 → 서비스에서 AppException(ErrorCode.상황별코드)
  현재 비밀번호 불일치·중복·없는 대상·재고 부족·상태상 불가
  → 상황마다 오류 코드를 따로 둔다 (화면이 코드로 분기할 수 있게)
  → 특정 칸·줄이 문제면 errors 에 담아 준다

예외: 로그인처럼 실패 이유를 알려 주면 안 되는 곳은 형식 검사를 하지 않고
      한 가지 오류(LOGIN_FAILED)로만 답한다. 서버 보호용 길이 상한만 @Valid 로 둔다.
```

---

## 테스트 규칙

### FIRST 원칙

| 원칙 | 핵심 |
|------|------|
| **F**ast | 외부 의존성 최소화, Mock/인메모리 활용 |
| **I**solated | 테스트 간 상태 공유 금지 |
| **R**epeatable | 환경 무관하게 동일 결과 |
| **S**elf-validating | assert로 명확한 pass/fail |
| **T**imely | 기능 구현과 동시에 작성 |

### BE 테스트 도구

| 어노테이션 | 용도 |
|------------|------|
| `@ExtendWith(MockitoExtension.class)` | Service 단위 테스트 |
| `@WebMvcTest` | Controller 통합 테스트 |
| `@DataJpaTest` | Repository 슬라이스 테스트 |
| `@SpringBootTest` | 풀 통합 테스트 |

- 테스트 없는 기능 구현 금지.

---

## 디자인 패턴 적용 규칙

> 레이어별 패턴 목록, 적용 시점, 판단 기준: `.claude/rules/backend-design-patterns.md`

- 패턴 없이 절차적으로 작성된 코드는 리뷰에서 반려한다.
- 조건 미해당 시 단순 구현 유지. 과도한 패턴 적용 금지.
