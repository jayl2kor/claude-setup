# settings.json 훅 예시

`setup-phase4-infra` 에이전트가 `.claude/settings.json` 생성 시 참조합니다.
조건에 따라 포함할 훅을 선택합니다. 조건 판단 로직은 `setup-phase4-infra.md`에 있습니다.

---

## Stop Hook (기본 Quality Gate) — 모든 프로젝트에 포함

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

---

## PostToolUse: Security Keywords — 보안 프로젝트에 추가

조건: `docs/requirements.md`에 인증·인가·API·결제·사용자 데이터 관련 항목이 있는 경우

**참고**: 이 훅은 파일 경로가 아닌 파일 **내용**에서 보안 키워드를 스캔합니다.
`$CLAUDE_TOOL_INPUT` 전체를 grep하는 것이 의도된 동작입니다.

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | grep -qiE \"password|secret|token|key|auth|jwt|session|credential\" && echo \"🔒 보안 관련 코드 변경 감지 — security-reviewer 에이전트를 실행하세요\" || true'"
    }
  ]
}
```

---

## PostToolUse: backend-developer — backend-developer 에이전트가 있는 경우에 추가

2단계 패턴: `file_path` 필드만 추출 후 경로 패턴 매칭 (파일 내용 false positive 방지)

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'printf \"%s\" \"$CLAUDE_TOOL_INPUT\" | grep -oE \"\\\"file_path\\\"[[:space:]]*:[[:space:]]*\\\"[^\\\"]+\\\"\" | grep -qiE \"/(routes|controllers|handlers|api|endpoints|views)/\" && echo \"🏗️  서버/API 코드 감지 — backend-developer 에이전트를 호출하세요\" || true'"
    }
  ]
}
```

---

## PostToolUse: frontend-developer — frontend-developer 에이전트가 있는 경우에 추가

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'printf \"%s\" \"$CLAUDE_TOOL_INPUT\" | grep -oE \"\\\"file_path\\\"[[:space:]]*:[[:space:]]*\\\"[^\\\"]+\\\"\" | grep -qiE \"/(components|pages|views)/.*\\.(tsx|jsx|vue|svelte)\" && echo \"🎨 UI 컴포넌트 코드 감지 — frontend-developer 에이전트를 호출하세요\" || true'"
    }
  ]
}
```

---

## PostToolUse: ml-engineer — ml-engineer 에이전트가 있는 경우에 추가

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "bash -c 'printf \"%s\" \"$CLAUDE_TOOL_INPUT\" | grep -oE \"\\\"file_path\\\"[[:space:]]*:[[:space:]]*\\\"[^\\\"]+\\\"\" | grep -qiE \"(train|model|pipeline|dataset|evaluate|inference)\" && echo \"🤖 ML 코드 감지 — ml-engineer 에이전트를 호출하세요\" || true'"
    }
  ]
}
```

---

## 2단계 패턴 동작 원리

1. `grep -oE '"file_path"[[:space:]]*:[[:space:]]*"[^"]+"'` — JSON에서 `file_path` 키-값 쌍만 추출
2. `grep -qiE "<경로 패턴>"` — 추출된 경로 값에만 패턴 매칭

파일 내용에 경로 관련 문자열이 포함되어 있어도 false positive가 발생하지 않습니다.

---

## Merge Rule

`docs/requirements.md`의 `Permissions` 섹션에 추가 allow/deny 항목이 있으면 기본 Stop Hook의 `permissions` 필드에 병합합니다.
`docs/requirements.md`의 `Hooks` 섹션에 커스텀 훅이 있으면 해당 이벤트 배열에 추가합니다 (덮어쓰지 않음).
