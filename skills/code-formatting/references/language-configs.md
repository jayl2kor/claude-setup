# Language Formatter & Linter Configs — Full Reference

Complete, annotated configuration files for each supported language.

---

## Python — pyproject.toml

`pyproject.toml` is the single config file for Black, Ruff, mypy, and pytest. Keep all Python tooling config here.

```toml
[tool.black]
line-length = 88          # Black's default — do not change without team consensus
target-version = ["py311"]
include = '\.pyi?$'

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = [
  "E",    # pycodestyle errors
  "W",    # pycodestyle warnings
  "F",    # pyflakes
  "I",    # isort
  "B",    # flake8-bugbear (catches real bugs, not just style)
  "C4",   # flake8-comprehensions
  "UP",   # pyupgrade — modernize syntax automatically
  "SIM",  # flake8-simplify
  "RUF",  # Ruff-native rules
]
ignore = [
  "E501",   # line length — Black handles this
  "B008",   # do not perform function calls in argument defaults (conflicts with FastAPI)
]

[tool.ruff.lint.isort]
known-first-party = ["myapp"]
force-sort-within-sections = true
```

**Python import ordering** (enforced by `ruff --select I`):

```python
# 1. __future__ imports
from __future__ import annotations

# 2. Standard library
import os
import sys
from pathlib import Path
from typing import Optional

# 3. Third-party packages
import httpx
import pydantic
from fastapi import FastAPI, HTTPException

# 4. First-party (your own packages)
from myapp.config import settings
from myapp.models import User

# 5. Local relative imports
from .utils import parse_date
from .validators import validate_email
```

One blank line between each section. Never mix sections. Never use wildcard imports (`from os import *`).

**Python docstring style** (Google):

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

    Example:
        >>> user = fetch_user(42)
        >>> print(user.email)
        alice@example.com
    """
```

Use Google-style (Args/Returns/Raises) over reStructuredText. It renders well in mkdocs-material and VS Code.

---

## JavaScript / TypeScript — .prettierrc and eslint.config.mjs

Prettier owns all whitespace decisions. ESLint owns correctness, imports, and type-safety rules. Do not configure ESLint to fight Prettier.

**`.prettierrc`**:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "bracketSpacing": true,
  "endOfLine": "lf"
}
```

**`eslint.config.mjs`** (ESLint flat config, v9+):

```js
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import prettier from 'eslint-config-prettier'  // disables rules Prettier owns
import importPlugin from 'eslint-plugin-import'

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  prettier,   // must be last — disables conflicting formatting rules
  {
    plugins: { import: importPlugin },
    rules: {
      // Correctness
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      // Import discipline
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      }],
      'import/no-duplicates': 'error',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
    languageOptions: {
      parserOptions: { project: true, tsconfigRootDir: import.meta.dirname },
    },
  },
)
```

**JavaScript/TypeScript import ordering**:

```typescript
// 1. Node built-ins (prefix with node: in ESM)
import path from 'node:path'
import { readFile } from 'node:fs/promises'

// 2. External packages
import express from 'express'
import { z } from 'zod'

// 3. Internal aliases (@/)
import { db } from '@/lib/database'
import type { User } from '@/types/user'

// 4. Relative imports — parent first, then siblings
import { parseDate } from '../utils/date'
import { validateEmail } from './validators'
```

**JSDoc for TypeScript public APIs**:

```typescript
/**
 * Fetches a paginated list of orders for the given user.
 *
 * @param userId - The UUID of the user whose orders to fetch.
 * @param options - Pagination and filter options.
 * @param options.limit - Maximum number of results (default: 20, max: 100).
 * @param options.cursor - Opaque cursor from the previous page's `nextCursor`.
 * @returns A page of orders and a cursor for the next page, or null if exhausted.
 * @throws {NotFoundError} If the user does not exist.
 *
 * @example
 * const page = await fetchOrders('uuid-123', { limit: 10 })
 * page.items.forEach(o => console.log(o.id))
 */
async function fetchOrders(
  userId: string,
  options: FetchOrdersOptions = {},
): Promise<OrderPage> { ... }
```

---

## Go — .golangci.yml

Go has no style debates: `gofmt` is the law. Configure your editor to run `goimports` on save (it runs gofmt and manages imports). Never manually format Go code.

**`.golangci.yml`**:

```yaml
run:
  timeout: 5m
  go: "1.23"

linters:
  enable:
    - errcheck      # unhandled errors
    - gosimple      # simplify code
    - govet         # suspicious constructs
    - ineffassign   # detect unused assignments
    - staticcheck   # comprehensive static analysis (replaces megacheck)
    - unused        # detect unused code
    - goimports     # import formatting
    - misspell      # spelling in comments/strings
    - gosec         # security issues
    - exhaustive    # enum switch exhaustiveness
    - wrapcheck     # errors must be wrapped with context
    - testifylint   # correct use of testify assertions

linters-settings:
  goimports:
    local-prefixes: github.com/your-org/your-repo
  wrapcheck:
    ignoreSigs:
      - .Errorf(
      - errors.New(
  govet:
    enable-all: true

issues:
  exclude-rules:
    - path: _test\.go
      linters: [wrapcheck, gosec]
```

**Go import ordering** (goimports enforces automatically):

```go
import (
    // 1. Standard library
    "context"
    "fmt"
    "net/http"

    // 2. Third-party
    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    // 3. First-party (same module)
    "github.com/your-org/your-repo/internal/domain"
    "github.com/your-org/your-repo/internal/repository"
)
```

**godoc style** — every exported symbol requires a comment; start with the identifier name, never "This":

```go
// Package user provides domain types and service logic for user management.
package user

// User represents an authenticated application user.
// The zero value is not valid; use NewUser to construct one.
type User struct {
    ID    uuid.UUID
    Email string
}

// NewUser constructs a validated User with a generated ID.
// It returns an error if email is malformed or empty.
func NewUser(email string) (*User, error) { ... }

// ErrNotFound is returned when a user lookup finds no matching record.
var ErrNotFound = errors.New("user: not found")
```

---

## Rust — rustfmt.toml and clippy.toml

`rustfmt` is mandatory. Every Rust project should have a `rustfmt.toml` committed at the repo root.

**`rustfmt.toml`**:

```toml
edition = "2021"
max_width = 100
use_small_heuristics = "Default"
imports_granularity = "Crate"   # group all imports from same crate
group_imports = "StdExternalCrate"
reorder_imports = true
reorder_modules = true
format_strings = false          # do not reformat string literals
wrap_comments = true
comment_width = 100
```

**`clippy.toml`** and deny attributes in `lib.rs`:

```toml
# clippy.toml
msrv = "1.75.0"
```

```rust
// lib.rs or main.rs
#![deny(
    clippy::all,
    clippy::pedantic,
    clippy::nursery,
    missing_docs,        // require doc comments on public items
    unsafe_code,         // ban unsafe unless explicitly allowed per module
)]
#![allow(
    clippy::module_name_repetitions,  // common in Rust idioms
    clippy::must_use_candidate,       // too noisy for most codebases
)]
```

**Rust doc comment style**:

```rust
/// Fetches a user by their primary key.
///
/// # Arguments
///
/// * `id` - The UUID of the user to fetch.
/// * `include_deleted` - When `true`, soft-deleted users are returned.
///
/// # Errors
///
/// Returns [`Error::NotFound`] if no user exists with the given `id`.
/// Returns [`Error::Database`] if the underlying query fails.
///
/// # Examples
///
/// ```rust
/// let user = repo.fetch_user(id, false).await?;
/// println!("{}", user.email);
/// ```
pub async fn fetch_user(&self, id: Uuid, include_deleted: bool) -> Result<User, Error> {
    // ...
}
```
