# 시각화 규칙 (research-vault AGENTS.md 발췌)

> 출처: research-vault `AGENTS.md` §시각화 규칙. 글로벌 스킬 자기완결성을 위해 본 skill 내부로 내재화한 사본. 원본 정책은 research-vault에서 관리되며, 본 사본은 작성 시점 스냅샷이다. (2026-05-16 감사: 글로벌에서 미해결되는 research-vault 백링크 `[[vault-kroki-diagram]]`를 평문 코드스팬으로 중립화 — 정책 텍스트 의미는 보존.)


볼트 내 모든 인라인 다이어그램은 자체 호스팅 Kroki 0.28.0 + Quartz `KrokiDiagrams` 트랜스포머 스택에서 검증된 30종 펜스 코드블록 중 하나를 사용한다. 도구 선택·펜스 규약·호환성 패치는 `vault-kroki-diagram` skill을 통해 적용한다.

| 상황 | 스킬 | 출력 형식 |
|------|------|-----------|
| 노트 내 인라인 다이어그램 (시스템 아키텍처·시퀀스·ERD·상태·간트·마인드맵·플로우차트·차트·네트워크·하드웨어·BPMN 등) | `vault-kroki-diagram` | 타입명 단독 펜스 코드블록 (` ```d2`, ` ```mermaid`, ` ```c4plantuml`, ` ```structurizr`, ` ```vegalite` 등) |
| 노트/개념 간 공간 배치, 마인드맵, 브레인스토밍 | `obsidian-canvas-creator` | `.canvas` 파일 |

**도구 선택 원칙:**
- 단일 default(Mermaid 일률 사용)를 강제하지 않는다. **use-case별로 `vault-kroki-diagram` §3 도구 선택 빠른표에서 적합 도구를 선택**한다. 주요 매핑:
  - 시스템 아키텍처(컴포넌트·관계 중심) → **D2** (`d2`)
  - C4 모델 단일 뷰 → **C4 PlantUML** (`c4plantuml`)
  - 다중 뷰 C4 일관 표현 → **Structurizr** (`structurizr`)
  - 시퀀스 다이어그램 → **Mermaid** (`mermaid`)
  - 상태 머신·플로우차트·간트·타임라인 → **Mermaid** (`mermaid`)
  - ER 다이어그램 → **DBML** (`dbml`) 또는 **Erd** (`erd`)
  - 마인드맵·WBS → **PlantUML** (`plantuml`)
  - 차트·데이터 시각화 → **Vega-Lite** (`vegalite`)
  - 손그림·브레인스토밍 → **Excalidraw** (`excalidraw`)
  - 네트워크 토폴로지·랙·패킷·바이트필드·결선 → NwDiag·RackDiag·PacketDiag·Bytefield·WireViz (도메인별)
  - 기타 카테고리는 skill의 §3 빠른표·`references/tool-selection.md` 참조

**기본 규약:**
- 펜스는 **타입명 단독**(예: ` ```d2`, ` ```mermaid`)으로 작성한다. `kroki-{type}` prefix는 Quartz 트랜스포머와 매칭되지 않는다 (`vault-kroki-diagram` M-01).
- 외부 예시(kroki.io·Mermaid Live Editor·Structurizr Lite 등)를 그대로 옮기지 않고 `vault-kroki-diagram` §4 호환성 패치(M-01 ~ M-06)를 점검한다. 특히 Mermaid 화살표 신문법(M-02·`->>`·`-->>`)·Structurizr `group` 키워드(M-03)·Mermaid/BPMN/Excalidraw worker 의존(M-04)·Vega 헤드리스 제약(M-06).
- **ASCII 텍스트 그림을 사용하지 않는다.** 박스 드로잉 문자(`┌ ┐ └ ┘ │ ─ ├ ┤ ┬ ┴`), 텍스트 화살표(`→ ← ↓ ↑`), 파이프(`|`)·대시(`-`)로 그린 테이블 모양 레이아웃 등을 포함한다. 구조도, 아키텍처 개요, **tmux/pane 레이아웃 다이어그램** 등도 반드시 위 빠른표에서 선정한 Kroki 펜스 도구로 작성한다.
- Canvas는 별도 파일로 생성되므로, 노트에서 `![[파일명]]`으로 임베드한다.
- 다이어그램 노드 텍스트에 이모지를 사용하지 않는다.
- Mermaid 노드 라벨에 `1.`, `2.` 등 **숫자+마침표** 패턴을 사용하지 않는다 (Obsidian이 마크다운 순서 목록으로 파싱). 순번이 필요하면 원형 숫자(`①②③`)나 괄호(`1)`)를 사용한다.

