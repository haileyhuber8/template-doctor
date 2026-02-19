# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** `packages/analyzer-core/src/run-analyzer.ts` (existing checks), `packages/app/configs/` (ruleset JSON), `infra/` (project's own Bicep infrastructure)
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

📌 Team update (2026-02-19): Provisioning error analysis identified top failing templates (azure-search-openai-demo, azure-gpt-rag, deploy-your-ai-application-in-production) and common Bicep issues (local auth enforcement, soft-delete conflicts, missing parameter defaults). Legolas will fix these template repos as part of Workstream A. — decided by Frodo & Gandalf

