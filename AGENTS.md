# AGENTS.md — Skills Repository 작업 지침

## 저장소 목적

회사 전체에서 공용으로 사용할 AI agent skills를 모아두고, 신규 skills를 작성·추가하는 저장소.
각 top-level 디렉토리 안에 `SKILL.md`가 있으면 하나의 스킬로 인식된다.
opencode, Claude Code 등에서 이 저장소 경로를 skills path로 설정하면 즉시 사용 가능.

## 스킬 구조

모든 스킬은 아래 구조를 따른다:

```
<skill-name>/
├── SKILL.md            # 스킬 정의 (YAML frontmatter + 마크다운 본문)
└── references/         # 지원 자료 (템플릿, 체크리스트, 가이드 등)
    ├── inputs.md       # (권장) 스킬 입력값 정의
    └── ...             # 기타 참조 자료
```

### SKILL.md 형식

```markdown
---
name: <skill-name>
description: Use when <trigger condition>.
---

# <Skill Title>

## What This Skill Does
## What This Skill Does Not Do
## Required Inputs
## Source Material
## Workflow
## Output Targets
## Reporting Requirements (or Quality Standards)
```

- `name`: 디렉토리명과 동일, 소문자 kebab-case
- `description`: "Use when..."으로 시작하는 한 문장 — AI 에이전트가 스킬 적용 여부를 판단하는 기준

## 네이밍 컨벤션

| 대상 | 규칙 | 예시 |
|------|------|------|
| 스킬 디렉토리 | 소문자 kebab-case, verb-noun 패턴 | `analyze-onprem-readiness`, `setup-postgres-ha` |
| SKILL.md | 대문자 파일명 | `SKILL.md` |
| references 내 파일 | 소문자 kebab-case `.md` | `inputs.md`, `report-template.md` |

## 신규 스킬 추가 (직접 작성)

1. repo root에 `<skill-name>/` 디렉토리 생성
2. `SKILL.md` 작성 — 위 형식의 frontmatter와 필수 섹션 포함
3. `references/` 디렉토리 생성, 최소 `inputs.md` 포함 권장
4. `analyze-onprem-readiness/` 스킬을 톤·깊이·구조의 참고 사례로 활용
5. description은 반드시 트리거 조건을 명시 ("Use when...")
6. Workflow는 구체적이고 번호가 매겨진 단계로 작성
7. "하는 것"과 "하지 않는 것"을 명확히 구분

## 외부 GitHub Repo에서 스킬 추가

오픈소스 GitHub repo URL을 받아 스킬을 추출·추가하는 절차:

### 1. 소스 분석

- repo URL을 fetch하거나 clone하여 내용 확인
- README, AGENTS.md, SKILL.md, `skills/` 디렉토리, 플러그인 정의 등에서 스킬 또는 스킬 유사 지침을 식별
- 스킬이 될 수 있는 것: 명시적 SKILL.md, AGENTS.md/README의 워크플로우 섹션, 플러그인 정의, 프롬프트 템플릿, 런북 등

### 2. 스킬 추출 및 변환

- 식별된 각 스킬마다 `<skill-name>/` 디렉토리를 표준 구조로 생성
- 소스 내용을 그대로 복사하지 않고, 이 저장소의 SKILL.md 형식으로 재작성
- 관련 참조 자료(템플릿, 체크리스트, 샘플 설정)는 `references/`에 배치
- **코드, 바이너리, 대용량 파일은 포함하지 않는다** — 문서와 템플릿만 추출

### 3. 출처 기록

`_imported/<source-repo-name>.md`에 아래 내용을 기록:

```markdown
# <Source Repo Name>

- **Source URL**: <GitHub URL>
- **License**: <라이선스>
- **Import Date**: <YYYY-MM-DD>
- **Extracted Skills**:
  - `<skill-name>`: <간략 설명>
- **Modifications**: <원본 대비 변경사항 요약>
- **Excluded Content**: <의도적으로 제외한 내용과 사유>
```

### 4. 스킬별 README.md 작성

외부 repo에서 가져온 스킬의 경우, `_imported/<source-repo-name>.md` 파일이 해당 스킬들에 대한 설명 문서 역할을 한다.
원본 repo의 목적, 스킬의 활용 맥락, 주의사항 등을 포함.

### 5. 검증

- 각 스킬 디렉토리에 유효한 frontmatter를 가진 `SKILL.md`가 있는지 확인
- `references/` 디렉토리가 존재하는지 확인
- 원본 repo 없이도 AI 에이전트가 스킬을 독립적으로 실행할 수 있는지 확인

## 품질 기준

- 모든 SKILL.md는 사전 컨텍스트 없이 AI 에이전트가 실행할 수 있을 만큼 self-contained
- 사실이 아닌 내용(엔드포인트, 시크릿, 인증 정보, 토폴로지)을 만들어내지 않는다
- 알 수 없는 값은 placeholder로 표시하고 명시적으로 호출
- 확인된 사실과 추론을 구분
- 참조 파일은 SKILL.md에서 링크로 연결, 인라인 임베딩 금지
- 스킬 간 목적이 겹치지 않도록 관리 — 유사하면 병합하거나 차별화

## 스킬이 아닌 파일

다음은 스킬로 인식되지 않는다:

- `AGENTS.md`, `README.md`, `.gitignore` (repo root)
- `_imported/` 디렉토리 (출처 기록)
- `SKILL.md`가 없는 모든 파일 또는 디렉토리
