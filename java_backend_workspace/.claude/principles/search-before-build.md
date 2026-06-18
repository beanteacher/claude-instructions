# 만들기 전에 검색하기 (Iron Rule)

- 익숙하지 않은 인프라/프레임워크 패턴을 도입하기 전에 **기존 솔루션을 먼저 확인**한다. 저장소에 이미 갖춰진 공통 자산을 중복 구현하지 않는다. 전형적 예:
  - **HTTP 클라이언트** → `global/config/WebClientConfig` 의 공통 빈
  - **로그인 식별** → `SecurityUtils.getLoginId()` 또는 `@LoginAccount` ArgumentResolver
  - **감사 컬럼** → `global/entity/BaseEntity` / `BaseTimeEntity`
  - **예외 표준** → 프로젝트 표준 `AppException` + `ErrorCode` + `GlobalExceptionHandler`
  - **페이지 응답** → 공통 페이지 래퍼 모델
  - **공통 레포지토리 기능** → `BaseRepository` (`findByIdOrThrow`)
- 제약 조건을 충족하는 경우 **검증된 로컬 패턴을 재사용**한다. `.claude/rules/backend-java-patterns.md` 의 예시가 1차 참조 대상이다.
- 비표준 접근 방식(새로운 HTTP 라이브러리, 대체 ORM, 커스텀 트랜잭션 매니저 등) 을 선택하는 경우 **원칙 기반 근거를 문서화**하고 PR 본문에 기록한다.
- 다음을 구분해서 판단한다:
  - **검증된 패턴** — 이미 저장소나 Spring Boot 생태계에서 일반화된 구성 (예: `@RequiredArgsConstructor` 주입, MapStruct 매퍼)
  - **검토가 필요한 새로운/인기 있는 패턴** — 도입 시 팀 리뷰 필요 (예: Spring Modulith, Virtual Threads 대량 적용)
  - **특정 제약 조건으로 정당화되는 독자적 접근 방식** — 프로젝트 고유 원자성 요건 등으로 만들어진 구조
- 외부 라이브러리 추가 판단 흐름:
  1. Spring Boot 스타터 / Spring 생태계에 이미 있는가? → 있으면 그걸 쓴다.
  2. 기존 의존성으로 15줄 안에 해결되는가? → 그러면 라이브러리를 추가하지 않는다.
  3. 추가해야 한다면 Lombok + MapStruct + QueryDSL annotation processor 와 충돌하지 않는지 먼저 확인한다.
