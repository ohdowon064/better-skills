# better-skills 개발 가이드

이 레포는 Claude Code 플러그인입니다. `skills/`와 `agents/`가 플러그인으로 제공되고, 런타임 데이터는 사용자 프로젝트의 `.claude/`에 생성됩니다.

## 경로 규칙

| 경로 | 위치 | 설명 |
|------|------|------|
| `skills/<core>/SKILL.md` | 플러그인 | 코어 스킬 (dev, feature-planner, verify-implementation, manage-skills) |
| `agents/*.md` | 플러그인 | 서브에이전트 정의 |
| `.claude/skills/verify-*/SKILL.md` | 사용자 프로젝트 | 런타임에 자동 생성되는 검증 스킬 |
| `.claude/skill-registry.json` | 사용자 프로젝트 | 스킬 레지스트리 SSOT |
| `.claude/verify-history.json` | 사용자 프로젝트 | 검증 실행 이력 |
| `docs/plans/PLAN_*.md` | 사용자 프로젝트 | 계획 문서 |

스킬/에이전트 내부에서 다른 플러그인 파일을 참조할 때는 `skills/`, `agents/` 접두사를 사용합니다. 사용자 프로젝트의 런타임 파일을 참조할 때는 `.claude/` 접두사를 사용합니다.

## SSOT

`.claude/skill-registry.json`이 verify 스킬 목록의 유일한 정식 소스입니다. 스킬 목록을 다른 곳에 중복 기록하지 마세요.

## 에이전트 쓰기 책임

| 디렉토리 | 쓰기 권한 |
|---------|----------|
| `.claude/skills/verify-*/` | skill-writer만 |
| `.claude/skill-registry.json` | feature-planner, manage-skills |
