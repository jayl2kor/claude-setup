---
description: Guided Claude environment setup for new projects. Creates CLAUDE.md, agents, skills, memory, and permissions configuration through interactive requirements gathering and intelligent skill matching. Use when starting a new project and wanting to configure the Claude environment.
argument-hint: <project description>
---

# Claude Setup

새 프로젝트를 위한 Claude 환경을 구성합니다. 대화형 요구사항 수집 → requirements.md 생성 → 스킬 리서치 → 환경 자동 구성 순서로 진행합니다.

**초기 요청**: $ARGUMENTS

**언어 감지**: 사용자의 입력 언어를 감지하여 동일한 언어로 모든 대화를 진행합니다. ($ARGUMENTS가 한국어면 한국어로, 영어면 영어로)

---

## Phase 1: 요구사항 수집

**목표**: 프로젝트를 이해하고 Claude 환경 설정에 필요한 정보를 수집합니다.

**액션**:
1. 할 일 목록 생성 (Phase 1~4 전체)
2. $ARGUMENTS가 있으면 초기 프로젝트 설명으로 사용. 없으면 먼저 프로젝트 설명을 요청
3. 아래 질문 클러스터를 **순서대로** 진행 (이전 클러스터 답변 후 다음 진행):

**Cluster A — 프로젝트 정체성** (한번에 물어볼 것):
- 이 프로젝트의 핵심 목적을 한 문장으로 설명해 주세요
- 주요 사용자는 누구인가요? (팀 내부 / 고객 / 개발자 등)
- 기술 스택은 무엇인가요? (언어, 프레임워크, 인프라)

**Cluster B — 워크플로우** (A 답변 후):
- 가장 자주 발생하는 개발 작업은 무엇인가요? (예: 기능 개발, 코드 리뷰, 배포, DB 마이그레이션)
- 특별히 주의가 필요하거나 확인이 필요한 민감한 작업이 있나요? (예: 프로덕션 배포, 결제 코드)
- 팀으로 사용하나요, 개인 프로젝트인가요?

**Cluster C — 도메인 전문성** (B 답변 후):
- 특수 도메인이 있나요? (ML/AI, 보안 크리티컬, 금융, 의료 등)
- 연동하는 외부 API나 서비스가 있나요?
- 자주 참고하는 공식 문서나 사양서가 있나요?

**완료 후**: 아래 형식으로 수집된 내용 요약 출력:

```
## 수집된 요구사항 요약

| 항목 | 내용 |
|------|------|
| 프로젝트명 | ... |
| 목적 | ... |
| 사용자 | ... |
| 스택 | ... |
| 주요 워크플로우 | ... |
| 민감 작업 | ... |
| 특수 도메인 | ... |
| 외부 서비스 | ... |
```

**[게이트]** "이 내용이 정확한가요? 수정이 필요하면 알려주세요. 확인해 주시면 requirements.md를 생성하겠습니다."
→ **사용자 확인을 받기 전까지 Phase 2로 진행하지 마세요.**

---

## Phase 2: requirements.md 생성

**목표**: Phase 1에서 수집한 내용을 Phase 4에서 직접 소비 가능한 구조화된 문서로 작성합니다.

**액션**:
1. 프로젝트 루트에 `requirements.md` 작성 (아래 스키마 사용)
2. `Skills Needed`와 `Agents Needed`를 **아래 결정 규칙**으로 채울 것
   - `source` 컬럼 값: `bundled` / `official` / `community` / `custom` (Phase 3에서 확정)
3. 파일 작성 후 내용 요약 출력

### Agents/Skills 결정 규칙

**항상 포함 (모든 프로젝트 기본값)**:
- Agent: `code-reviewer` — 코드 품질은 모든 프로젝트에 필요
- Skill: `code-formatting` — 일관된 코드 스타일
- Skill: `antipattern-detection` — 코드 리뷰·리팩토링 시

**스택 기반 매핑**:
| 조건 | 추가 항목 |
|------|----------|
| API 서버 (FastAPI, Express, Spring, Go HTTP 등) | Skill: `backend-patterns` |
| React / Vue / Next.js / Svelte / 프론트엔드 프레임워크 | Skill: `frontend-patterns` |
| UI 컴포넌트 설계 또는 디자인 시스템 언급 | Skill: `ui-design-principles`, `ux-design-principles` |
| 풀스택 (프론트+백 모두) | 위 frontend + backend 둘 다 |
| 웹 서비스 / 마케팅 사이트 / 콘텐츠 중심 서비스 | Skill: `seo` |
| Next.js / Nuxt / 서버사이드 렌더링 언급 | Skill: `seo` |

**워크플로우 기반 매핑**:
| 조건 | 추가 항목 |
|------|----------|
| E2E 테스트 / QA / 통합 테스트 언급 | Skill: `e2e-testing` |
| 기술 선택·아키텍처 결정이 필요한 단계 | Skill: `deep-research` |
| 코드 리뷰가 주요 워크플로우 | Agent: `code-reviewer` (이미 기본 포함) |

**도메인 기반 매핑**:
| 조건 | 추가 항목 |
|------|----------|
| 인증/인가 구현 | Skill: `security-review`, Agent: `security-reviewer` |
| 보안 크리티컬 (금융, 의료, 결제) | Skill: `security-review`, Agent: `security-reviewer` |
| 외부 API 다수 연동 | Skill: `deep-research` (연동 방법 조사용) |
| ML/AI 파이프라인 | Skill: `deep-research` + Agent: `create_custom` (도메인 전용) |
| 특수 도메인 (의료, 법률, 물류 등) | Agent: `create_custom` (도메인 전문 에이전트) |

**번들에 없는 경우**:
- 위 규칙으로 커버되지 않는 도메인/스택 → `source: tbd`, Phase 3 리서치로 확정

**requirements.md 스키마**:

```markdown
# Requirements: <프로젝트명>

Generated: <날짜>

## Project
- **name**:
- **purpose**:
- **users**:
- **repo_path**:

## Stack
- **language**:
- **framework**:
- **database**:
- **infrastructure**:
- **key_libraries**: []
- **external_apis**: []

## Workflows
- [ ] feature development
- [ ] code review
- [ ] deployment
- [ ] database migrations
- [ ] <기타 식별된 워크플로우>

## Agents Needed
| Agent | Purpose | Source | Action |
|-------|---------|--------|--------|
| code-reviewer | 코드 품질 검토 | bundled | apply |
| ... | ... | tbd | tbd |

## Skills Needed
| Skill | Purpose | Source | Action |
|-------|---------|--------|--------|
| backend-patterns | ... | bundled | apply |
| ... | ... | tbd | tbd |

## Memory Seeds
| File | Type | Content Summary |
|------|------|-----------------|
| project.md | project | 프로젝트 개요 및 목적 |
| stack.md | reference | 기술 스택 참조 |

## Permissions
allow:
  - Bash(git:*)
  - Bash(npm:*)
deny: []

## Hooks
| Event | Trigger | Action |
|-------|---------|--------|
```

**[게이트]** "requirements.md 파일을 검토해 주세요. 수정이 필요하면 알려주세요. 확인해 주시면 스킬 리서치를 시작하겠습니다."
→ **사용자 확인을 받기 전까지 Phase 3으로 진행하지 마세요.**

---

## Phase 3: 리서치

**목표**: 각 스킬/에이전트 항목에 대해 최적의 소스를 결정합니다.

### 3-1. 번들 매칭 (먼저 실행)

`requirements.md`의 `Skills Needed`와 `Agents Needed`를 이 플러그인의 번들 목록과 대조합니다.

**번들된 Agents**: `code-reviewer`, `security-reviewer`, `domain-researcher`

**번들된 Skills**:
- `backend-patterns` — API, DB, 마이크로서비스 아키텍처
- `frontend-patterns` — 컴포넌트 설계, 상태 관리
- `ui-design-principles` — 시각 디자인, 타이포그래피, 색상
- `ux-design-principles` — 사용성, 인터랙션, 접근성
- `code-formatting` — 린트, 포매팅, 코드 스타일
- `antipattern-detection` — 안티패턴 감지 및 교정
- `deep-research` — 도메인 심층 리서치
- `e2e-testing` — E2E 테스트 (Playwright/Cypress)
- `security-review` — OWASP, 보안 취약점 스캔
- `seo` — Core Web Vitals, 기술 SEO, 구조화 데이터

번들 매칭 항목: `source: bundled`, `action: apply`로 마킹

### 3-2. 외부 리서치 (번들에 없는 항목만)

번들에 없는 항목에 대해 `domain-researcher` 에이전트를 **최대 3개 병렬** 스폰합니다.

각 에이전트 프롬프트 예시:
- "Research whether an official or community skill exists for [SKILL_NAME] in the [DOMAIN] domain. Use case: [USE_CASE]. Check: 1) official 17 skills at https://github.com/anthropics/skills/tree/main/skills, 2) https://github.com/affaan-m/everything-claude-code skills/ directory. Return: { skill_name, recommendation: 'install_official|download_community|create_custom|skip', reason, install_command }"

### 3-3. 결과 제시

리서치 결과를 테이블로 출력:

```
## 리서치 결과

### Skills
| Skill | 소스 | 추천 액션 | 근거 |
|-------|------|-----------|------|
| backend-patterns | bundled | apply | 플러그인에 포함됨 |
| ... | ... | ... | ... |

### Agents
| Agent | 소스 | 추천 액션 | 근거 |
|-------|------|-----------|------|
| code-reviewer | bundled | apply | 플러그인에 포함됨 |
| ... | ... | ... | ... |
```

**[게이트]** "이 추천으로 진행할까요? 특정 항목의 액션을 변경하고 싶으시면 알려주세요."
→ **사용자 확인을 받기 전까지 Phase 4로 진행하지 마세요.**

---

## Phase 4: 설정 실행

**목표**: requirements.md와 리서치 결과를 바탕으로 Claude 환경 파일들을 생성합니다.

**반드시 아래 순서를 지켜서 생성합니다** (의존성 순서):

### 1. CLAUDE.md

프로젝트 루트에 `CLAUDE.md` 생성:
```markdown
# <프로젝트명>

<목적 한 줄 요약>

## Quick Start
<주요 빌드/실행 명령어>

## Architecture
<핵심 아키텍처 설명>

## Key Files
<중요 파일/디렉터리>

## Code Style
<코드 스타일 규칙>

## Testing
<테스트 실행 방법>

## Gotchas
<주의사항, 흔한 실수>

## Workflows
<주요 개발 워크플로우>

## Quality Gates

기능 구현 완료 후, PR 생성 전, 또는 주요 마일스톤마다 아래 커맨드를 순서대로 실행한다.

### 기능 구현 완료 시
1. `/simplify` — 구현한 코드의 중복 제거, 복잡도 감소, 가독성 개선
2. `code-reviewer` 에이전트 호출 — DRY, SOLID, 프로젝트 컨벤션 준수 검토

### 보안 관련 코드 변경 시 (인증/인가, API, 결제, 사용자 데이터)
3. `security-reviewer` 에이전트 호출 — OWASP Top 10, 취약점, 시크릿 노출 검토

### PR 생성 전
4. `/code-review` — PR 전체 코드 리뷰 (설치된 경우)
5. 테스트 실행: `<테스트 명령어>`

### CLAUDE.md 업데이트 필요 시
- `/revise-claude-md` — 새로 발견한 컨벤션, 빌드 명령 변경, 주의사항 반영

## Context 관리 (중요)

Claude의 컨텍스트 창은 유한하다. 큰 작업 단위가 끝날 때마다 아래 절차를 따른다.

### /clear 권장 시점
- 독립적인 기능 구현이 완료된 후
- 긴 디버깅 세션이 끝난 후
- 주제가 완전히 바뀌는 새 작업 시작 전
- 컨텍스트가 많이 소모되어 응답 품질이 떨어진다고 느낄 때

### /clear 전 필수 절차

1. **중요 컨텍스트를 memory에 저장** (Claude가 직접 실행):
   - 완료한 작업 요약 → `memory/session-log.md` 업데이트
   - 발견한 버그·해결책 → `memory/debugging.md`
   - 새로 파악한 컨벤션 → `memory/feedback-*.md`
   - 결정된 아키텍처 사항 → `memory/architecture.md`

2. **사용자에게 제안**: "작업이 완료됐습니다. `/clear`로 컨텍스트를 정리한 후 다음 작업을 시작하면 더 효율적입니다."

### /clear 후 세션 복원
새 세션 시작 시 Claude는 자동으로:
- CLAUDE.md 전체 로드
- `memory/MEMORY.md` 인덱스 첫 200줄 로드
- 필요한 memory 파일을 요청 시 추가 로드

→ **별도 설명 없이도 이전 컨텍스트가 복원됨**

## Installed Agents & Skills

| 컴포넌트 | 호출 방법 | 용도 |
|---------|----------|------|
| `code-reviewer` | "코드 리뷰해줘" / "review this" | 코드 품질, DRY, 컨벤션 |
| `security-reviewer` | "보안 검토해줘" / "security review" | OWASP, 취약점 스캔 |
<프로젝트 맞춤 에이전트 행 추가>
```

### 2. .claude/agents/*.md

`Agents Needed`의 각 항목 처리:
- `source: bundled` → `.claude/agents/<name>.md` 에 플러그인 번들 에이전트 내용 복사
- `action: create_custom` → **아래 절차대로** 신규 에이전트 작성

**create_custom 에이전트 작성 절차** (각 에이전트마다 반복):

**Step 1 — 도메인 Deep Research** (에이전트 작성 전 필수):

해당 에이전트의 도메인에 대해 WebFetch/WebSearch로 심층 조사:
- 도메인 아키텍처 패턴 (예: ML 파이프라인이면 데이터 전처리→학습→평가→배포 패턴)
- 해당 분야의 best practices, 표준, 가이드라인 (공식 문서, RFC, 저명한 기술 블로그)
- 흔한 실수와 안티패턴
- 이 에이전트가 판단해야 할 핵심 기준들

리서치 소스 우선순위:
1. 공식 문서, RFC, 표준 사양
2. 저명한 기술 조직의 엔지니어링 블로그 (Google, Meta, Netflix, Stripe 등)
3. GitHub 고품질 레포지토리의 설계 문서
4. 커뮤니티 베스트 프랙티스 (StackOverflow, HN 토론 등)

**Step 2 — 리서치 기반 에이전트 작성**:

수집된 도메인 지식을 에이전트 시스템 프롬프트에 녹여 작성:
- 단순 역할 설명이 아닌, 해당 도메인 전문가처럼 판단할 수 있는 구체적 기준 포함
- 리서치에서 찾은 아키텍처 패턴, 체크리스트, 안티패턴을 직접 반영
- 에이전트가 "이 코드/설계가 좋은지 나쁜지"를 도메인 지식 기반으로 판단할 수 있도록

에이전트 파일 포맷:
```markdown
---
description: <역할 설명. When to use 포함. examples 태그로 2개 예시 포함>
---

<도메인 deep-research 기반 에이전트 시스템 프롬프트>
```

**예시**: FastAPI 마이크로서비스용 `api-design-reviewer` 에이전트 생성 시:
- REST 성숙도 모델(Richardson), HTTP 메서드/상태코드 의미론, OpenAPI 스펙 연구
- FastAPI 공식 문서의 best practices, Pydantic v2 패턴 조사
- Google API Design Guide, Stripe API 설계 원칙 참고
→ 조사 결과를 에이전트 프롬프트에 구체적 판단 기준으로 반영

### 3. .claude/skills/*/SKILL.md

`Skills Needed`의 `create_custom` 항목에 대해 **도메인 deep-research 후** 스킬 작성.

**create_custom 스킬 작성 절차**:

**Step 1 — 도메인 Deep Research** (스킬 작성 전 필수):
- 해당 스킬 도메인의 최신 best practices를 WebFetch/WebSearch로 조사
- 구체적 기준, 판단 근거, 예시가 될 수 있는 자료 수집
- 번들 스킬(backend-patterns, security-review 등)과 동일한 수준의 전문성 목표

**Step 2 — `skill-creator` 가이드라인 준수하여 작성**:

`skill-creator` 스킬의 지침을 따라 작성한다:
- **Frontmatter description**: third-person ("This skill should be used when..."), 구체적 trigger phrases 포함
- **SKILL.md 본문**: imperative form (verb-first), 1,500~2,000단어(~200~280줄) 목표, 최대 3,000단어
- **Progressive disclosure**: 상세 코드 예시, 설정, 고급 패턴은 `references/` 서브디렉터리로 분리
- **Additional Resources 섹션**: SKILL.md 말미에 references/ 파일 목록과 각 용도 명시

스킬 파일 포맷:
```markdown
---
name: <skill-name>
description: This skill should be used when the user asks to "<trigger phrase 1>", "<trigger phrase 2>", or mentions <specific scenario>.
version: 1.0.0
---

# <스킬명>

<한 단락 개요>

## Core Concepts
<핵심 개념>

## <주요 절차>
<단계별 지침 — imperative form>

## Quick Reference
<자주 쓰는 패턴 표>

## Additional Resources
- **`references/patterns.md`** — <언제 읽어야 하는지>
```

상세 내용이 많을 경우 반드시 `references/` 파일 생성 후 SKILL.md에서 참조할 것.

### 4. ~/.claude/projects/<encoded-path>/memory/

메모리 파일 생성:
```bash
# 경로 인코딩: / → - 치환
ENCODED_PATH=$(echo "$PWD" | sed 's|/|-|g')
MEMORY_DIR="$HOME/.claude/projects/${ENCODED_PATH}/memory"
```

**항상 생성하는 기본 파일들** (모든 프로젝트):

`MEMORY.md` — 인덱스 (세션마다 자동 로드되는 첫 200줄):
```markdown
# Memory Index — <프로젝트명>

## 프로젝트 현황
- [session-log.md](session-log.md) — 완료된 작업 로그 및 현재 진행 상황
- [architecture.md](architecture.md) — 결정된 아키텍처·설계 사항

## 컨벤션 & 피드백
- [feedback-conventions.md](feedback-conventions.md) — 코드 스타일, 네이밍 규칙

## 참조
- [debugging.md](debugging.md) — 발견된 버그·해결책·주의사항
```

`session-log.md` — /clear 전 작업 요약을 축적하는 파일:
```markdown
---
name: session-log
description: /clear 전 저장한 작업 완료 로그. 새 세션 시작 시 현재 진행 상황 파악에 사용.
type: project
---

# Session Log

## <날짜> — 초기 설정
- claude-setup으로 프로젝트 환경 구성 완료
- 생성된 파일: CLAUDE.md, .claude/agents/, .claude/settings.json
```

`architecture.md` — 설계 결정 사항:
```markdown
---
name: architecture
description: 결정된 아키텍처·기술 선택 사항. 새 세션에서 설계 컨텍스트 복원에 사용.
type: project
---

# Architecture Decisions

<Phase 1에서 파악된 아키텍처 내용>
```

`feedback-conventions.md` — 컨벤션 피드백:
```markdown
---
name: feedback-conventions
description: 프로젝트 코드 컨벤션 및 사용자 피드백. 새 세션에서도 일관성 유지.
type: feedback
---

# Conventions & Feedback

<Phase 1에서 파악된 코드 스타일·컨벤션>
```

`debugging.md` — 디버깅 기록 (초기 빈 파일로 생성):
```markdown
---
name: debugging
description: 발견된 버그, 해결책, 주의사항. 같은 문제 반복 방지.
type: feedback
---

# Debugging Log

(작업 중 버그·해결책 발견 시 여기에 기록)
```

`requirements.md`의 `Memory Seeds` 항목에 추가 seed가 있으면 위 기본 파일들에 병합하여 생성.

### 5. .claude/settings.json

`Permissions`와 `Hooks` 섹션을 기반으로 생성한다.

**기본 포함 hooks** (모든 프로젝트):

```json
{
  "permissions": {
    "allow": [],
    "deny": []
  },
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo '\\n💡 Quality Gate 체크리스트:\\n  기능 구현 완료? → /simplify 실행 후 code-reviewer 호출\\n  보안 코드 변경? → security-reviewer 호출\\n  PR 준비? → /code-review 실행'"
          }
        ]
      }
    ]
  }
}
```

**보안 크리티컬 프로젝트 추가 hook** (`requirements.md`에 보안 관련 항목이 있는 경우):

```json
"PostToolUse": [
  {
    "matcher": "Write|Edit",
    "hooks": [
      {
        "type": "command",
        "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | grep -qiE \"password|secret|token|key|auth|jwt|session|credential\" && echo \"🔒 보안 관련 코드 변경 감지 — security-reviewer 에이전트를 실행하세요\" || true'"
      }
    ]
  }
]
```

`requirements.md`의 `Permissions`와 `Hooks` 섹션 내용을 위 기본값에 **병합**하여 최종 파일 작성.

### 완료 보고

모든 파일 생성 후 Setup Complete 매니페스트 출력:

```
## ✅ Setup Complete

생성된 파일:
| 파일 | 설명 |
|------|------|
| CLAUDE.md | 프로젝트 가이드 + Quality Gates |
| .claude/agents/code-reviewer.md | 코드 품질 리뷰 에이전트 |
| ... | ... |

## 🚀 바로 시작하기

**주요 커맨드 (Quality Gates)**
| 시점 | 커맨드/에이전트 | 용도 |
|------|----------------|------|
| 기능 구현 완료 후 | `/simplify` | 코드 정리, 중복 제거 |
| 기능 구현 완료 후 | `code-reviewer` 에이전트 | 품질·컨벤션 검토 |
| 보안 코드 변경 후 | `security-reviewer` 에이전트 | OWASP 취약점 검토 |
| PR 생성 전 | `/code-review` | PR 전체 리뷰 |
| CLAUDE.md 수정 필요 시 | `/revise-claude-md` | 가이드 업데이트 |

다음 단계:
- CLAUDE.md를 팀과 공유하세요 (git에 커밋)
- .claude/settings.json의 hooks가 정상 동작하는지 확인하세요
- 보안 크리티컬 코드 작성 시 security-reviewer를 먼저 호출하세요
| 파일 | 설명 |
|------|------|
| CLAUDE.md | 프로젝트 가이드 |
| .claude/agents/code-reviewer.md | 코드 품질 리뷰 에이전트 |
| ... | ... |

다음 단계:
- CLAUDE.md를 팀과 공유하세요 (git에 커밋)
- .claude/settings.json 권한을 검토하세요
- /deep-research <기술명> 으로 심층 리서치를 시작하세요
```
