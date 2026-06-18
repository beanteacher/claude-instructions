# 6. CHECKPOINT — 커밋과 상태 영속화

> 검증을 통과한 작업을 **커밋으로 굳히고, 진행 상태를 외부에 저장** 하는 단계.
> 핵심 원칙: 컨텍스트가 초기화돼도 **재개(resume)** 할 수 있어야 한다.

[← 05-verify](./05-verify.md) · [개요](./README.md) · [다음: 07-finalize →](./07-finalize.md)

---

## 이 단계의 목표

| 입력 | 행동 | 산출물 | 가드 |
|------|------|--------|------|
| VERIFY 통과한 작업 | 커밋 + 진행 상태 갱신 | 커밋 해시 + 갱신된 상태 파일 | 빌드 깨진 상태로 커밋 금지 |

---

## HOW 1 — 커밋 단위 = 작업 카드 단위

한 작업 카드([01-plan](./01-plan.md))가 한 커밋이 되는 게 이상적이다.

```
✅ 작업 단위 커밋
   test: OrderService 취소 상태전이 테스트 추가
   feat: OrderService 취소 상태전이 구현

❌ 뭉친 커밋
   feat: 주문 취소 전부 + 리팩터 + 포맷팅
```

- **타입이 다르면 커밋을 나눈다** (`test:` / `feat:` / `refactor:`).
- 보안·스키마 변경은 **별도 커밋** 으로 분리해 리뷰 포인트를 드러낸다.
- TDD 라면 "테스트 추가" 와 "구현" 을 나눠도 좋다 (RED→GREEN 추적성).

---

## HOW 2 — 커밋 메시지 규칙 (범용)

```
type: subject

[why — 무엇보다 왜]

[footer — 요구사항/이슈 ID]
```

| 타입 | 용도 |
|------|------|
| `feat` | 새 기능 |
| `fix` | 버그 수정 |
| `refactor` | 동작 불변, 구조 개선 |
| `test` | 테스트 추가/수정 |
| `perf` | 성능 |
| `docs` / `chore` / `style` / `ci` / `build` | 부수 |

- subject 는 **무엇을** 짧게, body 는 **왜** 를 설명.
- `WIP`, `update`, `fix typo` 같은 무의미 메시지 금지.
- 프로젝트가 scope 를 안 쓰면 `feat(api):` 같은 임의 scope 붙이지 않는다.

---

## HOW 3 — 진행 상태 영속화 (재개 핵심)

루프가 중단돼도 이어가려면 상태를 구조화해 외부(파일/DB)에 저장한다.

```json
{
  "goal": "주문 취소 API",
  "definition_of_done": [
    "PENDING 취소 시 CANCELED",
    "PAID 취소 시 409",
    "타인 주문 403"
  ],
  "tasks": [
    { "id": 1, "title": "OrderRepositoryTest", "status": "done",        "commit": "a1b2c3d" },
    { "id": 2, "title": "OrderRepository",      "status": "done",        "commit": "d4e5f6a" },
    { "id": 3, "title": "OrderServiceTest",     "status": "done",        "commit": "b7c8d9e" },
    { "id": 4, "title": "OrderService",         "status": "in_progress", "attempts": 1 },
    { "id": 5, "title": "OrderControllerTest",  "status": "pending" },
    { "id": 6, "title": "OrderController+DTO",  "status": "pending" }
  ],
  "current_phase": "ACT",
  "last_checkpoint_commit": "b7c8d9e",
  "updated_at": "2026-06-17T14:40:00+09:00"
}
```

상태에 꼭 담을 것:
- **DoD** (재개 시 목표 복원)
- **작업별 status + 커밋 해시** (어디까지 굳었나)
- **in_progress 작업의 attempts** (재시도 누적 유지 → 무한 루프 방지 연속성)
- **current_phase** (어느 단계에서 끊겼나)

---

## HOW 4 — 재개(resume) 절차

```
1. 상태 파일을 읽는다.
2. status == done 인 작업은 건너뛴다 (이미 커밋됨).
3. in_progress 작업부터 재개 — attempts 를 이어받는다.
4. last_checkpoint_commit 과 작업 트리가 일치하는지 확인
   (불일치 시: 미커밋 변경이 있는가? 손상되지 않았나 점검).
5. current_phase 에 맞는 단계 문서로 진입.
```

> 재개 가능성이 곧 **회복탄력성**이다. 세션이 끊겨도, 컨텍스트가 압축돼도 손실이 "마지막 체크포인트 이후" 로 제한된다.

---

## HOW 5 — 체크포인트 직전 가드

커밋하기 전에 반드시:

- [ ] 빌드/대상 테스트가 GREEN 인가 ([05-verify](./05-verify.md) 통과)
- [ ] 무관한 변경이 섞이지 않았는가 (`diff` 확인)
- [ ] 시크릿/임시 디버그 코드/주석 처리된 코드가 안 들어갔는가
- [ ] 빌드 산출물·생성 파일이 실수로 커밋되지 않는가 (ignore 대상)
- [ ] 커밋 메시지 타입이 변경 성격과 맞는가

---

## 안티패턴

| 안티패턴 | 결과 | 대신 |
|----------|------|------|
| 깨진 상태로 커밋 | 재개 시 깨진 지점부터 | VERIFY 통과 후에만 |
| 거대 뭉친 커밋 | 리뷰·롤백·이분탐색 곤란 | 작업/타입 단위 분리 |
| 상태 영속화 안 함 | 중단 = 처음부터 | 진행 상태 파일 유지 |
| attempts 리셋 | 무한 루프 방지 무력화 | attempts 누적 보존 |
| 시크릿/디버그 코드 커밋 | 보안·노이즈 | 커밋 전 diff 점검 |
