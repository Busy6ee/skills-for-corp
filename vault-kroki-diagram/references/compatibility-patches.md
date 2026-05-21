# Compatibility patches — full text (M-01 ~ M-06)

Six compatibility patches found when moving external sources (kroki.io official examples, Mermaid Live Editor, Structurizr Lite, etc.) into the self-hosted Kroki 0.28.0 + Quartz `KrokiDiagrams` transformer environment.

> **Single canonical source.** This file is the authoritative definition of M-01 ~ M-06. The §호환성 메모 section of `references/kroki-catalog.md` is a non-authoritative one-line index that follows this file. When a patch changes, edit ONLY this file.

---

## M-01: Fence convention is the bare type name

**One-line summary**: only the bare ` ```{type} ` form (e.g. ` ```blockdiag `, ` ```d2 `) matches the Quartz `KrokiDiagrams` transformer. Do not use a `kroki-{type}` prefix.

| Item | Detail |
|---|---|
| Rationale | The `KROKI_TYPES` whitelist in the Quartz transformer `quartz/plugins/transformers/kroki.ts` is defined with bare type names |
| Wrong examples | ` ```kroki-d2 ` · ` ```kroki-mermaid ` · ` ```krokid2 ` |
| Correct examples | ` ```d2 ` · ` ```mermaid ` · ` ```graphviz ` |
| Impact | A wrong fence renders as a plain code block and no Kroki call happens at all. No build error occurs, so visual verification is needed |

---

## M-02: Mermaid arrows must use the new syntax (`->>`, `-->>`)

**One-line summary**: the Kroki Mermaid 11.6.0 strict parser rejects old syntax such as `->`, `-->` inside a `loop` block (HTTP 400). Patch to the new syntax.

| Item | Detail |
|---|---|
| Rejected examples | `Alice->John: Hello` · `John-->Alice: Reply` |
| HTTP 400 message | `Expecting 'SOLID_ARROW' ...` |
| Passing examples | `Alice->>John: Hello` (solid + arrowhead) · `John-->>Alice: Reply` (dashed + arrowhead) |
| Scope | All Mermaid sequenceDiagram. kroki.io example #14 is a case where this patch was applied |
| When converting external sources | Pasting Mermaid Live Editor examples or old Mermaid docs verbatim almost always triggers this error |

**Verification**:
```bash
echo 'sequenceDiagram
  Alice->>John: Hello
  John-->>Alice: Reply' | curl -s -X POST http://localhost:8080/mermaid/svg \
  -H 'Content-Type: text/plain' --data-binary @-
```
HTTP 200 + a valid SVG means it passes.

---

## M-03: Remove the Structurizr DSL `enterprise` keyword

**One-line summary**: Structurizr 3.0.0 deprecated then fully removed the `enterprise` keyword. Use the `group "..."` keyword for the same grouping intent.

| Item | Detail |
|---|---|
| Rejected example | `enterprise "Big Bank plc" { ... }` |
| HTTP 400 message | `enterprise keyword was previously deprecated, and has now been removed - please use group instead` |
| Passing example | `group "Big Bank plc" { ... }` |
| Scope | Structurizr DSL workspace definitions overall |
| External sources | kroki.io example #29 · part of the official Structurizr tutorial |
| Source | [Structurizr DSL group docs](https://docs.structurizr.com/dsl/language#group) |

---

## M-04: Mermaid · BPMN · Excalidraw are split workers

**One-line summary**: only metadata (the `/version` response, binary path) is registered in the Kroki 0.28.0 core container; actual rendering is a call to a worker container. HTTP 503 when the worker is down.

| Tool | Worker image | Gateway env var |
|---|---|---|
| Mermaid | `yuzutech/kroki-mermaid:0.28.0` | `KROKI_MERMAID_HOST: mermaid` |
| BPMN | `yuzutech/kroki-bpmn:0.28.0` | `KROKI_BPMN_HOST: bpmn` |
| Excalidraw | `yuzutech/kroki-excalidraw:0.28.0` | `KROKI_EXCALIDRAW_HOST: excalidraw` |

The 27 core-integrated types (`d2`·`graphviz`·`plantuml` etc.) are handled by the gateway alone with no extra worker.

**Diagnosis**:
- HTTP 200: healthy
- HTTP 400: syntax error (M-02·M-03 etc.)
- HTTP 503: worker down. Check the worker container state with `docker compose ps` → `docker compose up -d <service>`

**When introducing a new diagram tool**: POST with an empty body → HTTP 503 signals a worker is needed. The worker image naming rule is `yuzutech/kroki-{type}:0.28.0`.

---

## M-05: Fixed-width SVGs (e.g. Vega) are shrunk for mobile via CSS

**One-line summary**: Vega SVG includes an inline `width="800"` attribute, causing horizontal scroll or clipping on narrow screens. A CSS rule in Quartz `custom.scss` forces viewBox-based auto-shrink.

```scss
.kroki-diagram svg {
  max-width: 100% !important;
  width: 100% !important;
  height: auto !important;
}
```

| Item | Detail |
|---|---|
| Affected tools | Vega · Vega-Lite · GraphViz (large) · other fixed-width SVG generators |
| Applied at | `quartz/styles/custom.scss` (already applied) |
| Mechanism | The SVG `viewBox` attribute is preserved, so forcing width via CSS keeps the aspect ratio and fits the viewport width |
| Author action | **No separate action needed**. This patch is permanently applied to the Quartz build assets |

---

## M-06: Client-JS-dependent Vega transforms do not work headless

**One-line summary**: dynamic-layout transforms such as Vega's `wordcloud` depend on client JS execution, so under Kroki server-side headless render every mark's coordinate and font size falls back to default (0,0)/0px. The SVG itself is generated at around 28KB but is visually blank.

| Item | Detail |
|---|---|
| Affected transforms | `wordcloud` · `force` · other random()·event·tick-dependent transforms |
| Passing pattern | Static input + static axes·scales·marks (simple bar·scatter·line charts etc.) |
| Diagnosis pattern | Even if the Kroki POST response is HTTP 200 + a normal-size SVG, a `transform="translate(0,0)"` + `font-size="0px"` pattern on text elements signals headless failure |
| Applied case | Catalog #11 — kroki.io original Word Cloud → replaced with the official Vega simple bar chart |
| Recommended avoidance | If the visualization requires dynamic layout, first check whether it can be expressed with standard Vega-Lite encoding. If not, fall back to a static image import or Mermaid block-beta |

---

## Checklist when introducing a new diagram tool

When adding a new tool to the vault, beyond the six above, check for additional compatibility issues in this order.

1. Is the tool type name in the Quartz `KROKI_TYPES` whitelist (30 types)? If not, it must be added to the transformer code
2. Core-integrated or split worker? Confirm with an empty-body POST whether it returns 503 (M-04)
3. Does POSTing an external example verbatim return HTTP 200? If 400, investigate the syntax change (a pattern like M-02·M-03)
4. 200 but the SVG is blank? Check for JS-dependent behavior (M-06)
5. Does it generate a fixed-width SVG? Confirm the `custom.scss` rule applies (M-05)

When a new compatibility note is found, add it here only — this file is the single canonical source. The `references/kroki-catalog.md` §호환성 메모 section is a one-line index that references this file and must not duplicate per-patch detail.
