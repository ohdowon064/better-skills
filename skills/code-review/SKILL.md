---
name: code-review
description: 코드 변경사항을 리뷰하여 설계 품질, 보안, 성능, 아키텍처 일관성을 평가합니다. PR 전 리뷰, 특정 파일/디렉토리 리뷰, 최근 변경 리뷰를 지원합니다. 키워드: review, 리뷰, code review, 코드 리뷰, PR, 보안, security, 설계, design.
argument-hint: "[선택: 파일/디렉토리 경로, --staged, --branch <branch>]"
---

# Code Review

## Purpose

사용자가 `/dev` 파이프라인 밖에서 직접 코드 리뷰를 요청할 때 사용한다. `code-reviewer` 에이전트를 호출하는 래퍼 스킬.

## Argument Parsing

```
(인수 없음)              → 최근 커밋되지 않은 변경 + 스테이징된 변경 리뷰
"src/auth/"              → 특정 경로의 최근 변경 리뷰
"--staged"               → git staged 변경만 리뷰
"--branch feature/auth"  → 현재 브랜치와 main/master 분기점 이후 전체 diff 리뷰
```

## Workflow

### Step 1: 리뷰 대상 Diff 수집

인수에 따라 적절한 git diff 명령어를 실행한다:

```bash
# 인수 없음
git diff HEAD

# 특정 경로
git diff HEAD -- <path>

# --staged
git diff --staged

# --branch
git merge-base main HEAD  # 분기점 찾기
git diff <merge-base>..HEAD
```

변경된 파일이 없으면 "변경사항이 없습니다" 메시지 출력 후 종료한다.

### Step 2: PLAN 문서 탐색 (선택)

```bash
ls docs/plans/PLAN_*.md 2>/dev/null
```

PLAN이 존재하면, 변경된 파일이 어떤 PLAN/Phase에 해당하는지 매핑한다:
- 매핑되면 PLAN 문서를 code-reviewer에게 컨텍스트로 전달한다
- 매핑되지 않으면 PLAN 없이 리뷰한다

### Step 3: code-reviewer Subagent 실행

> **사용하는 Subagent**: `agents/code-reviewer.md`

- PLAN이 있으면 `phase` 모드로, 없으면 `integration` 모드로 실행한다
- 전달 항목: diff 범위, PLAN 경로 (있으면), git ref

### Step 4: 결과 출력

code-reviewer의 결과를 사용자에게 표시한다.

`REQUEST_CHANGES`인 경우, AskUserQuestion으로 확인:
1. **자동 수정** — 리뷰 이슈를 반영하여 코드 수정
2. **건너뛰기** — 이번에는 수정하지 않음

### Step 5: 수정 적용 (선택)

자동 수정 선택 시:
1. 이슈별로 코드를 수정한다
2. 수정 후 다시 code-reviewer를 실행하여 이슈가 해결되었는지 확인한다

## Related Files

| File | Purpose |
|------|---------|
| `agents/code-reviewer.md` | Subagent: 코드 리뷰 실행 |
| `docs/plans/PLAN_*.md` | 컨텍스트: PLAN 대비 충실도 평가용 |
