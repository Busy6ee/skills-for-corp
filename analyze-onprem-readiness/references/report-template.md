# On-Prem Readiness Report Template

## 1. Project Summary

- Project name
- Review target
- Intended on-prem context

## 2. Evidence Reviewed

- README or deployment guides reviewed
- Sample config or env files reviewed
- Other relevant source files reviewed

## 3. On-Prem Readiness Assessment

- Likely deployment model
- Key blockers
- Key risks
- Confidence level

## 4. External Dependency Review

- Hosted APIs or SaaS requirements
- Auto-download behavior
- Telemetry or analytics behavior
- Remote search, MCP, or agent dependencies

## 5. Telemetry And User Data Collection Review

- Telemetry, analytics, crash reporting, or update calls found
- Source files or modules involved
- Data types likely to be sent outbound
- Expected behavior when outbound traffic is blocked
- Whether configuration alone can disable or redirect the behavior

## 6. Configuration Draft

- Required env vars
- Required config keys
- Features to disable or replace
- Placeholder values that still require user input

## 7. Problem Report

- Non-fatal blocked outbound behavior
- Functional issues caused by blocked outbound behavior
- Risks to startup, retries, logs, or user experience
- Items that must be escalated before production use

## 8. Conditional Code Modification Guidance

- Required only when configuration is insufficient
- Candidate code paths to patch
- Recommended mitigation direction
- Risks or tradeoffs of patching

## 9. Unknowns And Manual Checks

- Unverified assumptions
- Missing documentation
- Runtime checks still needed

## 10. Recommendation

- Suitable for further on-prem setup work, yes or no
- Immediate next steps
- Whether reusable skill generation is warranted now
