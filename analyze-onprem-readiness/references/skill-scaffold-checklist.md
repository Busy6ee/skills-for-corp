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

Additional checks for specific project types:

- For LLM serving projects: model offline mode settings (e.g. `HF_HUB_OFFLINE`) are included in config keys
- Proxy and certificate settings (`HTTP_PROXY`, `HTTPS_PROXY`, `REQUESTS_CA_BUNDLE`, etc.) are reflected in the config draft when the project makes outbound HTTP calls

Before generating the new skill:

- confirm the intended skill name
- confirm whether the skill is analysis-only or setup-oriented
- keep copied source material under `references/`
- avoid embedding long upstream docs directly into `SKILL.md`
- state which findings are still provisional
