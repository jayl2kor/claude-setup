---
name: skill-creator
description: This skill should be used when the user asks to "create a skill", "make a new skill", "add a skill to the plugin", "write a SKILL.md", "스킬 만들어줘", "스킬 작성해줘", "새 스킬 추가", or needs guidance on skill structure, progressive disclosure, frontmatter format, writing style, or Claude Code skill best practices.
version: 1.0.0
---

# Skill Creator

Create effective, well-structured skills for Claude Code plugins. Skills are modular packages that give Claude specialized knowledge and workflows. A well-crafted skill triggers reliably, stays lean in context, and delivers targeted guidance.

## Core Principles

**Progressive Disclosure** — Skills use a three-level loading system:
1. **Frontmatter** (name + description) — Always in context, ~100 words
2. **SKILL.md body** — Loaded when skill triggers, target 1,500–2,000 words, hard max 3,000
3. **references/ files** — Loaded on demand, unlimited size

**Keep SKILL.md lean.** Move everything that isn't core procedural guidance to `references/`. Claude only reads reference files when it needs them.

## Skill Directory Structure

```
skills/skill-name/
├── SKILL.md           # Required — core instructions
├── references/        # Optional — detailed docs, loaded as needed
│   ├── patterns.md
│   └── advanced.md
├── examples/          # Optional — working code, copy-paste ready
│   └── example.sh
└── scripts/           # Optional — utility scripts, executable
    └── validate.sh
```

Create only the directories you actually need.

## Writing the Frontmatter Description

The description is the most critical field — it determines when Claude activates the skill.

**Rules:**
- Use **third person**: "This skill should be used when the user asks to..."
- List **specific trigger phrases** in quotes: `"create a hook"`, `"add PreToolUse"`
- Be concrete, not vague

```yaml
---
name: skill-name
description: This skill should be used when the user asks to "create X", "configure Y", "add Z", or mentions [specific scenario]. Provides [brief value statement].
version: 1.0.0
---
```

**Good vs Bad:**
```yaml
# ✅ Good — third person, specific triggers
description: This skill should be used when the user asks to "create a hook", "add a PreToolUse hook", "validate tool use", or mentions hook events (PreToolUse, PostToolUse, Stop).

# ❌ Bad — vague, wrong person
description: Use this skill when working with hooks.
```

## Writing the SKILL.md Body

**Use imperative/infinitive form** (verb-first). Not second person.

```markdown
# ✅ Correct
To create a hook, define the event type first.
Configure the MCP server with authentication.
Validate settings before use.

# ❌ Incorrect
You should create a hook by defining the event type.
You need to configure the MCP server.
```

**Structure:**
1. One-paragraph overview (what this skill does, when to use it)
2. Core concepts (minimum needed to understand the domain)
3. Step-by-step procedure for the primary use case
4. Quick reference (table or checklist)
5. Additional Resources section — pointers to references/examples/scripts

## What Goes Where

| Content | Location |
|---------|----------|
| Core workflow steps | SKILL.md |
| Key concepts and definitions | SKILL.md |
| Most common patterns (top 3–5) | SKILL.md |
| Quick reference tables | SKILL.md |
| Detailed code examples (>20 lines) | references/ or examples/ |
| Language-specific configs | references/ |
| Advanced/edge case patterns | references/ |
| API documentation | references/ |
| Utility scripts | scripts/ |
| Template files, boilerplate | examples/ or assets/ |

## Always Reference Supporting Files

If you create references/ or examples/, SKILL.md must tell Claude they exist:

```markdown
## Additional Resources

- **`references/patterns.md`** — Common patterns and detailed examples. Read when implementing non-trivial cases.
- **`references/advanced.md`** — Advanced techniques and edge cases. Read when standard approaches are insufficient.
- **`examples/working-example.sh`** — Complete working script. Copy and adapt for your use case.
```

Without this, Claude won't know the files exist.

## Creation Workflow

**Step 1 — Understand concrete use cases**

Identify 3–5 specific things users will ask when this skill should trigger. These become trigger phrases in the description. If unclear, ask the user for examples.

**Step 2 — Plan resources**

For each use case, determine: what code gets rewritten repeatedly? → script. What documentation gets referenced repeatedly? → reference. What output template gets reused? → asset/example.

**Step 3 — Create structure**

```bash
mkdir -p skills/skill-name/{references,examples,scripts}
touch skills/skill-name/SKILL.md
```

**Step 4 — Write SKILL.md**

Start with frontmatter (third-person description, specific triggers), then write lean body in imperative form. Add pointers to supporting files at the bottom.

**Step 5 — Add supporting files**

Create references/, examples/, scripts/ content. Each reference file can be large (2,000–5,000+ words) — detail belongs here, not in SKILL.md.

**Step 6 — Validate**

Check against the validation checklist before finalizing.

## Validation Checklist

**Frontmatter:**
- [ ] `name` and `description` fields present
- [ ] Description uses third person ("This skill should be used when...")
- [ ] Description includes specific trigger phrases in quotes
- [ ] Not vague ("Use when working with X" is bad)

**SKILL.md body:**
- [ ] Uses imperative/infinitive form throughout (not "you should")
- [ ] Under 3,000 words (ideally 1,500–2,000)
- [ ] Detailed content moved to references/
- [ ] References section lists all supporting files with descriptions

**Supporting files:**
- [ ] All files referenced in SKILL.md actually exist
- [ ] Scripts are executable
- [ ] Examples are complete and correct

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Vague description | Add specific trigger phrases users would actually say |
| SKILL.md over 3,000 words | Move detailed examples and configs to references/ |
| Second-person writing ("you should") | Rewrite as imperative ("To do X, configure Y") |
| references/ files not mentioned in SKILL.md | Add "Additional Resources" section |
| One giant SKILL.md with no references/ | Split: core in SKILL.md, detail in references/ |

## Additional Resources

For complete examples of well-structured skills, read the plugin-dev plugin skills:
- `hook-development/` — Progressive disclosure, 3 references + 3 scripts
- `agent-development/` — AI-assisted creation workflow, strong trigger phrases
- `plugin-structure/` — Good organization, concise SKILL.md
