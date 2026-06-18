# 4. OBSERVE — 결과 수집

> 방금 한 행동(ACT)의 **실제 결과를 명령으로 실행해 출력을 읽는** 단계.
> 핵심 원칙: **추측 금지.** "통과했을 것이다" 가 아니라 "출력에 이렇게 찍혔다".

[← 03-act](./03-act.md) · [개요](./README.md) · [다음: 05-verify →](./05-verify.md)

---

## 이 단계의 목표

| 입력 | 행동 | 산출물 | 가드 |
|------|------|--------|------|
| 변경된 코드 | 검증 명령 실행 + 출력 파싱 | 구조화된 관찰 결과 | 출력을 읽지 않고 성공 가정 금지 |

---

## HOW 1 — 무엇을 실행하는가 (비용 순서대로)

값싼 것부터 돌려 빠르게 실패를 잡는다.

```
① 컴파일/타입체크   가장 값쌈, 가장 흔한 실패
       ./gradlew compileJava   |  tsc --noEmit  |  go build ./...

② 해당 단위 테스트   지금 작업한 그 테스트만 (전체 X)
       ./gradlew test --tests *.OrderServiceTest

③ 인접 회귀 테스트   영향 범위(02-context) 의 테스트
       (시그니처를 바꿨다면 그 호출처 테스트)

④ 린트/정적분석     (선택) 스타일·취약점

⑤ 전체 테스트/빌드   FINALIZE 에서. 매 사이클마다는 비쌈
```

> 매 내부 루프에서 ①②만 빠르게, 작업이 닫힐 때 ③, 전체는 [07-finalize](./07-finalize.md) 에서.

---

## HOW 2 — 출력을 신호로 분류한다

raw 로그를 그대로 두지 말고 분류한다.

| 신호 | 의미 | 다음 행동 |
|------|------|-----------|
| 컴파일 에러 | 문법·타입·시그니처 불일치 | 즉시 ACT 로 (가장 값싼 수정) |
| 테스트 실패 (assert) | 로직이 기대와 다름 | 원인 분석 → ACT |
| 테스트 실패 (exception) | 예상 못 한 NPE/설정 오류 | 스택트레이스 최상단 프레임부터 |
| 테스트 통과 | DoD 한 항목 충족 가능성 | VERIFY 로 |
| RED 가 안 뜸 (첫 실행 통과) | **테스트가 잘못됨** | 테스트를 의심하고 고친다 |
| 경고(warning) | 당장 실패 아님 | 기록만, 작업 닫을 때 판단 |

---

## HOW 3 — 실패 출력 읽는 법

```
1. 마지막 실패 요약부터 본다
   "Tests: 1 failed, 4 passed" → 어떤 테스트가?

2. assert 메시지: expected vs actual 을 본다
   expected: <CANCELED> but was: <PENDING>
   → 상태 전이가 안 일어남. Service 로직 누락 의심.

3. exception 이면 스택트레이스 "내 코드" 프레임을 찾는다
   프레임워크 내부 프레임은 건너뛰고 OrderService.java:42 같은 줄로.

4. 로그 노이즈 제거
   - 같은 에러 반복은 1개로 본다
   - 무관한 INFO/DEBUG 는 무시
   - "Caused by:" 체인의 최하단 근본 원인을 본다
```

---

## HOW 4 — RED 확인의 중요성 (TDD)

새 테스트는 **반드시 한 번 실패하는 걸 봐야** 한다.

```
테스트 작성 → 실행 → 실패 확인 ✅  (이 테스트가 "무언가를 진짜로 검증" 한다는 증거)
                  → 통과 ❌       (구현이 없는데 통과? = 테스트가 헛돌고 있음)
```

첫 실행 통과 시 점검:
- assert 가 실제로 결과를 보고 있나? (`assertTrue(true)` 같은 헛 assert 아닌가)
- 테스트가 실제 대상 메서드를 호출하나?
- mock 이 과해서 정작 로직을 안 타는 건 아닌가?

---

## 산출물 — 구조화된 관찰 결과

VERIFY 로 넘기는 형태:

```yaml
task: "OrderService 취소 상태전이"
ran:
  - cmd: "./gradlew compileJava"
    result: OK
  - cmd: "./gradlew test --tests *.OrderServiceTest"
    result: "1 failed, 4 passed"
failure:
  test: "취소_PAID주문이면_예외"
  expected: "WiseException(ORDER_NOT_CANCELABLE)"
  actual:   "no exception thrown"
  hint: "Service 에 PAID 상태 가드 분기 누락"
```

---

## 안티패턴

| 안티패턴 | 결과 | 대신 |
|----------|------|------|
| 명령 안 돌리고 "됐다" | 거짓 완료 | 실제 실행 + 출력 확인 |
| 매 사이클 전체 테스트 | 느려서 루프가 안 돈다 | 단위 먼저, 전체는 마지막 |
| 로그 안 읽고 재시도 | 같은 실패 반복 | expected/actual·근본원인 파악 |
| RED 생략 | 헛도는 테스트 | 실패를 먼저 눈으로 확인 |
| 스택트레이스 맨 위만 봄 | 진짜 원인 놓침 | "Caused by" 최하단까지 |
