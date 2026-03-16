# Project Guidelines

이 프로젝트는 통합 개발 라이프사이클 스킬 시스템을 사용합니다.

## Development Workflow

1. **계획 수립**: `/feature-planner`로 Phase 기반 개발 계획을 세우고, verify 스킬이 자동 생성됩니다
2. **개발 진행**: 각 Phase를 TDD (Red-Green-Refactor) 사이클로 구현합니다
3. **검증 실행**: `/verify-implementation`으로 Quality Gate를 검증합니다 (Phase별 또는 전체)
4. **스킬 유지보수**: `/manage-skills`로 검증 스킬을 코드베이스 변화에 맞춰 관리합니다

## Skills

| 스킬 | 설명 |
|------|------|
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
