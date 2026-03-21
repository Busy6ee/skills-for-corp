# AGENTS.md — Skills Repository 작업 지침

## 저장소 목적

회사 전체에서 공용으로 사용할 AI agent skills를 모아두고, 신규 skills를 작성·추가하는 저장소.
각 top-level 디렉토리 안에 `SKILL.md`가 있으면 하나의 스킬로 인식된다.
opencode, Claude Code 등에서 이 저장소 경로를 skills path로 설정하면 즉시 사용 가능.

## 스킬 구조

[Anthropic Skills 2.0 스펙](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)을 따른다.

```
<skill-name>/
├── SKILL.md              # 스킬 정의 (YAML frontmatter + 마크다운 본문, 필수)
├── reference.md          # 참조 자료 (필요 시, on-demand 로딩)
├── examples.md           # 사용 예시 (필요 시)
└── scripts/              # 유틸리티 스크립트 (필요 시, 실행용)
```

### SKILL.md 형식

```markdown
---
name: <skill-name>
description: <스킬이 하는 일과 언제 사용하는지. 3인칭으로 작성>
---

<자유 형식 마크다운 — 태스크에 맞는 섹션 구성>
```

#### Frontmatter 규칙

- `name`: 소문자, 숫자, 하이픈만 사용. 최대 64자. 디렉토리명과 동일
- `description`: 필수. 최대 1024자. 스킬이 하는 일 + 트리거 조건을 모두 포함. 3인칭으로 작성

#### Body 작성 원칙

- **자유 형식**: 고정 섹션 구조 없음. 태스크 특성에 맞게 섹션을 구성
- **간결하게**: Claude는 이미 똑똑하다. Claude가 모르는 것만 추가
- **500줄 이하**: 초과 시 참조 파일로 분리 (progressive disclosure)
- **참조 파일은 1단계 깊이**: SKILL.md에서 직접 참조. 참조→참조 체인 금지

## 네이밍 컨벤션

| 대상 | 규칙 | 예시 |
|------|------|------|
| 스킬 디렉토리 | 소문자 kebab-case | `analyze-onprem-readiness`, `test-driven-development` |
| SKILL.md | 대문자 파일명 | `SKILL.md` |
| 참조 파일 | 소문자 kebab-case, 내용을 설명하는 이름 | `testing-anti-patterns.md`, `plan-document-reviewer-prompt.md` |

## 신규 스킬 추가 (직접 작성)

1. repo root에 `<skill-name>/` 디렉토리 생성
2. `SKILL.md` 작성 — frontmatter(`name`, `description`)와 태스크에 맞는 본문
3. 필요 시 참조 파일 추가 (SKILL.md에서 링크로 연결)
4. description에 트리거 조건을 명확히 포함 (Claude가 100+ 스킬 중 선택할 수 있도록)
5. 기존 스킬과 목적이 겹치지 않는지 확인

## 외부 GitHub Repo에서 스킬 추가

오픈소스 GitHub repo URL을 받아 스킬을 추출·추가하는 절차:

### 1. 소스 분석

- repo URL을 fetch하거나 clone하여 내용 확인
- README, AGENTS.md, SKILL.md, `skills/` 디렉토리, 플러그인 정의 등에서 스킬 또는 스킬 유사 지침을 식별
- 스킬이 될 수 있는 것: 명시적 SKILL.md, AGENTS.md/README의 워크플로우 섹션, 플러그인 정의, 프롬프트 템플릿, 런북 등

### 2. 스킬 추출

- 식별된 각 스킬마다 `<skill-name>/` 디렉토리 생성
- **원본 스킬이 이미 Skills 2.0 스펙에 부합하면 원본 그대로 사용** — 불필요한 변환 금지
- 원본이 Skills 2.0에 맞지 않는 경우에만 스펙에 맞게 조정
- 참조 파일(프롬프트 템플릿, 가이드 등)은 원본 구조 유지
- **코드, 바이너리, 대용량 파일은 포함하지 않는다** — 문서와 템플릿만 추출 (스크립트는 스킬 실행에 필요한 경우 포함 가능)

### 3. 출처 기록

`_imported/<source-repo-name>.md`에 아래 내용을 기록:

```markdown
# <Source Repo Name>

- **Source URL**: <GitHub URL>
- **License**: <라이선스>
- **Import Date**: <YYYY-MM-DD>
- **Imported Skills**:
  - `<skill-name>`: <간략 설명>
- **Modifications**: <원본 대비 변경사항 요약 (없으면 "원본 그대로 사용")>
- **Excluded Content**: <의도적으로 제외한 내용과 사유>
```

### 4. 검증

- 각 스킬 디렉토리에 유효한 frontmatter를 가진 `SKILL.md`가 있는지 확인
- 원본 repo 없이도 AI 에이전트가 스킬을 독립적으로 실행할 수 있는지 확인
- description이 충분히 구체적인지 확인 (what + when)

## 품질 기준

- 모든 SKILL.md는 사전 컨텍스트 없이 AI 에이전트가 실행할 수 있을 만큼 self-contained
- SKILL.md body는 500줄 이하 — 초과 시 참조 파일로 분리
- 사실이 아닌 내용(엔드포인트, 시크릿, 인증 정보, 토폴로지)을 만들어내지 않는다
- 알 수 없는 값은 placeholder로 표시하고 명시적으로 호출
- 참조 파일은 SKILL.md에서 링크로 연결, 인라인 임베딩 금지
- 스킬 간 목적이 겹치지 않도록 관리 — 유사하면 병합하거나 차별화

## 스킬이 아닌 파일

다음은 스킬로 인식되지 않는다:

- `AGENTS.md`, `README.md`, `.gitignore` (repo root)
- `_imported/` 디렉토리 (출처 기록)
- `SKILL.md`가 없는 모든 파일 또는 디렉토리
