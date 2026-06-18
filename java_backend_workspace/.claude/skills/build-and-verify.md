---
name: "Build and Verify"
description: "Spring Boot 백엔드 빌드·검증 절차. Gradle 명령, QueryDSL Q-클래스 재생성, 테스트 실행, 완결성 체크리스트."
---

# Build & Verify Skill

> 매 회차 "완료"를 선언하기 전에 돌려야 하는 표준 검증 절차다.
> 빌드 도구는 **Gradle Wrapper** (`./gradlew` / Windows `gradlew.bat`). Java 17은 toolchain 고정.

---

## 1. 표준 검증 명령 (순서대로)

```bash
./gradlew compileJava     # QueryDSL Q-클래스를 src/main/generated/ 에 재생성 + 컴파일
./gradlew test            # JUnit 5 + Spring Security Test (단위/슬라이스)
./gradlew bootJar         # 배포 산출물 (배포 영향 변경 시)
```

- CI 빌드와 동일하게 맞추려면: `./gradlew clean bootJar -x test`
- `clean` 은 `src/main/generated/` (Q-클래스)를 삭제하므로, 이후 `compileJava` 로 재생성된다.

### 단일 테스트 (변경 근처 타깃 우선)

```bash
./gradlew test --tests com.{company}.{app}.product.service.ProductServiceTest
./gradlew test --tests com.{company}.{app}.product.service.ProductServiceTest.create_중복이면_예외
```

광범위 스위트보다 **변경된 동작 근처 타깃 테스트**를 우선한다.

### 보고서

```bash
start build/reports/tests/test/index.html   # Windows
open build/reports/tests/test/index.html    # macOS
```

---

## 2. 프로파일 전환

```bash
./gradlew bootRun                                        # 기본 local
./gradlew bootRun --args='--spring.profiles.active=development'
SPRING_PROFILES_ACTIVE=development ./gradlew bootRun
```

기본 프로파일 `local` (옵션: `development`, `production`).
통합 테스트용 DB/Redis 는 `docker-compose.yml`.

---

## 3. PASS 판정 기준

| 항목 | PASS 조건 |
|------|-----------|
| `compileJava` | 컴파일 에러 0, Q-클래스 재생성 성공 |
| `test` | 실패 0, 변경 동작에 대한 RED→GREEN 흔적 존재 |
| `bootJar` | jar 파일 생성 성공 (배포 영향 변경일 때) |

빌드/테스트 출력을 **직접 읽고** 판정한다. "통과했을 것"은 금지.

---

## 4. 완결성 체크리스트 (빌드 통과 ≠ 완료)

`.claude/principles/completeness-principle.md` 기준. 아래가 모두 충족돼야 진짜 완료다.

```
[ ] 엔티티·DTO·레포지토리·서비스·컨트롤러가 정합하게 연결
[ ] DDL 변경 시 스키마 파일 수정
[ ] SecurityConfig permitAll 영향 검토 (변경 시 별도 커밋 + 사유)
[ ] 성공·실패 양쪽 경로 테스트 커버
[ ] 로그에 민감 정보(토큰/비밀번호/개인정보) 미노출
[ ] application-{local,development,production}.yml 환경별 값 정합
[ ] CI/CD 파이프라인 영향 구성 변경 여부
```

---

## 5. 테스트 인프라

- **슬라이스/단위**: H2 인메모리 DB (`@AutoConfigureTestDatabase(replace=ANY)`). Oracle 사용 프로젝트는 `MODE=Oracle` 추가.
- **통합(시퀀스/Native query)**: TestContainers 로 보강 (CI 분리 가능).
- **Redis**: 단위/슬라이스는 Redis AutoConfiguration 제외 + `RedisTemplate` mock. 통합 시 TestContainers Redis.

---

## 6. 자주 막히는 지점

- Q-클래스 not found → `./gradlew compileJava` 로 재생성 (`src/main/generated/` 는 빌드 산출물, 손으로 커밋 금지).
- H2에서 DB 전용 기능이 다르게 동작 → 핵심 쿼리는 TestContainers 로 검증.
- 테스트가 첫 실행에 통과 → 테스트가 잘못된 것. RED를 먼저 확인.
