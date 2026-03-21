# Pre-Commit Setup — Full Reference

Complete `.pre-commit-config.yaml`, CI enforcement workflow, and common mistakes.

---

## .pre-commit-config.yaml

Install `pre-commit` (`pip install pre-commit` or `brew install pre-commit`) and run `pre-commit install` in the repo. Commit this file so every contributor gets the same hooks.

```yaml
repos:
  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  # JavaScript / TypeScript
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        types_or: [javascript, jsx, ts, tsx, css, json, yaml, markdown]

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.3.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        args: [--fix, --max-warnings=0]
        additional_dependencies:
          - eslint
          - typescript-eslint
          - eslint-config-prettier

  # Go
  - repo: https://github.com/dnephin/pre-commit-golang
    rev: v0.5.1
    hooks:
      - id: go-fmt
      - id: go-imports
      - id: golangci-lint

  # General
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
```

**Key decisions explained:**

- `--exit-non-zero-on-fix` on ruff: the commit is blocked when ruff auto-fixes, so the developer reviews the fix before committing. Remove this flag if you prefer auto-fix-and-continue behavior.
- `--max-warnings=0` on ESLint: zero tolerance for warnings prevents warning accumulation. Lower this only if you have legacy warnings you're actively working through.
- Pin with `rev:` — never use a floating branch ref like `main`. Floating refs break reproducibility when upstream releases a breaking change.
- `golangci-lint` in pre-commit can be slow on large repos. Consider moving it to CI only and keeping `go-fmt` + `go-imports` in the hook.

---

## CI Enforcement

Formatters must also run in CI to catch commits that bypass the pre-commit hook (via `--no-verify` or direct pushes to the remote).

```yaml
# .github/workflows/lint.yml
name: Lint
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Check formatting (Python)
        run: |
          pip install ruff black
          black --check .
          ruff check .

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Check formatting (JS/TS)
        run: |
          npm ci
          npx prettier --check .
          npx eslint . --max-warnings=0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.23"

      - name: Check formatting (Go)
        run: |
          test -z "$(gofmt -l .)"
          go vet ./...

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: v1.59
```

**Caching tip**: Use `actions/cache` for `~/.cache/pre-commit` if you run `pre-commit run --all-files` in CI — it avoids re-downloading hook environments on every run.

---

## Common Mistakes

### Formatter Configuration

**`.editorconfig` conflicts with Prettier** — Prettier ignores `.editorconfig` by default. If your repo uses both, align them manually or use the `prettier-editorconfig` package. Mismatched indent settings cause editors to reformat on open and then Prettier to reformat on save, producing noisy diffs.

**Running formatters in the wrong order** — Always run Prettier before ESLint (or use `eslint-config-prettier` to disable ESLint's formatting rules). Running ESLint `--fix` after Prettier can reintroduce formatting that Prettier would undo on the next run.

**Different line lengths in formatter vs. linter** — If `printWidth` in `.prettierrc` is 100 but ESLint's `max-len` is 80, you get constant false positives. One source of truth only: set it in the formatter and disable the linter's line-length rule.

### Import Ordering

**Mixing `import type` and value imports in TypeScript** — Put `import type` after value imports from the same package, or enable `verbatimModuleSyntax` in `tsconfig.json` to have TypeScript enforce the distinction automatically.

**Relative imports traversing more than two levels** — `../../../utils` is a signal to add a path alias. Configure `@/` in `tsconfig.json` paths and `eslint-plugin-import`'s `internal` group to point at your source root.

**Circular imports** — Catch these with `eslint-plugin-import/no-cycle` or `ruff --select ICN`. Circular imports cause initialization order bugs that are hard to debug and impossible to tree-shake.

### Pre-Commit Hooks

**Using `--no-verify` to skip hooks** — This is how style debt accumulates. Fix the root cause instead. If a hook is flaky or slow, fix the hook — don't teach the team to bypass it.

**Running the full test suite in a pre-commit hook** — Hooks should complete in under 10 seconds. Slow hooks get bypassed. Reserve integration tests and full test suites for CI. Pre-commit hooks should only run: formatting, linting, and fast static analysis.

**Not pinning hook revisions** — Floating `main` or `master` refs break reproducibility. A hook that passes today may fail tomorrow when upstream makes a breaking change. Always pin with `rev:` and update with `pre-commit autoupdate` periodically.

**Not running `pre-commit install` after cloning** — Add a `make setup` or `scripts/setup.sh` target that runs `pre-commit install` so new contributors get hooks automatically. Document this in your onboarding guide.

**Hook environment not committed** — Pre-commit caches hook environments in `~/.cache/pre-commit`. On CI, cache this directory keyed on the hash of `.pre-commit-config.yaml` to avoid re-downloading on every run.
