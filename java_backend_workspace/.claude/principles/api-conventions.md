# API 규약 (Iron Rule)

- **경계에서 요청 입력을 검증한다. 클라이언트 페이로드를 절대 신뢰하지 않는다.**
  - 모든 요청 DTO 는 record + Bean Validation (`@NotNull`, `@NotBlank`, `@Size`, `@Positive`, `@Pattern` 등) 으로 선언.
  - 컨트롤러 파라미터에 `@Valid` 를 붙이고, 실패는 `GlobalExceptionHandler` 가 `VALIDATION_ERROR` 로 변환하도록 한다.
  - 식별자는 `@PathVariable` 로 받고 **서비스 레벨에서 소유권·권한을 재검증**한다 (URL 에 있다는 이유로 통과시키지 않는다).
- **명시적 에러 코드가 포함된 안정적인 응답 형태를 반환한다.**
  - 성공 응답은 도메인 DTO 를 `ResponseEntity.ok(...)` 로 반환.
  - 실패 응답은 `ErrorResponse` (코드 + 메시지 + 타임스탬프) 형태로 `GlobalExceptionHandler` 가 생성. 컨트롤러에서 직접 `ResponseEntity.badRequest().body(...)` 를 만들지 않는다.
  - 새 오류는 `ErrorCode` enum 에 코드를 추가한 뒤 `new AppException(ErrorCode.X)` 로 던진다.
- **핸들러는 가볍게 유지한다. 비즈니스 로직은 서비스/도메인 레이어로 이동한다.**
  - 컨트롤러 메서드는 기본적으로 3~5줄. 서비스 호출과 응답 래핑 외의 로직을 두지 않는다.
  - 분기가 3개 이상 생기면 서비스 내부 private helper 또는 Strategy 패턴으로 분리.

---

## URL 규칙

- `/api/v1/{resource}` 패턴을 기본으로 사용한다. 예:
  ```
  /api/v1/products
  /api/v1/products/{id}
  /api/v1/orders
  /api/v1/auth/login
  ```
- 외부 공개 API 가 없거나 내부 운영 콘솔 전용이면 `/api/v1/` 세그먼트를 생략하고 도메인 경로로 시작할 수 있다. 버전 정책은 프로젝트 시작 시 결정하고 `CLAUDE.md` 에 명시한다.
- **공개 경로는 반드시 `SecurityConfig` 의 `permitAll` 목록에 명시**한다. 기본은 인증 요구.
- HTTP method 시맨틱을 지킨다.
  ```
  GET    조회
  POST   생성
  PUT    전체 수정
  PATCH  부분 수정
  DELETE 삭제
  ```
- 상태 변경을 "POST /xxx/do-something" 같은 동사형 경로로 만들지 않는다. 리소스 중심으로 `PUT /xxx/{id}/state` 또는 전용 하위 리소스로 모델링.

---

## 응답 형태

- 단건: 도메인 DTO (예: `ProductDto.Response`).
- 목록(페이지 없음): `List<ProductDto.Response>`.
- 목록(페이지): `ApiResponse<Page<ProductDto.Response>>` 또는 프로젝트 표준 페이지 래퍼.
- 단순 성공 확인: `ResponseEntity.ok(true)` 또는 `ApiResponse.success(...)`.
- 실패: `GlobalExceptionHandler` 가 `ErrorResponse` 로 일괄 변환. `{ success, message, timestamp }` 형태 유지.

---

## Pagination / Sorting

- Spring Data `Pageable` 을 컨트롤러 파라미터로 바인딩.
- 기본값은 `application.yml` 의 `spring.data.web.pageable`:
  ```yaml
  default-page-size: 20
  max-page-size: 100
  one-indexed-parameters: true
  ```
- 요청은 `?page=1&size=20&sort=id,desc` 형태를 따른다.

---

## Webhook / 공개 경로

- 공개 경로(webhook 등)는 `permitAll` 이지만 **애플리케이션 단에서도 인증 처리를 해야 한다.**
  - webhook: 서명 헤더 / 공유 토큰 검증.
- 공격자가 임의 payload 로 호출할 수 있다는 전제로 작성한다.

---

## 예시

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ApiResponse<ProductDto.Response>> create(
            @RequestBody @Valid ProductDto.CreateRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(productService.create(request)));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<Page<ProductDto.Response>>> search(
            ProductDto.SearchParams params, Pageable pageable) {
        return ResponseEntity.ok(ApiResponse.success(productService.search(params, pageable)));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductDto.Response>> getById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(productService.getById(id)));
    }
}
```
