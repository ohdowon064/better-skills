---
name: feature-planner
description: 기능 개발 계획을 Phase 기반으로 수립(CREATE)하고, 기존 계획을 수정(UPDATE)하며, 완료 처리(COMPLETE)합니다. 각 Phase에 대한 verify 스킬을 자동 생성합니다. 기존 레포와 신규 레포 모두 지원합니다. 기능 구현, 개발 계획, 태스크 분해, 로드맵, TDD 전략, 계획 수정, 계획 완료, Phase 추가/삭제를 요청할 때 이 스킬을 사용하세요. 키워드: plan, 계획, phases, breakdown, strategy, roadmap, 설계, 구현, feature, 기능, update, 수정, complete, 완료.
argument-hint: "[선택: update PLAN_name, complete PLAN_name, list]"
---

# Feature Planner

## Purpose

기능 요구사항을 분석하여 Phase 기반 개발 계획을 수립합니다:

- 각 Phase는 독립적으로 동작하는 완전한 기능을 전달
- TDD Red-Green-Refactor 사이클이 모든 Phase에 내장
- Quality Gate가 다음 Phase 진행을 통제
- 계획 확정 후 Phase별 verify 스킬을 자동 생성하여, verify-implementation과 연동
- **기존 레포**와 **신규 레포** 모두 지원

## Modes

이 스킬은 4가지 모드로 동작한다. 인수로 모드를 지정한다:

```
(인수 없음)              → CREATE 모드 (새 계획 수립)
"update PLAN_auth"       → UPDATE 모드 (기존 계획 수정)
"complete PLAN_auth"     → COMPLETE 모드 (계획 완료 처리)
"list"                   → 기존 계획 목록 표시
```

### UPDATE 모드

기존 계획을 수정한다. 다음 시나리오를 지원한다:

**Phase 추가/삽입:**
1. 기존 PLAN 문서를 읽는다
2. 사용자에게 새 Phase의 위치(기존 Phase 사이 또는 끝)와 내용을 확인한다
3. Phase 번호를 재정렬한다 (삽입 시 이후 Phase 번호 증가)
4. 새 Phase의 verify 스킬을 `skill-writer`로 생성한다
5. 영향받은 기존 verify 스킬의 이름을 업데이트한다 (phase-2 → phase-3 등)
6. 레지스트리 동기화

**Phase 수정:**
1. 기존 PLAN 문서를 읽는다
2. 수정할 Phase를 사용자에게 확인한다
3. Tasks, Quality Gate, 대상 파일 등을 수정한다
4. 해당 Phase의 verify 스킬을 `skill-writer` UPDATE 모드로 수정한다

**Phase 삭제:**
1. 사용자에게 삭제할 Phase를 확인한다
2. 해당 Phase의 verify 스킬을 레지스트리에서 삭제하고 파일도 삭제한다
3. Phase 번호를 재정렬한다
4. 레지스트리 동기화

**요구사항 변경:**
1. 기존 PLAN 문서의 Overview/Success Criteria를 수정한다
2. 영향받는 Phase를 식별한다
3. 영향받는 Phase의 Tasks와 Quality Gate를 조정한다
4. 대응하는 verify 스킬도 업데이트한다

### COMPLETE 모드

모든 Phase가 완료된 계획을 정리한다:

1. PLAN 문서의 모든 Phase Status가 ✅ Complete인지 확인한다
   - 미완료 Phase가 있으면 경고하고 중단한다
2. PLAN 문서의 Status를 "✅ Completed"로 변경한다
3. 모든 `verify-phase-N-*` 스킬을 범용 스킬로 통합한다:
   - `skill-writer` CREATE로 범용 `verify-<name>` 스킬 생성 (Phase 특화 검사 제거)
   - 기존 `verify-phase-N-*` 스킬의 파일 삭제 + 레지스트리에서 제거
   - 새 범용 스킬을 레지스트리에 추가
4. 레지스트리 동기화 (skill-registry.json)
5. 완료 보고서 출력

### LIST 모드

기존 계획 문서를 목록으로 표시하고, 복수 PLAN 간 충돌을 감지한다:

```bash
ls docs/plans/PLAN_*.md 2>/dev/null
```

**Multi-PLAN Summary Table:**

각 문서의 Status, Phase 진행률, 최종 검증 결과를 요약 테이블로 출력한다:

```markdown
## 현재 계획 목록

| PLAN | Status | 진행률 | 현재 Phase | 최종 검증 | 수정 파일 수 |
|------|--------|--------|-----------|----------|-------------|
| PLAN_auth | 🔄 In Progress | 40% (2/5) | Phase 3 | ✅ 2026-03-15 | 12 |
| PLAN_search | 🔄 In Progress | 20% (1/5) | Phase 2 | ❌ 2026-03-16 | 8 |
| PLAN_ui | ✅ Completed | 100% (3/3) | — | ✅ 2026-03-14 | 15 |
```

**파일 충돌 감지:**

복수 PLAN이 동시에 🔄 In Progress인 경우, 각 PLAN의 Phase Tasks에서 수정 대상 파일(File(s) 필드)을 추출하여 교차 비교한다:

```bash
# 각 PLAN의 현재/미래 Phase에서 대상 파일 추출
grep -A2 "File(s):" docs/plans/PLAN_auth.md | grep '`' | sed 's/.*`\(.*\)`.*/\1/'
grep -A2 "File(s):" docs/plans/PLAN_search.md | grep '`' | sed 's/.*`\(.*\)`.*/\1/'
```

동일 파일을 둘 이상의 PLAN이 수정하려는 경우 충돌 경고를 표시한다:

```markdown
### ⚠️ 파일 충돌 감지

다음 파일이 여러 PLAN에서 동시에 수정 대상입니다:

| 파일 | PLAN | Phase |
|------|------|-------|
| `src/middleware/auth.ts` | PLAN_auth | Phase 3 |
| `src/middleware/auth.ts` | PLAN_search | Phase 2 |

**권장 조치:**
- 충돌하는 Phase의 개발 순서를 조정하세요 (한쪽을 먼저 완료)
- 또는 파일을 분리하여 각 PLAN이 독립된 파일을 수정하도록 리팩토링하세요
```

충돌이 없으면 "충돌 없음 — 병렬 개발 안전" 메시지를 표시한다.

## Planning Workflow (CREATE 모드)

### Step 0: Project Context Detection (Subagent 분석)

계획 수립에 앞서 프로젝트의 현재 상태를 파악한다. **`codebase-scanner`** Subagent를 실행하여 프로젝트를 분석한다.

> **사용하는 Subagent**: `agents/codebase-scanner.md`

#### 레포 타입 판별

먼저 레포의 상태를 판별한다:

```bash
# git 초기화 여부
git log --oneline -1 2>/dev/null
# 커밋 수 확인
git rev-list --count HEAD 2>/dev/null
# 소스 코드 존재 여부
ls src/ lib/ app/ 2>/dev/null
```

| 타입 | 조건 | 의미 |
|------|------|------|
| BRAND_NEW | git 미초기화 또는 커밋 0개 | 완전히 빈 레포 |
| FRESH_START | 커밋 5개 이하, 소스 코드 없음 | 초기 설정만 완료 |
| CONFIG_ONLY | 커밋 있음, 설정 파일만 존재 | CI/CD, 린터 등만 세팅 |
| EXISTING | 소스 코드 존재 | 이미 개발 진행 중인 레포 |

#### Subagent 실행

`codebase-scanner`에게 다음 분석 항목을 요청한다:
- **프로젝트 구조 분석** — 매니페스트, 패키지 매니저, 린터/포매터, 디렉토리 구조
- **테스트 환경 감지** — 테스트 프레임워크, CI/CD, 타입 체커, 실행 명령어 도출
- **기존 코드 패턴 분석** — EXISTING 타입일 때만. 코딩 컨벤션, 아키텍처, 주요 의존성

하나의 `codebase-scanner` 인스턴스가 모든 분석을 통합 수행하므로, 중복 파일 읽기가 없고 컨텍스트 공유가 자연스럽다.

#### 컨텍스트 종합

분석 결과를 종합하여 **프로젝트 컨텍스트**를 구성한다. 감지된 정보를 아래 템플릿에 채워넣는다 (감지되지 않은 항목은 "미감지"로 표시):

```markdown
## Project Context

- **레포 타입**: [BRAND_NEW / FRESH_START / CONFIG_ONLY / EXISTING]
- **언어/생태계**: [감지된 언어 및 런타임]
- **패키지 매니저**: [감지된 패키지 매니저]
- **테스트 프레임워크**: [감지된 테스트 프레임워크]
- **린터**: [감지된 린터/포매터]
- **타입 체커**: [감지된 타입 체커 또는 "해당 없음"]
- **CI/CD**: [감지된 CI/CD 또는 "미설정"]
- **테스트 명령어**: `[감지된 test 명령어]`
- **커버리지 명령어**: `[감지된 coverage 명령어]`
- **린트 명령어**: `[감지된 lint 명령어]`
- **빌드 명령어**: `[감지된 build 명령어]`
- **아키텍처**: [감지된 아키텍처 패턴 또는 "분석 필요"]
```

이 컨텍스트는 이후 Phase의 Quality Gate 검증 명령어를 자동으로 채워넣는 데 사용된다. 감지된 실제 명령어가 플레이스홀더 대신 들어간다.

#### 레포 타입별 계획 전략

| 레포 타입 | Phase 1 내용 | Quality Gate |
|-----------|-------------|-------------|
| BRAND_NEW | 프로젝트 스캐폴딩 (git init, 패키지 매니저 초기화, 디렉토리 구조) | AskUserQuestion으로 기술 스택 확인 후 설정 |
| FRESH_START | 핵심 기능부터 시작 (스캐폴딩 불필요) | 감지된 설정 파일에서 명령어 자동 추출 |
| CONFIG_ONLY | FRESH_START와 동일 + 기존 CI/CD 설정 반영 | 기존 도구 설정을 검증 명령어에 활용 |
| EXISTING | 기존 코드와 충돌하지 않는 설계, 기존 패턴 준수 | 기존 테스트/린터를 그대로 사용 |

BRAND_NEW인 경우, AskUserQuestion으로 사용자에게 기술 스택을 확인한다:
- 언어/프레임워크 선택
- 테스트 프레임워크 선택
- 패키지 매니저 선택

### Step 1: Requirements Analysis

1. 관련 파일을 읽어 코드베이스 아키텍처 파악 (Step 0의 컨텍스트 활용)
2. 의존성과 통합 지점 식별
3. 복잡도와 리스크 평가
4. 적절한 규모 판단 (Small/Medium/Large)

### Step 2: Phase Breakdown with TDD Integration

기능을 3-7개 Phase로 분해한다. 각 Phase는:

- **Test-First**: 테스트를 구현보다 먼저 작성
- 독립적으로 동작하는 기능을 전달
- 최대 1-4시간 소요
- Red-Green-Refactor 사이클을 따름
- 측정 가능한 테스트 커버리지 목표가 있음
- 독립적으로 롤백 가능
- 명확한 성공 기준이 있음

**Phase 구조:**

```
Phase N: [이름]
├── Goal: 이 Phase가 전달하는 동작하는 기능
├── Test Strategy: 테스트 타입, 커버리지 목표, 테스트 시나리오
├── Tasks:
│   ├── RED: 실패하는 테스트 먼저 작성
│   ├── GREEN: 테스트를 통과시키는 최소 구현
│   └── REFACTOR: 테스트가 통과하는 상태에서 코드 개선
├── Quality Gate: TDD 준수 + 검증 기준
├── Dependencies: 시작 전 필요한 것
└── Coverage Target: 이 Phase의 커버리지 목표
```

**Phase 규모 가이드:**

| 규모 | Phase 수 | 총 시간 | 예시 |
|------|---------|--------|------|
| Small | 2-3개 | 3-6시간 | 다크 모드 토글, 새 폼 컴포넌트 |
| Medium | 4-5개 | 8-15시간 | 사용자 인증, 검색 기능 |
| Large | 6-7개 | 15-25시간 | AI 검색, 실시간 협업 |

### Step 3: User Approval

**중요**: AskUserQuestion을 사용하여 사용자의 명시적 승인을 받는다.

확인 사항:
- Phase 분해가 프로젝트에 적합한지
- 제안된 접근법에 우려가 없는지
- 계획 문서 생성을 진행해도 되는지

사용자가 승인한 후에만 문서를 생성한다.

### Step 4: Plan Document Creation

승인된 Phase 분해를 바탕으로 계획 문서를 직접 생성한다.

**실행 절차:**
1. `skills/feature-planner/plan-template.md`를 읽는다
2. 프로젝트 컨텍스트(Step 0)의 감지된 명령어로 Quality Gate의 Validation Commands를 채운다
3. 각 Phase를 TDD Red-Green-Refactor 구조로 작성한다
4. Risk Assessment, Rollback Strategy, Verification Status 섹션을 채운다
5. `docs/plans/PLAN_<feature-name>.md`로 저장한다

**작성 규칙:**
- Quality Gate의 검증 명령어는 반드시 프로젝트 컨텍스트에서 감지된 실제 명령어를 사용한다
- `[your test command]` 같은 플레이스홀더를 남기지 않는다
- 감지되지 않은 명령어는 해당 체크 항목을 "수동 확인 필요"로 표시한다
- 모든 체크박스는 미체크 상태로 생성한다
- 각 Phase의 Tasks에 대상 파일 경로를 구체적으로 명시한다

**문서에 포함되는 내용:**
- Overview와 목표
- Architecture Decisions (근거와 트레이드오프)
- 전체 Phase 분해 (체크박스 포함)
- Quality Gate 체크리스트 (**Step 0에서 감지한 실제 명령어 사용**)
- Risk Assessment 테이블
- Phase별 Rollback Strategy
- Progress Tracking 섹션
- Verification Status 섹션 (verify-implementation 연동)

**UPDATE 시 계획 문서 수정 절차:**
1. 기존 PLAN 문서를 읽는다
2. 수정 유형에 따라:
   - ADD_PHASE: 지정된 위치에 새 Phase 삽입, 이후 Phase 번호 재정렬
   - MODIFY_PHASE: 해당 Phase의 Tasks/Quality Gate/대상 파일 수정
   - DELETE_PHASE: Phase 제거 후 번호 재정렬, Verification Status에서도 제거
   - UPDATE_REQUIREMENTS: Overview/Success Criteria 수정, 영향받는 Phase 식별
3. Progress Tracking의 전체 진행률을 재계산한다
4. Last Updated 날짜를 갱신한다

**COMPLETE 시 계획 문서 완료 처리:**
1. 모든 Phase의 Status가 ✅ Complete인지 확인한다
2. Status를 "✅ Completed"로, Overall Progress를 100%로 변경한다
3. Estimated Completion 날짜를 실제 완료일로 업데이트한다

### Step 5: Verify Skill Generation (Subagent 병렬 생성)

계획이 확정되면, 각 Phase의 Quality Gate를 기반으로 verify 스킬을 **병렬로** 생성한다.

각 Phase마다 **`skill-writer`** Subagent를 1개씩 실행하여 병렬 생성한다 (최대 7개 동시).

> **사용하는 Subagent**: `agents/skill-writer.md`

각 `skill-writer`에게 다음 정보를 전달한다:
1. **모드**: CREATE
2. Phase 번호, 이름, 대상 파일 경로 목록
3. Quality Gate 체크리스트
4. 프로젝트 컨텍스트 (Step 0에서 감지한 명령어들)

생성할 verify 스킬의 구조:

```yaml
---
name: verify-phase-N-<name>
description: Phase N (<이름>)의 Quality Gate를 검증합니다. <Phase 내용> 구현 후 사용.
---
```

필수 섹션:
- **Purpose**: 이 Phase에서 검증할 항목 목록
- **When to Run**: Phase N 개발 중/완료 후
- **Related Files**: Phase에서 생성/수정할 파일 경로 (계획에서 추출)
- **Workflow**: Step 0에서 감지한 실제 명령어를 사용한 검사 단계
- **Exceptions**: 위반이 아닌 케이스

**병렬 실패 처리:**

N개의 `skill-writer` 중 일부가 실패한 경우:
1. 성공한 스킬은 그대로 유지한다
2. 실패한 스킬만 순차 재시도 1회 실행한다
3. 재실패 시 해당 Phase의 verify 스킬을 "수동 생성 필요"로 표시하고, 나머지 스킬로 계속 진행한다

```markdown
### ⚠️ Verify 스킬 생성 부분 실패

| Phase | 스킬 | 상태 |
|-------|------|------|
| Phase 1 | verify-phase-1-models | ✅ 생성 완료 |
| Phase 2 | verify-phase-2-api | ❌ 생성 실패 (수동 생성 필요) |
| Phase 3 | verify-phase-3-ui | ✅ 생성 완료 |
```

실패한 스킬 없이도 해당 Phase의 TDD 개발은 진행 가능하다. 검증 단계(Step 3c)에서 해당 스킬이 없으면 SKIP 처리된다.

**모든 병렬 생성이 완료된 후** (부분 실패 포함), 성공한 스킬에 대해 **`.claude/skill-registry.json`의 `skills` 배열에 새 스킬 객체를 추가**한다 (SSOT — Single Source of Truth).

> verify-implementation은 런타임에 skill-registry.json을 읽으므로, 별도 업데이트가 불필요하다.

## Quality Gate Standards

각 Phase는 다음 Phase로 진행하기 전에 반드시 검증해야 한다. Step 0에서 감지한 실제 명령어를 사용한다.

### Coverage Configuration

커버리지 임계값은 프로젝트 특성에 따라 설정 가능하다. Step 0의 codebase-scanner가 기존 프로젝트의 커버리지 설정을 감지하여 자동으로 채운다. 감지되지 않으면 기본값을 사용한다:

| 설정 | 기본값 | 감지 소스 | 의미 |
|------|--------|----------|------|
| 전체 커버리지 목표 | 80% | jest.config의 coverageThreshold, .nycrc, pytest.ini 등 | 프로젝트 전체 라인 커버리지 |
| 신규 코드 커버리지 목표 | 80% | 전체 목표와 동일 (별도 설정이 없으면) | Phase에서 추가/수정된 파일 커버리지 |
| 커버리지 하락 허용 | No | — | Phase 전후 전체 커버리지 비교 |

BRAND_NEW 레포에서는 AskUserQuestion으로 사용자에게 커버리지 목표를 확인한다.

### Checklist

**TDD Compliance** (핵심 — verify-implementation이 자동 검증):
- [ ] 테스트를 프로덕션 코드보다 먼저 작성했는가 (git 이력 기반 자동 검증)
- [ ] Red-Green-Refactor 사이클을 따랐는가 (커밋 패턴 자동 분석)
- [ ] 신규 코드 커버리지 ≥ 설정 목표
- [ ] 전체 커버리지 하락 없음
- [ ] 주요 사용자 흐름에 대한 통합 테스트 존재

**Build & Tests**:
- [ ] 빌드/컴파일 에러 없음
- [ ] 모든 기존 테스트 통과
- [ ] 새 기능에 대한 새 테스트 추가
- [ ] 테스트 스위트 실행 시간 적절 (<5분)

**Code Quality**:
- [ ] 린트 에러/경고 없음
- [ ] 타입 체크 통과 (해당 시)
- [ ] 코드 포매팅 일관성

**Security & Performance**:
- [ ] 알려진 보안 취약점 없음
- [ ] 성능 저하 없음

## Risk Assessment

각 리스크에 대해 문서화:
- Probability: Low/Medium/High
- Impact: Low/Medium/High
- Mitigation Strategy: 구체적 조치

## Rollback Strategy

각 Phase에 대해 실패 시 복구 방법을 문서화:
- 어떤 코드 변경을 되돌려야 하는지
- DB 마이그레이션 롤백 (해당 시)
- 설정 변경 복원
- 제거할 의존성

## Related Files

| File | Purpose |
|------|---------|
| `skills/dev/SKILL.md` | 전체 파이프라인 오케스트레이터 (이 스킬을 자동 호출) |
| `skills/feature-planner/plan-template.md` | 계획 문서 생성 템플릿 |
| `skills/verify-implementation/SKILL.md` | 검증 실행 스킬 (생성된 verify 스킬 등록) |
| `skills/manage-skills/SKILL.md` | 스킬 유지보수 (생성된 verify 스킬 등록) |
| `agents/codebase-scanner.md` | Step 0 Subagent: 프로젝트 컨텍스트 통합 분석 |
| `agents/skill-writer.md` | Step 5 Subagent: verify 스킬 생성 (병렬) |
| `docs/plans/PLAN_*.md` | 생성된 계획 문서들 |
