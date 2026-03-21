---
name: analyze-onprem-readiness
description: Use when reviewing an open-source project for internal, private-network, or on-premise deployment requirements before creating or updating a reusable setup skill.
---

# On-Prem Readiness Analysis

Use this skill when the user wants to evaluate a new open-source project for on-premise use before committing to a reusable setup skill.

## What This Skill Does

- Reviews the minimum source material needed to assess on-prem readiness
- Produces a structured analysis report
- Inspects outbound telemetry, analytics, auto-update, crash reporting, and user-data collection behavior when the evidence is in code or docs
- Drafts evidence-backed local configuration changes with placeholders for unknown values
- Produces a problem report for blocked outbound or user-data collection behavior
- Produces code modification guidance only when settings or documented toggles are not enough
- Generates a repository-local setup skill only when the user explicitly requests that final step

## What This Skill Does Not Do By Default

- Does not install packages, start services, or deploy the project
- Does not claim compatibility without evidence from the reviewed materials
- Does not invent endpoints, secrets, model names, or internal topology
- Does not create `skills/<name>/` output unless the user explicitly asks for it

## Required Inputs

Read [`references/inputs.md`](references/inputs.md) before starting. If required values are missing, ask only for the missing values.

## Source Material

Use only the minimum material needed for the requested review. Start with:

- the target repository or local project path provided by the user
- local README or deployment docs
- sample config files, env examples, or install instructions
- relevant source files when telemetry, tracking, auto-update, or other outbound behavior cannot be confirmed from docs alone

Use [`references/report-template.md`](references/report-template.md) as the output shape and [`references/skill-scaffold-checklist.md`](references/skill-scaffold-checklist.md) before generating any reusable skill draft.

## Workflow

1. Confirm the review target and whether the user wants:
   - analysis only
   - analysis plus config draft
   - or analysis plus config draft plus skill scaffold generation
2. Read only the minimum necessary docs and config samples.
3. Inspect code paths related to telemetry, analytics, crash reporting, auto-update, user tracking, or other outbound behavior when the reviewed docs are not sufficient.
4. Separate verified facts from inference.
5. Produce the analysis report using the repository template.
6. Draft local configuration changes only where the evidence supports them.
7. Produce a problem report for blocked outbound, telemetry, or user-data collection behavior.
8. Add code modification guidance only when:
   - a setting or documented toggle is not enough
   - and the blocked behavior causes functional issues, delay, retries, noisy errors, or unavoidable data collection attempts
9. Leave unknown values as placeholders and call them out explicitly.
10. Stop after the report, config draft, and any required remediation guidance unless the user explicitly requests a reusable skill.
11. If the user does request a reusable skill, follow the scaffold checklist and keep copied source material under `references/`.

## Output Targets

- Inline reply:
  - analysis report summary
  - config draft summary
  - problem report for outbound or data-collection issues
  - code modification guidance only when required
- Optional saved artifacts:
  - `docs/reports/<date>-<project>-onprem-analysis.md`
  - `docs/drafts/<date>-<project>-config-draft.<ext>`
- Optional generated skill:
  - `skills/<new-skill>/SKILL.md`
  - `skills/<new-skill>/references/`

## Reporting Requirements

Always make these distinctions explicit:

- confirmed fact vs inference
- supported local configuration vs unresolved placeholder
- on-prem capable vs on-prem verified
- non-fatal blocked outbound behavior vs behavior requiring remediation
- setting-based mitigation vs code-change-required mitigation
- report-only mode vs reusable skill generation mode
