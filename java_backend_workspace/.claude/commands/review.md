---
name: review
description: "Spring Boot 백엔드 PR/변경 코드 리뷰. 심각도 우선, principles/rules 위반 우선 탐지."
argument-hint: "[optional: PR 번호, 파일 경로, 도메인(product 등), 또는 변경 범위]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

<objective>
Spring Boot 백엔드 저장소의 변경 사항을
**스타일 체크리스트가 아니라 리스크 탐지 워크플로우**로 리뷰한다.
`.claude/principles/*` 위반은 머지 반려 근거가 된다.
</objective>

<context>
Scope: $ARGUMENTS

References:
- @CLAUDE.md
- @.claude/skills/code-review-skill.md
- @.claude/skills/build-and-verify.md
- @.claude/rules/backend-conventions.md
- @.claude/rules/backend-java-patterns.md
- @.claude/rules/backend-design-patterns.md
- @.claude/principles/security.md
- @.claude/principles/database.md
- @.claude/principles/error-handling.md
- @.claude/principles/api-conventions.md
- @.claude/principles/project-structure.md
- @.claude/principles/code-style.md
- @.claude/principles/completeness-principle.md
- @.claude/principles/search-before-build.md
</context>

<process>
1. 리뷰 범위 결정
   - $ARGUMENTS 가 PR 번호이면 `gh pr view {N}` / `gh pr diff {N}` 으로 변경 사항 수집
   - 파일 경로 / 도메인 / 모듈이 주어지면 해당 영역만 대상
   - 인자 없으면 staged + unstaged 변경 (`git diff --stat`, `git diff main...HEAD`)

2. 영향 범위 분류 — 변경 파일을 다음 축으로 매핑
   - 도메인: 프로젝트 도메인 패키지 (예: product / order / user / auth / global)
   - 레이어: controller / service / repository / mapper / domain / dto
   - 인프라: global/security (SecurityConfig, JWT 필터), global/config (WebClient, Redis, QueryDSL)
   - 스키마: 프로젝트 스키마 파일들
   - 배포 파이프라인: application*.yml · build.gradle · CI/CD 설정

3. 2-Pass 리뷰 (`code-review-skill.md` 체크리스트 기반)
   - **Pass 1 — 머지 차단 (principles)**
     - 보안 (`security.md`): 시크릿 하드코딩, permitAll 추가, webhook 서명 검증
     - DB (`database.md`): ID 생성 전략, additive 스키마, 스키마 파일 동기화
     - 예외 (`error-handling.md`): `AppException` + `ErrorCode` + `GlobalExceptionHandler` 통일
     - API (`api-conventions.md`): URL 규칙, 응답 envelope 일관성, `ResponseEntity` 관행
     - 구조 (`project-structure.md`): 패키지 경계, 의존 방향
     - 완결성 (`completeness-principle.md`): DDL 동기화, permitAll, 부수효과 처리 누락 여부
   - **Pass 2 — 일반 규칙 (rules)**
     - `backend-conventions.md` (레이어 책임, 트랜잭션, DTO record, ResponseEntity 관행)
     - `backend-java-patterns.md` (product 도메인 기준 템플릿)
     - `backend-design-patterns.md` (Strategy/Factory/Template Method/Observer/Repository Custom)
     - 테스트: Repository → Service → Controller TDD, H2 인메모리 DB

4. 병렬 검토 패스 (필요 시 Task 위임)
   - correctness / regression risk
   - security & data handling (JWT 회전, Redis, 입력 바인딩)
   - test adequacy & observability (TDD 순서, 회귀 테스트, 로그)
   - 배포 파이프라인 영향 (afterCommit 이벤트, 외부 시스템 연동)

5. fix-first 휴리스틱
   - 단순/저위험 발견 (오타, import, 포매팅, 미사용 코드) → AUTO-FIXED
   - 모호/고위험 발견 (보안 필터, 스키마, 트랜잭션 경계, 외부 파이프라인) → NEEDS INPUT (수정하지 않음)

6. 결과 산출
   - `code-review-skill.md` 의 "리뷰 템플릿" 형식으로 작성
   - 발견사항을 심각도 순서로 정렬
     - 🔴 blocker (Must Fix) — principles 위반 / 데이터·보안 리스크
     - 🟡 warning (Should Fix) — rules 위반 / 설계 결함
     - 🟢 info (Consider) — 개선 제안
   - 각 항목에 다음을 포함
     - 파일/라인 (`src/main/java/.../X.java:NN`)
     - 근거 문서 인용 (`.claude/principles/...` 또는 `.claude/rules/...`)
     - 실패 모드(왜 문제인가) + 구체 수정 방향
   - 발견사항이 없으면 그 사실을 명시하고 잔존 리스크를 짧게 나열

7. 보고서 출력 위치
   - PR 리뷰: `gh pr comment {N} --body "$(cat review.md)"` 사용 시 본문은 채팅으로도 함께 출력
   - 로컬 변경 리뷰: 채팅으로 직접 출력 (별도 파일 저장은 사용자가 요청한 경우에만)
   - 승인(`gh pr review --approve`) / 변경 요청(`--request-changes`) 은 **반드시 사용자 확인 후** 실행

8. 마무리 점검 (사용자에게 묻기 전에 셀프체크)
   - 스키마 변경이면 관련 파일이 함께 수정되었는가
   - SecurityConfig permitAll 이 추가/수정되었으면 사유와 영향 범위가 적혔는가
   - afterCommit 이벤트 / 외부 시스템 연동에 영향이 있는가
   - 빌드 검증 명령(`./gradlew compileJava`, `./gradlew test`, `./gradlew bootJar`) 이 PR 본문/머지 체크리스트에 표기되었는가
</process>
