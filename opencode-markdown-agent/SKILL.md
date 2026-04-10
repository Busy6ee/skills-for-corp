---
name: opencode-markdown-agent
description: Use when creating, editing, or reviewing OpenCode Markdown agent definitions (.md files in .opencode/agents/ or ~/.config/opencode/agents/). Use when configuring subagents for vLLM-served models, setting tool permissions, writing system prompts for code-generation agents, or debugging agent auto-invocation issues.
---

# OpenCode Markdown Agent — Definition & System Prompt Authoring

## Overview

OpenCode agents can be defined as standalone Markdown files where **frontmatter = config** and **body = system prompt**. This skill covers the full authoring workflow: frontmatter options, system prompt structure, tool/permission configuration, and domain-specific prompt patterns for code-generation agents.

Source: [OpenCode Agent Docs](https://opencode.ai/docs/ko/agents/)

## File Placement

```
~/.config/opencode/agents/   ← Global (all projects)
.opencode/agents/            ← Per-project
```

**Filename = agent name.** `rtl-coder.md` registers as `rtl-coder`.

## Frontmatter Reference

```yaml
---
description: (REQUIRED) English — primary agent reads this to decide auto-invocation
mode: subagent              # primary | subagent | all (default: all)
model: provider/model-id    # must match opencode.json provider
temperature: 0.0            # 0.0–1.0
top_p: 0.95                 # 0.0–1.0
tools:
  write: true
  edit: true
  bash: false
  read: true
  glob: true
  grep: true
  mcp_*: false              # wildcard supported
permission:
  edit: allow               # allow | ask | deny
  bash:
    "*": deny
    "git diff": allow
    "make *": allow
  webfetch: deny
  task:
    "*": deny
    "explore": allow
steps: 20                   # max agentic iterations
hidden: false               # true = hide from @ autocomplete
color: "#2196F3"            # hex or theme name
---
```

Any unrecognized key is **passed through** to the provider as a model option (e.g., `reasoningEffort: high` for OpenAI).

## Frontmatter Options Quick Reference

| Option | Type | Purpose | Code Agent Default |
|--------|------|---------|-------------------|
| `description` | string | **Required.** Auto-invocation trigger | Domain-specific English |
| `mode` | string | Agent type | `subagent` |
| `model` | string | `provider/model-id` | Match opencode.json |
| `temperature` | float | Randomness | `0.0` (deterministic code) |
| `tools` | object | Enable/disable per tool | write+edit+read, bash off |
| `permission` | object | Per-tool auth level | bash deny for safety |
| `steps` | int | Max iterations | 15–25 (code), 10 (review) |
| `hidden` | bool | Hide from @ menu | `false` |
| `color` | string | UI badge color | Per-role differentiation |

## System Prompt Structure

The body after frontmatter is the system prompt. Structure it as:

```markdown
(Role statement — one sentence)

## Domain Knowledge
(Key facts the model needs — conventions, architecture, APIs)

## Rules
(Behavioral constraints — what to do and how)

## Output Format
(Expected structure of responses)

## Do NOT
(Explicit prohibitions — prevents hallucination drift)
```

### Prompt Design Principles

1. **Embed domain knowledge directly** — subagent models (especially local vLLM) cannot read external skill files or CLAUDE.md
2. **description in English** — the primary agent (LLM) reads it to decide invocation; English maximizes matching accuracy
3. **Prohibitions are critical** — local models hallucinate more than frontier models; explicit "Do NOT" sections prevent common failures
4. **Keep prompts under 2000 tokens** — longer prompts degrade local model instruction following

## Agent Archetypes

### Code Generator (write-capable)

```yaml
tools:
  write: true
  edit: true
  bash: false
  read: true
  glob: true
  grep: true
permission:
  edit: allow
  bash:
    "*": deny
temperature: 0.0
steps: 20
```

Use for: RTL coder, Chisel module writer, gem5 SimObject creator, testbench generator.

### Code Reviewer (read-only)

```yaml
tools:
  write: false
  edit: false
  bash: false
  read: true
  glob: true
  grep: true
permission:
  edit: deny
  bash:
    "*": deny
temperature: 0.1
steps: 10
```

Use for: RTL reviewer, Chisel convention checker, security auditor.

### Builder (full access, guarded)

```yaml
tools:
  write: true
  edit: true
  bash: true
  read: true
permission:
  edit: allow
  bash:
    "*": ask
    "git status*": allow
    "make *": allow
    "mill *": allow
temperature: 0.0
steps: 30
```

Use for: Build automation, CI helpers — bash allowed but guarded.

## Provider Prerequisite

Markdown agents define prompt + config only. **Provider must be registered in `opencode.json`:**

```json
{
  "provider": {
    "gemma4": {
      "api": "openai",
      "url": "http://localhost:8002/v1",
      "models": ["google/gemma-4-31B-it"]
    }
  }
}
```

Then reference as `model: gemma4/google/gemma-4-31B-it` in frontmatter.

Remove any `"agent"` block from opencode.json when switching to Markdown — duplicate names conflict.

## Embedding External Knowledge in Prompts

When the subagent model runs on local vLLM, it **cannot access** skill files, CLAUDE.md, or MCP servers available to the primary agent. All domain rules must be embedded in the system prompt body.

### Strategy: Compress skill content into prompt sections

```
Source skills (16 files, ~25K words)
  ↓ Extract key rules, naming conventions, patterns
  ↓ Remove examples that duplicate rules
  ↓ Merge overlapping content
Agent prompt (~1500 words)
```

### What to include

| Include | Skip |
|---------|------|
| Naming conventions | Detailed tutorials |
| Code patterns (brief) | Multi-paragraph explanations |
| Gotchas / Do-NOT rules | Historical context |
| Architecture overview | Full API reference |
| Key data structures | Build system details |

### Keep prompt in sync

When source skills update, the agent prompt needs manual sync. Add a note in the agent file:

```markdown
<!-- Prompt source: chisel-* (8) + xiangshan-* (8) skills, synced 2026-04-10 -->
```

## Complete Example — Chisel Code Agent

`.opencode/agents/chisel-coder.md`:

```markdown
---
description: Use when writing or modifying Chisel/Scala hardware designs, especially XiangShan-style RISC-V processor modules
mode: subagent
model: gemma4/google/gemma-4-31B-it
temperature: 0.0
tools:
  write: true
  edit: true
  bash: false
  read: true
  glob: true
  grep: true
permission:
  edit: allow
  bash:
    "*": deny
steps: 25
color: "#E91E63"
---

You are an expert Chisel hardware designer with deep knowledge of the XiangShan RISC-V processor codebase.

## Chisel Fundamentals

- Explicit width: `UInt(8.W)`, `SInt(10.W)`, `Bool()`
- Comparison: `===` / `=/=` (NOT `==` / `!=`)
- Register: `RegInit(0.U(4.W))` preferred, `RegEnable(value, enable)` for gated capture
- Handshake: `DecoupledIO[T]` (ready/valid/bits), `Flipped()` for consumer

## XiangShan Conventions

- Modules: UpperCamelCase, IOs: *IO suffix, Bundles: *Bundle suffix
- `fromXxx` = Flipped(), `toXxx` = normal direction
- Base classes: `XSModule`, `XSBundle` (not plain Module/Bundle)
- Pipeline: `val s1_valid = RegNext(s0_valid, false.B)`
- Flush: check `io.redirect.valid`, use `robIdx.needFlush(redirect)`
- Counters: `XSPerfAccumulate("name", condition)` for all key events

## Do NOT

- Use plain `Module` in XiangShan context
- Forget `Flipped()` on `fromXxx` ports
- Omit width on literals
- Use `reduce` when `reduceTree` is appropriate
```

## Invocation

- **Auto**: primary agent matches `description` → calls via Task tool
- **Manual**: `@agent-name <instruction>` in chat
- **Switch**: Tab key cycles primary agents; subagents via @ only

## Gotchas

| Pitfall | Fix |
|---------|-----|
| Agent not appearing in @ menu | Check file is in `.opencode/agents/` and has valid frontmatter |
| Auto-invocation not triggering | Make `description` more specific; use English keywords matching user intent |
| Model returning errors | Verify `model` matches `provider/model-id` in opencode.json exactly |
| bash commands executing despite deny | Check both `tools.bash: false` AND `permission.bash.*: deny` are set |
| JSON + Markdown conflict | Remove `"agent"` block from opencode.json when using Markdown agents |
| Prompt too long for local model | Keep system prompt under 2000 tokens; compress skill content |
| Agent ignoring conventions | Add explicit "Do NOT" section; local models need negative constraints |
| `steps` too low | Code generation needs 15–25; review needs 10; increase if agent truncates mid-task |
