# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** All packages — reviews everything for risk. `packages/analyzer-core/src/run-analyzer.ts`, `packages/server/src/routes/`, `packages/app/configs/`, `schemas/`
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### 2026-02-19: Provisioning plan risk review
- **Frontend category hardcoding is a systemic risk**: Both `adapt.ts` and `dashboard-renderer.ts` use hardcoded category arrays. ANY new category added to the backend will be invisible in the frontend unless both files are updated simultaneously. This is not documented anywhere as a required step. Must watch for this pattern in every future category addition.
- **False positive rate is the #1 risk for new analyzer checks**: Checks that fire on >20% of real templates are worse than no check — they create noise, erode trust, and cause template authors to ignore all findings. The most dangerous checks are those that enforce aspirational standards (e.g., `@description` on every Bicep param) rather than detecting actual defects.
- **Regex-based Bicep analysis has hard limits**: Bicep implicit dependencies (`dependsOn` inferred from symbolic references) cannot be detected by regex. Any check that requires semantic understanding of Bicep (module resolution, variable tracing, implicit deps) will produce unacceptable false positive rates. The team should plan for Bicep CLI JSON output parsing as a Phase 4 investment.
- **Compliance percentage is a denominator game**: Adding new checks to the analyzer changes the denominator in the compliance formula. Even if a template's actual quality hasn't changed, its score drops. This needs either (a) decoupling new categories from the top-level score during rollout, or (b) proactive communication to template authors.
- **Batch scan + auto-filer blast radius**: A single bad check running across a batch scan generates N false GitHub issues (one per scanned template). There is no automatic recall mechanism — filed issues can't be automatically closed. Every new check needs a shadow/dry-run phase before enabling auto-filing.
- **Data analysis fixability estimates tend toward optimism**: Frodo's 40-45% "fixable in repos" estimate is a ceiling, not a floor. Many ARM errors attributed to template defects are actually runtime-conditional (subscription policies, region availability). Realistic template-fixable percentage is closer to 25-30%.

