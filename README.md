# Corporate Shared Skills

회사 전체에서 공용으로 사용하는 AI agent skills 저장소.
Claude Code, opencode 등 AI 코딩 도구에서 바로 사용할 수 있다.

## Quick Start

### 1. 저장소 클론

```bash
git clone <this-repo-url> ~/skills-for-corp
```

### 2. 도구별 설정

**opencode** — `opencode.json` (글로벌 또는 프로젝트):

```json
{
  "skills": {
    "paths": ["~/skills-for-corp"]
  }
}
```

**Claude Code** — 프로젝트의 `.claude/skills/`에 심볼릭 링크:

```bash
ln -s ~/skills-for-corp ~/.claude/skills/corp
```

또는 프로젝트별로:

```bash
ln -s ~/skills-for-corp .claude/skills/corp
```

### 3. 사용

설정 후 AI 에이전트가 자동으로 스킬을 인식하고 적절한 상황에서 활용한다.

## Available Skills

| Skill | Description |
|-------|-------------|
| [analyze-onprem-readiness](analyze-onprem-readiness/SKILL.md) | 오픈소스 프로젝트의 온프레미스 배포 준비 상태 분석 |

## Skill Structure

각 스킬은 다음 구조를 따른다:

```
<skill-name>/
├── SKILL.md          # 스킬 정의 (YAML frontmatter + 마크다운)
└── references/       # 지원 자료 (템플릿, 체크리스트, 가이드)
```

## Contributing

스킬 추가 및 관리 규칙은 [AGENTS.md](AGENTS.md)를 참고.

- **직접 작성**: 표준 구조에 맞춰 `<skill-name>/` 디렉토리 생성
- **외부 repo에서 추출**: AGENTS.md의 "외부 GitHub Repo에서 스킬 추가" 절차를 따름
- 외부에서 가져온 스킬의 출처는 `_imported/` 디렉토리에 기록

## Imported Skills

`_imported/` 디렉토리에는 외부 GitHub 저장소에서 추출한 스킬의 출처, 라이선스, 변경사항이 기록되어 있다.
