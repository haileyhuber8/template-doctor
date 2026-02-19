# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** `packages/analyzer-core/src/run-analyzer.ts` (core engine, ~528 lines), `packages/analyzer-core/src/types.ts` (interfaces), `packages/server/src/routes/` (Express routes), `packages/app/configs/` (ruleset JSON files)
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### 2026-02-19: Analyzer architecture deep-dive for provisioning fix plan

**Architecture patterns discovered:**
- `run-analyzer.ts` (528 lines) is a single monolithic function with all checks inline. There are helper functions (`checkReadmeRequirements`, `checkForManagedIdentity`, `detectAuthenticationMethods`, `analyzeAuthenticationMethods`) but no modular check registration system.
- The category system is mature: 6 categories (`repositoryManagement`, `functionalRequirements`, `deployment`, `security`, `testing`, `agents`) with a `categoryMap` dict that maps issue categories to display categories. Adding a new category is trivial — just add to `categoryKeys` array and `categoryMap`.
- Four ruleset configs (`dod`, `partner`, `custom`, `docs`) live as JSON in `packages/app/configs/`. They share the same `AnalyzerConfig` interface. The `securityBestPractices` field already shows how to add boolean toggles — we'll use the same pattern for `provisioningChecks`.
- The type system (`AnalysisIssue`, `CompliantItem`, `AnalysisResult`) is clean and extensible. Issues have `id`, `severity`, `category`, `message`, `error`, `details`. The `id` field is key for Sam's issue-filing deduplication.
- The AZD validation pipeline (`packages/server/src/services/azd-validation.ts`) already parses workflow artifacts for `azd up`/`azd down` results. This is runtime validation; our provisioning checks are static analysis — complementary, not overlapping.

**Key design decisions made:**
- Add `provisioning` as a new category (not a new ruleset) — this gives all 4 rulesets access to provisioning checks.
- Extract checks into a modular file system (`packages/analyzer-core/src/checks/provisioning.ts`) instead of continuing to grow the monolithic `runAnalyzer()`. This is the first step toward a proper check registry pattern.
- Checks use a `(files, config, issues, compliant) => void` function signature — stateless, pure, testable.
- Each provisioning check gets a `provisioning-` prefixed ID for Sam's issue-filing system to apply labels automatically.

**How the current analyzer fits for provisioning checks:**
- The analyzer is well-suited for static file analysis (Bicep patterns, YAML structure, file presence). It already does this for README, LICENSE, azure.yaml, workflow files, and Bicep security.
- It is NOT suited for live/runtime checks (quota querying, RBAC validation, soft-delete state). Those belong in pre-provision hooks (A5) and AZD tooling (#6800).
- The existing `detectAuthenticationMethods()` function (lines 127-145) is a good reference pattern for how provisioning checks should scan Bicep files — simple regex arrays, deterministic, no external dependencies.
- The category percentage calculation (lines 497-504) works automatically for new categories — no code changes needed in the percentage logic.

