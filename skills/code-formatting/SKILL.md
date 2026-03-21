---
name: code-formatting
description: This skill should be used when the user asks to configure linters, formatters, or pre-commit hooks; review code style and naming conventions; set up ESLint, Prettier, Black, Ruff, gofmt, or rustfmt; enforce import ordering; write docstrings or JSDoc; or audit a codebase for consistent style across languages. Trigger phrases include "set up formatting", "configure linter", "add pre-commit hooks", "fix code style", "naming conventions", "import ordering", "set up ESLint", "set up Prettier", "set up Black".
version: 1.0.0
---

# Code Formatting

Expert-level formatting standards, linter configuration, and auto-formatter philosophy across Python, JavaScript/TypeScript, Go, and Rust. Consistent formatting is a prerequisite for readable diffs, productive code review, and automated tooling.

## When to Activate

- Setting up or auditing a linter or formatter configuration
- Resolving disagreements about naming conventions or line length
- Adding pre-commit hooks that enforce formatting
- Writing docstrings, JSDoc, or godoc comments
- Reviewing import ordering in any language
- Choosing between conflicting style guides

---

## Core Philosophy: Trust the Formatter

Auto-formatters (Prettier, Black, gofmt, rustfmt) exist to eliminate style debates. The correct policy:

1. **Pick a formatter. Run it on save. Enforce it in CI. Never override it manually.**
2. Configure the formatter once in a committed config file — do not rely on editor defaults.
3. Treat formatter output as ground truth. When a human and the formatter disagree, the formatter wins.
4. Reserve linter rules for correctness and antipatterns, not style. Style is the formatter's job.

The only legitimate reason to disable a formatter rule is when it produces output that breaks a tool (e.g., a SQL query that must stay on one line for a parser). Document these with a comment and keep them rare.

---

## Language Quick Reference

| Language | Line Length | Formatter | Linter | Config File |
|---|---|---|---|---|
| Python | 88 | Black | Ruff | `pyproject.toml` |
| JavaScript/TS | 80–100 | Prettier | ESLint | `.prettierrc`, `eslint.config.mjs` |
| Go | No limit | gofmt / goimports | golangci-lint | `.golangci.yml` |
| Rust | 100 | rustfmt | Clippy | `rustfmt.toml`, `clippy.toml` |
| CSS/SCSS | 80–100 | Prettier | Stylelint | `.prettierrc` |
| Markdown | 120 or off | Prettier | — | `.prettierrc` (disable prose wrap) |

Full config files with annotations: [references/language-configs.md](references/language-configs.md)

---

## Naming Conventions

### Python

| Construct | Convention | Example |
|---|---|---|
| Variables, functions | `snake_case` | `user_count`, `get_user()` |
| Classes | `PascalCase` | `UserRepository` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Private attrs/methods | Leading underscore | `_internal_cache` |
| Type aliases | `PascalCase` | `UserId = NewType("UserId", int)` |
| Modules/packages | `snake_case`, short | `user_service.py` |

### JavaScript / TypeScript

| Construct | Convention | Example |
|---|---|---|
| Variables, functions | `camelCase` | `userId`, `getUser()` |
| Classes, interfaces, types | `PascalCase` | `UserRepository`, `ApiResponse` |
| Constants (module-level) | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| React components | `PascalCase` | `UserCard`, `OrderList` |
| Custom hooks | `useCamelCase` | `useCurrentUser` |
| Files (components) | `PascalCase.tsx` | `UserCard.tsx` |
| Files (utilities/hooks) | `camelCase.ts` | `useAuth.ts`, `formatDate.ts` |

### Go

| Construct | Convention | Example |
|---|---|---|
| Packages | lowercase, single word | `user`, `httputil` |
| Exported identifiers | `PascalCase` | `UserRepository`, `ErrNotFound` |
| Unexported identifiers | `camelCase` | `userCache`, `parseDate` |
| Interfaces | noun or `-er` suffix | `Reader`, `UserStore` |
| Error variables | `ErrPascalCase` | `ErrNotFound`, `ErrUnauthorized` |
| Receiver names | 1–2 letter abbreviation | `func (u *User)`, `func (r *Repository)` |

### Rust

| Construct | Convention | Example |
|---|---|---|
| Types, traits, enums | `PascalCase` | `UserRepository`, `OrderStatus` |
| Variables, functions | `snake_case` | `user_count`, `fetch_user()` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Modules, files | `snake_case` | `user_repository.rs` |
| Lifetime params | Short `'a`, descriptive `'conn` | `fn fetch<'conn>` |

---

## Import Ordering

**Python** (enforced by `ruff --select I`): five sections, one blank line between each:

```
__future__ → stdlib → third-party → first-party → relative
```

Never mix sections. Never use wildcard imports (`from os import *`).

**JavaScript/TypeScript** (enforced by `eslint-plugin-import/order`):

```
Node built-ins (node:*) → external packages → internal aliases (@/) → relative
```

Use `import type` for type-only imports in TypeScript; enable `verbatimModuleSyntax` to enforce it.

**Go** (enforced by `goimports`): three groups — stdlib → third-party → first-party. Set `local-prefixes: github.com/your-org/your-repo` in `.golangci.yml` so goimports knows your module boundary.

**Rust** (enforced by `rustfmt`): set `imports_granularity = "Crate"` to group all imports from the same crate, and `group_imports = "StdExternalCrate"` for stdlib/external separation.

Full import examples for all languages: [references/language-configs.md](references/language-configs.md)

---

## Doc Comment Style

**Python** — Google style (Args/Returns/Raises/Example sections). Preferred over reStructuredText for inline readability and mkdocs-material rendering:

```python
def fetch_user(user_id: int, *, include_deleted: bool = False) -> User | None:
    """Fetch a single user by primary key.

    Args:
        user_id: The integer primary key of the user.
        include_deleted: When True, also returns soft-deleted users.

    Returns:
        The User instance, or None if not found.

    Raises:
        DatabaseError: If the database connection is unavailable.
    """
```

**TypeScript/JavaScript** — JSDoc for public APIs in JS. In TypeScript, types carry most of the burden — focus JSDoc on behavior, not types. Document `@throws` and `@example` on public SDK methods.

**Go** — every exported symbol requires a godoc comment. Start with the identifier name, never "This". The first sentence becomes the package index summary.

```go
// NewUser constructs a validated User with a generated ID.
// It returns an error if email is malformed or empty.
func NewUser(email string) (*User, error) { ... }
```

**Rust** — `///` triple-slash comments with `# Arguments`, `# Errors`, `# Examples` sections. Doc examples are compiled and run by `cargo test`.

---

## Pre-Commit Integration Overview

Install `pre-commit` globally and commit `.pre-commit-config.yaml` to enforce formatting before every commit. The config covers Ruff + Black (Python), Prettier + ESLint (JS/TS), gofmt + golangci-lint (Go), and general hooks (trailing whitespace, merge conflict markers, private key detection).

Also enforce in CI — pre-commit hooks can be skipped with `--no-verify`; CI cannot.

Full `.pre-commit-config.yaml`, CI workflow snippet, and common mistakes: [references/precommit-setup.md](references/precommit-setup.md)

---

## Common Mistakes (Quick Reference)

**Formatter config:**
- `.editorconfig` conflicting with Prettier — align manually or use `prettier-editorconfig`
- Different line lengths in formatter vs. linter — one source of truth only
- Running `eslint --fix` after `prettier --write` and getting conflicts — always use `eslint-config-prettier`

**Naming:**
- Abbreviations in public APIs (`usrCnt`, `ord`) — spell it out
- Boolean variables without `is`/`has`/`can` prefix — use `isDeleted`, `hasPermission`
- Generic names (`data`, `info`, `temp`, `obj`) — name what the value actually represents

---

## References

- [references/language-configs.md](references/language-configs.md) — Full `pyproject.toml`, `.prettierrc`, `eslint.config.mjs`, `.golangci.yml`, `rustfmt.toml` with annotations
- [references/precommit-setup.md](references/precommit-setup.md) — Full `.pre-commit-config.yaml`, CI enforcement snippet, common pre-commit mistakes
