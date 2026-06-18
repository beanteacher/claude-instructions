---
name: "Code Review"
description: "GitHub PR 코드 리뷰 — Spring Boot 백엔드 컨벤션 기반 체계적 분석, 심각도 우선 피드백, 템플릿 기반 리뷰 코멘트 작성"
---

# Code Review Skill (Spring Boot 백엔드)

## 목적
Spring Boot 백엔드 저장소의 Pull Request 를 본 프로젝트 컨벤션과 원칙(`/.claude/rules`, `/.claude/principles`) 에 맞춰 일관되게 리뷰한다.

리뷰 기준 우선순위:
1. **`/.claude/principles/*`** — 위반 시 머지 반려 (보안·DB·예외·API·구조·완결성)
2. **`/.claude/rules/*`** — 일반 규칙 (컨벤션, 디자인 패턴, 테스팅)

---

## 루프 검증 게이트 연계 (Loop Engineering)

이 스킬은 두 가지 맥락에서 동일한 체크리스트로 동작한다:

- **수동 PR 리뷰** — 아래 "코드 리뷰 프로세스" 그대로. 사람이 호출.
- **무인 루프의 검증 게이트** — 이 스킬의 심층 체크리스트(A~P)를 PASS/FAIL 판정 기준으로 사용한다. 검증 절차(빌드/테스트 실행)는 `.claude/skills/build-and-verify.md` 를 함께 참조한다.

루프에서 호출될 때의 차이:
- 발견사항을 PR 코멘트로 포스팅하는 대신 **executor 에게 반환**해 수정 루프를 돈다.
- `🔴 Must Fix` 가 하나라도 있으면 회차는 **미완료(FAIL)**.
- 승인/머지/`main` push 는 **사람 확인 후**에만 (무인 단계에서 자동 수행 금지).

---

## 코드 리뷰 프로세스

### 1. PR 정보 수집
```bash
# 열려있는 PR 확인
gh pr list

# 특정 PR 상세 정보
gh pr view {PR_NUMBER} --json number,title,author,baseRefName,headRefName,additions,deletions,files

# PR 작성자 확인
gh pr view {PR_NUMBER} --json author --jq '.author.login'
```

### 2. 변경 사항 분석
```bash
# PR diff 확인
gh pr diff {PR_NUMBER}

# 또는 로컬 브랜치 기반
git diff {BASE_BRANCH}...HEAD              # 전체 diff
git diff {BASE_BRANCH}...HEAD --stat       # 파일별 통계
git diff {BASE_BRANCH}...HEAD --name-only  # 변경 파일 목록
git log {BASE_BRANCH}..HEAD --oneline      # 커밋 목록
```

### 3. 영향 범위 식별

변경된 파일을 도메인 + 레이어로 분류한다:

```
도메인:    각 프로젝트의 도메인 패키지 (예: product / order / user / auth / global)
레이어:    controller / service / repository / mapper / domain / dto
           global/security · global/config · global/exception
설정/스키마: 스키마 파일 (*.sql, migration 등)
           SecurityConfig (permitAll) · application*.yml · build.gradle
```

### 4. 심층 검토 수행 (아래 체크리스트)

### 5. 리뷰 문서 작성 (아래 템플릿)

### 6. PR에 리뷰 포스팅
```bash
gh pr comment {PR_NUMBER} --body "$(cat review.md)"
gh pr review {PR_NUMBER} --approve --body "리뷰 내용"            # 사용자 확인 필수
gh pr review {PR_NUMBER} --request-changes --body "리뷰 내용"
```

---

## 심층 검토 체크리스트

### A. 구조 / 아키텍처 (`.claude/principles/project-structure.md`)

- [ ] 새 코드가 `com.{company}.{app}.<domain>.<layer>` 경계에 맞게 배치되었는가
- [ ] Package-by-feature 원칙을 깼는가 (도메인 횡단으로 `controller/service/...` 묶지 않았는가)
- [ ] Controller → Service → Repository 단방향 의존이 깨지지 않았는가 (역의존·순환 의존 없음)
- [ ] 도메인 간 호출이 같은 bounded context 내부인가, 아니면 `ApplicationEventPublisher` 로 분리되었는가
- [ ] `global/` 모듈에만 도메인 공통 빈을 두었는가 (특정 도메인이 다른 도메인을 직접 참조하지 않는가)

### B. API / Controller (`.claude/principles/api-conventions.md`, `.claude/rules/backend-conventions.md`)

- [ ] 프로젝트 API URL 규칙(`CLAUDE.md` 명시 기준)을 따르는가
- [ ] 응답 envelope 방식이 프로젝트 표준을 따르는가 (`ApiResponse<T>` 또는 직접 반환 — 프로젝트 내에서 일관성 유지)
- [ ] 컨트롤러가 엔티티를 직접 반환하고 있지 않는가 (반드시 DTO)
- [ ] `@Valid` 가 요청 DTO 에 누락되지 않았는가
- [ ] try/catch 로 응답을 포장하지 않고 공통 예외(`AppException`)를 던져 `GlobalExceptionHandler` 에 위임하는가
- [ ] URL 이 REST 컨벤션을 따르는가 (kebab-case 허용)

### C. Service / 트랜잭션 (`.claude/rules/conventions.md`, `.claude/rules/java-patterns.md`)

- [ ] 쓰기 메서드에 `@Transactional`, 읽기 전용에 `@Transactional(readOnly = true)` 가 적절히 붙었는가
- [ ] `rollbackFor = Exception.class` 가 필요한 체크 예외 경로에 명시되었는가
- [ ] `@Transactional` 이 컨트롤러나 레포지토리에 잘못 붙어있지 않은가
- [ ] `open-in-view = false` 환경에서 뷰/컨트롤러 지연 로딩을 가정하지 않는가
- [ ] 인터페이스 + 구현체 분리가 정당한가 (구현 2개 이상 또는 런타임 교체 필요한 경우만 — 그 외에는 구체 `@Service` 클래스)
- [ ] 생성자 주입 (`@RequiredArgsConstructor` + `private final`) 만 사용했는가, production 에서 `@Autowired` 필드 주입 없는가
- [ ] `var` / 추측성 인터페이스 분리가 들어가 있지 않은가

### D. Repository / Persistence (`.claude/principles/database.md`)

- [ ] Entity 가 `BaseEntity` / `BaseTimeEntity` 를 상속했는가
- [ ] ID 생성 전략이 프로젝트 DB 에 맞게 선택되었는가 (MySQL: `IDENTITY`, Oracle: `SEQUENCE`)
- [ ] 연관관계 기본 fetch 가 `LAZY` 인가, 즉시 로딩은 명시적인 근거가 있는가
- [ ] 동적 쿼리는 QueryDSL `Repository Custom + Impl` 패턴으로 분리되었는가
- [ ] 리소스 조회 시 `BaseRepository.findByIdOrThrow(id)` 를 사용했는가 (`findById().orElseThrow(...)` 중복 작성 금지)
- [ ] `@Data`, 양방향 setter 등 안티패턴이 엔티티에 들어있지 않은가

### E. DDL / Schema (`.claude/principles/database.md`)

- [ ] 스키마 변경 시 프로젝트 스키마 파일(들)을 함께 수정했는가
- [ ] additive 변경 (ADD COLUMN / NEW TABLE) 인가, 파괴적 변경이면 사유와 마이그레이션 절차가 PR 본문에 있는가
- [ ] 프로젝트 스키마 관리 방식(Flyway/Liquibase/수동 SQL)을 따르고 있는가

### F. DTO / Mapping (`.claude/rules/conventions.md`, `.claude/rules/mapping.md`)

- [ ] DTO 가 `dto/{Domain}Dto.java` 단일 파일 + inner record 구조인가 (`dto/request/`, `dto/response/` 분리 금지)
- [ ] Controller / Service 안에 record 가 직접 선언되어 있지 않은가
- [ ] record 에 Lombok `@Builder` / Setter 가 들어있지 않은가 (불변)
- [ ] 변환 경로 우선순위를 지켰는가
  - 비즈니스 의미 → 도메인 팩토리 (`Entity.createFromXxx(...)`)
  - 외부 파라미터 결합 → record `of()`
  - 순수 1:1 매핑 → MapStruct (`{Domain}DtoMapper`)
- [ ] MapStruct 매퍼에 분기 / 계산 / 검증 로직이 들어가 있지 않은가
- [ ] `@Mapper(componentModel = "spring")` 이 누락되지 않았는가
- [ ] `from()` / `of()` 와 MapStruct 가 같은 변환을 중복 정의하고 있지 않은가

### G. 빈 재사용

- [ ] `ObjectMapper`, `WebClient`, `RestTemplate`, `JPAQueryFactory`, `RedisTemplate`, `PasswordEncoder`, `CorsConfigurationSource` 를 서비스에서 직접 `new` 하지 않았는가
- [ ] 같은 타입의 여러 빈이 필요할 때 빈 이름 또는 `@Qualifier` 로 명시했는가
- [ ] 환경별 값이 3개 이상 묶이면 `@ConfigurationProperties` 로 승격되었는가 (`@Value` 남발 금지)

### H. 예외 처리 (`.claude/principles/error-handling.md`)

- [ ] 비즈니스 오류가 공통 `AppException` 으로 통일되었는가 (엔티티별 커스텀 예외 클래스를 새로 만들지 않았는가)
- [ ] 재사용 가능한 오류는 `ErrorCode` enum 에 항목을 추가해서 `AppException(ErrorCode.X)` 로 던졌는가
- [ ] 컨트롤러 / 서비스에서 예외를 catch 후 무시하거나 응답으로 포장하고 있지 않은가
- [ ] `GlobalExceptionHandler` 에서 처리하지 못하는 새 예외 타입을 임의로 도입하지 않았는가

### I. 보안 (`.claude/principles/security.md`)

- [ ] 시크릿 (JWT 비밀키, DB 비밀번호, 외부 API 키) 이 코드/yml 에 하드코딩되지 않았는가
- [ ] `SecurityConfig` 의 `permitAll` 경로 변경이 PR 본문에 명시적 사유와 함께 있는가 (기본은 인증 필수)
- [ ] 외부 webhook 수신 시 서명·토큰 검증이 있는가
- [ ] JWT 만료/회전 로직 변경 시 토큰 무효화 영향이 검토되었는가
- [ ] 사용자 입력이 SQL 에 문자열 결합으로 들어가지 않는가 (JPA / QueryDSL 파라미터 바인딩 사용)
- [ ] 로그에 민감 정보 (토큰, 비밀번호, 개인정보) 가 출력되지 않는가
- [ ] `SecurityContextHolder` 직접 접근 대신 `SecurityUtils.getLoginId()` 또는 `@LoginAccount` 를 사용했는가

### J. 디자인 패턴 적용 (`.claude/rules/backend-design-patterns.md`)

- [ ] 동일 인터페이스의 분기가 3개 이상이면 Strategy + Factory 로 분리되었는가
- [ ] 처리 흐름은 같고 일부 단계만 다르면 Template Method 또는 private helper 분해로 정리되었는가
- [ ] 외부 시스템 연동이 Adapter 로 격리되었는가
- [ ] 패턴이 과도하게 도입되지 않았는가 (조건 미해당 시 단순 구현)

### K. 이벤트 / afterCommit

- [ ] DB 변경 후 부수효과(외부 알림, 동기화 등)를 서비스에서 직접 호출하지 않고 `ApplicationEventPublisher` 로 분리했는가
- [ ] 리스너가 `@TransactionalEventListener(phase = AFTER_COMMIT)` 인가
- [ ] 신규 이벤트가 `record` 로 정의되어 있는가

### L. 외부 시스템 파이프라인 (프로젝트별 해당 항목 확인)

- [ ] 외부 시스템 연동(파일 생성, 동기화, 알림 발송 등)이 afterCommit 훅으로 트리거되는가
- [ ] 원자성이 필요한 작업이 적절한 트랜잭션/보상 로직으로 감싸졌는가

### M. 도메인 불변조건 (프로젝트별 핵심 도메인)

- [ ] 핵심 도메인의 상태 전이 규칙이 지켜지는가
- [ ] 도메인 enum(`AttributeConverter` 포함)이 일관되게 사용되었는가
- [ ] 중복 검증 경로가 유지되었는가

### N. 코드 스타일 / 완결성 (`.claude/principles/code-style.md`, `.claude/principles/completeness-principle.md`)

- [ ] 변경 diff 가 작업 의도에 비해 과도하게 크지 않은가 (불필요 리팩터 / 포매팅 변경 섞이지 않음)
- [ ] 메서드 시그니처를 의미 없이 다중 라인으로 풀지 않았는가
- [ ] import 와일드카드 (`.*`) 가 들어있지 않은가
- [ ] 들여쓰기 4 spaces 유지
- [ ] 기존 공통 자산을 재사용했는가 (`BaseRepository`, `GlobalExceptionHandler` 등)
- [ ] 완결성 체크리스트 통과 — DDL 동기화, permitAll 영향 등이 빠짐없이 처리되었는가

### O. 테스트 (`.claude/rules/testing.md`)

- [ ] TDD 계층 순서를 따랐는가 (Repository → Service → Controller)
- [ ] 도메인/API 변경에 회귀 테스트가 추가되었는가
- [ ] 보안 필터 / 인증 흐름 변경 시 `@WithMockUser` 또는 `SecurityMockMvcRequestPostProcessors` 기반 테스트가 있는가
- [ ] 슬라이스 테스트는 `@DataJpaTest` (H2 Oracle 모드) / `@WebMvcTest` / `@ExtendWith(MockitoExtension.class)` 등 적절한 어노테이션을 사용했는가
- [ ] 통합 테스트가 필요한 경로 (시퀀스, Native query, 외부 파이프라인) 에 보강이 있는가
- [ ] 테스트 코드의 `@Autowired` 필드 주입 허용은 슬라이스/통합 테스트로 한정되었는가

### P. Git / 커밋 (`.claude/rules/git-workflow.md`)

- [ ] 커밋 메시지에 **scope 가 없는가** (예: `feat: ...` ⭕ / `feat(backend): ...` ❌)
- [ ] 허용 타입 (`feat / fix / refactor / perf / docs / test / chore / style / ci / build`) 만 사용했는가
- [ ] subject 가 마침표로 끝나지 않는가, 본문이 "왜" 를 설명하는가
- [ ] 여러 의도가 한 커밋에 섞이지 않았는가
- [ ] 민감 변경 (보안 필터, SecurityConfig, permitAll) 이 별도 커밋으로 분리되었는가

---

## 피드백 등급

### 우선순위 분류

| 등급 | 의미 | 머지 조건 |
|------|------|-----------|
| 🔴 **Must Fix** | 머지 불가 (보안 위반, DB/스키마 파괴, 데이터 손실, principles 위반) | 수정 후 재리뷰 |
| 🟡 **Should Fix** | 수정 권장 (rules 위반, 설계 결함, 누락된 테스트, 하드코딩) | 수정 또는 사유 설명 |
| 🟢 **Consider** | 개선 제안 (네이밍, 가독성, 부분 리팩터링) | 머지 후 개선 가능 |

### 평가 등급

| 등급 | 조건 |
|------|------|
| 5/5 | Must Fix, Should Fix 없음. 즉시 머지 가능 |
| 4/5 | Should Fix 1~2건, Must Fix 없음. 피드백 확인 후 머지 |
| 3/5 | Must Fix 1건 또는 Should Fix 3건+. 수정 후 재검토 |
| 2/5 | Must Fix 다수 또는 principles 위반. 대폭 수정 필요 |
| 1/5 | 재작성 필요 |

---

## 피드백 작성 규칙

### 원칙
1. **요약 먼저** — 바쁜 사람도 빠르게 파악 가능하게
2. **구체적 위치** — 파일명:라인번호 명시
3. **해결책 제시** — 문제만 지적하지 말고 개선안 코드 포함
4. **잘한 점도 언급** — 긍정적 피드백 포함
5. **기술적 근거** — 모든 피드백에 "왜"를 포함, 가능하면 본 프로젝트 규칙 문서를 인용 (`.claude/principles/...`, `.claude/rules/...`)

### 코드 위치 표시
```
파일명:라인번호         예) src/main/java/com/{company}/{app}/product/service/ProductService.java:62
파일명:시작-끝라인       예) src/main/java/com/{company}/{app}/global/security/SecurityConfig.java:48-71
```

### 피드백 포맷 예시
````markdown
🔴 **Must Fix** — `src/main/java/com/{company}/{app}/product/service/ProductService.java:62`

**문제**: `findById(id).orElseThrow(...)` 로 조회하고 있어 `BaseRepository.findByIdOrThrow` 컨벤션을 우회합니다.
오류 메시지가 도메인마다 갈라지고, 추후 공통 처리 (감사 로그, 트레이싱) 도 적용되지 않습니다.
참고: `.claude/rules/backend-conventions.md` "예외 처리 규칙".

**현재 코드**:
```java
Product product = productRepository.findById(id)
        .orElseThrow(() -> new EntityNotFoundException("..."));
```

**수정 제안**:
```java
Product product = productRepository.findByIdOrThrow(id);
```
````

---

## 리뷰 템플릿

````markdown
# PR Review: [PR 제목]

## 변경 요약
- X commits, Y files (+Z lines, -W lines)
- 주요 변경: [도메인 / 레이어 단위 요약 — 예: product 서비스에 할인율 계산 추가, SecurityConfig permitAll 수정]

## 영향 범위
- 도메인: [예: product, order]
- 레이어: [예: service, repository, mapper]
- 인프라/스키마: [예: 스키마 파일, SecurityConfig permitAll]

## 종합 평가
- **평가**: X/5
- **머지 가능 여부**: ✅ 즉시 머지 / ⏸️ 수정 후 머지 / ❌ 머지 보류

## 잘된 점
- [예: 변환 경로를 도메인 팩토리 + MapStruct 로 깔끔히 분리, afterCommit 이벤트로 부수효과 분리]

## 피드백

### 🔴 Must Fix

#### 1. [이슈 제목]
**위치**: `파일:라인`
**근거**: `.claude/principles/...` 또는 `.claude/rules/...`
**문제**: [설명]
**수정 제안**:
~~~java
// 수정 코드
~~~

### 🟡 Should Fix

#### 1. [이슈 제목]
**위치**: `파일:라인`
**근거**: `.claude/rules/...`
**문제**: [설명]
**수정 제안**:
~~~java
// 수정 코드
~~~

### 🟢 Consider

#### 1. [이슈 제목]
**위치**: `파일:라인`
**제안**: [설명]

## Action Items

| 우선순위 | 이슈 | 파일:라인 | 조치 |
|---------|------|----------|------|
| 🔴 Must | [이슈] | [위치] | [조치] |
| 🟡 Should | [이슈] | [위치] | [조치] |
| 🟢 Consider | [이슈] | [위치] | [조치] |

## 보안 점검
- permitAll / SecurityConfig 변경: [있음/없음 + 사유]
- 시크릿 / 토큰 노출: [확인 결과]
- 입력 검증 / SQL 바인딩: [확인 결과]

## DB / 스키마 점검
- 스키마 파일 수정 여부: [✅/❌]
- additive vs 파괴적 변경: [확인 결과]

## 머지 체크리스트
- [ ] Must Fix 항목 모두 수정
- [ ] `./gradlew compileJava` / `./gradlew test` / `./gradlew bootJar` 통과
- [ ] DDL 동기화 완료
- [ ] permitAll / 보안 영향 검토 완료
- [ ] 회귀 테스트 추가 / 통과

---
🤖 *This review was generated by [Claude Code](https://claude.ai/code)*
````

---

## PR 코멘트 포스팅 가이드

### 올바른 방법
```bash
# 방법 1: 변수로 읽어서 전달
CONTENT=$(cat review.md)
gh pr comment {PR_NUMBER} --body "$CONTENT"

# 방법 2: stdin 파이프
cat review.md | gh pr comment {PR_NUMBER} --body-file -

# 방법 3: Here Document
gh pr comment {PR_NUMBER} --body "$(cat <<'EOF'
[리뷰 내용]
EOF
)"
```

### 주의사항
- `--body-file 파일명` 대신 `--body "$CONTENT"` 사용 (파일 업로드 방지)
- PR 승인(`--approve`)은 **반드시 사용자 확인 후** 진행
- 중첩 코드 블록 시 외부: 백틱(```), 내부: 물결표(~~~) 사용

---

## 빠른 참조 커맨드

```bash
# PR 정보
gh pr view {PR_NUMBER} --json number,title,author,baseRefName,headRefName,additions,deletions
gh pr diff {PR_NUMBER}

# 변경 통계
git diff {BASE_BRANCH}...HEAD --stat
git log {BASE_BRANCH}..HEAD --oneline

# 리뷰 포스팅
gh pr comment {PR_NUMBER} --body "$(cat review.md)"
gh pr review {PR_NUMBER} --approve --body "리뷰 내용"
gh pr review {PR_NUMBER} --request-changes --body "리뷰 내용"

# 기존 리뷰 코멘트 조회
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments

# 빌드/테스트 검증
./gradlew compileJava
./gradlew test
./gradlew test --tests com.{company}.{app}.product.service.ProductServiceTest.someMethod
./gradlew bootJar
```
