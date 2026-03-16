---
name: verify-implementation
description: 등록된 모든 verify 스킬을 Subagent로 병렬 실행하여 통합 검증 리포트를 생성합니다. Phase별 또는 전체 검증을 지원하고, 이슈 발견 시 자동 수정 옵션을 제공합니다. 기능 구현 후, PR 제출 전, 코드 리뷰 중, 품질 점검 시 반드시 이 스킬을 사용하세요. 키워드: verify, 검증, validation, check, quality, 품질, test, gate, 확인.
argument-hint: "[선택: phase-1, phase-2, ... 또는 --current-phase]"
---

# Verify Implementation

## Purpose

등록된 모든 `verify-*` 스킬을 실행하여 통합 검증 리포트를 생성한다:

1. **병렬 검증** — 독립적인 verify 스킬은 Subagent로 동시 실행하여 검증 시간 대폭 단축
2. **Phase-aware** — 특정 Phase의 스킬만 선택적으로 실행 가능
3. **통합 리포트** — 모든 결과를 하나의 리포트로 취합
4. **자동 수정** — 이슈 발견 시 auto-fix / 개별 리뷰 / 건너뛰기 옵션
5. **재검증** — 수정 후 Before/After 비교
6. **계획 문서 연동** — 검증 결과를 `docs/plans/PLAN_*.md`의 Verification Status에 자동 반영

## Execution Timing

- 기능 구현 완료 후
- PR 제출 전
- 코드 리뷰 중
- Quality Gate 검증 시
- Phase 완료 확인 시

## Target Skills

실행 대상 스킬 목록은 **`manage-skills/SKILL.md`의 Registered Verify Skills 테이블**을 단일 소스(SSOT)로 참조한다.

**스킬 로딩 절차:**
1. `.claude/skills/manage-skills/SKILL.md`를 읽는다
2. "Registered Verify Skills" 섹션의 테이블을 파싱한다
3. 라이프사이클이 ACTIVE 또는 CREATED인 스킬만 실행 대상에 포함한다
4. ARCHIVED 스킬은 제외한다

**SSOT를 읽을 수 없는 경우 (파일 없음/파싱 실패):**
폴백으로 파일 시스템에서 직접 스캔한다:
```bash
ls .claude/skills/verify-*/SKILL.md 2>/dev/null
```

## Workflow

### Step 1: Argument Parsing & Skill Selection

인수를 파싱하여 실행 대상을 결정한다:

```
인수 없음                    → 위 테이블의 모든 스킬 실행
"phase-N"                    → verify-phase-N-* 패턴에 매칭되는 스킬만 실행
"--current-phase"            → docs/plans/PLAN_*.md에서 Status가 🔄 In Progress인 Phase를 찾아
                               해당 Phase의 verify 스킬만 실행 (복수 PLAN 지원 — 아래 참조)
"PLAN_auth"                  → 특정 PLAN의 전체 verify 스킬만 실행
"PLAN_auth phase-2"          → 특정 PLAN의 특정 Phase verify 스킬만 실행
```

#### 복수 PLAN 동시 진행 처리

`--current-phase`를 사용할 때 여러 PLAN이 동시에 🔄 In Progress 상태일 수 있다:

```bash
# 진행 중인 PLAN 찾기
grep -l "🔄 In Progress" docs/plans/PLAN_*.md 2>/dev/null
```

**단일 PLAN인 경우:** 해당 PLAN에서 In Progress Phase의 verify 스킬을 실행한다.

**복수 PLAN인 경우:** AskUserQuestion으로 사용자에게 선택을 요청한다:

```markdown
현재 진행 중인 PLAN이 2개입니다. 어떤 PLAN을 검증할까요?

1. **PLAN_auth** — Phase 2 (API 엔드포인트) 진행 중
2. **PLAN_search** — Phase 1 (인덱싱) 진행 중
3. **모두 검증** — 두 PLAN의 진행 중인 Phase를 모두 검증
```

"모두 검증" 선택 시 각 PLAN의 Phase 스킬을 모두 수집하여 병렬 실행한다. 통합 리포트에서는 PLAN별로 그룹화하여 표시한다:

```markdown
### 요약

#### PLAN_auth (Phase 2)
| 스킬 | 결과 | 이슈 수 |
|------|------|---------|
| verify-phase-2-auth-api | ✅ PASS | 0 |

#### PLAN_search (Phase 1)
| 스킬 | 결과 | 이슈 수 |
|------|------|---------|
| verify-phase-1-search-index | ❌ FAIL | 1 |
```

**등록된 스킬이 0개인 경우:**

에러 대신 가이드 메시지를 표시한다:

```markdown
## 검증 스킬이 아직 없습니다

다음 방법으로 검증 스킬을 생성할 수 있습니다:

1. `/feature-planner`로 기능 계획을 수립하면 Phase별 verify 스킬이 자동 생성됩니다
2. `/manage-skills`를 실행하면 프로젝트 분석 후 기본 verify 스킬을 제안합니다
```

실행 대상 스킬 목록을 사용자에게 표시한다:

```markdown
## 검증 대상

| # | 스킬 | 설명 | 모드 |
|---|------|------|------|
| 1 | verify-phase-1-models | Phase 1 모델 검증 | 병렬 |
| 2 | verify-phase-2-logic | Phase 2 로직 검증 | 병렬 |
| 3 | verify-build | 빌드 검증 | 병렬 |

**실행 모드**: 3개 스킬 병렬 실행
```

### Step 2: Dependency Analysis & Parallel Execution

실행 대상 스킬을 **독립 그룹**과 **의존 체인**으로 분류한다.

**독립 스킬 판별 기준:**
- 다른 verify 스킬의 결과를 참조하지 않는 스킬 → 독립
- 특정 Phase의 PASS를 전제하는 스킬 → 의존

**병렬 실행:**

독립 스킬 각각에 대해 **`test-runner`** Subagent를 실행한다. 독립 스킬은 **동시에** 실행하여 검증 시간을 대폭 단축한다.

> **사용하는 Subagent**: `.claude/agents/test-runner.md`

각 `test-runner`에게 다음 정보를 전달한다:
1. 해당 verify 스킬의 SKILL.md 경로
2. 프로젝트 루트 경로
3. **TDD 검증 모드 활성화** — Phase 스킬인 경우 (`verify-phase-N-*` 패턴), Phase 시작 시점의 git ref를 함께 전달하여 TDD Compliance Check를 수행하도록 한다

Subagent가 스킬을 읽고, Workflow의 모든 검사를 수행하여, 각 검사의 PASS/FAIL/EXEMPT 상태와 상세 내용을 반환한다. Phase 스킬인 경우 추가로 TDD 순서 검증 결과도 반환한다.

이것이 기존 순차 실행 대비 핵심 개선점이다. 5개 스킬이 있으면 5배 빠르게 끝난다.

**스킬이 7개 초과인 경우**, 7개씩 배치로 나눠 실행한다.

**의존 스킬**은 선행 스킬의 결과를 확인한 후 순차 실행한다.

**test-runner 실패/타임아웃 처리:**

N개의 `test-runner` 중 일부가 실패(에러 반환, 응답 없음, 60초 내 미응답)한 경우:
1. 성공한 스킬의 결과는 그대로 유지한다
2. 실패한 스킬만 순차 재시도 1회 실행한다
3. 재실패 시 해당 스킬을 `⚠️ AGENT_FAILURE` 상태로 표시하고, 나머지 스킬 결과로 PASS/FAIL을 판정한다

통합 리포트에 표시:

```markdown
| 스킬 | 결과 | 이슈 수 | 소요 시간 |
|------|------|---------|----------|
| verify-phase-1-models | ✅ PASS | 0 | 5s |
| verify-phase-2-logic | ⚠️ AGENT_FAILURE | — | timeout |

**주의**: 1개 스킬의 검증이 완료되지 않았습니다. 해당 스킬을 직접 확인하거나 재실행하세요.
```

AGENT_FAILURE 스킬은 PASS/FAIL 집계에서 제외한다. 전체 스킬이 AGENT_FAILURE이면 "검증 불가 — 재실행 필요"를 보고한다.

### Step 3: Integrated Report

모든 스킬 실행이 완료되면 통합 리포트를 생성한다:

```markdown
## 통합 검증 리포트

**실행 시각**: 2026-03-16 14:30
**대상 스킬**: 5개 (병렬 4 + 순차 1)
**소요 시간**: 23초

### 요약

| 스킬 | 결과 | 이슈 수 | 소요 시간 |
|------|------|---------|----------|
| verify-phase-1-models | ✅ PASS | 0 | 5s |
| verify-phase-2-logic | ❌ FAIL | 2 | 8s |
| verify-build | ✅ PASS | 0 | 12s |
| verify-lint | ⚠️ WARN | 1 | 3s |
| verify-integration | ✅ PASS | 0 | 15s |

**전체 결과**: 3 PASS / 1 FAIL / 1 WARN

### TDD Compliance (Phase 스킬에 대해서만)

| Phase | Test-First 순서 | R-G-R 증거 | REFACTOR 측정 | 신규 코드 커버리지 |
|-------|----------------|------------|--------------|-------------------|
| Phase 1 | ✅ 5/5 파일 | ✅ DETECTED | ✅ 평균 -18% 길이 | 87% (목표 80%) ✅ |
| Phase 2 | ⚠️ 3/5 파일 | ✅ DETECTED | ⚠️ REFACTOR 미수행 | 78% (목표 80%) ❌ |

**TDD 위반 상세:**
- `src/services/payment.ts`: 소스가 테스트보다 2커밋 먼저 (커밋 abc123)
- `src/utils/validator.ts`: 대응 테스트 파일 없음

### 이슈 상세

(각 FAIL/WARN 스킬의 이슈를 상세히 기술)
```

### Step 4: User Action Confirmation

이슈가 발견된 경우, AskUserQuestion을 사용하여 사용자에게 액션을 확인한다:

선택지:
1. **전체 자동 수정** — 모든 이슈를 자동으로 수정
2. **개별 리뷰** — 각 이슈를 하나씩 확인하며 수정/스킵 결정
3. **건너뛰기** — 이번에는 수정하지 않음

### Step 5: Fix Application

사용자가 선택한 방식에 따라 수정을 적용한다:
- 전체 자동 수정: 모든 이슈에 대해 권장 수정 방법을 적용
- 개별 리뷰: 각 이슈를 사용자에게 보여주고 수정/스킵 결정

### Step 6: Post-Fix Revalidation

수정이 적용된 경우, 영향받은 스킬만 **재실행**하여 Before/After를 비교한다:

```markdown
## 재검증 결과

| 스킬 | Before | After |
|------|--------|-------|
| verify-phase-2-logic | ❌ FAIL (2 issues) | ✅ PASS |
```

### Step 7: Plan Document Update

검증 결과를 대응하는 계획 문서에 자동 반영한다.

`docs/plans/PLAN_*.md`에서 **Verification Status** 섹션을 찾아 업데이트:

```markdown
## ✅ Verification Status (Auto-updated by verify-implementation)

| Phase | Verify Skill | Last Run | Result | Issues |
|-------|-------------|----------|--------|--------|
| Phase 1 | `verify-phase-1-models` | 2026-03-16 14:30 | ✅ PASS | 0 |
| Phase 2 | `verify-phase-2-logic` | 2026-03-16 14:30 | ✅ PASS (fixed) | 0 |
```

해당 Phase의 Quality Gate 체크박스도 검증 결과에 따라 업데이트한다.

**TDD Compliance 체크박스도 자동 업데이트:**
- Test-First 순서 검증 결과 → "Tests written FIRST and initially failed" 체크박스
- 신규 코드 커버리지 결과 → "Coverage meets requirements" 체크박스
- R-G-R 증거 결과 → "Code improved while tests still pass" 체크박스

### Step 8: Execution History & Skill Effectiveness

검증 실행 결과를 `.claude/verify-history.md`에 기록하여, 스킬 효과성을 추적한다.

**실행 이력 기록:**

```markdown
## Verify Execution History

| 날짜 | 스킬 | 결과 | 이슈 수 | 사용자 액션 | 소요 시간 |
|------|------|------|---------|------------|----------|
| 2026-03-16 | verify-phase-1-models | PASS | 0 | — | 5s |
| 2026-03-16 | verify-phase-2-logic | FAIL | 2 | auto-fix | 8s |
| 2026-03-17 | verify-phase-2-logic | PASS | 0 | — | 7s |
```

**스킬 효과성 분석:**

실행 이력이 5회 이상 쌓인 스킬에 대해 효과성 지표를 계산한다:

| 지표 | 계산 방법 | 의미 |
|------|----------|------|
| FAIL 비율 | FAIL 횟수 / 전체 실행 | 높으면 → 스킬이 실제 이슈를 잡고 있음 (유용) |
| 연속 PASS | 최근 N회 연속 PASS | 높으면 → 스킬이 더 이상 가치를 못 내고 있음 (GRADUATE 후보) |
| Skip 비율 | 사용자 Skip / FAIL 횟수 | 높으면 → false positive가 많음 (스킬 수정 필요) |
| 평균 이슈 수 | 총 이슈 / FAIL 횟수 | 낮으면 → 사소한 이슈만 잡음 |

**자동 제안:**

- 연속 PASS 10회 이상 → "이 스킬은 최근 이슈를 발견하지 못하고 있습니다. GRADUATE 또는 ARCHIVE를 고려하세요."
- Skip 비율 50% 이상 → "이 스킬의 false positive 비율이 높습니다. Exceptions 섹션 업데이트를 권장합니다."
- FAIL 비율 80% 이상 → "이 스킬이 거의 항상 FAIL합니다. 검사 기준이 너무 엄격하거나 근본적 코드 문제가 있을 수 있습니다."

이 분석은 통합 리포트의 마지막에 "스킬 건강 상태" 섹션으로 포함된다.

### Step 9: False Positive 자동 학습

`.claude/verify-history.md`의 실행 이력에서 사용자가 Skip한 항목을 분석하여, 반복적인 false positive에 대해 Exceptions 추가를 자동 제안한다.

**분석 절차:**

1. verify-history.md에서 최근 이력을 읽는다
2. 동일 스킬의 동일 검사 항목이 **3회 연속 Skip**된 경우를 식별한다
3. 해당 항목에 대해 자동으로 Exceptions 추가를 제안한다

```markdown
### 🔄 False Positive 학습 제안

다음 검사 항목이 반복적으로 Skip되었습니다. Exceptions에 추가할까요?

| 스킬 | 검사 항목 | 연속 Skip | 제안 |
|------|----------|----------|------|
| verify-auth | JWT 만료 시간 하드코딩 검사 | 5회 | 환경변수 방식도 허용하도록 Exceptions 추가 |
| verify-lint | console.log 사용 검사 | 3회 | debug 모듈 사용 시 면제하도록 Exceptions 추가 |
```

AskUserQuestion으로 사용자에게 확인한다:
1. **예, 모두 추가** — 제안된 모든 항목을 각 스킬의 Exceptions에 추가
2. **개별 선택** — 각 항목을 하나씩 확인
3. **아니오** — 이번에는 건너뛰기

승인된 항목에 대해 `skill-writer` Subagent를 UPDATE 모드로 실행하여 해당 스킬의 Exceptions 섹션에 새 항목을 추가한다.

**verify-history.md 기록 형식 (Skip 추적용):**

기존 이력 테이블에 "Skip된 항목" 컬럼을 활용한다:

```markdown
| 날짜 | 스킬 | 결과 | 이슈 수 | 사용자 액션 | Skip된 항목 | 소요 시간 |
|------|------|------|---------|------------|------------|----------|
| 2026-03-16 | verify-auth | FAIL | 2 | skip | JWT 만료 검사 | 5s |
```

### Step 10: Cross-Skill Recommendations

검증 결과를 분석하여 다른 스킬의 실행을 추천한다.

**manage-skills 추천 조건:**

검증 과정에서 다음 중 하나라도 발견되면, 리포트 마지막에 `/manage-skills` 실행을 추천한다:

1. **커버되지 않은 변경 파일** — `git diff HEAD --name-only`로 최근 변경된 파일 중 어떤 verify 스킬의 Related Files에도 포함되지 않는 파일이 있는 경우
2. **깨진 스킬 참조** — verify 스킬의 Related Files에 존재하지 않는 파일이 참조된 경우 (스킬이 stale)
3. **반복 FAIL** — 동일 스킬이 3회 연속 FAIL이면 스킬 자체의 수정이 필요할 수 있음
4. **Phase 완료 감지** — 모든 체크박스가 완료된 Phase가 있으면 GRADUATE 처리 필요

추천 메시지 형식:

```markdown
---

### 💡 추천 액션

다음 이유로 `/manage-skills` 실행을 권장합니다:
- 최근 변경된 3개 파일이 어떤 verify 스킬에도 커버되지 않음: `src/new-module.ts`, `src/helpers/format.ts`, `src/config/db.ts`
- `verify-phase-1-models`의 Related Files에 삭제된 파일 참조 1건

> `/manage-skills`를 실행하면 스킬 커버리지를 점검하고 필요한 업데이트를 수행합니다.
```

**feature-planner 추천 조건:**

등록된 verify 스킬이 0개이고, `docs/plans/` 디렉토리에 계획 문서도 없는 경우:

```markdown
### 💡 추천 액션

검증 스킬과 계획 문서가 모두 없습니다.
> `/feature-planner`로 기능 계획을 수립하면 Phase별 verify 스킬이 자동 생성됩니다.
```

## Exceptions

다음은 **문제가 아니다**:

1. **이 스킬 자체** — verify-implementation은 실행 대상에서 항상 제외한다 (무한 루프 방지)
2. **manage-skills** — 관리 스킬은 검증 대상이 아니다
3. **feature-planner** — 계획 스킬은 검증 대상이 아니다
4. **Exceptions에 명시된 케이스** — 각 verify 스킬의 Exceptions 섹션에 나열된 항목은 EXEMPT 처리
5. **비활성(ARCHIVED) 스킬** — 스킬 라이프사이클이 ARCHIVED인 스킬은 실행하지 않음

## Related Files

| File | Purpose |
|------|---------|
| `.claude/skills/dev/SKILL.md` | 전체 파이프라인 오케스트레이터 (이 스킬을 자동 호출) |
| `.claude/skills/manage-skills/SKILL.md` | 스킬 유지보수 (실행 대상 목록 동기화) |
| `.claude/skills/feature-planner/SKILL.md` | 계획 수립 (verify 스킬 생성 트리거) |
| `.claude/agents/test-runner.md` | Subagent: 개별 verify 스킬 실행 (병렬) |
| `.claude/verify-history.md` | 검증 실행 이력 (스킬 효과성 추적) |
| `CLAUDE.md` | 프로젝트 가이드라인 |
| `docs/plans/PLAN_*.md` | 계획 문서 (Verification Status 업데이트 대상) |
