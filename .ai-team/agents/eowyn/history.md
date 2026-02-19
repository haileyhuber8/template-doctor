# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** `packages/app/tests/` (Playwright tests), `tests/unit/` (unit tests), `playwright.config.js`, `vitest.config.mjs`, `scripts/smoke-api.sh`
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

📌 Team update (2026-02-19): New `provisioning` analyzer category with 10 static checks planned. Eowyn will write tests for each check (Bicep compilation, SKU/quota risk, ARM pre-flight, role assignments, PowerShell hooks, local auth, soft-delete, parameter validation, azure.yaml validation, Container App config). Test files expected in `packages/analyzer-core/` and Playwright integration tests. — decided by Frodo & Gandalf

