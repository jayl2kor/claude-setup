# 에이전트·스킬 파일 포맷 템플릿

`setup-phase4-files` 에이전트가 커스텀 에이전트·스킬 생성 시 참조합니다.

---

## 에이전트 파일 포맷

파일 경로: `.claude/agents/<name>.md`

```markdown
---
description: <역할 설명 (third-person). When to use 포함. examples 태그로 2개 예시 포함>

<example>
Context: <사용 맥락>
user: <구체적인 요청>
assistant: <에이전트가 하는 일 — 판단 기준과 결과물 포함>
</example>

<example>
Context: <다른 사용 맥락>
user: <다른 구체적인 요청>
assistant: <에이전트가 하는 일>
</example>
---

# <에이전트명>

<도메인 deep-research 기반 에이전트 시스템 프롬프트>
- 단순 역할 설명이 아닌, 해당 도메인 전문가처럼 판단할 수 있는 구체적 기준 포함
- 리서치에서 찾은 아키텍처 패턴, 체크리스트, 안티패턴 직접 반영
- "이 코드/설계가 좋은지 나쁜지"를 도메인 지식 기반으로 판단 가능하도록
```

---

## 스킬 파일 포맷

파일 경로: `.claude/skills/<name>/SKILL.md`

```markdown
---
name: <skill-name>
description: This skill should be used when the user asks to "<trigger phrase 1>", "<trigger phrase 2>", or mentions <specific scenario>.
version: 1.0.0
---

# <스킬명>

<한 단락 개요>

## Core Concepts

<핵심 개념 — 정의, 핵심 판단 기준>

## <주요 절차명>

<단계별 지침 — imperative form (verb-first)>

## Quick Reference

<자주 쓰는 패턴 표>

## Additional Resources

- **`references/patterns.md`** — <상세 코드 예시, 고급 패턴. 복잡한 케이스 구현 시 읽기>
- **`references/checklist.md`** — <체크리스트. PR 생성 전 검토 시 읽기>
```

**Progressive Disclosure 원칙**:
- SKILL.md 본문: 1,500~2,000단어(~200~280줄) 목표, 최대 3,000단어
- 상세 코드 예시(>20줄), 설정 파일, 고급 패턴은 `references/` 서브디렉터리로 분리
- Additional Resources 섹션에서 각 참조 파일의 목적과 "언제 읽어야 하는지" 명시
