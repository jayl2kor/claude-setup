---
description: Guided Claude environment setup for new projects. Creates CLAUDE.md, agents, skills, memory, and permissions configuration through interactive requirements gathering and intelligent skill matching. Use when starting a new project and wanting to configure the Claude environment.
argument-hint: <project description>
---

# Claude Setup — Orchestrator

새 프로젝트의 Claude 환경을 5단계로 구성합니다.

**초기 요청**: $ARGUMENTS

**언어 감지**: 사용자 입력 언어로 모든 대화를 진행합니다. ($ARGUMENTS가 한국어 → 한국어, 영어 → 영어)

---

## 실행 순서

각 단계는 전용 에이전트에 위임합니다. 게이트는 이 오케스트레이터가 담당합니다.

### Pre-flight

`setup-preflight` 에이전트를 호출합니다.

반환값:
- `mode`: `new` | `update` | `reset` | `abort`
- `existing_files`: 감지된 파일 목록
- `backup_path`: Reset 시 백업 경로
- `existing_requirements_summary`: Update 시 기존 requirements.md 요약

`abort`이면 즉시 종료합니다.

---

### Phase 1: 요구사항 수집

`setup-phase1` 에이전트를 호출합니다. 전달할 컨텍스트:
- `project_description`: $ARGUMENTS
- `mode`: Pre-flight에서 반환된 mode
- `existing_requirements_summary`: Update 모드인 경우 Pre-flight 반환값

에이전트 완료 후 **[게이트]** 출력:
> "이 내용이 정확한가요? 수정이 필요하면 알려주세요. 확인해 주시면 docs/requirements.md를 생성하겠습니다."

→ **사용자 확인을 받기 전까지 Phase 2로 진행하지 마세요.**

---

### Phase 2: requirements.md 생성

`setup-phase2` 에이전트를 호출합니다. 전달할 컨텍스트:
- Phase 1에서 반환된 requirements summary 전체

에이전트 완료 후 **[게이트]** 출력:
> "docs/requirements.md 파일을 검토해 주세요. 수정이 필요하면 알려주세요. 확인해 주시면 스킬 리서치를 시작하겠습니다."

→ **사용자 확인을 받기 전까지 Phase 3으로 진행하지 마세요.**

---

### Phase 3: 리서치

`setup-phase3` 에이전트를 호출합니다. 전달할 컨텍스트:
- `requirements_md_path`: "docs/requirements.md"

에이전트 완료 후 **[게이트]** 출력:
> "이 추천으로 진행할까요? 특정 항목의 액션을 변경하고 싶으시면 알려주세요."

→ **사용자 확인을 받기 전까지 Phase 4로 진행하지 마세요.**

---

### Phase 4: 설정 실행

두 에이전트를 **순서대로** 호출합니다 (의존성 순서 엄수):

**Step 1** — `setup-phase4-files` 에이전트 호출. 전달할 컨텍스트:
- `requirements_md_path`: "docs/requirements.md"
- Phase 3에서 반환된 research_results (승인된 skills/agents 목록)

**Step 2** — `setup-phase4-infra` 에이전트 호출. 전달할 컨텍스트:
- `requirements_md_path`: "docs/requirements.md"
- Step 1에서 반환된 generated_agents_list

두 에이전트 완료 후 **[게이트]** 출력:
> "환경 구성이 완료됐습니다. 이제 구현 계획을 Plan Mode로 수립하겠습니다. 진행할까요?"

→ **사용자 확인을 받기 전까지 Phase 5로 진행하지 마세요.**

---

### Phase 5: 프로젝트 계획 수립

`setup-phase5` 에이전트를 호출합니다. 전달할 컨텍스트:
- `requirements_md_path`: "docs/requirements.md"
- Phase 4 전체에서 생성된 파일 목록 (setup-phase4-files + setup-phase4-infra 반환값 합산)
