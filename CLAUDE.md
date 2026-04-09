# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Corporate shared skills repository for AI agents (Claude Code, opencode, etc.). Contains reusable skill definitions following the Anthropic Skills 2.0 specification. This is a documentation-only repository — no build, test, or lint commands.

## Skill Format (Anthropic Skills 2.0)

Each skill lives in a kebab-case directory with a required `SKILL.md`:

```
<skill-name>/
├── SKILL.md              # Required: YAML frontmatter + markdown body
└── references/           # Optional: supporting docs for progressive disclosure
```

**SKILL.md frontmatter** (required fields):
```yaml
---
name: skill-name           # kebab-case, max 64 chars
description: "..."         # Trigger-based ("Use when..."), max 1024 chars
---
```

**Body constraints:** Max 500 lines. Longer content goes to `references/`. Focus on gotchas and insider knowledge, not public facts.

## Skill Design Principles (3-Axis)

1. **Trigger** — description must answer "WHEN should this activate?" not "WHAT does it do"
2. **Structure** — progressive disclosure: description → body → references
3. **Execution** — scripts as composable tools, not procedural instructions

## Adding Skills

- **New skill:** Create directory + `SKILL.md` with proper frontmatter. Add entry to README.md skill list.
- **Imported skill:** Extract only relevant documentation from source repo. Track source in `_imported/<source>.md` with URL, license, import date, modifications, and exclusions.
- **Exclusions when importing:** Scripts (unless essential), agent/hook definitions, test scenarios, plugin configs, server code.

## Key Conventions

- All directory and file names use kebab-case (except `SKILL.md` which is uppercase)
- Skills must be self-contained — executable without external context
- No invented endpoints, secrets, or internal topology; placeholder values must be marked explicitly
- Bilingual repo (Korean + English); skill content should be in English for model accuracy

## Architecture

- **31 skill directories** across 4 categories:
  - Superpowers (14): Development workflow skills (TDD, planning, debugging, code review)
  - Chisel HDL (8): Hardware design language patterns from chisel-book
  - XiangShan (8): RISC-V CPU design project conventions
  - Custom (1): analyze-onprem-readiness
- **`_imported/`**: Metadata files tracking external skill sources (URL, license, modifications)
- **`AGENTS.md`**: Full specification and guidelines in Korean for skill authoring

## Integration

**Claude Code:**
```bash
ln -s ~/skills-for-corp ~/.claude/skills/corp
```

**opencode:**
```json
{ "skills": { "paths": ["~/skills-for-corp"] } }
```

Skills are discovered automatically by the presence of `SKILL.md` in top-level directories.
