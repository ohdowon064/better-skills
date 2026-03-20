---
name: dev
description: 기획서 또는 기능 요구사항을 받아 계획 수립부터 TDD 개발, 검증, 스킬 유지보수까지 전체 파이프라인을 자동으로 오케스트레이션합니다. 사용자는 기획서를 넣고 핵심 의사결정에만 개입하면 됩니다. 키워드: dev, develop, 개발, 구현, implement, build, 만들어, 기능, feature, 기획서, spec, 요구사항.
argument-hint: "[기능 설명 또는 기획서 내용]"
---

# Dev — Full Pipeline Orchestrator

## Purpose

기획서 또는 기능 요구사항 하나를 입력받아 **완성된 기능**이 나올 때까지 전체 개발 파이프라인을 자동으로 실행한다:

```
기획서 입력 → 계획 수립 → [Phase별 TDD 개발 → 검증 → 코드 리뷰 → 수정]×N → 통합 리뷰 → 완료 처리 → 스킬 정리
```

사용자가 개입하는 지점은 **4곳만**:

1. **계획 승인** — Phase 분해가 적절한지 확인
2. **FAIL 시 액션 선택** — 자동 수정 / 개별 리뷰 / 건너뛰기
3. **코드 리뷰 REQUEST_CHANGES** — 리뷰 이슈 수정 / 건너뛰기
4. **예상치 못한 블로커** — 해결 방법 결정

나머지는 모두 자동으로 흐른다.

## Workflow

### Step 1: 입력 분석

기획서/요구사항을 분석하여 실행 전략을 결정한다:

```
텍스트 입력          → 그대로 feature-planner에 전달
파일 경로 (.md 등)   → 파일을 읽어 내용 추출 후 전달
이미 PLAN 존재       → 기존 PLAN의 미완료 Phase부터 재개 (Step 3으로 이동)
```

**기존 PLAN 감지:**

```bash
ls docs/plans/PLAN_*.md 2>/dev/null
```

진행 중인 PLAN이 있으면 AskUserQuestion:
- **기존 PLAN 이어서 진행** — 미완료 Phase부터 재개
- **새 PLAN 생성** — 새 기능으로 계획 수립

### Step 2: 계획 수립 (feature-planner 실행)

`feature-planner` 스킬을 CREATE 모드로 실행한다.

**자동 전달 항목:**
- 기획서/요구사항 전문
- 실행 모드: CREATE

**이 단계에서 사용자 개입:**
- codebase-scanner가 BRAND_NEW 레포 감지 시 → 기술 스택 선택 (feature-planner가 AskUserQuestion)
- Phase 분해 완료 후 → 계획 승인 (feature-planner가 AskUserQuestion)

**기능 시작 git ref 기록:**
```bash
git rev-parse HEAD
```
feature-planner 실행 직전에 현재 커밋 ref를 기록하여, PLAN 문서의 `Feature Start Ref` 필드에 저장한다. 이 ref는 통합 코드 리뷰(Step 3.5)에서 전체 diff 범위로 사용된다.

**결과물:**
- `docs/plans/PLAN_<name>.md` 생성됨 (Feature Start Ref 포함)
- `verify-phase-N-*` 스킬 생성됨
- skill-registry.json 업데이트됨

**⛔ 위 결과물이 모두 생성 완료된 후에만 다음 단계로 진행한다. PLAN 문서나 verify 스킬 생성을 백그라운드로 돌리고 개발을 먼저 시작하지 않는다.**

### Step 2.5: 복수 PLAN 파일 충돌 감지

새 PLAN이 생성된 후, 기존에 🔄 In Progress 상태인 다른 PLAN이 있는지 확인한다:

```bash
grep -l "🔄 In Progress" docs/plans/PLAN_*.md 2>/dev/null
```

**진행 중인 PLAN이 2개 이상인 경우:**

각 PLAN의 Phase Tasks에서 수정 대상 파일(File(s) 필드)을 추출하여 교차 비교한다:

```bash
# 각 PLAN의 현재/미래 Phase에서 대상 파일 추출
grep -A2 "File(s):" docs/plans/PLAN_*.md | grep '`' | sed 's/.*`\(.*\)`.*/\1/'
```

동일 파일을 둘 이상의 PLAN이 수정하려는 경우 경고를 표시한다:

```markdown
### ⚠️ 파일 충돌 감지

다음 파일이 현재 PLAN과 진행 중인 다른 PLAN에서 동시에 수정 대상입니다:

| 파일 | 현재 PLAN | 충돌 PLAN | Phase |
|------|----------|----------|-------|
| `src/middleware/auth.ts` | PLAN_payment | PLAN_auth | Phase 3 |

**권장 조치:**
1. 충돌하는 Phase의 개발 순서를 조정 (한쪽을 먼저 완료)
2. 파일을 분리하여 각 PLAN이 독립된 파일을 수정하도록 리팩토링
```

AskUserQuestion으로 확인한다:
1. **그대로 진행** — 충돌을 인지하고 개발 계속
2. **순서 조정** — 충돌 PLAN을 먼저 완료한 후 진행

충돌이 없으면 메시지 없이 Step 3으로 바로 진행한다.

### Step 3: Phase 순회 루프

PLAN 문서를 읽어 Phase 목록을 추출한다. Status가 ⏳ Pending 또는 🔄 In Progress인 첫 번째 Phase부터 시작한다.

```
각 Phase에 대해:
    3a. Phase 상태를 🔄 In Progress로 변경
    3b. TDD 개발 실행
    3c. 검증 실행
    3d. 코드 리뷰
    3e. 검증+리뷰 통과 시 → Phase 상태를 ✅ Complete로 변경, 다음 Phase로
        검증 실패 시 → 수정 후 3c 재실행
        리뷰 REQUEST_CHANGES → 수정 후 3c 재실행
        검증+리뷰 합산 3회 실패 시 → 사용자에게 블로커 보고, 계속/중단 선택
```

### Step 3a: Phase 상태 업데이트

PLAN 문서에서 현재 Phase의 Status를 🔄 In Progress로 변경한다.
Last Updated 날짜도 오늘로 갱신한다.

**Phase 시작 git ref 기록:**
```bash
git rev-parse HEAD
```
결과를 PLAN 문서의 해당 Phase `Phase Start Ref` 필드에 기록한다. 이 ref는 이후 verify-implementation의 TDD Compliance Check(Step 3c)와 code-reviewer(Step 3d)에서 diff 범위로 사용된다.

### Step 3b: TDD 개발 실행

PLAN 문서의 해당 Phase Tasks 섹션을 읽고, 순서대로 수행한다:

**🔴 RED — 실패하는 테스트 작성:**
- Phase의 RED 섹션에 나열된 테스트 파일을 작성한다
- 테스트가 실패하는지 확인한다 (테스트 명령어 실행)
- 실패 확인 후 커밋: `test: add failing tests for Phase N - <name>`

**🟢 GREEN — 테스트 통과시키는 최소 구현:**
- Phase의 GREEN 섹션에 나열된 소스 파일을 구현한다
- 테스트가 통과하는지 확인한다
- 통과 확인 후 커밋: `feat: implement Phase N - <name>`

**🔵 REFACTOR — 코드 품질 개선:**
- Phase의 REFACTOR 섹션 체크리스트를 수행한다
- 테스트가 여전히 통과하는지 확인한다
- 커밋: `refactor: clean up Phase N - <name>`

**각 단계에서 커밋 메시지에 TDD 키워드를 포함하여, 추후 test-runner의 TDD Compliance Check가 자동 감지할 수 있게 한다.**

체크박스 업데이트:
- 각 Task 완료 시 PLAN 문서의 해당 체크박스를 체크한다

### Step 3c: 검증 실행 (verify-implementation 자동 호출)

`verify-implementation` 스킬을 현재 Phase로 실행한다.

**자동 전달 항목:**
- 인수: `phase-N` (현재 Phase 번호)

**검증 항목:**
- Quality Gate 전체 (빌드, 테스트, 린트, 타입체크)
- TDD Compliance (Test-First 순서, R-G-R 증거, REFACTOR 측정, 커버리지)

**결과 분기:**

```
✅ 전체 PASS → Step 3d로 이동 (코드 리뷰)
❌ FAIL 있음 → 사용자에게 액션 확인 (verify-implementation의 Step 4)
               → 수정 적용 후 재검증
⚠️ WARN만   → 경고 표시 후 Step 3d로 이동 (코드 리뷰, 진행 가능)
```

### Step 3d: 코드 리뷰 (code-reviewer 자동 호출)

verify-implementation이 PASS한 후, `code-reviewer` Subagent를 phase 모드로 실행한다.

> **사용하는 Subagent**: `agents/code-reviewer.md`

**전달 항목:**
- 모드: phase
- Phase 번호
- PLAN 문서 경로
- Phase 시작 git ref (PLAN 문서의 `Phase Start Ref` 필드에서 읽음)

**결과 분기:**

```
APPROVE           → PLAN의 Verification Status 테이블에 Review 결과 기록 → Step 3e로 이동
COMMENT           → Review 결과 기록 → 제안 사항을 사용자에게 표시, Step 3e로 이동 (진행 가능)
REQUEST_CHANGES   → Review 결과 기록 → 이슈를 사용자에게 표시하고, AskUserQuestion으로 확인:
                    1. 자동 수정 — 리뷰 이슈를 반영하여 코드 수정 후 3c 재실행
                    2. 건너뛰기 — 리뷰 이슈를 무시하고 3e로 이동
```

**PLAN Verification Status 업데이트:**

code-reviewer 결과를 받은 후, PLAN 문서의 Verification Status 테이블에서 해당 Phase의 `Review` 컬럼을 업데이트한다 (APPROVE / COMMENT / REQUEST_CHANGES).

### Step 3e: Phase 완료 처리

검증+리뷰 통과 후:
1. PLAN 문서에서 Phase Status를 ✅ Complete로 변경
2. Progress Tracking 섹션 업데이트 (퍼센트 갱신)
3. 다음 Phase가 있으면 → Step 3a로 돌아감
4. 모든 Phase 완료 시 → Step 3.5로 이동

### Step 3.5: 통합 코드 리뷰

모든 Phase가 완료된 후, Step 4 진입 전에 `code-reviewer` Subagent를 integration 모드로 실행한다.

> **사용하는 Subagent**: `agents/code-reviewer.md`

**전달 항목:**
- 모드: integration
- PLAN 문서 경로
- 기능 시작 git ref (PLAN 문서의 `Feature Start Ref` 필드에서 읽음)

**결과 분기:**

```
APPROVE           → Step 4로 이동
COMMENT           → 제안 사항을 사용자에게 표시, Step 4로 이동
REQUEST_CHANGES   → 이슈를 사용자에게 표시하고, AskUserQuestion으로 확인:
                    1. 수정 — 코드 수정 후, 영향받은 Phase 스킬만 재검증
                    2. 건너뛰기 — Step 4로 이동
```

### Step 4: 완료 처리 (feature-planner COMPLETE 자동 호출)

모든 Phase가 완료되면 `feature-planner` 스킬을 COMPLETE 모드로 실행한다.

**자동 전달 항목:**
- 인수: `complete PLAN_<name>`

**수행 내용:**
- PLAN 문서 Status를 ✅ Completed로 변경

### Step 5: 스킬 유지보수 (manage-skills 자동 호출)

완료 처리 후 `manage-skills` 스킬을 자동으로 실행한다.

**수행 내용:**
- verify-phase-N-* 스킬을 범용 verify-* 스킬로 통합 (기존 Phase 스킬 삭제)
- 전체 변경 파일의 스킬 커버리지 점검
- 커버되지 않은 파일에 대한 새 스킬 제안
- skill-registry.json 정합성 검증

### Step 6: 최종 보고

전체 파이프라인 완료 후 요약 보고서를 출력한다:

```markdown
## 개발 완료 보고서

**기능**: [기능명]
**PLAN**: PLAN_<name>
**소요 시간**: [시작 ~ 종료]

### Phase 실행 결과

| Phase | 상태 | TDD 준수 | 커버리지 | 검증 시도 | 리뷰 |
|-------|------|---------|---------|----------|------|
| Phase 1: 모델 설계 | ✅ Complete | ✅ | 92% | 1회 | APPROVE |
| Phase 2: API 구현 | ✅ Complete | ✅ | 87% | 2회 | COMMENT (2건) |
| Phase 3: UI 연동 | ✅ Complete | ⚠️ | 81% | 1회 | APPROVE |

### 통합 코드 리뷰
- 결과: APPROVE
- Cross-Phase 이슈: 0건

### 생성된 산출물
- 계획 문서: `docs/plans/PLAN_auth.md`
- 생성된 스킬: verify-auth, verify-api (범용 스킬로 통합)
- 총 커밋: 15개 (TDD 패턴 준수)

### 스킬 상태
- 통합: 3개 (Phase 스킬 → 범용 스킬)
- 삭제: 3개 (기존 Phase 스킬)
- 활성: 2개 (범용 verify 스킬)
```

## 재개 모드

이전에 중단된 개발을 이어서 진행할 수 있다:

```
/dev                   → 기존 PLAN 감지 시 재개 옵션 제시
/dev PLAN_auth         → 특정 PLAN의 미완료 Phase부터 재개
```

재개 시:
1. PLAN 문서를 읽어 마지막 ✅ Complete Phase를 찾는다
2. 다음 Phase부터 Step 3a로 진입한다
3. 🔄 In Progress Phase가 있으면 해당 Phase의 미완료 Task부터 이어간다

## 에러 핸들링

**빌드/테스트 실패가 3회 연속일 때:**

```markdown
### ⚠️ 블로커 발생

Phase 2 검증이 3회 연속 실패했습니다.

**마지막 실패 사유:**
- verify-phase-2-api: 통합 테스트 2개 실패
  - `POST /api/auth/login` 응답 형식 불일치
  - `GET /api/auth/me` 인증 토큰 검증 실패

선택하세요:
1. **계속 시도** — 다른 접근법으로 수정 시도
2. **이 Phase 건너뛰기** — 다음 Phase로 이동 (권장하지 않음)
3. **중단** — 현재 상태 저장 후 개발 중단
```

**의존성 설치 실패, 환경 문제 등:**
- 에러 내용을 사용자에게 보고하고 해결 방법을 제안한다
- 사용자가 해결한 후 `/dev PLAN_<name>`으로 재개할 수 있다

### 서브에이전트 실패 처리

파이프라인 중 서브에이전트가 실패(응답 없음, 에러 반환, 타임아웃)할 때의 전략:

**Step 2 — feature-planner 내부:**

| 서브에이전트 | 실패 시 전략 |
|-------------|-------------|
| codebase-scanner | 폴백: 직접 `ls`, `cat package.json` 등으로 최소 컨텍스트 수집 후 계속 |
| skill-writer (병렬 N개) | 성공한 스킬은 유지, 실패한 스킬만 순차 재시도 1회. 재실패 시 해당 Phase의 verify 스킬을 "수동 생성 필요"로 표시하고 계속 진행 |

**Step 3c — verify-implementation 내부:**

| 서브에이전트 | 실패 시 전략 |
|-------------|-------------|
| test-runner (병렬 N개) | 실패한 스킬을 순차 재시도 1회. 재실패 시 해당 스킬을 SKIP 처리하고, 통합 리포트에 "⚠️ 검증 불가 (에이전트 실패)" 표시. 나머지 스킬 결과로 PASS/FAIL 판정 |

**Step 3d — code-reviewer:**

| 서브에이전트 | 실패 시 전략 |
|-------------|-------------|
| code-reviewer (phase) | 재시도 1회. 재실패 시 "⚠️ 코드 리뷰 미완료"로 표시하고 Step 3e로 진행 (비핵심 경로) |

**Step 3.5 — code-reviewer:**

| 서브에이전트 | 실패 시 전략 |
|-------------|-------------|
| code-reviewer (integration) | 재시도 1회. 재실패 시 사용자에게 "통합 리뷰 실패 — COMPLETE 진행 여부" 확인 |

**Step 5 — manage-skills 내부:**

| 서브에이전트 | 실패 시 전략 |
|-------------|-------------|
| skill-writer (병렬 N개) | 성공한 스킬은 유지, 실패한 스킬만 순차 재시도 1회. 재실패 시 보고서에 "업데이트 실패" 표시 |

**공통 원칙:**
1. 재시도는 최대 1회, 동일 입력으로 실행
2. 핵심 경로(feature-planner의 PLAN 생성)의 실패는 해당 단계를 중단하고 사용자에게 보고
3. 비핵심 경로(code-reviewer phase)의 실패는 건너뛰고 계속 진행
4. 병렬 실행 중 부분 실패는 성공한 결과를 유지하고 실패분만 재시도
5. 모든 실패는 최종 보고서의 "에이전트 실행 상태" 섹션에 기록

## Exceptions

다음은 이 스킬이 처리하지 않는다:

1. **배포** — CI/CD 파이프라인 실행, 프로덕션 배포는 범위 밖
2. **PR 생성** — git push와 PR 생성은 사용자에게 안내만 한다
3. **DB 마이그레이션 실행** — 마이그레이션 파일은 생성하지만 실행은 사용자 확인 후
4. **환경 변수 설정** — `.env` 파일에 필요한 변수를 안내하지만 실제 값은 사용자가 입력

## Related Files

| File | Purpose |
|------|---------|
| `skills/feature-planner/SKILL.md` | Step 2, 4: 계획 수립 및 완료 처리 |
| `skills/verify-implementation/SKILL.md` | Step 3c: Phase별 검증 실행 |
| `skills/manage-skills/SKILL.md` | Step 5: 스킬 유지보수 |
| `agents/codebase-scanner.md` | feature-planner/manage-skills가 사용 |
| `agents/skill-writer.md` | feature-planner/manage-skills가 사용 |
| `agents/test-runner.md` | verify-implementation이 사용 |
| `agents/code-reviewer.md` | Step 3d, 3.5: Phase별/통합 코드 리뷰 |
| `docs/plans/PLAN_*.md` | 계획 문서 (생성/읽기/업데이트) |
