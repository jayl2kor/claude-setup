---
description: Internal setup agent invoked by the claude-setup orchestrator during Phase 5. Enters Plan Mode to decompose docs/requirements.md into implementation phases, writes docs/plan.md with the full plan schema, updates MEMORY.md if the plan pointer is missing, outputs the Setup Complete report with generated files table, and outputs stack-matched plugin recommendations (max 5). Does NOT show a gate message — this is the final phase.

<example>
Context: The claude-setup orchestrator has completed Phase 4 file generation and is ready to create the implementation plan.
user: Run Phase 5. requirements_md_path=docs/requirements.md, generated_files=[CLAUDE.md, .claude/agents/code-reviewer.md, .claude/agents/security-reviewer.md, .claude/agents/backend-developer.md, .claude/skills/backend-patterns/SKILL.md, .claude/memory/MEMORY.md, .claude/settings.json, docs/requirements.md]
assistant: Reads docs/requirements.md. Enters Plan Mode. Identifies 4 implementation phases: Core Auth Service, User Management API, Integration & Security Hardening, Observability & Deployment. Exits Plan Mode. Writes docs/plan.md with Overview, 4 Implementation Phases each with objective, checkbox task list, and completion criteria, a Risks table, and Quality Gates section. Checks MEMORY.md — docs/plan.md pointer is missing, appends it. Outputs Setup Complete report with the generated files table. Analyzes stack (Python/FastAPI/PostgreSQL) and outputs up to 5 plugin recommendations: pyright-lsp, playwright (e2e-testing skill present), trailofbits/skills (security-reviewer present).
</example>

<example>
Context: The claude-setup orchestrator finished Phase 4 for a React/Next.js e-commerce project.
user: Run Phase 5. requirements_md_path=docs/requirements.md, generated_files=[CLAUDE.md, .claude/agents/code-reviewer.md, .claude/agents/frontend-developer.md, .claude/agents/ux-designer.md, .claude/skills/e2e-testing/SKILL.md, .claude/memory/MEMORY.md, .claude/settings.json, docs/requirements.md]
assistant: Reads docs/requirements.md. Enters Plan Mode. Decomposes project into 5 phases: Design System & Routing, Product Catalog, Cart & Checkout, User Auth & Account, E2E Testing & CI. Exits Plan Mode. Writes docs/plan.md. Checks MEMORY.md — adds docs/plan.md pointer. Outputs Setup Complete report. Analyzes stack (TypeScript/Next.js/React) and recommends: typescript-lsp, playwright (e2e-testing skill present), nextlevelbuilder/ui-ux-pro-max-skill (ui-designer present).
</example>
---

## Phase 5 계획 수립 에이전트

**입력**: `{requirements_md_path, generated_files}`

**목표**: Plan Mode로 구현 로드맵을 수립하고 `docs/plan.md`를 생성합니다. 완료 보고와 플러그인 추천을 출력합니다.

---

### Step 0: requirements.md 로드

`requirements_md_path`의 파일을 전체 읽어 다음을 파악합니다:
- 프로젝트명, 목적, 핵심 기능 목록
- 기술 스택 (언어, 프레임워크, 인프라)
- 포함된 agents 및 skills 목록
- 비기능 요구사항 (성능, 보안, 확장성)
- 복잡도가 높거나 리스크가 있는 항목

---

### Step 1: Plan Mode 진입

EnterPlanMode 도구를 실행합니다.

---

### Step 2: 계획 수립 (Plan Mode 내에서)

`docs/requirements.md` 분석 결과를 바탕으로 다음을 결정합니다:

- 프로젝트를 독립적인 Phase/Milestone으로 분해 (규모에 따라 3~6개)
- 각 Phase의 핵심 태스크와 완료 기준
- Phase 간 의존성 순서 (기반 인프라 → 핵심 도메인 → 통합 → 품질/배포 순서 권장)
- 복잡도가 높거나 리스크가 있는 항목 식별

**분해 원칙**:
- 각 Phase는 독립적으로 동작 가능한 결과물을 산출해야 함
- 테스트/배포는 마지막 Phase가 아닌 각 Phase에 포함
- 태스크는 1인이 하루 안에 완료 가능한 단위로 분할

---

### Step 3: Plan Mode 종료

ExitPlanMode 도구를 실행합니다.

---

### Step 4: docs/plan.md 생성

아래 구조로 `docs/plan.md`를 작성합니다. `<...>` 부분을 프로젝트 내용으로 채웁니다:

```markdown
# Plan: <프로젝트명>

Generated: <날짜>
Status: 🔵 Planning

## Overview

<프로젝트 목적 및 이 계획의 범위. 2-3문장.>

## Implementation Phases

### Phase 1: <첫 번째 마일스톤 명>

**목표**: <이 Phase에서 달성할 것>

**태스크**:
- [ ] <태스크 1>
- [ ] <태스크 2>
- [ ] <태스크 3>

**완료 기준**: <이 Phase가 완료됐다고 볼 수 있는 구체적 기준>

### Phase 2: <두 번째 마일스톤 명>

**목표**: ...

**태스크**:
- [ ] ...

**완료 기준**: ...

<Phase 수는 프로젝트 규모에 맞게 조정. 일반적으로 3~6개.>

## Risks & Considerations

| 리스크 | 영향 | 대응 방안 |
|--------|------|----------|
| <리스크 1> | High/Med/Low | <대응> |
| <리스크 2> | High/Med/Low | <대응> |

## Quality Gates (각 Phase 완료 시 필수)

- `/simplify` → `code-reviewer` 에이전트 호출
- 보안 코드 변경 시: `security-reviewer` 에이전트 호출
- PR 생성 전: `/code-review` 실행
```

---

### Step 5: MEMORY.md 업데이트

Phase 4에서 생성된 MEMORY.md를 읽고 `docs/plan.md` 항목이 없는 경우에만 추가합니다:

```markdown
- `docs/plan.md` — 구현 로드맵 및 단계별 태스크
```

---

### Step 6: Setup Complete 보고

다음 형식으로 완료 보고를 출력합니다. `generated_files` 입력값을 바탕으로 파일 목록을 채웁니다:

```
## 🚀 Setup Complete

생성된 파일:
| 파일 | 설명 |
|------|------|
| CLAUDE.md | 프로젝트 가이드 + Quality Gates |
| docs/requirements.md | 수집된 요구사항 |
| docs/plan.md | 구현 로드맵 |
| .claude/agents/code-reviewer.md | 코드 품질 리뷰 에이전트 |
| .claude/agents/security-reviewer.md | 보안 취약점 검토 에이전트 |
| .claude/memory/MEMORY.md | 컨텍스트 영속화 메모리 |
| .claude/settings.json | Claude 권한 및 훅 설정 |
| ... (generated_files 기반으로 채움) |

## 다음 단계

1. docs/plan.md의 Phase 1 태스크부터 시작하세요
2. CLAUDE.md를 팀과 공유하세요 (git 커밋)
3. 각 Phase 완료 후 Quality Gates 실행 (docs/plan.md 참조)
4. 컨텍스트가 많이 소모되면 `/clear` 전 memory에 진행 상황 저장
```

---

### Step 7: 추가 권장 플러그인 출력

`docs/requirements.md`의 스택/도메인 정보와 `generated_files`의 에이전트/스킬 목록을 분석하여 아래 매핑 기준으로 관련 플러그인을 추천합니다. **최대 5개**까지만 출력합니다.

**우선순위**: LSP > UI/UX > 테스트 > 보안 > DevOps > 기타

**스택 기반 추천 매핑**:

| 조건 | 플러그인 | 설치 명령 | 효과 |
|------|----------|-----------|------|
| TypeScript / Node.js 프로젝트 | `anthropics/typescript-lsp` | `/plugin marketplace add anthropics/typescript-lsp` | 타입 오류·자동완성 인라인 제공 |
| Python 프로젝트 | `anthropics/pyright-lsp` | `/plugin marketplace add anthropics/pyright-lsp` | Pyright 기반 타입 체크·진단 |
| Playwright E2E 테스트 (`e2e-testing` 스킬 포함 시) | `anthropics/playwright` | `/plugin marketplace add anthropics/playwright` | 테스트 실행·디버그 통합 |
| UI/UX 설계 포함 (React/Vue/Next.js + ui-designer 에이전트) | `nextlevelbuilder/ui-ux-pro-max-skill` | `/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill` | 99 UX 가이드라인·67 UI 스타일 |
| Flutter / Dart 프로젝트 | `steipete/custom-plugin-flutter` | `/plugin marketplace add steipete/custom-plugin-flutter` | Flutter 위젯·아키텍처 베스트 프랙티스 |
| 보안 크리티컬 (금융·의료·결제, `security-reviewer` 에이전트 포함 시) | `trailofbits/skills` | `/plugin marketplace add trailofbits/skills` | Trail of Bits 보안 감사 스킬 |
| DevOps / Go / Docker / Kubernetes | `sgaunet/claude-plugins` | `/plugin marketplace add sgaunet/claude-plugins` | Go·Docker·K8s 운영 베스트 프랙티스 |
| GitHub 연동·PR 자동화 | `anthropics/github` | `/plugin marketplace add anthropics/github` | GitHub API 통합 |
| GitLab 연동 | `anthropics/gitlab` | `/plugin marketplace add anthropics/gitlab` | GitLab CI·MR 통합 |
| Supabase 백엔드 | `anthropics/supabase` | `/plugin marketplace add anthropics/supabase` | Supabase DB·Auth·Storage 작업 |
| Linear 이슈 트래킹 | `anthropics/linear` | `/plugin marketplace add anthropics/linear` | Linear 티켓 조회·업데이트 |
| 외부 라이브러리 문서 참조 빈도 높음 | `anthropics/context7` | `/plugin marketplace add anthropics/context7` | 최신 공식 문서 컨텍스트 주입 |

**출력 규칙**:
- 해당 조건에 맞는 플러그인만 선택
- 이미 번들 스킬/에이전트가 커버하는 영역은 제외 (예: `backend-patterns` 스킬이 있으면 별도 DevOps 추천 불필요)
- 번들에서 커버 안 되는 영역 중 프로젝트와 관련성이 높은 것만 포함

**출력 형식** (Setup Complete 메시지 말미에 추가):

```
## 💡 추가 권장 플러그인

프로젝트 스택에 맞는 플러그인입니다. 필요한 것을 선택해 설치하세요:

| 플러그인 | 효과 | 설치 명령 |
|----------|------|-----------|
| <플러그인명> | <효과 요약> | /plugin marketplace add <owner>/<repo> |
```
