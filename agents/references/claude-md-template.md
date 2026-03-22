# CLAUDE.md 템플릿

`setup-phase4-files` 에이전트가 CLAUDE.md 생성 시 참조합니다.
`<...>` 부분을 `docs/requirements.md` 내용으로 채웁니다.
`## Installed Agents & Skills`와 `## Agent Triggers` 섹션은 에이전트가 동적으로 생성합니다.

---

```markdown
# <프로젝트명>

<목적 한 줄 요약>

## Quick Start

<주요 빌드/실행 명령어>

## Architecture

<핵심 아키텍처 설명 — 주요 컴포넌트, 데이터 흐름, 레이어 구조>

## Key Files

<중요 파일/디렉터리 목록 — 경로와 역할>

## Code Style

<코드 스타일 규칙 — 포매터, 린터, 네이밍 컨벤션>

## Testing

<테스트 실행 방법 — 단위/통합/E2E 명령어>

## Gotchas

<주의사항 및 흔한 실수 — 처음 기여자가 겪는 함정>

## Workflows

<주요 개발 워크플로우 — 기능 추가, 버그 수정, 배포 절차>

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
   - 새로 파악한 컨벤션 → `memory/feedback-conventions.md`
   - 결정된 아키텍처 사항 → `memory/architecture.md`

2. **사용자에게 제안**: "작업이 완료됐습니다. `/clear`로 컨텍스트를 정리한 후 다음 작업을 시작하면 더 효율적입니다."

### /clear 후 세션 복원
새 세션 시작 시 Claude는 자동으로:
- CLAUDE.md 전체 로드
- `memory/MEMORY.md` 인덱스 첫 200줄 로드
- 필요한 memory 파일을 요청 시 추가 로드

→ **별도 설명 없이도 이전 컨텍스트가 복원됨**

## Installed Agents & Skills

<동적 생성 — setup-phase4-files.md가 채움>

## Agent Triggers

<동적 생성 — setup-phase4-files.md가 채움>
```
