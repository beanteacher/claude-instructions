# 프로젝트 구조 (Iron Rule)

- **기존 모듈 경계를 존중한다.** 저장소의 패키지 구조(`com.{company}.{app}.<domain>.<layer>`)는 이미 안정화되어 있다. 도메인 간 경계를 넘어 코드를 뒤섞지 않는다.
- 새 파일은 **가장 가까운 기능 도메인 근처**에 배치한다. 기존 도메인 폴더가 있으면 새 패키지를 만들지 않는다.
- **강력한 근거 없이 최상위 패키지를 새로 만들지 않는다.** 도메인 목록은 완만하게만 확장한다. 추가가 꼭 필요하면 PR 에 사유를 남긴다.
- **아키텍처 맵과 일관된 의존성 방향을 유지한다.**
  - `controller → service → repository → domain`
  - 모든 레이어는 `global/*` (공통 모듈)을 참조 가능.
  - 엔티티는 컨트롤러 경계를 넘지 않는다 (DTO + MapStruct).
  - 같은 레이어 간 가로지르기(`order/service` 가 `product/service` 를 호출) 는 허용되지만, **역방향 호출**은 이벤트로 우회한다.
- **구조가 불명확한 경우** 다음 순서로 1차 참조:
  1. `.claude/rules/backend-conventions.md` — 레이어 책임·명명 규칙
  2. `.claude/rules/backend-java-patterns.md` — 코드 템플릿
  3. `.claude/rules/backend-design-patterns.md` — 패턴 상세
  4. 기존 도메인(예: `product/`) 의 실제 코드
- 도메인 내부 서브패키지는 아래 집합으로 제한한다. 새 서브패키지를 만들고 싶으면 기존 구조로 소화 가능한지 먼저 확인한다.
  ```
  controller / service / repository / mapper / domain / dto
  (+ 필요 시 event / listener)
  ```
- 도메인이 커져 3개 이상 하위 개념으로 나뉘는 경우(예: `order` → `item / payment / delivery`) **하위 슬라이스마다 동일한 서브패키지를 반복**한다. 도메인 최상위에 layer 폴더를 두고 그 아래 도메인을 섞는 역구조는 금지한다.
