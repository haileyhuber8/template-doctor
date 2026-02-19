# Session: 2026-02-19-provisioning-analysis

**Requested by:** Hailey Victory

## Who Worked

- **Frodo** — Kusto error analysis of azd provisioning failures
- **Gandalf** — Provisioning fix plan (four-workstream architecture)
- **Sauron** — Risk review of proposed changes

## What Was Produced

- `provisioning-error-analysis.md` — 90,086 failure analysis across 26 error categories; top failing templates, per-resource failure rates, fixability breakdown
- `provisioning-fix-plan.md` — Four-workstream plan: template fixes, analyzer checks, AZD coordination, documentation
- `azd-provisioning-error-analysis.kqlx` — Kusto queries used for the telemetry analysis

## Key Findings

- 90,086 azd provisioning failures over 90 days; ARM deployment failures dominate at 61%
- Overall provisioning failure rate stable at ~52% over 12 months — not self-correcting
- ~40% of errors fixable by template authors via static analysis checks
- Top impact checks: Bicep compilation (8,778 errors), ARM pre-flight (15,199), PowerShell hooks (4,915), role assignments (2,064)

## Decisions Made

- New `provisioning` category added to the analyzer with 10 static checks
- Phased 6-week execution: P0 checks (framework + local auth + soft-delete) ship first
- Modular check architecture at `packages/analyzer-core/src/checks/provisioning.ts`
