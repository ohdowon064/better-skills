# better-skills

Claude Code를 위한 자율 개발 파이프라인 스킬 시스템.

기획서를 넣으면 계획 수립, TDD 개발, 검증, 스킬 유지보수까지 자동으로 실행됩니다.

```
/dev 사용자 인증 시스템을 구현해줘. 로그인/회원가입/비밀번호 재설정이 필요해.
```

---

## 기능

### `/dev` — 원커맨드 파이프라인

하나의 명령으로 전체 개발 사이클이 돌아갑니다.

```
기획서 입력 → 계획 수립 → Phase별 TDD 개발 → 검증 → 완료 처리 → 스킬 정리
```

사용자가 개입하는 지점은 3곳뿐입니다: 계획 승인, FAIL 시 액션 선택, 예상치 못한 블로커. 나머지는 Claude가 자율적으로 처리합니다.

텍스트, 마크다운 파일, 기존 PLAN 재개 등 다양한 입력 형식을 지원합니다.

```
/dev docs/auth-spec.md          # 파일로 전달
/dev PLAN_auth                  # 중단된 작업 재개
```

### `/feature-planner` — Phase 기반 계획 수립

기능 요구사항을 3~7개 Phase로 자동 분해합니다. 각 Phase는 독립적으로 동작하는 단위이며, Red-Green-Refactor TDD 사이클이 내장되어 있습니다.

프로젝트 컨텍스트를 자동 감지하여 빈 레포부터 기존 레포까지 모두 대응합니다. `codebase-scanner` 서브에이전트가 언어, 패키지 매니저, 테스트 프레임워크, 린터, CI/CD 설정을 한 번에 분석하고, 감지된 실제 명령어가 Quality Gate에 자동으로 채워집니다.

계획이 확정되면 Phase별 verify 스킬이 `skill-writer` 서브에이전트를 통해 병렬로 생성됩니다.

CREATE, UPDATE, COMPLETE, LIST 4가지 모드를 지원합니다. LIST 모드에서는 복수 PLAN 간 파일 충돌도 감지합니다.

### `/verify-implementation` — 병렬 검증 엔진

등록된 모든 verify 스킬을 `test-runner` 서브에이전트로 동시에 실행합니다. 5개 스킬이 있으면 순차 실행 대비 5배 빠릅니다. 7개 초과 시 배치로 나눠 실행합니다.

검증 결과를 통합 리포트로 취합하고, 이슈 발견 시 자동 수정 / 개별 리뷰 / 건너뛰기를 선택할 수 있습니다. 수정 후에는 영향받은 스킬만 재실행하여 Before/After를 비교합니다.

TDD Compliance Check가 내장되어 있어 git 이력 기반으로 Test-First 순서, Red-Green-Refactor 사이클 증거, REFACTOR 정량 측정(파일 길이, 함수 수, 중복, 평균 함수 길이), 신규 코드 커버리지를 자동 검증합니다.

### `/manage-skills` — 스킬 유지보수 & 레지스트리 관리

코드베이스 변화에 맞춰 verify 스킬의 드리프트를 탐지하고 수정합니다. `.claude/skill-registry.json`이 전체 시스템의 Single Source of Truth(SSOT)로 기능하며, verify-implementation과 CLAUDE.md는 이 파일을 런타임에 참조합니다.

스킬이 없는 콜드 스타트(빈 레포, 기존 레포 모두)도 처리합니다. evals.json과 실제 스킬 내용의 불일치를 감지하는 Evals Drift Detection도 포함되어 있습니다.

---

## 장점

### 자율 실행, 최소 개입

기획서 하나만 넣으면 됩니다. Claude가 계획을 세우고, 테스트를 먼저 작성하고, 구현하고, 검증하고, 실패하면 수정하고, 완료되면 스킬을 정리합니다. 사용자는 의사결정이 필요한 3곳에서만 개입합니다.

### TDD가 프로세스에 내장

TDD는 권장 사항이 아니라 파이프라인의 필수 단계입니다. 모든 Phase가 RED(실패하는 테스트 작성) → GREEN(최소 구현) → REFACTOR(코드 개선) 순서로 진행되며, git 이력 기반으로 Test-First 순서가 자동 검증됩니다. REFACTOR의 효과도 파일 길이, 함수 수, 중복, 평균 함수 길이를 before/after로 정량 측정합니다.

### 서브에이전트 병렬 실행

verify 스킬 검증, 스킬 생성, 코드베이스 분석을 서브에이전트가 병렬로 처리합니다. 순차 실행 대비 검증 시간이 스킬 수에 비례하여 단축됩니다.

### 자가 진화하는 스킬 시스템

스킬은 정적이지 않습니다. 사용 이력에 따라 효과성이 분석되고, 연속 PASS 10회 이상이면 GRADUATE 후보로, Skip 비율이 높으면 Exceptions 업데이트를 제안합니다. 3회 연속 Skip된 검사 항목은 False Positive으로 학습하여 자동으로 면제 추가를 제안합니다.

### 스킬 라이프사이클 관리

모든 verify 스킬은 CREATED → ACTIVE → GRADUATED → ARCHIVED 생애주기를 따릅니다. Phase 개발 중에는 `verify-phase-N-*` 형태로 활동하고, Phase 완료 후에는 범용 `verify-*` 스킬로 승격(GRADUATE)됩니다. 더 이상 필요 없는 스킬은 ARCHIVED되어 실행 대상에서 자동 제외됩니다.

### 버전 관리와 롤백

스킬이 업데이트될 때마다 `.claude/skill-versions/`에 스냅샷이 자동 보관됩니다. 업데이트 후 문제가 발생하면 이전 버전과 diff를 비교하거나 즉시 롤백할 수 있습니다. 최대 10개 버전을 유지하고 오래된 것부터 자동 정리됩니다.

---

## 특징

### 프로젝트 환경 자동 감지

`codebase-scanner` 서브에이전트가 프로젝트를 분석하여 레포 타입(BRAND_NEW, FRESH_START, CONFIG_ONLY, EXISTING)을 판별합니다. 패키지 매니저, 테스트 프레임워크, 린터, 타입 체커, CI/CD를 자동으로 감지하고, 감지된 실제 명령어가 Quality Gate에 채워집니다. `npm test` 같은 플레이스홀더가 아니라 프로젝트의 실제 명령어가 들어갑니다.

### 복수 PLAN 동시 진행

여러 기능을 병렬로 개발할 수 있습니다. LIST 모드에서 진행 중인 PLAN들의 수정 대상 파일을 교차 비교하여 충돌을 사전에 감지합니다. `--current-phase`로 검증할 때 복수 PLAN이 있으면 선택할 수 있습니다.

### Cross-Skill 추천 피드백 루프

각 스킬이 실행 결과에 따라 다른 스킬의 실행을 자동으로 추천합니다. verify-implementation이 커버되지 않은 파일을 발견하면 manage-skills를, manage-skills가 새 스킬을 생성하면 verify-implementation을 추천합니다. 이 피드백 루프가 스킬 시스템의 자가 진화를 만듭니다.

### Evals 드리프트 감지

`evals/evals.json`의 테스트 케이스와 실제 스킬 내용의 불일치를 3가지 유형으로 감지합니다: UNCOVERED(새 기능에 테스트 없음), STALE(삭제된 기능을 참조), OUTDATED(현재 동작과 불일치). 자동 업데이트 옵션으로 STALE 제거, OUTDATED 갱신, UNCOVERED 테스트 초안 생성이 가능합니다.

### 재개 모드

중단된 개발을 이어서 진행할 수 있습니다. `/dev PLAN_auth`로 특정 PLAN의 마지막 완료된 Phase 다음부터 재개합니다. In Progress 상태의 Phase가 있으면 미완료 Task부터 이어갑니다.

### 블로커 자동 보고

빌드/테스트 실패가 3회 연속이면 블로커 보고서와 함께 계속 시도 / Phase 건너뛰기 / 중단 옵션을 제시합니다. 의존성 설치 실패나 환경 문제도 해결 방법을 제안하고, 해결 후 재개할 수 있습니다.

---

## 구조

```
.claude/
├── skills/
│   ├── dev/SKILL.md                    # 파이프라인 오케스트레이터
│   ├── feature-planner/
│   │   ├── SKILL.md                    # 계획 수립
│   │   └── plan-template.md            # 계획 문서 템플릿
│   ├── verify-implementation/SKILL.md  # 통합 검증 엔진
│   ├── manage-skills/SKILL.md          # 스킬 유지보수
│   └── verify-*/SKILL.md              # 자동 생성되는 검증 스킬들
├── agents/
│   ├── codebase-scanner.md             # 프로젝트 컨텍스트 분석
│   ├── plan-writer.md                  # 계획 문서 작성
│   ├── skill-writer.md                 # verify 스킬 생성/수정
│   ├── test-runner.md                  # 검증 실행 (병렬)
│   ├── version-manager.md              # 스킬 버전 이력/롤백/정리
│   └── evals-checker.md                # evals 드리프트 감지/수정
├── skill-registry.json                 # 스킬 레지스트리 SSOT (verify 스킬 목록/메타데이터)
├── skill-versions/                     # 스킬 버전 스냅샷
└── verify-history.json                 # 검증 실행 이력 (최근 100건 rotate)
```

## 라이선스

MIT
