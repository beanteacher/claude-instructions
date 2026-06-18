# 데이터베이스 규칙 (Iron Rule)

- **브라운필드 시스템에서 안전한 배포를 위해 추가적(additive) 스키마 변경을 선호한다.**
  - 컬럼 추가는 `NULL` 허용 + 기본값(default) 전략.
  - 컬럼 삭제/이름 변경은 두 단계로 나눈다: (1) 새 컬럼을 먼저 추가하고 코드에서 양쪽 호환, (2) 이후 릴리스에서 기존 컬럼 제거.
  - 타입 변경은 새 컬럼 + 데이터 마이그레이션 + 구 컬럼 제거 순서로 분리.
- **되돌릴 수 있는 전략의 마이그레이션을 사용한다.**
  - 프로젝트 스키마 관리 방식(Flyway, Liquibase, 수동 SQL)은 `CLAUDE.md` 에 명시한다.
  - 파괴적 변경은 영향 범위와 **롤백 SQL** 을 PR 본문에 첨부한다.
- **스키마 마이그레이션과 대규모 기능 리팩터링을 하나의 작업에 섞지 않는다.** 커밋 레벨에서도 분리한다.
  - `refactor: ...` 커밋에는 스키마 변경을 포함하지 않는다.
  - `feat: ...` + 스키마 추가 시, DDL 커밋은 별도 커밋(`chore: product 테이블에 discount_rate 컬럼 추가`) 으로 뺀다.
- **쿼리 로직을 중앙화한다.** 같은 조건을 `@Query` / QueryDSL / Native SQL 세 곳에 중복 작성하지 않는다. 동적 조건은 `XxxRepositoryCustom` + `XxxRepositoryImpl` 로 통일.
- **핵심 영속성 경로와 에러 케이스에 대한 테스트를 추가한다.**
  - `@DataJpaTest` 슬라이스에서 저장·조회·제약조건 위반을 검증.
  - DB 전용 기능(시퀀스, Native query 등)은 H2 호환 모드로 커버되지 않을 수 있으니, 필요하면 TestContainers 로 별도 통합 테스트를 작성.

---

## JPA 엔티티 주의사항

- **ID 생성 전략**은 프로젝트 DB 에 맞게 선택한다.
  - MySQL/MariaDB: `GenerationType.IDENTITY`
  - Oracle: `GenerationType.SEQUENCE` + `@SequenceGenerator(allocationSize = 1)`
  - 예:
    ```java
    // Oracle
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "PRODUCT_SEQ")
    @SequenceGenerator(name = "PRODUCT_SEQ", sequenceName = "PRODUCT_SEQ", allocationSize = 1)
    private Long id;

    // MySQL
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    ```
- **예약어 컬럼**은 `@Column(name = "\"desc\"")` 처럼 따옴표로 감싼다.
- **Boolean 컬럼**은 DB 타입에 따라 `@Convert` 또는 네이티브 boolean 컬럼을 사용한다.
- **Pagination** 은 `Pageable` + Spring Data 자동 변환을 사용한다. DB 전용 페이징 문법을 손으로 작성하지 않는다.
- **대량 삭제/업데이트** (`@Modifying`) 는 영속성 컨텍스트 캐시와 어긋나므로, 수행 직후 `EntityManager.clear()` 또는 관련 캐시를 무효화한다.

---

## SQL 로깅

- SQL 로깅(p6spy, Hibernate show-sql 등)은 `local` / `development` 프로필에서만 활성화. 운영에서는 비활성화.
- 성능 프로파일링을 위해 운영에 잠깐 켜야 한다면, 작업 종료 후 반드시 복구하고 커밋 로그에 사유를 남긴다.
