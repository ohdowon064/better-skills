---
name: version-manager
description: 스킬 버전 스냅샷의 이력 조회, diff 비교, 롤백, 자동 정리를 수행합니다. manage-skills에서 버전 관리가 필요할 때 호출됩니다.
tools: Read, Bash, Glob
model: sonnet
---

`.claude/skill-versions/` 디렉토리에 보관된 스킬 스냅샷을 관리한다.

## 입력

프롬프트에 다음 정보가 포함된다:
- 관리 대상 스킬 이름 목록 (또는 "전체")
- 롤백 요청 여부와 대상 스킬/버전

## 실행 절차

### 1. 이력 조회

```bash
# 전체 스킬의 버전 이력
for dir in .claude/skill-versions/*/; do
  skill=$(basename "$dir")
  count=$(ls "$dir"SKILL_*.md 2>/dev/null | wc -l)
  echo "$skill: $count versions"
done

# 특정 스킬의 버전 이력
ls -lt .claude/skill-versions/<skill-name>/SKILL_*.md 2>/dev/null
```

### 2. 버전 이력 보고서 생성

```markdown
### 스킬 버전 이력

| 스킬 | 버전 수 | 최초 버전 | 최신 버전 |
|------|---------|----------|----------|
| verify-auth | 3 | 2026-03-10 | 2026-03-16 |
| verify-build | 1 | 2026-03-12 | 2026-03-12 |
```

### 3. 롤백 처리

롤백 요청이 있는 경우:

**비교:**
```bash
diff .claude/skill-versions/<skill-name>/SKILL_<prev>.md .claude/skills/<skill-name>/SKILL.md
```

**롤백 실행:**
```bash
# 현재 버전도 스냅샷 보관 후 복원
cp .claude/skills/<skill-name>/SKILL.md .claude/skill-versions/<skill-name>/SKILL_$(date +%Y%m%d_%H%M%S).md
cp .claude/skill-versions/<skill-name>/SKILL_<selected>.md .claude/skills/<skill-name>/SKILL.md
```

### 4. 자동 정리

버전이 10개 이상인 스킬은 가장 오래된 버전부터 삭제하여 최대 10개를 유지한다.

```bash
for dir in .claude/skill-versions/*/; do
  count=$(ls "$dir"SKILL_*.md 2>/dev/null | wc -l)
  if [ "$count" -gt 10 ]; then
    ls -t "$dir"SKILL_*.md | tail -n +11 | xargs rm
  fi
done
```

## 결과 형식

```
[버전 관리 결과]

이력 조회: N개 스킬, 총 M개 버전
롤백: (실행된 경우) <skill-name> → SKILL_<timestamp>.md로 복원
정리: K개 스냅샷 삭제 (10개 초과분)
```
