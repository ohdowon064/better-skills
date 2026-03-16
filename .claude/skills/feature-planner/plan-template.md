# Implementation Plan: [Feature Name]

**Status**: 🔄 In Progress
**Repo Type**: [BRAND_NEW | FRESH_START | CONFIG_ONLY | EXISTING]
**Started**: YYYY-MM-DD
**Last Updated**: YYYY-MM-DD
**Estimated Completion**: YYYY-MM-DD

---

**⚠️ CRITICAL INSTRUCTIONS**: After completing each phase:
1. ✅ Check off completed task checkboxes
2. 🧪 Run `/verify-implementation --current-phase` to validate quality gates
3. ⚠️ Verify ALL quality gate items pass
4. 📅 Update "Last Updated" date above
5. 📝 Document learnings in Notes section
6. ➡️ Only then proceed to next phase

⛔ **DO NOT skip quality gates or proceed with failing checks**

---

## 📋 Overview

### Feature Description
[What this feature does and why it's needed]

### Success Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

---

## 🛠️ Project Context

### Detected Environment
| Item | Value |
|------|-------|
| **Repo Type** | [BRAND_NEW / FRESH_START / CONFIG_ONLY / EXISTING] |
| **Language** | [e.g., TypeScript] |
| **Package Manager** | [e.g., pnpm] |
| **Test Framework** | [e.g., vitest] |
| **Linter** | [e.g., eslint] |
| **Type Checker** | [e.g., tsc] |
| **CI/CD** | [e.g., GitHub Actions] |

### Validation Commands
```bash
# Test
[detected test command]

# Coverage
[detected coverage command]

# Lint
[detected lint command]

# Type Check
[detected type check command]

# Build
[detected build command]

# Security Audit
[detected audit command]
```

---

## 🏗️ Architecture Decisions

| Decision | Rationale | Trade-offs |
|----------|-----------|------------|
| [Decision 1] | [Why this approach] | [What we're giving up] |

---

## 📦 Dependencies

### Required Before Starting
- [ ] Dependency 1: [Description]

### External Dependencies
- Package/Library 1: version X.Y.Z

---

## 🧪 Test Strategy

### TDD Principle
Write tests FIRST, then implement to make them pass.

### Test Pyramid
| Test Type | Coverage Target | Purpose |
|-----------|-----------------|---------|
| **Unit Tests** | ≥80% | Business logic, models, core algorithms |
| **Integration Tests** | Critical paths | Component interactions, data flow |
| **E2E Tests** | Key user flows | Full system behavior validation |

---

## 🚀 Implementation Phases

### Phase 1: [Phase Name]
**Goal**: [Specific working functionality this phase delivers]
**Estimated Time**: X hours
**Status**: ⏳ Pending
**Verify Skill**: `verify-phase-1-<name>`

#### Tasks

**🔴 RED: Write Failing Tests First**
- [ ] **Test 1.1**: Write unit tests for [specific functionality]
  - File(s): `[path/to/test/file]`
  - Expected: Tests FAIL (red) because feature doesn't exist yet

**🟢 GREEN: Implement to Make Tests Pass**
- [ ] **Task 1.2**: Implement [component/module]
  - File(s): `[path/to/source/file]`
  - Goal: Make Test 1.1 pass with minimal code

**🔵 REFACTOR: Clean Up Code**
- [ ] **Task 1.3**: Refactor for code quality
  - Checklist:
    - [ ] Remove duplication
    - [ ] Improve naming clarity
    - [ ] Add inline documentation

#### Quality Gate ✋

**⚠️ STOP: Do NOT proceed to Phase 2 until ALL checks pass**

Run: `/verify-implementation phase-1`

**TDD Compliance**:
- [ ] Tests written FIRST and initially failed
- [ ] Production code written to make tests pass
- [ ] Code improved while tests still pass
- [ ] Coverage meets requirements

**Build & Tests**:
- [ ] Build: `[detected build command]` — no errors
- [ ] Tests: `[detected test command]` — all passing
- [ ] Coverage: `[detected coverage command]` — meets target

**Code Quality**:
- [ ] Lint: `[detected lint command]` — no errors
- [ ] Type Check: `[detected type check command]` — passes
- [ ] Formatting consistent

---

<!-- Repeat Phase structure for Phase 2, 3, ... -->

---

## ⚠️ Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|---------------------|
| [Risk 1] | Low/Med/High | Low/Med/High | [Specific steps] |

---

## 🔄 Rollback Strategy

### If Phase 1 Fails
- Undo code changes in: [list files]
- Restore configuration: [specific settings]

---

## ✅ Verification Status (Auto-updated by verify-implementation)

| Phase | Verify Skill | Last Run | Result | Issues |
|-------|-------------|----------|--------|--------|
| Phase 1 | `verify-phase-1-<name>` | - | ⏳ Not run | - |
| Phase 2 | `verify-phase-2-<name>` | - | ⏳ Not run | - |

---

## 📊 Progress Tracking

### Completion Status
- **Phase 1**: ⏳ 0%
- **Phase 2**: ⏳ 0%

**Overall Progress**: 0%

### Time Tracking
| Phase | Estimated | Actual | Variance |
|-------|-----------|--------|----------|
| Phase 1 | X hours | - | - |
| **Total** | X hours | - | - |

---

## 📝 Notes & Learnings

### Implementation Notes
- [Add insights discovered during implementation]

### Blockers Encountered
- **Blocker 1**: [Description] → [Resolution]
