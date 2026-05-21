---
name: vault-kroki-diagram
description: Use when adding, editing, or reviewing inline diagrams in Obsidian vault notes (`*.md`) rendered by a self-hosted Kroki + Quartz pipeline (architecture, sequence, ERD, state, gantt, mindmap, flowchart, chart, network, hardware, BPMN, C4, etc.). Triggers include "draw a diagram", "visualize", "sequence", "flowchart", "ERD", "mindmap", "gantt", "C4" and Korean equivalents "다이어그램 그려줘" · "시각화" · "시퀀스" · "플로우차트" · "ERD" · "마인드맵" · "간트". Prefer over generic Mermaid/PlantUML knowledge for vault diagram work. Not for Obsidian Canvas (`.canvas`) or external SVG/PNG imports.
---

# vault-kroki-diagram

This skill is the guidance for authoring, editing, and reviewing every diagram fence code block in an Obsidian vault so it stays consistent with a self-hosted Kroki + Quartz build pipeline.

> [!note] Self-contained (global skill)
> All referenced assets (catalog, visualization policy, tool selection, compatibility patches) are internalized under `references/`, so this skill does not depend on any research-vault file. The only external dependency is the live Kroki endpoint used by the optional §6 verification (a runtime, not a file); fence authoring, patches, and tool selection work fully without that endpoint. `references/kroki-catalog.md` and `references/visualization-policy.md` are internalized Korean-language copies (kept Korean to preserve verification and source fidelity).

## When to use

Use when authoring or reviewing an inline diagram fence inside an Obsidian vault note (`*.md`) on the self-hosted Kroki + Quartz stack: system architecture, sequence, ERD, state, gantt, mindmap, flowchart, data chart, network/hardware, BPMN, C4 and similar.

**Do NOT use for:**

- Obsidian Canvas / JSON Canvas (`.canvas` spatial boards) — use the `obsidian-canvas-creator` · `json-canvas` skills.
- External static SVG/PNG attachments — only when the content cannot be expressed as a Kroki fence (see §7).
- Generic Mermaid syntax learning unrelated to this stack — `mermaid-visualizer`. For vault work still prefer this skill (it enforces self-hosted patches such as M-02).

## 1. Environment premises (must know)

| Item | Value |
|---|---|
| Kroki core | `yuzutech/kroki:0.28.0` (docker compose) |
| Worker split | `kroki-mermaid` · `kroki-bpmn` · `kroki-excalidraw` (3 workers) |
| Render path | At Quartz build time the `KrokiDiagrams` transformer does fence → Kroki POST → inline SVG embed |
| Browser-dependent JS | **None**. All diagrams are pre-rendered as static SVG |
| Whitelist | 30 types (`actdiag`·`blockdiag`·`bpmn`·`bytefield`·`c4plantuml`·`d2`·`dbml`·`diagramsnet`·`ditaa`·`dot`·`erd`·`excalidraw`·`graphviz`·`mermaid`·`nomnoml`·`nwdiag`·`packetdiag`·`pikchr`·`plantuml`·`rackdiag`·`seqdiag`·`structurizr`·`svgbob`·`symbolator`·`tikz`·`umlet`·`vega`·`vegalite`·`wavedrom`·`wireviz`) |
| Reference catalog | `references/kroki-catalog.md` — master catalog containing 33 verified fence sources (internalized copy of the research-vault source note, Korean) |

> [!note] 30 vs 33
> **30** = the Quartz `KROKI_TYPES` whitelist (distinct fence type names that render). **33** = verified fence *sources* in the catalog; some tools have multiple catalog entries (e.g. C4 PlantUML appears as catalog #27 · #28 · #30). The two counts are not contradictory.

## 2. Fence convention (M-01)

Use the **type name alone** as the fence identifier. The `kroki-` prefix is NOT matched by the transformer.

- Correct: a fence opened with `d2`, `mermaid`, `graphviz`, etc. — type name only.
- Wrong: `kroki-d2`, `kroki-mermaid` — not matched; renders as a plain code block (no Kroki call, no build error, so it needs visual verification).

Only bare-type-name fences match the Quartz `KROKI_TYPES` whitelist. When adding a new diagram, use one of the 30 whitelisted type names above as the fence identifier.

## 3. Tool selection quick table

| Purpose | Recommended tool | Fence |
|---|---|---|
| System architecture (components·relations) | D2 | `d2` |
| Sequence diagram (interaction) | Mermaid (worker) | `mermaid` |
| Sequence diagram (server-side static) | SeqDiag | `seqdiag` |
| ER diagram | Erd or DBML | `erd` / `dbml` |
| State machine | Mermaid or UMlet | `mermaid` / `umlet` |
| Gantt / timeline | Mermaid | `mermaid` |
| Mindmap | PlantUML | `plantuml` |
| C4 model (Context/Container/Component) | C4 PlantUML | `c4plantuml` |
| Large-scale SW architecture modeling | Structurizr DSL | `structurizr` |
| Hand-drawn style (brainstorming) | Excalidraw (worker) | `excalidraw` |
| Network topology | NwDiag | `nwdiag` |
| Rack / hardware layout | RackDiag | `rackdiag` |
| TCP/IP packet layout | PacketDiag | `packetdiag` |
| BPMN business process | BPMN (worker) | `bpmn` |
| Data visualization (charts) | Vega-Lite preferred / Vega for low-level | `vegalite` / `vega` |
| ASCII art → SVG | Svgbob | `svgbob` |
| Digital timing diagram | WaveDrom | `wavedrom` |
| Byte field (protocol) | Bytefield | `bytefield` |
| Circuit / hardware wiring | WireViz / Symbolator | `wireviz` / `symbolator` |
| General graph (nodes·edges) | GraphViz | `graphviz` |
| Free shapes · lightweight UML | Nomnoml · Pikchr | `nomnoml` / `pikchr` |

For detailed selection criteria and trade-offs see `references/tool-selection.md`.

## 4. Six compatibility patches (mandatory)

When authoring a new diagram, do not paste external sources (e.g. kroki.io official examples, Mermaid Live Editor) verbatim — check the patches below first. Full detail: `references/compatibility-patches.md`.

| ID | One-line summary | When it applies |
|---|---|---|
| M-01 | Fence is the bare type name; the `kroki-` prefix is forbidden | All diagrams |
| M-02 | Arrows inside Mermaid `loop` must use the new syntax `->>` · `-->>` (old syntax `->`·`-->` is rejected, HTTP 400) | Mermaid sequenceDiagram |
| M-03 | The Structurizr DSL `enterprise` keyword was removed → use `group "..."` | Structurizr workspace |
| M-04 | Mermaid·BPMN·Excalidraw run as split workers. HTTP 503 when the worker is down | When using those 3 |
| M-05 | Fixed-width SVGs (e.g. Vega) are shrunk for mobile via `max-width: 100%` CSS in Quartz `custom.scss` (already applied) | Vega·Vega-Lite·large GraphViz |
| M-06 | Client-JS-dependent Vega transforms (e.g. `wordcloud`) do not work in headless render → use static marks only | Dynamic Vega charts |

## 5. Workflow

When you receive a request to add or modify a diagram, proceed in this order.

1. **Identify intent** — What is being visualized? Identify candidate tools from the §3 tool-selection quick table. If the user did not specify a tool, present a recommendation first.
2. **Consult the catalog** — In `references/kroki-catalog.md`, look up verified examples for that tool (sections 01~33) to confirm the syntax form. If a verified fence block for the same tool already exists, follow that pattern.
3. **Write the fence** — Write the code block as a bare-type-name fence (§2).
4. **Apply compatibility patches** — Check and apply whichever of the six §4 patches affect that tool. Mermaid, Structurizr, and Vega in particular break frequently when external examples are copied verbatim.
5. **Tidy note metadata** — If added to an Obsidian note, confirm the frontmatter `tags` already include `workflow/design` or `documentation`. Visualization policy follows `references/visualization-policy.md`. The frontmatter ontology convention differs per vault, so follow the target vault's `AGENTS.md` (when present).

## 6. Verification (optional)

When you need to confirm a fence actually renders, use the script in the "## 일괄 POST 검증 (Bash)" section of `references/kroki-catalog.md` (the catalog is Korean; that heading is literal). For an immediate single-diagram check, the one-liner below works (requires a live Kroki endpoint — a runtime, not a file, dependency).

```bash
KROKI_URL="${KROKI_URL:-http://localhost:8080}"
echo 'diagram source' | curl -s -o /tmp/check.svg -w "%{http_code}\n" \
  -X POST "$KROKI_URL/{type}/svg" \
  -H 'Content-Type: text/plain' --data-binary @-
```

HTTP 200 + an SVG larger than 100 bytes is healthy. 503 means the worker is down (M-04); 400 means a syntax error (M-02·M-03 etc.).

## 7. Relationship with other visualization assets

- **Obsidian Canvas / JSON Canvas**: a separate `.canvas` file, not a fence block. For graph visualization and idea organization. Not in scope for this skill. Use the `obsidian-canvas-creator` · `json-canvas` skills.
- **External SVG / PNG attachments**: import static image files directly under `<vault>/attachments/` or `99_templates/`. When the content can be expressed as a Kroki fence, prefer the fence code block over a static image (single source, diffability).
- **mermaid-visualizer (global skill)**: for learning generic Mermaid syntax. This skill additionally enforces self-hosted Kroki environment consistency (patches like M-02), so prefer this skill for vault work.

## 8. Common mistakes and handling

| Symptom | Cause | Fix |
|---|---|---|
| Fence renders as plain code (no SVG) | Fence label is in `kroki-{type}` form (M-01 violation) | Correct it to `{type}` alone |
| HTTP 503 response | Mermaid·BPMN·Excalidraw worker is down (M-04) | Start the relevant worker service in the docker compose stack |
| Mermaid sequenceDiagram HTTP 400 | Old `->` · `-->` syntax (M-02) | Replace with the new `->>` · `-->>` syntax |
| Structurizr `enterprise` HTTP 400 | DSL 3.0.0 keyword removed (M-03) | Replace with `group "..."` |
| Vega SVG is 28KB but visually blank | JS-dependent transform such as `wordcloud` (M-06) | Replace with a static-marks chart |
| Vega chart horizontally scrolls on mobile | Fixed-width SVG (M-05) | Quartz `custom.scss` CSS already applied; no separate action |

## 9. References

**Internalized references (self-contained — no external file dependency):**

- `references/visualization-policy.md` — visualization rules (excerpt copy of research-vault `AGENTS.md` §시각화 규칙; the single source of policy; Korean)
- `references/kroki-catalog.md` — master catalog of 33 verified fence sources (internalized copy of the research-vault source note; Korean)
- `references/tool-selection.md` — detailed per-tool use-case matrix
- `references/compatibility-patches.md` — full text of compatibility patches M-01 ~ M-06

This skill is self-contained: the four `references/*.md` files listed above are the only dependencies. Earlier drafts also carried research-vault-internal `[[…]]` backlinks (KMS / Quartz / Kroki infrastructure decision notes); these did not resolve in global use and were removed during the 2026-05-16 audit.
