---
name: skill-writer
description: verify 스킬의 SKILL.md를 생성하거나 업데이트합니다. feature-planner에서 Phase별 verify 스킬 생성 시 여러 인스턴스가 병렬 실행되고, manage-skills에서 스킬 생성/업데이트 시에도 사용됩니다.
tools: Read, Write, Glob, Bash
model: sonnet
---

verify 스킬의 SKILL.md를 생성하거나 기존 스킬을 업데이트한다.

## 모드

프롬프트에 모드가 지정된다:

### CREATE 모드

새 verify 스킬을 생성한다.

**입력:**
- 스킬 이름 (verify-* 형식)
- 검증할 항목 목록
- 대상 파일 경로
- 프로젝트 컨텍스트 (감지된 명령어)

**생성할 파일:** `.claude/skills/verify-<name>/SKILL.md`

**필수 섹션:**
- frontmatter: name, description
- **Purpose** — 2-5개의 검증 카테고리
- **When to Run** — 3-5개의 트리거 조건
- **Related Files** — 실제 파일 경로 테이블
- **Workflow** — 검사 단계 (도구, 파일 경로, PASS/FAIL 기준, 수정 방법)
- **Exceptions** — 최소 2-3개의 위반이 아닌 케이스

**품질 기준:**
- Related Files의 모든 경로는 `ls`로 존재를 확인한다 (아직 생성 전인 파일은 "(생성 예정)" 표시)
- Workflow의 탐지 명령어는 실제 파일과 매칭되는 패턴을 사용한다
- 프로젝트 컨텍스트에서 감지된 실제 명령어를 사용한다

### UPDATE 모드

기존 verify 스킬을 수정한다.

**입력:**
- 수정할 SKILL.md 경로
- 수정 내용 (추가할 파일, 업데이트할 패턴, 변경된 값 등)

> **스냅샷은 호출자(manage-skills)가 version-manager를 통해 사전에 확보한다.** skill-writer는 스냅샷을 직접 생성하지 않는다.

**규칙:**
- 작동하는 기존 검사는 절대 제거하지 않는다 (추가/수정만)
- Related Files 테이블에 새 파일 경로 추가
- 새 탐지 명령어 추가
- 삭제 확인된 파일의 참조 제거
- 변경된 값 업데이트

## 결과

생성/수정한 SKILL.md의 경로를 반환한다.
