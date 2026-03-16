---
name: feature-planner
description: 기능 개발 계획을 Phase 기반으로 수립하고, 각 Phase에 대한 verify 스킬을 자동 생성합니다. 기존 진행 중인 레포와 신규 빈 레포 모두 지원합니다. 사용자가 기능 구현, 개발 계획, 태스크 분해, 로드맵 작성, TDD 전략, 작업 구조화를 요청할 때 반드시 이 스킬을 사용하세요. 키워드: plan, 계획, phases, breakdown, strategy, roadmap, 설계, 구현, feature, 기능.
---

# Feature Planner

## Purpose

기능 요구사항을 분석하여 Phase 기반 개발 계획을 수립합니다:

- 각 Phase는 독립적으로 동작하는 완전한 기능을 전달
- TDD Red-Green-Refactor 사이클이 모든 Phase에 내장
- Quality Gate가 다음 Phase 진행을 통제
- 계획 확정 후 Phase별 verify 스킬을 자동 생성하여, verify-implementation과 연동
- **기존 레포**와 **신규 레포** 모두 지원

## Planning Workflow

### Step 0: Project Context Detection (Subagent 분석)

계획 수립에 앞서 프로젝트의 현재 상태를 파악한다. **`codebase-scanner`** Subagent를 실행하여 프로젝트를 분석한다.

> **사용하는 Subagent**: `.claude/agents/codebase-scanner.md`

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

### Step 4: Plan Document Creation (Subagent)

승인된 Phase 분해를 바탕으로 **`plan-writer`** Subagent를 실행하여 계획 문서를 생성한다.

> **사용하는 Subagent**: `.claude/agents/plan-writer.md`

`plan-writer`에게 다음 정보를 전달한다:
1. 프로젝트 컨텍스트 (Step 0 결과)
2. 기능 요구사항
3. 승인된 Phase 분해
4. `plan-template.md` 경로

`plan-writer`가 `docs/plans/PLAN_<feature-name>.md` 문서를 생성하고 경로를 반환한다.

문서에 포함되는 내용:
- Overview와 목표
- Architecture Decisions (근거와 트레이드오프)
- 전체 Phase 분해 (체크박스 포함)
- Quality Gate 체크리스트 (**Step 0에서 감지한 실제 명령어 사용**)
- Risk Assessment 테이블
- Phase별 Rollback Strategy
- Progress Tracking 섹션
- Verification Status 섹션 (verify-implementation 연동)

### Step 5: Verify Skill Generation (Subagent 병렬 생성)

계획이 확정되면, 각 Phase의 Quality Gate를 기반으로 verify 스킬을 **병렬로** 생성한다.

각 Phase마다 **`skill-writer`** Subagent를 1개씩 실행하여 병렬 생성한다 (최대 7개 동시).

> **사용하는 Subagent**: `.claude/agents/skill-writer.md`

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

**모든 병렬 생성이 완료된 후**, 순차적으로 레지스트리를 업데이트:
1. `manage-skills/SKILL.md`의 등록된 검증 스킬 테이블에 추가
2. `verify-implementation/SKILL.md`의 실행 대상 스킬 테이블에 추가
3. `CLAUDE.md`의 Skills 테이블에 추가

## Quality Gate Standards

각 Phase는 다음 Phase로 진행하기 전에 반드시 검증해야 한다. Step 0에서 감지한 실제 명령어를 사용한다.

**TDD Compliance** (핵심):
- [ ] 테스트를 프로덕션 코드보다 먼저 작성했는가
- [ ] Red-Green-Refactor 사이클을 따랐는가
- [ ] 비즈니스 로직 유닛 테스트 커버리지 ≥80%
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
| `.claude/skills/feature-planner/plan-template.md` | 계획 문서 생성 템플릿 |
| `.claude/skills/verify-implementation/SKILL.md` | 검증 실행 스킬 (생성된 verify 스킬 등록) |
| `.claude/skills/manage-skills/SKILL.md` | 스킬 유지보수 (생성된 verify 스킬 등록) |
| `.claude/agents/codebase-scanner.md` | Step 0 Subagent: 프로젝트 컨텍스트 통합 분석 |
| `.claude/agents/plan-writer.md` | Step 4 Subagent: 계획 문서 작성 |
| `.claude/agents/skill-writer.md` | Step 5 Subagent: verify 스킬 생성 (병렬) |
| `CLAUDE.md` | 프로젝트 가이드라인 (Skills 테이블 업데이트) |
| `docs/plans/PLAN_*.md` | 생성된 계획 문서들 |
