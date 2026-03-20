# better-skills 프로젝트 전체 검토 요청

## 프로젝트 개요

이 레포는 Claude Code 플러그인으로, 기획서 하나를 입력하면 계획 수립 → TDD 개발 → 검증 → 코드 리뷰 → 스킬 유지보수까지 자동으로 오케스트레이션하는 자율 개발 파이프라인입니다.

## 파일 구조

```
better-skills/
├── CLAUDE.md                              # 개발 가이드 (경로 규칙, SSOT, 쓰기 책임)
├── README.md                              # 프로젝트 문서
├── skills/
│   ├── dev/SKILL.md                       # 전체 파이프라인 오케스트레이터
│   ├── feature-planner/
│   │   ├── SKILL.md                       # 계획 수립 (CREATE/UPDATE/COMPLETE/LIST)
│   │   └── plan-template.md               # PLAN 문서 템플릿
│   ├── verify-implementation/SKILL.md     # 통합 검증 엔진 (병렬 실행)
│   ├── manage-skills/SKILL.md             # 스킬 드리프트 탐지 & 유지보수
│   └── code-review/SKILL.md               # 사용자 직접 호출 코드 리뷰
├── agents/
│   ├── codebase-scanner.md                # 프로젝트 환경 분석 (sonnet)
│   ├── plan-writer.md                     # 계획 문서 작성 (sonnet)
│   ├── skill-writer.md                    # verify 스킬 CRUD (sonnet)
│   ├── test-runner.md                     # 검증 실행 + TDD 순서 검증 (sonnet)
│   └── code-reviewer.md                   # 설계/보안/성능 리뷰 (opus)
└── .claude-plugin/
    └── plugin.json                        # 플러그인 매니페스트
```

## 파이프라인 흐름

```
/dev "기능 설명"
  → feature-planner (CREATE)
      → codebase-scanner → 사용자 승인 → plan-writer → skill-writer ×N
  → Phase 루프 ×3~7
      → 3a. Phase 상태 업데이트
      → 3b. TDD 개발 (RED → GREEN → REFACTOR)
      → 3c. verify-implementation → test-runner ×N 병렬
      → 3d. code-reviewer (phase 모드)
      → 3e. Phase 완료
  → 3.5. code-reviewer (integration 모드)
  → feature-planner (COMPLETE) — Phase 스킬 → 범용 스킬 통합
  → manage-skills — 커버리지 점검, 레지스트리 정합성
  → 최종 보고서
```

## 런타임 데이터 (사용자 프로젝트의 .claude/)

```
.claude/
├── skills/verify-*/SKILL.md     # 자동 생성된 검증 스킬
├── skill-registry.json          # 스킬 레지스트리 SSOT
docs/plans/PLAN_*.md             # 계획 문서
```

## 최근 완료된 개선사항

1. **version-manager 제거** — git이 이미 버전 관리를 하므로 .claude/skill-versions/ 스냅샷 시스템 제거
2. **lifecycle 제거** — CREATED/ACTIVE/GRADUATED/ARCHIVED 4단계 → 레지스트리에 존재하면 활성, 삭제하면 비활성. skill-registry.json에서 lifecycle, updatedAt 필드 제거
3. **verify-history.json 제거** — 효과성 분석 지표(FAIL 비율, 연속 PASS, Skip 비율)가 실질적 액션을 트리거하지 않음. false positive 학습은 Skip 시점에 즉석 Exceptions 제안으로 대체
4. **code-reviewer 도입** — opus 모델 에이전트 신규 생성. Phase별 리뷰(Step 3d) + 통합 리뷰(Step 3.5). 사용자 직접 호출용 /code-review 스킬도 추가

## 검토 요청 사항

모든 파일을 읽고 다음 관점에서 분석해주세요:

### 1. 일관성 검사

- **Cross-reference 정합성**: 파일 A가 파일 B의 특정 Step이나 기능을 참조할 때, 실제로 B에 해당 내용이 있는지. 제거된 기능(version-manager, lifecycle, verify-history.json)의 잔존 참조가 없는지.
- **에이전트 모델 일관성**: CLAUDE.md, README.md, 각 에이전트 파일의 model 필드가 일치하는지 (codebase-scanner: sonnet, plan-writer: sonnet, skill-writer: sonnet, test-runner: sonnet, code-reviewer: opus).
- **스킬-에이전트 호출 관계**: 각 스킬이 참조하는 에이전트가 실제로 존재하고, 에이전트의 입출력 형식이 호출하는 스킬의 기대와 일치하는지.
- **Related Files 테이블**: 각 파일의 Related Files가 현재 파일 구조와 일치하는지. 삭제된 파일(version-manager.md, evals/ 등)이 남아있지 않은지.

### 2. 설계 문제

- **책임 중복**: 동일 로직이 여러 파일에 중복 기술되어 있는지. 예: 스킬 통합(INTEGRATE) 로직이 feature-planner COMPLETE, manage-skills Step 7, dev Step 4에 각각 기술되는데 서로 일관적인지.
- **단일 실패점**: 특정 에이전트나 파일이 실패하면 전체 파이프라인이 멈추는 지점이 있는지. 에러 핸들링이 모든 경로를 커버하는지.
- **PLAN 문서 상태 관리**: PLAN 문서의 이모지 기반 상태(⏳, 🔄, ✅)가 유일한 상태 소스인데, 이 방식의 취약점이 있는지.
- **skill-registry.json 스키마**: lifecycle, updatedAt 제거 후 남은 필드(name, description, coverPatterns, plan, phase, createdAt)가 충분한지. 빠진 정보가 있는지.

### 3. 누락/미흡 영역

- **feature-planner COMPLETE 모드와 code-reviewer의 연동**: 통합 리뷰(Step 3.5) 후 COMPLETE가 실행되는데, 리뷰에서 발견된 이슈가 COMPLETE 과정에 반영되는 경로가 있는지.
- **manage-skills와 code-reviewer**: manage-skills가 스킬을 업데이트할 때 code-reviewer를 거쳐야 하는지, 아니면 기계적 업데이트이므로 불필요한지.
- **plan-template.md**: code-reviewer 도입이 반영되어 있는지 (CRITICAL INSTRUCTIONS, Quality Gate, Verification Status 테이블).
- **에이전트 간 데이터 전달**: feature-planner → plan-writer, feature-planner → skill-writer 등의 데이터 전달이 명확히 정의되어 있는지, 아니면 "프롬프트로 전달"이라는 모호한 표현만 있는지.

### 4. 실행 가능성

- **Phase 시작 git ref**: dev Step 3d에서 "Phase 시작 git ref"를 code-reviewer에 전달하는데, 이 ref를 어디서 어떻게 기록하고 추적하는지 명확한 메커니즘이 있는지.
- **재개 모드**: `/dev PLAN_auth`로 재개할 때 Phase 상태 복원이 PLAN 문서의 이모지 파싱에만 의존하는데, 이게 충분한지.
- **병렬 실행 제약**: skill-writer, test-runner의 병렬 실행 시 파일 충돌(같은 파일을 동시에 수정)이 가능한지.

### 5. 간결성

- **제거 가능한 복잡성**: 현재 설계에서 불필요하게 복잡한 부분이 있는지. 예: Step 번호가 3a, 3b, 3c, 3d, 3e, 3.5로 복잡한데 단순화 가능한지.
- **문서 길이**: 각 SKILL.md가 적정 길이인지. 프롬프트로 사용되는 만큼 토큰 효율이 중요한데, 불필요한 예시나 반복이 있는지.

## 결과 형식

발견된 이슈를 다음 테이블로 정리해주세요:

```markdown
| # | 파일 | 라인 | 심각도 | 유형 | 이슈 | 제안 |
|---|------|------|--------|------|------|------|
```

심각도: CRITICAL (파이프라인이 깨짐) / HIGH (논리적 불일치) / MEDIUM (개선 필요) / LOW (사소)
유형: 일관성 / 설계 / 누락 / 실행가능성 / 간결성
