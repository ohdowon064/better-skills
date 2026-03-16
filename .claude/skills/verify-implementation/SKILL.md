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

현재 등록된 검증 스킬 목록이다. manage-skills 또는 feature-planner가 스킬을 생성/삭제할 때 이 테이블을 업데이트한다.

(아직 등록된 검증 스킬이 없습니다)

<!-- 스킬이 추가되면 아래 형식으로 등록:
| # | 스킬 | 설명 |
|---|------|------|
| 1 | `verify-phase-1-foundation` | Phase 1 기반 구조 검증 |
| 2 | `verify-build` | 빌드/컴파일 검증 |
-->

## Workflow

### Step 1: Argument Parsing & Skill Selection

인수를 파싱하여 실행 대상을 결정한다:

```
인수 없음            → 위 테이블의 모든 스킬 실행
"phase-N"            → verify-phase-N-* 패턴에 매칭되는 스킬만 실행
"--current-phase"    → docs/plans/PLAN_*.md에서 Status가 🔄 In Progress인 Phase를 찾아
                       해당 Phase의 verify 스킬만 실행
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

각 `test-runner`에게 해당 verify 스킬의 SKILL.md 경로를 전달한다. Subagent가 스킬을 읽고, Workflow의 모든 검사를 수행하여, 각 검사의 PASS/FAIL/EXEMPT 상태와 상세 내용을 반환한다.

이것이 기존 순차 실행 대비 핵심 개선점이다. 5개 스킬이 있으면 5배 빠르게 끝난다.

**스킬이 7개 초과인 경우**, 7개씩 배치로 나눠 실행한다.

**의존 스킬**은 선행 스킬의 결과를 확인한 후 순차 실행한다.

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
| `.claude/skills/manage-skills/SKILL.md` | 스킬 유지보수 (실행 대상 목록 동기화) |
| `.claude/skills/feature-planner/SKILL.md` | 계획 수립 (verify 스킬 생성 트리거) |
| `.claude/agents/test-runner.md` | Subagent: 개별 verify 스킬 실행 (병렬) |
| `CLAUDE.md` | 프로젝트 가이드라인 |
| `docs/plans/PLAN_*.md` | 계획 문서 (Verification Status 업데이트 대상) |
