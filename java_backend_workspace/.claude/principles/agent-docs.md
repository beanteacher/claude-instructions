# 에이전트 우선 문서화 (Iron Rule)

- 에이전트를 위한 문서를 먼저 작성한다: **명시적 입력, 출력, 제약조건, 게이트.**
  - 입력: 어떤 파일·패키지·엔티티가 영향을 받는가?
  - 출력: 어떤 산출물이 만들어져야 하는가? (코드, DDL, 테스트, `application-*.yml` 변경 등)
  - 제약: `open-in-view=false`, Stateless JWT, permitAll 허용 경로, afterCommit 이벤트 등 본 저장소의 불변조건.
  - 게이트: "완료" 판단 기준 — `./gradlew test`, `./gradlew compileJava` 성공 등.
- **서술형 산문보다 체크리스트, 스키마, 예시를 선호한다.**
  - "~를 고려하여 잘 처리한다" 같은 모호한 문장 대신:
    ```
    [ ] SequenceGenerator 선언
    [ ] ErrorCode enum 추가
    [ ] @TransactionalEventListener(phase = AFTER_COMMIT) 리스너 연결
    ```
- **파일 경로와 명령어 예시를 구체적으로 유지한다.**
  - `src/main/java/com/{company}/{app}/product/service/ProductService.java:42` 처럼 경로·라인을 명시.
  - 명령은 복붙 가능한 형태로: `./gradlew test --tests com.{company}.{app}.product.service.ProductServiceTest.save_중복이면_예외`.
- **근거는 짧은 글머리표로, 실행 단계는 결정적(deterministic) 순서**로 작성한다.
  - 근거: "왜 이 규칙이 필요한가" → 한두 줄 bullet.
  - 실행 단계: 1 → 2 → 3 숫자 순서. 사이클이 있으면 "X 가 실패하면 Y 로 돌아간다" 로 명시.
- 에이전트가 **사람에게 설명한다고 가정한다.** 그 반대가 아니다. 즉, 문서는 사람이 에이전트를 가르치는 목적이 아니라, 에이전트가 새 컨텍스트에서도 정확하게 재현할 수 있도록 **자기 완결적**으로 기술한다.

---

## 에이전트 참조 맵 (우선순위 순)

1. `CLAUDE.md` — 프로젝트 요약, 빌드·실행 방법, 아키텍처 개요, 보안 정책.
2. `.claude/conventions/architecture.md` — 패키지·레이어 지도.
3. `.claude/conventions/spring-boot.md` — 명명·레이어·API 규칙.
4. `.claude/conventions/library-stack.md` — 스택·의존성·설정.
5. `.claude/rules/backend-conventions.md` — 규칙 전용 (ResponseEntity 반환, 예외 처리, 레이어 책임 등).
6. `.claude/rules/backend-java-patterns.md` — 실제 코드 템플릿 (product 도메인 기준).
7. `.claude/rules/backend-design-patterns.md` — 패턴 상세.
8. `.claude/principles/*.md` — 엄격 규칙 (보안·에러·DB·완전성·코드스타일·프로젝트구조·search-before-build·agent-docs·api-conventions).

---

## PR / 작업 노트 템플릿 (권장)

```
## 변경 요약
- (한두 줄, 왜 바꾸는가)

## 영향 범위
- 코드: <패키지/파일>
- DDL: <스키마 변경 파일>
- 보안: <SecurityConfig 변경 여부>

## 검증
- [ ] ./gradlew test --tests <targets>
- [ ] ./gradlew bootJar (로컬 빌드 성공)
- [ ] 통합 테스트 또는 로컬 기동 시 관찰한 로그
```
