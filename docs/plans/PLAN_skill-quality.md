# Implementation Plan: Skill Quality Improvements

**Status**: ✅ Completed
**Repo Type**: EXISTING
**Feature Start Ref**: (to be filled)
**Started**: 2026-04-20
**Last Updated**: 2026-04-20
**Estimated Completion**: 2026-04-20

---

**⚠️ CRITICAL INSTRUCTIONS**: After completing each phase:
1. ✅ Check off completed task checkboxes
2. 📅 Update "Last Updated" date above
3. 📝 Document learnings in Notes section
4. ➡️ Only then proceed to next phase

⛔ **DO NOT skip quality gates or proceed with failing checks**

---

## 📋 Overview

### Feature Description
코드 리뷰 결과에서 도출된 10개 개선항목을 3개 Phase로 분류하여 수정한다. 모든 변경은 Markdown 프롬프트 파일 수정이며, 빌드/테스트가 없는 프로젝트 특성상 수동 검토로 품질을 확인한다.

### Success Criteria
- [ ] code-review SKILL.md가 커밋 없는 repo에서도 안전하게 동작
- [ ] 4개 에이전트의 edge case가 모두 처리됨
- [ ] 스킬 간 테이블 스키마, 카운터, 디렉토리 생성 일관성 확보
- [ ] dev와 feature-planner 간 파일 충돌 감지 로직 중복 제거

---

## 🛠️ Project Context

### Detected Environment
| Item | Value |
|------|-------|
| **Repo Type** | EXISTING |
| **Language** | Markdown (prompt-engineering) |
| **Package Manager** | N/A |
| **Test Framework** | N/A |
| **Linter** | N/A |
| **Type Checker** | N/A |
| **CI/CD** | N/A |

### Validation Commands
```bash
# 이 프로젝트는 Markdown 파일만으로 구성되어 빌드/테스트/린트가 없음.
# 품질 검증은 수동 검토 + git diff로 수행.
```

---

## 🏗️ Architecture Decisions

| Decision | Rationale | Trade-offs |
|----------|-----------|------------|
| 3 Phase로 분리 | 심각도(HIGH→MEDIUM) 순서로 우선순위 반영 | Phase 간 의존성 없어 독립 실행 가능 |
| TDD 미적용 | Markdown-only 프로젝트, 실행 가능한 코드 없음 | 자동 검증 불가, 수동 리뷰 필요 |
| 파일 충돌 감지를 feature-planner LIST로 위임 | SSOT 원칙, 중복 코드 제거 | dev에서 feature-planner LIST를 호출하는 의존성 추가 |

---

## 🚀 Implementation Phases

### Phase 1: Critical Fix — code-review git diff 안정화
**Goal**: 커밋 없는 repo에서 `git diff HEAD` 실패 방지 + 설명 불일치 수정
**Estimated Time**: 30분
**Status**: ⏳ Pending
**Phase Start Ref**: -

#### Tasks

- [ ] **Task 1.1**: `skills/code-review/SKILL.md` — Step 1 인수 없음 모드 수정
  - File(s): `skills/code-review/SKILL.md`
  - 변경 내용:
    1. `git diff HEAD` → 커밋 유무 분기 추가: `git rev-parse HEAD 2>/dev/null`로 먼저 확인
    2. 커밋 없으면 `git diff` (staged) + `git diff --staged` 조합으로 폴백
    3. 인수 없음 설명을 "최근 커밋되지 않은 변경 + 스테이징된 변경 리뷰"로 명확화

#### Quality Gate ✋
- [ ] `git diff HEAD` 폴백 로직이 BRAND_NEW repo 시나리오를 커버하는지 확인
- [ ] 인수별 설명과 실제 git 명령어가 일치하는지 확인

---

### Phase 2: Agent Robustness — 에이전트 견고성 강화
**Goal**: 4개 에이전트의 edge case 처리 및 사용성 개선
**Estimated Time**: 1시간
**Status**: ⏳ Pending
**Phase Start Ref**: -

#### Tasks

- [ ] **Task 2.1**: `agents/code-reviewer.md` — standalone 모드 추가
  - File(s): `agents/code-reviewer.md`
  - 변경 내용:
    1. 기존 `phase`, `integration` 모드 외에 `standalone` 모드 추가
    2. standalone 모드: PLAN 없이 일반 코드 품질 기준으로 리뷰
    3. 입력 섹션에 "PLAN 경로가 없으면 standalone 모드로 동작" 명시

- [ ] **Task 2.2**: `agents/codebase-scanner.md` — base branch 자동 감지
  - File(s): `agents/codebase-scanner.md`
  - 변경 내용:
    1. `git diff main...HEAD` 하드코딩 제거
    2. base branch 자동 감지 로직 추가: `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`
    3. 감지 실패 시 `main` → `master` 순서로 폴백

- [ ] **Task 2.3**: `agents/skill-writer.md` — 예시 출력 + 검증 단계 추가
  - File(s): `agents/skill-writer.md`
  - 변경 내용:
    1. CREATE 모드에 예시 SKILL.md 출력 구조 추가 (frontmatter + 5개 필수 섹션 예시)
    2. 생성 후 검증 단계 추가: 파일 존재 확인 + 필수 섹션 존재 여부 grep

- [ ] **Task 2.4**: `agents/test-runner.md` — 타임아웃 설정 가능
  - File(s): `agents/test-runner.md`
  - 변경 내용:
    1. 하드코딩 60초 → "프로젝트 컨텍스트의 타임아웃 설정값, 기본 60초" 로 변경
    2. 규칙 섹션에 타임아웃 오버라이드 입력 필드 추가

#### Quality Gate ✋
- [ ] code-reviewer에 3개 모드(phase, integration, standalone)가 모두 정의됨
- [ ] codebase-scanner의 base branch 감지에 폴백 체인이 있음
- [ ] skill-writer에 예시 출력과 검증 단계가 있음
- [ ] test-runner 타임아웃이 설정 가능하게 변경됨

---

### Phase 3: Consistency — 스킬 간 일관성 및 누락 보완
**Goal**: 스킬 간 테이블 스키마, 카운터, 디렉토리 생성 등 불일치 해소
**Estimated Time**: 1시간
**Status**: ⏳ Pending
**Phase Start Ref**: -

#### Tasks

- [ ] **Task 3.1**: `skills/dev/SKILL.md` — retry 카운터 메커니즘 명시
  - File(s): `skills/dev/SKILL.md`
  - 변경 내용:
    1. Step 3 진입 시 `retry_count = 0` 초기화 명시
    2. Step 3c FAIL, Step 3d REQUEST_CHANGES에서 `retry_count += 1` 명시
    3. `retry_count >= 3`일 때 블로커 보고 트리거 명시
    4. Step 3e(Phase 완료) 시 `retry_count = 0`으로 리셋 명시

- [ ] **Task 3.2**: `skills/verify-implementation/SKILL.md` — Review 컬럼 일관성
  - File(s): `skills/verify-implementation/SKILL.md`
  - 변경 내용:
    1. Step 7의 Verification Status 테이블 예시에 Review 컬럼 추가
    2. plan-template.md의 스키마(Phase | Verify Skill | Last Run | Result | Review | Issues)와 일치시킴
    3. verify-implementation은 Review 컬럼을 기존 값 그대로 보존하고 건드리지 않음을 명시

- [ ] **Task 3.3**: `skills/feature-planner/SKILL.md` — docs/plans 디렉토리 생성
  - File(s): `skills/feature-planner/SKILL.md`
  - 변경 내용:
    1. Step 4 (Plan Document Creation)에서 파일 저장 전 `mkdir -p docs/plans` 추가
    2. BRAND_NEW/FRESH_START 레포에서도 안전하게 동작

- [ ] **Task 3.4**: `skills/dev/SKILL.md` + `skills/feature-planner/SKILL.md` — 파일 충돌 감지 중복 제거
  - File(s): `skills/dev/SKILL.md`, `skills/feature-planner/SKILL.md`
  - 변경 내용:
    1. dev의 Step 2.5 "복수 PLAN 파일 충돌 감지" 섹션에서 직접 grep/sed 로직 제거
    2. 대신 feature-planner LIST 모드를 호출하여 충돌 감지 결과를 받도록 변경
    3. dev에는 "feature-planner를 LIST 모드로 실행하여 충돌을 확인한다" + 결과 분기만 남김

#### Quality Gate ✋
- [ ] dev SKILL.md에 retry_count 초기화, 증가, 리셋, 트리거 조건이 모두 명시됨
- [ ] verify-implementation Step 7 테이블과 plan-template.md 테이블 스키마가 일치함
- [ ] feature-planner Step 4에 mkdir -p docs/plans가 포함됨
- [ ] dev Step 2.5에서 grep/sed 직접 로직이 제거되고 feature-planner LIST 호출로 대체됨

---

## ⚠️ Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|---------------------|
| Phase 간 교차 수정으로 merge conflict | Low | Low | Phase가 서로 다른 파일/섹션을 수정하므로 충돌 가능성 낮음 |
| dev Step 2.5 위임으로 feature-planner 의존성 추가 | Low | Medium | feature-planner LIST는 이미 존재하는 기능이므로 안정적 |

---

## 🔄 Rollback Strategy

### If Phase 1 Fails
- `git checkout -- skills/code-review/SKILL.md`

### If Phase 2 Fails
- `git checkout -- agents/code-reviewer.md agents/codebase-scanner.md agents/skill-writer.md agents/test-runner.md`

### If Phase 3 Fails
- `git checkout -- skills/dev/SKILL.md skills/verify-implementation/SKILL.md skills/feature-planner/SKILL.md`

---

## ✅ Verification Status (Auto-updated by verify-implementation, dev)

| Phase | Verify Skill | Last Run | Result | Review | Issues |
|-------|-------------|----------|--------|--------|--------|
| Phase 1 | (수동 검토) | 2026-04-20 | ✅ PASS | - | 0 |
| Phase 2 | (수동 검토) | 2026-04-20 | ✅ PASS | - | 0 |
| Phase 3 | (수동 검토) | 2026-04-20 | ✅ PASS | - | 0 |

---

## 📊 Progress Tracking

### Completion Status
- **Phase 1**: ✅ 100%
- **Phase 2**: ✅ 100%
- **Phase 3**: ✅ 100%

**Overall Progress**: 100%

### Time Tracking
| Phase | Estimated | Actual | Variance |
|-------|-----------|--------|----------|
| Phase 1 | 30min | - | - |
| Phase 2 | 1h | - | - |
| Phase 3 | 1h | - | - |
| **Total** | 2.5h | - | - |

---

## 📝 Notes & Learnings

### Implementation Notes
- Markdown-only 프로젝트이므로 TDD/빌드/테스트 불필요
- 각 Phase는 독립적으로 실행 가능 (교차 의존성 없음)

### Blockers Encountered
- (없음)
