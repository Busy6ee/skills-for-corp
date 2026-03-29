# On-Prem Readiness Analysis Inputs

## Required

| Input | Why it is needed |
|------|------|
| Review target repository, URL, or local path | Defines the project being analyzed |
| Desired output mode | Distinguishes analysis only from analysis plus config draft |
| Whether a config draft is wanted | Controls whether the skill should stop at analysis or also draft settings |

## Optional

| Input | Why it is useful |
|------|------|
| Preferred output location | Needed if the user wants artifacts saved under `docs/` |
| Deployment constraints | Clarifies air-gapped, internal-only, proxy-only, or private-network assumptions |
| Existing internal standards | Helps compare project behavior against team restrictions |
| Whether source code is available for inspection | Needed when telemetry or tracking behavior must be confirmed in code |
| Request to generate a reusable skill | Required before creating `skills/<name>/` output |
| GPU/serving environment info | GPU specs, model storage path, serving backend — needed for LLM serving projects (vLLM, TGI, etc.) |
| Proxy/certificate environment | Internal proxy address, CA certificate path, network constraints |
| Preferred skill name | Needed only if reusable skill generation is requested |

## Defaults

- Produce an analysis report by default
- Produce a config draft when the reviewed evidence is sufficient
- Inspect telemetry, analytics, auto-update, and user-data collection behavior when the source is available or the docs are inconclusive
- Do not generate a reusable skill unless the user explicitly requests it
- Leave unresolved values as placeholders instead of guessing
- Call out every inference as an inference
