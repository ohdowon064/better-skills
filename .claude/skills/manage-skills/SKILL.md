---
name: manage-skills
description: 세션 변경사항과 계획 문서를 분석하여 검증 스킬의 드리프트를 탐지하고 수정합니다. 새 스킬 생성, 기존 스킬 업데이트, 레지스트리 동기화를 수행합니다. 기존 레포와 신규 레포 모두 지원하며, 스킬이 0개인 콜드 스타트도 처리합니다. 새로운 패턴/규칙 도입 후, PR 전 스킬 커버리지 점검, 스킬 정리/동기화가 필요할 때 반드시 이 스킬을 사용하세요. 키워드: manage, skills, 스킬, 관리, 유지보수, drift, coverage, 커버리지, 동기화, sync.
argument-hint: "[선택사항: 특정 스킬 이름 또는 집중할 영역]"
---

# Manage Skills

## Purpose

현재 세션에서 변경된 내용과 계획 문서를 분석하여 검증 스킬의 드리프트를 탐지하고 수정한다:

1. **커버리지 누락** — 어떤 verify 스킬에서도 참조하지 않는 변경된 파일
2. **유효하지 않은 참조** — 삭제되거나 이동된 파일을 참조하는 스킬
3. **누락된 검사** — 기존 검사에서 다루지 않는 새로운 패턴/규칙
4. **오래된 값** — 더 이상 일치하지 않는 설정값 또는 탐지 명령어
5. **계획 문서 동기화** — feature-planner가 생성한 계획과 verify 스킬의 정합성

## Execution Timing

- 새로운 패턴이나 규칙을 도입하는 기능을 구현한 후
- 기존 verify 스킬을 수정하고 일관성을 점검하고 싶을 때
- PR 전에 verify 스킬이 변경된 영역을 커버하는지 확인할 때
- 검증 실행 시 예상했던 이슈를 놓쳤을 때
- 주기적으로 스킬을 코드베이스 변화에 맞춰 정렬할 때

## Registered Verify Skills (Single Source of Truth)

**이 테이블이 verify 스킬 목록의 유일한 정식 소스(SSOT)이다.** verify-implementation과 CLAUDE.md는 이 테이블을 참조한다.

스킬 생성/삭제 시 이 테이블만 업데이트하면, verify-implementation과 CLAUDE.md는 런타임에 이 테이블을 읽어 동기화한다.

(아직 등록된 검증 스킬이 없습니다)

<!-- 스킬이 추가되면 아래 형식으로 등록:
| 스킬 | 설명 | 커버 파일 패턴 | 라이프사이클 |
|------|------|---------------|------------|
| `verify-phase-1-models` | Phase 1 모델 검증 | `src/models/**/*.ts` | ACTIVE |
| `verify-build` | 빌드/컴파일 검증 | `*` | ACTIVE |
-->

## Workflow

### Step 0: Cold Start Detection

먼저 프로젝트의 현재 상태를 확인한다.

```bash
# git 존재 여부
git log --oneline -1 2>/dev/null

# 기존 verify 스킬 존재 여부
ls .claude/skills/verify-*/SKILL.md 2>/dev/null
```

**등록된 스킬이 0개이고 git history도 없는 경우 (BRAND_NEW 레포):**

프로젝트 구조를 분석하여 기본 verify 스킬 세트를 제안한다:

```markdown
## 기본 검증 스킬 제안

프로젝트에 검증 스킬이 없습니다. 다음 기본 스킬을 생성할까요?

1. **verify-build** — 빌드/컴파일이 에러 없이 완료되는지 검증
2. **verify-test** — 모든 테스트가 통과하고 커버리지 목표를 달성하는지 검증
3. **verify-lint** — 린트/포매팅 규칙을 준수하는지 검증
```

AskUserQuestion으로 사용자에게 어떤 기본 스킬을 생성할지 확인한 후 Step 6으로 이동하여 스킬을 생성한다.

**등록된 스킬이 0개이지만 git history가 있는 경우 (EXISTING 레포):**

기존 코드베이스를 분석하여 도메인별 verify 스킬도 함께 제안한다:
- 기본 3개 (build, test, lint) + 프로젝트의 실제 구조에서 도출한 도메인 스킬
- 프로젝트의 주요 디렉토리와 패턴을 분석하여, 해당 프로젝트에 의미 있는 검증 영역을 식별한다
- 제안할 때는 반드시 프로젝트의 실제 파일 경로와 패턴을 근거로 제시한다

### Step 1: Session Change Analysis (Subagent 분석)

**`codebase-scanner`** Subagent를 실행하여 다음 항목을 분석한다:

> **사용하는 Subagent**: `.claude/agents/codebase-scanner.md`

`codebase-scanner`에게 다음 분석 범위를 요청한다:
- **git 변경사항 수집** — 커밋되지 않은 변경, 브랜치 분기 이후 변경 파일 수집 및 그룹화
- **verify 스킬 갭 분석** — 등록된 각 스킬의 Related Files 경로 존재 여부, 탐지 명령어 유효성, 변경된 값 감지
- **계획 문서 동기화 점검** — Phase 상태 확인, 완료된 Phase의 스킬 상태, 새 계획의 미생성 스킬

하나의 `codebase-scanner`가 모든 분석을 통합 수행하여 결과를 반환한다.

### Step 2: File-to-Skill Mapping

`codebase-scanner`의 결과(변경 파일 목록)와 **Registered Verify Skills** 테이블을 대조하여 매핑을 구축한다.

파일이 스킬과 매칭되는 조건:
- 해당 스킬의 커버 파일 패턴과 일치
- 해당 스킬이 참조하는 디렉토리 내에 위치
- 해당 스킬의 탐지 명령어에 사용된 패턴과 일치

```markdown
### File → Skill Mapping

| 스킬 | 트리거 파일 (변경된 파일) | 액션 |
|------|--------------------------|------|
| verify-api | `router.ts`, `handler.ts` | CHECK |
| verify-ui | `Button.tsx` | CHECK |
| (스킬 없음) | `package.json`, `.eslintrc.js` | UNCOVERED |
```

### Step 3: Coverage Gap Analysis

영향받은(CHECK) 각 스킬에 대해 `codebase-scanner`의 갭 분석 결과를 활용하여 갭을 정리한다:

1. **누락된 파일 참조** — 변경 파일이 스킬의 Related Files에 없는 경우
2. **오래된 탐지 명령어** — 패턴이 현재 파일 구조와 불일치
3. **커버되지 않은 새 패턴** — 변경된 파일에서 스킬이 검사하지 않는 새 규칙
4. **삭제된 파일의 잔여 참조** — Related Files에 없는 파일이 여전히 참조됨
5. **변경된 값** — 스킬이 검사하는 특정 값이 수정됨

```markdown
| 스킬 | 갭 유형 | 상세 |
|------|---------|------|
| verify-api | 파일 누락 | `src/server/newHandler.ts`가 Related Files에 없음 |
| verify-ui | 새 패턴 | 새 컴포넌트가 검사되지 않는 규칙을 사용 |
```

### Step 4: CREATE vs UPDATE Decision

다음 결정 트리를 적용한다:

```
커버되지 않은(UNCOVERED) 각 파일 그룹에 대해:
    IF 기존 스킬의 도메인과 관련된 파일인 경우:
        → UPDATE (기존 스킬 커버리지 확장)
    ELSE IF 3개 이상의 관련 파일이 공통 규칙/패턴을 공유하는 경우:
        → CREATE (새 verify 스킬 생성)
    ELSE:
        → EXEMPT (스킬 불필요)
```

`codebase-scanner`의 계획 문서 동기화 결과도 반영한다:
- 완료된 Phase의 스킬 → GRADUATE 제안
- 삭제된 계획의 스킬 → ARCHIVE 제안

결과를 사용자에게 제시한다:

```markdown
### 제안 액션

**UPDATE** (N개)
- `verify-api` — 누락된 파일 참조 2개 추가, 탐지 패턴 업데이트

**CREATE** (M개)
- 새 스킬 필요 — <패턴 설명> 커버 (X개 미커버 파일)

**GRADUATE** (K개)
- `verify-phase-1-models` → Phase 1 완료, 범용 `verify-models`로 승격 제안

**EXEMPT**
- `package.json` — 설정 파일, 면제
```

AskUserQuestion을 사용하여 확인한다.

### Step 5: Update Existing Skills (Subagent)

사용자가 업데이트를 승인한 각 스킬에 대해 **`skill-writer`** Subagent를 UPDATE 모드로 실행한다.

> **사용하는 Subagent**: `.claude/agents/skill-writer.md`

각 `skill-writer`에게 전달하는 정보:
1. **모드**: UPDATE
2. 수정할 SKILL.md 경로
3. 수정 내용 (추가할 파일, 업데이트할 패턴, 변경된 값 등)

**규칙:**
- **추가/수정만** — 작동하는 기존 검사는 절대 제거하지 않음
- Related Files 테이블에 새 파일 경로 추가
- 변경된 파일에서 발견된 패턴에 대한 새 탐지 명령어 추가
- 코드베이스에서 삭제가 확인된 파일의 참조 제거
- 변경된 특정 값 업데이트

여러 스킬을 업데이트해야 하는 경우, `skill-writer`를 **병렬로** 실행한다.

### Step 6: Create New Skills (Subagent 병렬 생성)

새로 생성할 각 스킬에 대해 **`skill-writer`** Subagent를 CREATE 모드로 실행한다.

> **사용하는 Subagent**: `.claude/agents/skill-writer.md`

1. **탐색** — 관련 변경 파일을 읽어 패턴을 이해한다

2. **사용자에게 스킬 이름 확인** — AskUserQuestion을 사용한다

   **이름 규칙:**
   - 반드시 `verify-`로 시작 (예: `verify-auth`, `verify-api`)
   - kebab-case 사용
   - 사용자가 `verify-` 없이 이름을 주면 자동으로 접두사 추가

3. **병렬 생성** — 여러 스킬을 생성할 경우 `skill-writer`를 동시에 실행한다

   각 `skill-writer`에게 전달하는 정보:
   - **모드**: CREATE
   - 스킬 이름
   - 검증할 항목 목록
   - 대상 파일 경로
   - 프로젝트 컨텍스트 (감지된 명령어)

4. **레지스트리 업데이트** — 모든 병렬 생성이 완료된 후 업데이트:

   **4a. 이 파일 자체 (`manage-skills/SKILL.md`) — SSOT:**
   - Registered Verify Skills 테이블에 새 행 추가
   - 첫 스킬 추가 시 "(아직 등록된 검증 스킬이 없습니다)" 텍스트 제거 후 테이블 생성

   **4b. `CLAUDE.md`:**
   - Skills 테이블에 새 행 추가

   > verify-implementation은 런타임에 이 파일의 SSOT 테이블을 읽으므로 별도 업데이트 불필요

### Step 7: Skill Lifecycle Management

`codebase-scanner`의 계획 문서 동기화 결과와 **`.claude/verify-history.md`** 실행 이력을 바탕으로 스킬 라이프사이클을 관리한다.

**실행 이력 기반 판단:**
- `.claude/verify-history.md`가 존재하면 읽어서 각 스킬의 효과성 지표를 확인한다
- 연속 PASS 10회 이상인 스킬 → GRADUATE 후보로 추가 제안
- Skip 비율 50% 이상인 스킬 → Exceptions 업데이트를 함께 제안
- FAIL 비율 80% 이상인 스킬 → 검사 기준 완화 또는 코드 수정 필요 알림

스킬 라이프사이클:

```
CREATED   → feature-planner가 생성. 아직 검증 미실행.
ACTIVE    → 해당 Phase 개발 중. verify-implementation이 실행.
GRADUATED → Phase 완료. 범용 verify 스킬로 승격.
            Phase별 세부 검사를 유지하되, 스킬 이름에서 phase-N 제거.
            예: verify-phase-1-models → verify-models
ARCHIVED  → 기능 완성 후 더 이상 불필요. 실행 대상에서 제외.
```

GRADUATE 시:
- 기존 `verify-phase-N-<name>` 스킬의 내용을 `verify-<name>` 스킬로 복사
- Phase 특화 검사는 제거하고 범용 검사만 유지
- 기존 Phase 스킬은 ARCHIVED로 표시

### Step 8: Skill Version Management (Subagent)

**`version-manager`** Subagent를 실행하여 `.claude/skill-versions/`의 스냅샷을 관리한다.

> **사용하는 Subagent**: `.claude/agents/version-manager.md`

**전달 항목:**
- 이번 세션에서 업데이트된 스킬 이름 목록
- `.claude/verify-history.md`에서 업데이트 후 FAIL이 증가한 스킬이 있으면 롤백 후보로 표시

**Subagent가 수행하는 내용:**
- 전체 스킬의 버전 이력 조회 및 보고서 생성
- 업데이트 후 FAIL 증가 스킬에 대해 롤백 제안 (AskUserQuestion으로 확인)
- 롤백 실행 시 현재 버전도 스냅샷 보관 후 복원
- 10개 초과 스냅샷 자동 정리

### Step 9: Validation

모든 편집 후:

1. 수정된 모든 SKILL.md 파일을 다시 읽기
2. 마크다운 형식 확인 (닫히지 않은 코드 블록, 테이블 열 일관성)
3. Related Files의 각 경로에 대해 파일 존재 확인:
   ```bash
   ls <file-path> 2>/dev/null || echo "MISSING: <file-path>"
   ```
4. 업데이트된 각 스킬에서 탐지 명령어 하나를 드라이런
5. Registered Verify Skills (SSOT) 테이블과 실제 `.claude/skills/verify-*/SKILL.md` 파일이 일치하는지 확인 (테이블에 있지만 파일 없음 / 파일 있지만 테이블에 없음)

### Step 10: Summary Report

```markdown
## 세션 스킬 유지보수 보고서

### 분석된 변경 파일: N개

### 업데이트된 스킬: X개
- `verify-<name>`: N개의 새 검사 추가, Related Files 업데이트

### 생성된 스킬: Y개
- `verify-<name>`: <패턴> 커버

### 승격된 스킬: K개
- `verify-phase-1-models` → `verify-models` (GRADUATED)

### 업데이트된 레지스트리:
- `manage-skills/SKILL.md`: Registered Verify Skills 테이블
- `verify-implementation/SKILL.md`: Target Skills 테이블
- `CLAUDE.md`: Skills 테이블

### EXEMPT (적용 스킬 없음):
- `path/to/file` — 면제 (사유)
```

### Step 11: Cross-Skill Recommendations

스킬 유지보수 결과를 분석하여 다른 스킬의 실행을 추천한다.

**verify-implementation 추천 조건:**

다음 중 하나라도 해당하면, 보고서 마지막에 `/verify-implementation` 실행을 추천한다:

1. **스킬이 신규 생성됨** — 새로 만든 verify 스킬이 실제로 동작하는지 검증 필요
2. **스킬이 업데이트됨** — 수정된 스킬의 탐지 명령어가 정상 작동하는지 확인 필요
3. **GRADUATE 처리됨** — 범용화된 스킬의 검증 범위가 올바른지 확인 필요

```markdown
---

### 💡 추천 액션

다음 이유로 `/verify-implementation` 실행을 권장합니다:
- 새로 생성된 스킬 2개의 동작 확인 필요: `verify-auth`, `verify-api`
- 업데이트된 스킬 1개의 재검증 필요: `verify-models`

> `/verify-implementation`을 실행하면 새로 생성/수정된 스킬이 정상 동작하는지 즉시 확인할 수 있습니다.
```

**feature-planner 추천 조건:**

분석 결과 대규모 미커버 영역이 발견되었지만, 기존 스킬의 확장으로는 커버할 수 없는 경우:

```markdown
### 💡 추천 액션

미커버 영역이 새로운 기능 모듈로 보입니다.
> `/feature-planner`로 해당 기능의 개발 계획을 수립하면 체계적인 verify 스킬이 자동 생성됩니다.
```

### Step 12: Evals Drift Detection (Subagent)

**`evals-checker`** Subagent를 실행하여 `evals/evals.json`과 실제 스킬 내용의 불일치를 감지한다.

> **사용하는 Subagent**: `.claude/agents/evals-checker.md`

**전달 항목:**
- 이번 세션에서 생성/수정/삭제된 스킬 목록
- 자동 업데이트 모드 여부 (AskUserQuestion으로 사용자에게 확인)

AskUserQuestion 선택지:
1. **자동 업데이트** — STALE 제거, OUTDATED 갱신, UNCOVERED 테스트 초안 생성
2. **보고만** — 불일치 목록만 표시

**Subagent가 수행하는 내용:**
- evals.json과 코어 스킬(dev, feature-planner, verify-implementation, manage-skills)의 내용 대조
- 3가지 불일치 유형 감지: UNCOVERED(새 기능에 테스트 없음), STALE(삭제된 기능 참조), OUTDATED(현재 동작과 불일치)
- 자동 업데이트 모드 시 evals.json 직접 수정

## Quality Standards

생성/업데이트된 모든 스킬은 다음을 갖추어야 한다:

- **실제 파일 경로** (`ls`로 검증), 플레이스홀더가 아닌 것
- **작동하는 탐지 명령어** — 현재 파일과 매칭되는 실제 패턴
- **PASS/FAIL 기준** — 각 검사에 대해 명확한 조건
- **최소 2-3개의 예외** — 위반이 아닌 것에 대한 설명
- **일관된 형식** — 기존 스킬과 동일한 frontmatter, 섹션 헤더, 테이블 구조

## Exemptions

다음은 **문제가 아니다**:

1. **Lock 파일 및 생성된 파일** — `package-lock.json`, `yarn.lock`, 빌드 출력물은 스킬 커버리지 불필요
2. **일회성 설정 변경** — 버전 범프, 린터 설정의 사소한 변경
3. **문서 파일** — `README.md`, `CHANGELOG.md`, `LICENSE`
4. **테스트 픽스처** — `fixtures/`, `__fixtures__/`, `test-data/` 내 파일
5. **벤더/서드파티 코드** — `vendor/`, `node_modules/`
6. **CI/CD 설정** — `.github/`, `.gitlab-ci.yml`, `Dockerfile`
7. **CLAUDE.md 자체** — 문서 업데이트이므로 검증 불필요
8. **영향받지 않은 스킬** — UNAFFECTED로 표시된 스킬은 검토 불필요

## Related Files

| File | Purpose |
|------|---------|
| `.claude/skills/dev/SKILL.md` | 전체 파이프라인 오케스트레이터 (이 스킬을 자동 호출) |
| `.claude/skills/verify-implementation/SKILL.md` | 통합 검증 스킬 (Target Skills 동기화) |
| `.claude/skills/feature-planner/SKILL.md` | 계획 수립 스킬 (verify 스킬 생성 트리거) |
| `.claude/skills/manage-skills/SKILL.md` | 이 파일 자체 (Registered Verify Skills 관리) |
| `.claude/agents/codebase-scanner.md` | Subagent: 변경사항/스킬갭/계획동기화 통합 분석 |
| `.claude/agents/skill-writer.md` | Subagent: verify 스킬 생성/업데이트 (병렬) |
| `.claude/agents/version-manager.md` | Subagent: 스킬 버전 이력 조회/롤백/정리 |
| `.claude/agents/evals-checker.md` | Subagent: evals 드리프트 감지/자동 수정 |
| `.claude/verify-history.md` | 검증 실행 이력 (스킬 효과성 분석) |
| `.claude/skill-versions/` | 스킬 버전 스냅샷 디렉토리 (이력 조회/롤백) |
| `evals/evals.json` | 스킬 테스트 케이스 (evals 드리프트 감지 대상) |
| `CLAUDE.md` | 프로젝트 가이드라인 (Skills 테이블 관리) |
| `docs/plans/PLAN_*.md` | 계획 문서 (Phase 상태 확인용) |
