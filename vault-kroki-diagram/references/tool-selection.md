# Tool selection — detailed matrix

The 33 tools verified on the self-hosted Kroki 0.28.0 + Quartz environment, organized by use case and trade-off. Make a first tool decision from this document before authoring a new diagram; pull verified fence sources from the `references/kroki-catalog.md` catalog (internalized copy; Korean).

## 1. System architecture

| Tool | Fence | Good for | Drawback |
|---|---|---|---|
| **D2** | `d2` | Component·relation focus. Clean default style. Vault default | Some learning curve |
| GraphViz | `graphviz` | Free node·edge layout. Large graphs | Less visual polish |
| Structurizr | `structurizr` | Large-scale SW architecture modeling. Consistent C4 4-level (Context→Code) | Needs DSL learning. Heavy for a single diagram |
| C4 PlantUML | `c4plantuml` | Quick single C4 view (Context·Container·Component) | Less consistent than Structurizr |
| Nomnoml | `nomnoml` | Lightweight UML | Limited expressiveness |

**Priority**: single diagram → **D2**. Single C4 view → **C4 PlantUML**. Multi-view (L1~L4) consistent representation → **Structurizr**.

## 2. Interaction · flow

| Tool | Fence | Good for | Drawback |
|---|---|---|---|
| **Mermaid sequenceDiagram** | `mermaid` | Sequence diagram default. Rich participant·alt·loop | Worker-dependent (M-04). Needs new syntax (M-02) |
| SeqDiag | `seqdiag` | Core-integrated. No worker needed | Less expressive than Mermaid |
| Mermaid flowchart | `mermaid` | General flowchart | Worker-dependent |
| ActDiag | `actdiag` | Swim-lane activity diagram | Dated |
| BPMN | `bpmn` | Standard business-process notation | Worker-dependent. Very long source |

**Priority**: sequence → **Mermaid**. Static environment without workers → **SeqDiag**. When formal business-process notation is required → **BPMN**.

## 3. Data model

| Tool | Fence | Good for |
|---|---|---|
| **DBML** | `dbml` | Relational DB schema. dbdiagram.io compatible |
| Erd | `erd` | ER diagram (Chen notation) |
| Mermaid erDiagram | `mermaid` | Mermaid-integrated environment. Simple ERD |

**Priority**: DB schema → **DBML**. Academic/conceptual ER → **Erd**.

## 4. State · lifecycle

| Tool | Fence | Good for |
|---|---|---|
| **Mermaid stateDiagram-v2** | `mermaid` | State machine default |
| UMlet | `umlet` | Strict standard UML notation |
| PlantUML state | `plantuml` | Standard UML + extra expressiveness |

## 5. Planning · time

| Tool | Fence | Good for |
|---|---|---|
| **Mermaid gantt** | `mermaid` | Gantt default |
| Mermaid timeline | `mermaid` | Simple timeline |
| WaveDrom | `wavedrom` | Digital signal timing (not soft schedules) |

## 6. Classification · hierarchy

| Tool | Fence | Good for |
|---|---|---|
| **PlantUML mindmap** | `plantuml` | Mindmap default |
| PlantUML wbs | `plantuml` | Work Breakdown Structure |
| Mermaid mindmap | `mermaid` | Mermaid-integrated environment |

## 7. Network · hardware

| Tool | Fence | Good for |
|---|---|---|
| NwDiag | `nwdiag` | Network topology |
| RackDiag | `rackdiag` | Server rack layout |
| PacketDiag | `packetdiag` | TCP/IP packet layout |
| Bytefield | `bytefield` | Byte-level protocol fields |
| WireViz | `wireviz` | Cable wiring diagram |
| Symbolator | `symbolator` | Digital IC pin diagram |

## 8. Data visualization (charts)

| Tool | Fence | Good for | Caution |
|---|---|---|---|
| **Vega-Lite** | `vegalite` | Declarative chart default. Bar·scatter·line·histogram | |
| Vega | `vega` | Low-level visualization. Complex custom charts | JS-dependent transforms such as `wordcloud` do not work (M-06) |

**Priority**: almost all charts → **Vega-Lite**. When low-level Vega is needed, restrict to static marks·axes·scales.

## 9. Hand-drawn · free visualization

| Tool | Fence | Good for |
|---|---|---|
| Excalidraw | `excalidraw` | Hand-drawn style. Visualizing brainstorming output (worker-dependent) |
| Svgbob | `svgbob` | Convert ASCII art to SVG |
| Ditaa | `ditaa` | ASCII box diagrams |
| Pikchr | `pikchr` | Free shapes. SQLite-project notation |

## 10. Other

| Tool | Fence | Good for |
|---|---|---|
| TikZ | `tikz` | LaTeX formulas·precise figures |
| diagramsnet | `diagramsnet` | draw.io XML import |

## Tool selection decision tree

Vault default choice by requirement (when the user specifies a particular tool, prefer that):

- **System / SW structure** → D2 (single diagram) · Structurizr (multi-view L1~L4)
- **Interaction** → Mermaid sequenceDiagram (apply M-02)
- **Data model** → DBML
- **State** → Mermaid stateDiagram-v2
- **Time / schedule** → Mermaid gantt
- **Chart** → Vega-Lite (avoid M-06)
- **Mindmap** → PlantUML mindmap
- **Network / hardware** → NwDiag · RackDiag · PacketDiag · Bytefield · WireViz · Symbolator (by domain)
- **Hand-drawn** → Excalidraw (needs M-04 worker)
