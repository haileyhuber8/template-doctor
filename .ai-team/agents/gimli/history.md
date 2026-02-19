# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** `packages/analyzer-core/src/run-analyzer.ts` (~528 lines, core engine), `packages/analyzer-core/src/types.ts` (interfaces), `packages/server/src/routes/analyze.ts` (432 lines, analysis API), `packages/server/src/routes/validation.ts` (746 lines, validation workflow)
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

📌 Team update (2026-02-19): Provisioning error analysis found 26 error categories across 90K failures; ~40% fixable via static checks. New `provisioning` analyzer category with 10 checks planned (local auth, soft-delete, parameter validation, azure.yaml, SKU detection, etc.). Gimli will implement these checks in `packages/analyzer-core/src/checks/provisioning.ts`. P0 checks ship weeks 1-2, remainder weeks 3-6. — decided by Frodo & Gandalf

