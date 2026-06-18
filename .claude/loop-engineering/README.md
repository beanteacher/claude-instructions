# Loop Engineering — AI 에이전트 작업 사이클 설계 가이드

> **목적**: 백엔드 프로젝트를 AI 에이전트로 개발할 때 쓰는 **범용** 루프 설계 문서 세트.
> 특정 언어·프레임워크에 종속되지 않는다. (예시는 Spring Boot / TDD 맥락을 인용하지만 다른 스택으로 그대로 치환 가능)

**Loop Engineering 이란**: AI 에이전트가 `관찰 → 계획 → 행동 → 검증 → 반복`하는 핵심 사이클을 의도적으로 설계·튜닝하는 작업. 루프의 품질이 결과물의 품질을 결정한다.

---

## 문서 구성

이 가이드는 **개요(이 문서) + 단계별 상세 8개**로 구성된다. 각 단계 문서는 "어떻게(HOW)" 를 기법·체크리스트·템플릿·예시·안티패턴으로 다룬다.

| # | 문서 | 단계 | 한 줄 요약 |
|---|------|------|-----------|
| 0 | [00-intake.md](./00-intake.md) | INTAKE | 요구사항을 받아 **완료의 정의(DoD)** 로 변환 |
| 1 | [01-plan.md](./01-plan.md) | PLAN | DoD 를 **작업 단위로 분해** + 의존성 정렬 |
| 2 | [02-context.md](./02-context.md) | CONTEXT | 코드 · 규칙 · 기존 자산 **탐색 (만들기 전에 검색)** |
| 3 | [03-act.md](./03-act.md) | ACT | **한 단위**만 실행 (TDD: RED → 최소 구현) |
| 4 | [04-observe.md](./04-observe.md) | OBSERVE | 빌드 · 테스트 · 로그 **실제 출력 수집** |
| 5 | [05-verify.md](./05-verify.md) | VERIFY | **다중 게이트** 검증 + 재시도/에스컬레이션 판단 |
| 6 | [06-checkpoint.md](./06-checkpoint.md) | CHECKPOINT | 커밋 + **상태 영속화 (재개 가능하게)** |
| 7 | [07-finalize.md](./07-finalize.md) | FINALIZE | 통합 검증 + **증거 기반 보고** |

---

## 핵심 철학

| 원칙 | 의미 |
|------|------|
| **검증 기반(Evidence-driven)** | "됐다"는 추측 금지. 실제 빌드·테스트 출력으로만 완료를 판정. |
| **작은 단위(Small steps)** | 한 사이클 = 한 기능. 여러 작업 몰아치기 금지. |
| **상태 영속화(Persistence)** | 컨텍스트가 초기화돼도 진행 상태를 외부에 저장해 이어서 작업. |
| **종료 가능(Terminable)** | 모든 루프는 명확한 종료 조건과 탈출구(에스컬레이션)를 가진다. |
| **재사용 우선(Search first)** | 만들기 전에 기존 자산을 검색. 중복 생성 금지. |

---

## 전체 루프 구조 (2단 루프)

```
┌──────────────────────────────────────────────────────────────┐
│  0. INTAKE       요구사항 수집 · 완료 정의(DoD) 확정          │
│       ↓                                                        │
│  1. PLAN         작업 분해 + 의존성 정렬                      │
│       ↓                                                        │
│  ┌─→ 2. CONTEXT  관련 코드 · 규칙 · 기존 자산 탐색            │
│  │     ↓                                                       │
│  │  ┌→ 3. ACT    한 단위 작업 실행                            │
│  │  │    ↓                                                     │
│  │  │  4. OBSERVE 컴파일 · 테스트 · 로그 결과 수집            │ ← 내부 루프
│  │  │    ↓                                                     │   (GREEN 까지 회전)
│  │  │  5. VERIFY  목표 충족? 규칙 위반? ──No──┐               │
│  │  │    ↓ Yes                                 │               │
│  │  └─── 재시도/수정 ←────────────────────────┘               │
│  │       ↓ (MAX_RETRY 초과 시 → ESCALATE)                     │
│  │     6. CHECKPOINT  커밋 · 상태 저장                        │ ← 외부 루프
│  │       ↓                                                     │   (작업 단위)
│  └────── 남은 작업 있음? ──Yes                                │
│          ↓ No                                                  │
│  7. FINALIZE     최종 통합 검증 + 증거 기반 보고             │
└──────────────────────────────────────────────────────────────┘
```

- **내부 루프 (3→4→5)**: 한 작업을 GREEN 으로 만들 때까지 빠르게 회전. **자기수정(self-correction)** 이 일어나는 곳.
- **외부 루프 (2→6)**: 작업 단위로 체크포인트를 찍고 다음 작업으로 진행.

---

## 반드시 넣어야 할 안전장치 4종

| 장치 | 막는 문제 | 구현 위치 |
|------|-----------|-----------|
| **종료 조건** | 무한 루프 | [05-verify](./05-verify.md) — MAX_RETRY + DoD 충족 |
| **상태 영속화** | 컨텍스트 손실 | [06-checkpoint](./06-checkpoint.md) — 진행 상태 파일 |
| **검증 게이트** | 거짓 완료 | [05-verify](./05-verify.md) — 실제 테스트로만 판정 |
| **에스컬레이션** | 무한 재시도 | [05-verify](./05-verify.md) — N회 실패 시 사람에게 |

---

## 의사코드 골격

```python
MAX_RETRY = 3

def agent_loop(request):
    dod   = intake(request)            # 0. 완료 정의 확정     → 00-intake.md
    tasks = plan(request, dod)         # 1. 작업 분해           → 01-plan.md

    for task in tasks:                 # ── 외부 루프 (작업 단위) ──
        context  = explore(task)       # 2. 탐색 우선           → 02-context.md
        attempts = 0

        while attempts < MAX_RETRY:    # ── 내부 루프 (GREEN 까지) ──
            result = act(task, context)            # 3. 실행    → 03-act.md
            obs    = observe(result)               # 4. 관찰    → 04-observe.md
            ok, reason = verify(obs, task.dod)     # 5. 검증    → 05-verify.md
            if ok:
                break
            attempts += 1
            context = refine(context, reason)      # 자기수정
        else:
            escalate(task, reason)     # 안전장치: N회 실패 → 사람에게
            return report_blocked(task)

        checkpoint(task)               # 6. 커밋 + 상태 저장    → 06-checkpoint.md

    return finalize(dod)               # 7. 통합 검증 + 보고    → 07-finalize.md
```

---

## 다른 스택으로 치환

루프 구조 자체는 동일하고 **OBSERVE 의 실행 명령만** 바뀐다.

| 단계 | Spring Boot | Node/TS | Go |
|------|-------------|---------|-----|
| 컴파일 | `./gradlew compileJava` | `tsc --noEmit` | `go build ./...` |
| 단위 테스트 | `./gradlew test --tests X` | `vitest run X` | `go test ./pkg -run X` |
| 빌드 산출물 | `./gradlew bootJar` | `npm run build` | `go build -o bin/app` |
| 린트 | checkstyle / spotless | `eslint .` | `golangci-lint run` |

---

## 한 줄 요약

> **좋은 에이전트 루프 = 작은 단위로 행동하고(ACT), 실제로 검증하고(VERIFY), 진행을 저장하고(CHECKPOINT), 막히면 멈출 줄 안다(ESCALATE).**
