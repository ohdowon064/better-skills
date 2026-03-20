# better-skills

Claude Code를 위한 자율 개발 파이프라인 플러그인.

기획서를 넣으면 계획 수립, TDD 개발, 검증, 스킬 유지보수까지 자동으로 실행됩니다.

```
/dev 사용자 인증 시스템을 구현해줘. 로그인/회원가입/비밀번호 재설정이 필요해.
```

## 요구사항

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI가 설치되어 있어야 합니다.

## 설치

```bash
# 프로젝트 스코프로 설치 (해당 프로젝트에서만 사용)
claude plugin install /path/to/better-skills --scope project

# 사용자 스코프로 설치 (모든 프로젝트에서 사용)
claude plugin install /path/to/better-skills

# GitHub에서 직접 설치
claude plugin install github:ohdowon064/better-skills --scope project
```

설치 후 Claude Code가 `skills/`와 `agents/` 디렉토리를 자동으로 인식합니다.

## 빠른 시작

### 기능 하나를 통째로 개발

```bash
# 기획서를 텍스트로 전달
/dev 사용자 인증 시스템을 구현해줘. 로그인/회원가입/비밀번호 재설정이 필요해.

# 마크다운 파일로 전달
/dev docs/auth-spec.md

# 중단된 작업 재개
/dev PLAN_auth
```

사용자가 개입하는 지점은 3곳뿐입니다:

1. **계획 승인** — Phase 분해가 적절한지 확인
2. **FAIL 시 액션 선택** — 자동 수정 / 개별 리뷰 / 건너뛰기
3. **예상치 못한 블로커** — 해결 방법 결정

### 개별 스킬 직접 호출

```bash
# 계획만 수립
/feature-planner

# 기존 계획 수정
/feature-planner update PLAN_auth

# 계획 완료 처리
/feature-planner complete PLAN_auth

# 검증만 실행
/verify-implementation
/verify-implementation phase-1
/verify-implementation --current-phase

# 스킬 점검
/manage-skills

# 코드 리뷰
/code-review
/code-review --staged
/code-review --branch feature/auth
```

## 파이프라인 흐름

```
기획서 입력 → 계획 수립 → Phase별 TDD 개발 → 검증 → 코드 리뷰 → 통합 리뷰 → 완료 처리 → 스킬 정리
                │              │            │          │          │
                │              │            │          │          └─ code-reviewer (integration)
                │              │            │          └─ code-reviewer (phase)
                │              │            └─ verify-implementation
                │              └─ RED → GREEN → REFACTOR
                └─ feature-planner
```

`/dev`가 이 전체 흐름을 자동으로 오케스트레이션합니다.

## 스킬

| 스킬 | 설명 |
|------|------|
| `/dev` | 전체 파이프라인 오케스트레이터. 기획서 → 완성된 기능까지 자동 실행 |
| `/feature-planner` | 기능을 3~7개 Phase로 분해. CREATE / UPDATE / COMPLETE 모드 |
| `/verify-implementation` | verify 스킬을 서브에이전트로 병렬 실행하여 통합 검증 리포트 생성 |
| `/manage-skills` | 코드 변경에 맞춰 verify 스킬의 드리프트 탐지 및 수정 |
| `/code-review` | 코드 리뷰. 최근 변경, 특정 경로, staged, 브랜치 diff 지원 |

## 서브에이전트

| 에이전트 | 역할 | 사용처 |
|---------|------|--------|
| `codebase-scanner` | 프로젝트 환경 통합 분석 (구조, 테스트, 린터, CI/CD, git 변경사항) | feature-planner, manage-skills |
| `skill-writer` | verify 스킬 생성/업데이트 (병렬 실행) | feature-planner, manage-skills |
| `test-runner` | 개별 verify 스킬 실행 + TDD 순서 검증 | verify-implementation |
| `code-reviewer` | 코드 설계/보안/성능 리뷰 (Phase별 + 통합) | dev, code-review |

## 주요 개념

### TDD 내장

TDD는 권장이 아니라 파이프라인의 필수 단계입니다.

- 모든 Phase가 RED → GREEN → REFACTOR 순서로 진행
- git 이력 기반으로 Test-First 순서를 자동 검증
- REFACTOR 효과를 정량 측정 (파일 길이, 함수 수, 평균 함수 길이, 중복)

### 스킬 라이프사이클

스킬의 상태는 레지스트리 존재 여부로 결정됩니다:

- **레지스트리에 있음** = 활성 (verify-implementation 실행 대상)
- **레지스트리에 없음** = 비활성 (이력은 git에 보존)

Phase 완료 시 `verify-phase-N-*` 스킬은 범용 `verify-*` 스킬로 **통합**됩니다 (새 스킬 생성 + 기존 삭제). 불필요한 스킬은 레지스트리와 파일에서 삭제되며, 필요 시 `git log`로 복구할 수 있습니다.

### Exceptions 자동 제안

검증에서 이슈를 Skip하면 해당 검사를 스킬의 Exceptions에 추가할지 즉시 확인합니다. 승인하면 스킬이 자동 업데이트되어 동일한 false positive가 반복되지 않습니다.

### 프로젝트 환경 자동 감지

`codebase-scanner`가 레포 타입을 자동 판별합니다:

| 타입 | 조건 |
|------|------|
| BRAND_NEW | git 미초기화 또는 커밋 0개 |
| FRESH_START | 커밋 5개 이하, 소스 코드 없음 |
| CONFIG_ONLY | 커밋 있음, 설정 파일만 존재 |
| EXISTING | 소스 코드 존재 |

감지된 실제 명령어가 Quality Gate에 채워집니다. `npm test` 같은 플레이스홀더가 아니라 프로젝트의 실제 명령어가 들어갑니다.

### SSOT 레지스트리

`.claude/skill-registry.json`이 verify 스킬 목록의 유일한 정식 소스입니다. verify-implementation, feature-planner는 모두 이 파일을 런타임에 참조합니다.

## 디렉토리 구조

### 플러그인 (이 레포)

```
better-skills/
├── .claude-plugin/
│   ├── plugin.json                     # 플러그인 매니페스트
│   └── marketplace.json                # 마켓플레이스 등록 정보
├── skills/
│   ├── dev/SKILL.md                    # 파이프라인 오케스트레이터
│   ├── feature-planner/
│   │   ├── SKILL.md                    # 계획 수립
│   │   └── plan-template.md            # 계획 문서 템플릿
│   ├── verify-implementation/SKILL.md  # 통합 검증 엔진
│   ├── manage-skills/SKILL.md          # 스킬 유지보수
│   └── code-review/SKILL.md            # 코드 리뷰 (사용자 직접 호출)
├── agents/
│   ├── codebase-scanner.md             # 프로젝트 컨텍스트 분석
│   ├── skill-writer.md                 # verify 스킬 생성/수정
│   ├── test-runner.md                  # 검증 실행 (병렬)
│   └── code-reviewer.md               # 코드 리뷰 (설계/보안/성능)
```

### 런타임에 프로젝트에 생성되는 파일

```
your-project/
├── .claude/
│   ├── skills/verify-*/SKILL.md        # 자동 생성된 검증 스킬
│   └── skill-registry.json             # 스킬 레지스트리 SSOT
└── docs/plans/PLAN_*.md                # 계획 문서
```

## 사용 예시

### 빈 프로젝트에서 시작

```bash
mkdir my-app && cd my-app && git init
claude plugin install /path/to/better-skills --scope project

/dev Todo 앱의 백엔드 API를 만들어줘.
# → 기술 스택 선택 → Phase 분해 → 승인 → 자동 개발
```

### 기존 프로젝트에 기능 추가

```bash
cd my-existing-app
claude plugin install /path/to/better-skills --scope project

/dev 검색 기능을 추가해줘. 제목과 본문에서 키워드 검색이 필요해.
# → 기존 코드 분석 → 기존 패턴 준수 → Phase 분해 → 자동 개발
```

### 복수 기능 병렬 개발

```bash
/dev 인증 시스템 만들어줘.
# (인증 개발 중단)
/dev 결제 시스템 만들어줘.
# → 기존 PLAN_auth와 파일 충돌 감지 → 사용자 선택
```

### 개별 스킬 활용

```bash
# 계획만 수립하고 검토
/feature-planner

# 기존 계획에 Phase 추가
/feature-planner update PLAN_auth

# 현재 Phase만 검증
/verify-implementation --current-phase

# 스킬 커버리지 점검
/manage-skills
```

## 에러 핸들링

### 블로커 자동 보고

빌드/테스트 실패가 3회 연속이면 블로커 보고서와 함께 선택지를 제시합니다:

1. **계속 시도** — 다른 접근법으로 수정
2. **Phase 건너뛰기** — 다음 Phase로 이동
3. **중단** — 현재 상태 저장 후 `/dev PLAN_<name>`으로 나중에 재개

### 서브에이전트 실패

| 경로 | 전략 |
|------|------|
| 핵심 경로 (feature-planner PLAN 생성) | 실패 시 사용자에게 보고 후 중단 |
| 병렬 실행 (skill-writer, test-runner) | 성공한 결과 유지, 실패분만 순차 재시도 1회 |
| 비핵심 경로 (code-reviewer) | 재시도 1회, 재실패 시 건너뛰고 계속 진행 |
| 폴백 가능 (codebase-scanner) | 직접 `git diff`, `ls` 등으로 최소 컨텍스트 수집 후 계속 |

## Cross-Skill 추천

각 스킬은 실행 결과에 따라 다른 스킬의 실행을 자동으로 추천합니다:

- `verify-implementation` → 커버되지 않은 파일 발견 시 `/manage-skills` 추천
- `manage-skills` → 새 스킬 생성/수정 후 `/verify-implementation` 추천
- 스킬/계획이 없으면 → `/feature-planner` 추천

## 라이선스

MIT
