# claude-setup

> 새 프로젝트를 위한 Claude 환경을 자동으로 설정해주는 Claude Code 플러그인

`/claude-setup <프로젝트 설명>` 한 번으로:
- 대화형 요구사항 수집 및 `requirements.md` 생성
- 관련 공식/커뮤니티 skill 자동 리서치
- `CLAUDE.md`, agents, skills, memory, 권한 설정 자동 구성

---

## 설치

```bash
# 로컬에서 직접 설치
claude plugin install /path/to/claude-setup

# 또는 저장소 클론 후 설치
git clone <repo-url>
claude plugin install ./claude-setup
```

---

## 사용법

```bash
# 새 프로젝트 디렉터리에서
/claude-setup "FastAPI 기반 사용자 인증 마이크로서비스"
/claude-setup "React + TypeScript e-commerce 플랫폼"
/claude-setup "Go CLI tool for Kubernetes cluster management"

# 설명 없이 시작 (대화로 유도)
/claude-setup
```

### 진행 단계

| Phase | 내용 | 출력 |
|-------|------|------|
| 1 | 대화형 요구사항 수집 (3개 질문 클러스터) | 요약 테이블 |
| 2 | requirements.md 생성 | `requirements.md` |
| 3 | 번들/외부 skill 리서치 및 추천 | 추천 테이블 |
| 4 | Claude 환경 파일 생성 | Setup Complete 매니페스트 |

각 Phase 전환 시 사용자 확인을 요청합니다.

---

## 번들 포함 항목

### Skills (9개)

| Skill | 용도 |
|-------|------|
| `deep-research` | 체계적 기술 조사 방법론 |
| `backend-patterns` | REST API, clean arch, CQRS, DB 패턴 |
| `frontend-patterns` | 컴포넌트 설계, 상태 관리, 성능 최적화 |
| `ui-design-principles` | Material Design/HIG, 타이포그래피, 색상 이론 |
| `ux-design-principles` | Nielsen 10 heuristics, WCAG, 인지 부하 이론 |
| `code-formatting` | 언어별 린트/포매팅 (Python, JS/TS, Go, Rust) |
| `antipattern-detection` | GoF 안티패턴, SOLID 위반, 언어별 패턴 |
| `e2e-testing` | Playwright 베스트 프랙티스, 테스트 피라미드, CI |
| `security-review` | OWASP Top 10, 인증/인가, 시크릿 관리 |

### Agents (3개)

| Agent | 용도 |
|-------|------|
| `code-reviewer` | 코드 품질, DRY, 유지보수성 평가 |
| `security-reviewer` | OWASP 취약점 감지, 보안 코드 리뷰 |
| `domain-researcher` | 외부 공식/커뮤니티 skill 검색 및 평가 |

> 번들 agents/skills는 setup 시 자동 적용되며, 독립적으로도 사용 가능합니다.

---

## 생성되는 파일

```
your-project/
├── CLAUDE.md                        # 프로젝트 가이드
├── requirements.md                  # 수집된 요구사항
└── .claude/
    ├── agents/
    │   ├── code-reviewer.md
    │   └── <프로젝트 맞춤 에이전트>.md
    ├── skills/
    │   └── <프로젝트 맞춤 스킬>/
    └── settings.json                # 권한, hooks 설정

~/.claude/projects/<project>/memory/
├── MEMORY.md
└── <seed 파일들>.md
```

---

## 참고 자료

- [Claude Code Best Practices](https://code.claude.com/docs/ko/best-practices)
- [Claude Code Memory](https://code.claude.com/docs/ko/memory)
- [Official Claude Skills](https://github.com/anthropics/skills/tree/main/skills)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code)
