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
- `description`: 필수. 최대 1024자. 3인칭으로 작성. **요약이 아닌 방아쇠(Trigger)**로 작성
  - AI가 100+ 스킬 중에서 이 스킬을 꺼낼지 결정하는 조건문이다
  - AI가 이미 아는 상식("변수명을 잘 지어라" 등)은 적지 말 것 — 토큰 낭비
  - Bad: `"코드 리뷰를 도와주는 스킬입니다"` (요약 — 언제 쓰는지 알 수 없음)
  - Good: `"PR이 올라왔을 때 보안 취약점을 검토한다. 사용자가 '리뷰해줘'라고 말할 때 호출"` (트리거)

#### Body 작성 원칙

- **자유 형식**: 고정 섹션 구조 없음. 태스크 특성에 맞게 섹션을 구성
- **간결하게**: AI는 이미 똑똑하다. AI가 모르는 것만 추가
- **Gotcha 중심**: 스킬에서 가장 가치 있는 부분은 **Gotchas(자주 빠지는 함정)** — AI가 퍼블릭 데이터로는 알 수 없는 사내 엣지 케이스 (예: 특정 메서드 호출 순서, 사내 라이브러리의 숨겨진 제약)
- **500줄 이하**: 초과 시 참조 파일로 분리
- **점진적 공개(Progressive Disclosure)**: 모든 정보를 한 번에 주입하지 않는다
  1. SKILL.md의 description만으로 트리거 판단
  2. 스킬 활성화 시 SKILL.md 본문 로드
  3. 필요할 때만 references/, examples/ 를 로드
- **참조 파일은 1단계 깊이**: SKILL.md에서 직접 참조. 참조→참조 체인 금지
- **스크립트 활용**: 절차적 지시("A 다음 B 다음 C를 해라")로 가두지 말고, 데이터 페칭/포맷팅/검증용 스크립트를 scripts/에 제공. AI는 준비된 스크립트를 레고 블록처럼 조립해 문제 해결. 스크립트는 단독 실행 가능하게 작성 (의존성, 인자 명시)

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

## 스킬 설계 원칙

### 효과적인 스킬의 3축 구조

| 축 | 핵심 | 구현 |
|---|------|------|
| **Trigger** | 언제 이 스킬이 활성화되는가 | description에 명확한 조건문 |
| **Structure** | 정보를 어떻게 조직하는가 | 점진적 공개 폴더 구조 + Gotcha 기반 마크다운 |
| **Execution** | 실제로 무엇을 실행하는가 | scripts/로 도구 연동, 외부 CLI/API 제어 |

### 진화적 개발

완벽한 스킬은 배포되지 않는다 — 진화할 뿐이다.

1. **Start Small**: 몇 줄의 지시사항으로 작게 시작
2. **Edge Case 발견**: 실사용 중 빠지는 함정 발견
3. **Gotcha 추가**: 함정에 대한 단 한 줄 추가
4. **Evolve**: 반복하며 참조 파일·스크립트로 확장

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
