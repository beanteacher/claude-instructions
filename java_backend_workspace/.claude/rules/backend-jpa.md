# 백엔드 JPA·쿼리 규칙 (규칙 전용)

> 엔티티·Repository·쿼리·트랜잭션 규칙만 둔다. 레이어·DTO·예외 같은 일반 규칙은 `backend-conventions.md`,
> 스키마 변경·쿼리 중앙화 원칙은 `.claude/principles/database.md` 를 따른다.

---

## 엔티티

- `@Getter` + `@NoArgsConstructor(access = AccessLevel.PROTECTED)`. `@Setter`·`@Data` 금지.
- 만들 때는 `static create(...)`, 값을 바꿀 때는 뜻이 드러나는 도메인 메서드(`changePassword(...)`)를 쓴다.
- ID 는 DB 에 맞춘다 (MySQL `IDENTITY`, Oracle `SEQUENCE`).
- 연관관계를 쓰면 `fetch = LAZY`. 즉시 로딩은 이유를 주석으로 남긴다.
- 스키마 검사(`ddl-auto: validate`)를 켠 프로젝트에서 JPA 기본 타입과 다른 컬럼은 `columnDefinition` 으로 맞춘다.
  `CHAR(n)` → `"char(n)"`, `TINYINT` → `"tinyint"`, `TEXT` → `"text"`. 안 맞추면 켜질 때 실패한다.

---

## 쿼리 작성 방식

```
메서드 이름 쿼리로 되면 → 그대로 쓴다
  findBy·existsBy·countBy·deleteBy + 조건 몇 개

안 되면 → QueryDSL
  GROUP BY·집계, 조인, 동적 조건, 여러 컬럼만 골라 읽기
  → {Entity}RepositoryCustom (인터페이스) + {Entity}RepositoryImpl (구현) 로 나누고
    {Entity}Repository 가 JpaRepository 와 {Entity}RepositoryCustom 을 함께 상속한다
  → JPAQueryFactory 는 설정 클래스에 Bean 하나로 둔다

쓰지 않는다
  → JPQL 문자열 @Query — 오타·타입 오류가 실행할 때에야 드러난다
  → Native SQL — DB 전용 기능이 꼭 필요할 때만, 이유를 주석으로 남긴다
  → Specification — 동적 조건도 QueryDSL 로 통일한다
```

---

## 쿼리 결과 모양

```
엔티티 그대로 읽으면 → 엔티티를 돌려준다
일부 컬럼·집계 값이면 → 명시적 record 로 돌려준다
  → {Domain}Dto 안에 "저장소 → 서비스 (조회 결과)" 묶음을 따로 두고 거기에 선언한다
  → 이름은 담긴 뜻으로 짓는다 (예: CategoryProductCount)
  → Projections.constructor(Record.class, ...) 로 담는다
    (@QueryProjection 은 DTO 가 QueryDSL 에 묶이므로 쓰지 않는다. 생성자 불일치는 Repository 테스트가 잡는다)

쓰지 않는다
  → Tuple·Object[] 를 Repository 밖으로 내보내기
  → Map 으로 바꿔 돌려주기 — 키·값이 무엇인지 타입에 안 보인다
  → Repository 안에 인터페이스 프로젝션 선언하기
```

```java
// ProductDto
// ── 저장소 → 서비스 (조회 결과) ──
public record CategoryProductCount(Long categoryId, long productCount) {
}

// ProductRepositoryImpl
public List<ProductDto.CategoryProductCount> countByCategory() {
    return queryFactory
        .select(Projections.constructor(ProductDto.CategoryProductCount.class, product.categoryId, product.count()))
        .from(product)
        .groupBy(product.categoryId)
        .fetch();
}
```

---

## 트랜잭션

- 클래스 레벨 `@Transactional` + 읽기 메서드에 `@Transactional(readOnly = true)`.

---

## Repository 예시

```java
public interface ProductRepository extends JpaRepository<Product, Long>, ProductRepositoryCustom {

    boolean existsByCategoryIdIn(Collection<Long> categoryIds);
}

public interface ProductRepositoryCustom {

    List<ProductDto.CategoryProductCount> countByCategory();
}

@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom {

    private final JPAQueryFactory queryFactory;
    ...
}
```

---

## 테스트

- 메서드 이름 쿼리·QueryDSL 쿼리·DB 제약 조건은 `@DataJpaTest` 로 확인한다.
