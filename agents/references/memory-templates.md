# Memory 시드 파일 템플릿

`setup-phase4-infra` 에이전트가 메모리 파일 생성 시 참조합니다.
각 섹션의 `<...>` 부분을 `docs/requirements.md` 내용으로 채웁니다.

---

## MEMORY.md — 세션마다 자동 로드되는 인덱스 (첫 200줄)

파일 경로: `~/.claude/projects/<encoded-path>/memory/MEMORY.md`

```markdown
# Memory Index — <프로젝트명>

## 프로젝트 현황

- [session-log.md](session-log.md) — 완료된 작업 로그 및 현재 진행 상황
- [architecture.md](architecture.md) — 결정된 아키텍처·설계 사항
- `docs/plan.md` — 구현 로드맵 및 단계별 태스크 (Phase 5에서 생성)

## 컨벤션 & 피드백
- [feedback-conventions.md](feedback-conventions.md) — 코드 스타일, 네이밍 규칙

## 참조
- [debugging.md](debugging.md) — 발견된 버그·해결책·주의사항
```

---

## session-log.md — /clear 전 작업 요약을 축적하는 파일

파일 경로: `~/.claude/projects/<encoded-path>/memory/session-log.md`

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

---

## architecture.md — 설계 결정 사항

파일 경로: `~/.claude/projects/<encoded-path>/memory/architecture.md`

```markdown
---
name: architecture
description: 결정된 아키텍처·기술 선택 사항. 새 세션에서 설계 컨텍스트 복원에 사용.
type: project
---

# Architecture Decisions

<docs/requirements.md의 아키텍처 관련 내용으로 채움>
```

---

## feedback-conventions.md — 컨벤션 피드백

파일 경로: `~/.claude/projects/<encoded-path>/memory/feedback-conventions.md`

```markdown
---
name: feedback-conventions
description: 프로젝트 코드 컨벤션 및 사용자 피드백. 새 세션에서도 일관성 유지.
type: feedback
---

# Conventions & Feedback

<docs/requirements.md의 코드 스타일·컨벤션 내용으로 채움>
```

---

## debugging.md — 디버깅 기록 (초기 빈 파일)

파일 경로: `~/.claude/projects/<encoded-path>/memory/debugging.md`

```markdown
---
name: debugging
description: 발견된 버그, 해결책, 주의사항. 같은 문제 반복 방지.
type: feedback
---

# Debugging Log

(작업 중 버그·해결책 발견 시 여기에 기록)
```
