# Corporate Shared Skills

회사 전체에서 공용으로 사용하는 AI agent skills 저장소.
Claude Code, opencode 등 AI 코딩 도구에서 바로 사용할 수 있다.
[Anthropic Skills 2.0 스펙](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)을 따른다.

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

### 자체 스킬

| Skill | Description |
|-------|-------------|
| [analyze-onprem-readiness](analyze-onprem-readiness/SKILL.md) | 오픈소스 프로젝트의 온프레미스 배포 준비 상태 분석 |

### Imported: [superpowers](https://github.com/obra/superpowers) (MIT)

| Skill | Description |
|-------|-------------|
| [brainstorming](brainstorming/SKILL.md) | 아이디어를 디자인/스펙으로 발전시키는 협업 프로세스 |
| [dispatching-parallel-agents](dispatching-parallel-agents/SKILL.md) | 독립적 문제를 병렬 에이전트로 동시 해결 |
| [executing-plans](executing-plans/SKILL.md) | 작성된 구현 계획 실행 및 리뷰 체크포인트 |
| [finishing-a-development-branch](finishing-a-development-branch/SKILL.md) | 개발 브랜치 완료 — 머지/PR/폐기 옵션 제시 |
| [receiving-code-review](receiving-code-review/SKILL.md) | 코드 리뷰 피드백 수신 — 기술적 검증 우선 |
| [requesting-code-review](requesting-code-review/SKILL.md) | 코드 리뷰 요청 — 리뷰어 서브에이전트 디스패치 |
| [subagent-driven-development](subagent-driven-development/SKILL.md) | 서브에이전트 기반 개발 — 태스크별 구현+2단계 리뷰 |
| [systematic-debugging](systematic-debugging/SKILL.md) | 근본 원인 분석 우선의 체계적 디버깅 프로세스 |
| [test-driven-development](test-driven-development/SKILL.md) | 테스트 우선 개발 (Red-Green-Refactor) |
| [using-git-worktrees](using-git-worktrees/SKILL.md) | Git worktree로 격리된 작업 공간 생성 |
| [using-superpowers](using-superpowers/SKILL.md) | 스킬 프레임워크 사용법 — 스킬 발견 및 호출 가이드 |
| [verification-before-completion](verification-before-completion/SKILL.md) | 완료 선언 전 검증 필수 — 증거 기반 상태 보고 |
| [writing-plans](writing-plans/SKILL.md) | 구현 계획서 작성 — 세분화된 태스크 |
| [writing-skills](writing-skills/SKILL.md) | 새 스킬 작성 — TDD 기반 프로세스 문서화 |

## Skill Structure

[Anthropic Skills 2.0](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) 스펙을 따른다:

```
<skill-name>/
├── SKILL.md              # 스킬 정의 (YAML frontmatter + 자유 형식 마크다운)
├── reference.md          # 참조 자료 (필요 시)
└── scripts/              # 유틸리티 스크립트 (필요 시)
```

## Contributing

스킬 추가 및 관리 규칙은 [AGENTS.md](AGENTS.md)를 참고.

- **직접 작성**: `<skill-name>/` 디렉토리에 `SKILL.md` 생성
- **외부 repo에서 추출**: AGENTS.md의 "외부 GitHub Repo에서 스킬 추가" 절차를 따름
- 외부에서 가져온 스킬의 출처는 `_imported/` 디렉토리에 기록

## Imported Skills

`_imported/` 디렉토리에는 외부 GitHub 저장소에서 추출한 스킬의 출처, 라이선스, 변경사항이 기록되어 있다.
