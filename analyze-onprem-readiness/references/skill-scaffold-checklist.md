# Skill Scaffold Checklist

Generate a reusable skill only when all of the following are true:

- The user explicitly asks for `skills/<name>/` output
- The analysis report already exists or can be quoted from the current turn
- The required config keys are backed by reviewed source material
- Unresolved values are preserved as placeholders
- The new skill can follow the repository structure:
  - `skills/<name>/SKILL.md`
  - `skills/<name>/references/inputs.md`
  - `skills/<name>/references/guide.md`
  - optional sample config files under `references/`

Before generating the new skill:

- confirm the intended skill name
- confirm whether the skill is analysis-only or setup-oriented
- keep copied source material under `references/`
- avoid embedding long upstream docs directly into `SKILL.md`
- state which findings are still provisional
