# better-skills

Claude Code 플러그인 형태의 자율 개발 파이프라인 스킬 시스템입니다.

## Development Workflow

**기본 사용법:** `/dev`에 기획서나 기능 요구사항을 전달하면, 계획 수립부터 TDD 개발, 검증, 스킬 정리까지 자동으로 실행됩니다. 사용자는 계획 승인과 FAIL 시 액션 선택에만 개입하면 됩니다.

```
/dev 사용자 인증 시스템을 구현해줘. 로그인/회원가입/비밀번호 재설정이 필요해.
```

**개별 스킬 직접 호출도 가능합니다:**

1. **계획 수립**: `/feature-planner` — Phase 기반 개발 계획 수립, verify 스킬 자동 생성
2. **개발 + 검증**: `/verify-implementation` — Quality Gate 검증 (Phase별 또는 전체)
3. **스킬 유지보수**: `/manage-skills` — 검증 스킬을 코드베이스 변화에 맞춰 관리

## Skills

| 스킬 | 설명 |
|------|------|
| `dev` | **전체 파이프라인 오케스트레이터** — 기획서 입력 → 계획 → TDD 개발 → 검증 → 완료까지 자동 실행 |
| `feature-planner` | 기능 요구사항을 Phase 기반 TDD 개발 계획으로 분해하고, Phase별 verify 스킬을 자동 생성 |
| `verify-implementation` | 등록된 verify 스킬을 Subagent로 병렬 실행하여 통합 검증 리포트 생성 |
| `manage-skills` | 세션 변경사항/계획 문서를 분석하여 verify 스킬의 드리프트를 탐지하고 수정 |

## Auto-Recommendations

각 스킬은 실행 결과에 따라 다른 스킬의 실행을 자동으로 추천합니다:

- `/verify-implementation` 실행 후 → 커버되지 않은 변경 파일 발견 시 `/manage-skills` 추천
- `/manage-skills` 실행 후 → 신규/수정 스킬이 생성되면 `/verify-implementation` 추천
- 두 스킬 모두 → 계획/스킬이 없으면 `/feature-planner` 추천

추천이 표시되면 따르는 것을 권장합니다. 이 피드백 루프가 스킬 시스템의 자가 진화를 가능하게 합니다.

## Conventions

- 검증 스킬은 `verify-` 접두사를 사용합니다 (예: `verify-auth`, `verify-build`)
- 계획 문서는 `docs/plans/PLAN_<feature-name>.md` 형식으로 저장됩니다
- 스킬 라이프사이클: CREATED → ACTIVE → GRADUATED → ARCHIVED
