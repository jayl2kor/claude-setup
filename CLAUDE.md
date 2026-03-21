# claude-setup Plugin

새 프로젝트의 Claude 환경 설정을 자동화하는 플러그인. `/claude-setup <설명>` 한 번으로 대화형 요구사항 수집 → `docs/requirements.md` 생성 → skill 리서치 → Claude 환경 자동 구성 → Plan Mode로 `docs/plan.md` 구현 계획 수립까지 수행한다.

## 구조

```
claude-setup/
├── .claude-plugin/plugin.json       # 플러그인 매니페스트
├── commands/claude-setup.md         # /claude-setup 메인 오케스트레이터 (Phase 1~5)
├── agents/
│   ├── domain-researcher.md         # 외부 skill 검색 (Phase 3 병렬 실행)
│   ├── code-reviewer.md             # 코드 품질 리뷰 (독립 사용 가능)
│   └── security-reviewer.md         # 보안 취약점 검토 (독립 사용 가능)
└── skills/
    ├── deep-research/               # 체계적 기술 조사 방법론
    ├── backend-patterns/            # REST, clean arch, CQRS, DB 패턴
    ├── frontend-patterns/           # 컴포넌트 설계, 상태 관리, 성능
    ├── ui-design-principles/        # 시각 디자인, 타이포그래피, 색상
    ├── ux-design-principles/        # 사용성, 인터랙션, 접근성
    ├── code-formatting/             # 린트, 포매팅, 언어별 스타일 가이드
    ├── antipattern-detection/       # GoF 안티패턴, SOLID 위반, 언어별 패턴
    ├── e2e-testing/                 # Playwright, 테스트 피라미드, CI 통합
    └── security-review/             # OWASP Top 10, 인증/인가, 시크릿 관리
```

## 핵심 규칙

- **Phase 게이트**: Phase 1→2, 2→3, 3→4, 4→5 각 전환 시 사용자 확인 필수. 확인 전 절대 진행 금지
- **번들 우선**: Phase 3에서 번들 스킬 먼저 매칭, 미매칭 항목만 외부 리서치
- **병렬 제한**: domain-researcher 에이전트 최대 3개 동시 실행
- **파일 생성 순서**: CLAUDE.md → agents → skills → memory → settings.json → docs/plan.md (의존성 순서 엄수)
- **경로 참조**: 하드코딩 금지, `${CLAUDE_PLUGIN_ROOT}` 사용
- **언어 감지**: 사용자 입력 언어로 대화 진행 (한국어 입력 → 한국어 응답)

## 번들 Skills/Agents

모든 번들 파일은 해당 도메인의 deep-research(공식 문서, RFC, 베스트 프랙티스)를 기반으로 작성됨.

| 컴포넌트 | 타입 | 트리거 |
|---------|------|--------|
| `code-reviewer` | agent | 코드 리뷰, PR 검토 시 |
| `security-reviewer` | agent | 인증/인가, API, 민감 데이터 코드 작성 시 |
| `ui-designer` | agent | UI 구현, 화면 레이아웃, 디자인 시스템 설계 시 |
| `ux-designer` | agent | 사용자 플로우, 인터랙션, 접근성 설계 시 |
| `domain-researcher` | agent | 외부 skill 검색 필요 시 (Phase 3) |
| `deep-research` | skill | 기술 선택, 아키텍처 결정 시 |
| `backend-patterns` | skill | API 서버, DB, 마이크로서비스 설계 시 |
| `frontend-patterns` | skill | UI 컴포넌트, 상태 관리, 성능 최적화 시 |
| `code-formatting` | skill | 코드 작성/리뷰, 포매팅 적용 시 |
| `antipattern-detection` | skill | 코드 리뷰, 리팩토링, 아키텍처 검토 시 |
| `e2e-testing` | skill | E2E 테스트 작성, CI 파이프라인 설정 시 |
| `security-review` | skill | 보안 코드 리뷰, 취약점 스캔 시 |

## 테스트

```bash
# 로컬 설치
claude plugin install /Users/user/git/claude-setup

# 테스트 실행
mkdir /tmp/test-project && cd /tmp/test-project
/claude-setup "FastAPI 기반 사용자 인증 마이크로서비스"

# 각 Phase 게이트 동작 확인
# requirements.md 생성 확인
# .claude/ 디렉터리 구조 확인
```

## 스킬 추가/수정 시

1. 해당 도메인 deep-research 선행 (WebFetch/WebSearch로 공식 문서, RFC, 베스트 프랙티스 조사)
2. 구체적이고 실행 가능한 지침 작성 (단순 요약 X, 판단 기준 포함)
3. 좋은 예시 vs 나쁜 예시 포함
4. `commands/claude-setup.md`의 번들 목록 업데이트
