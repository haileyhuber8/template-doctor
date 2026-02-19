# Project Context

- **Owner:** Hailey Victory (haileyhuber8)
- **Project:** Template Doctor — validates and analyzes Azure Developer CLI (azd) templates to increase provisioning/deployment success rates
- **Stack:** TypeScript, Node.js, Express, Vite SPA, MongoDB/Cosmos DB, Docker, Playwright, Kusto
- **Mission:** Mine Kusto telemetry for common provisioning errors, create analyzer checks to detect templates with those issues, file GitHub issues to drive fixes
- **Key Paths:** Kusto Workbench (VS Code extension) for telemetry queries, `packages/analyzer-core/src/run-analyzer.ts` for understanding existing checks
- **Created:** 2025-07-25

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### 2026-02-19: Kusto telemetry data structure and provisioning error analysis

**Database structure:**
- Primary cluster: `ddazureclients.kusto.windows.net`, database: `DevCli`
- Connection ID: `conn_1770234500628_s1l64ndmf`
- Key tables: `AzdProvisionErrorsByTemplate` (per-failure events with ARM traces), `DailyAzdProvisionsByTemplate` (daily aggregates), `AzdProvisionsByAzServiceAndTemplate` (pre-aggregated service-level stats, all-time), `AzdOperations` (all azd command executions)
- ARM traces are packed into a `Trace` dynamic column — use `| mv-expand Row = Trace | evaluate bag_unpack(Row, columnsConflict='replace_source')` to unpack. The unpacked columns use `AzureResourceProvider`, `AzureResourceType`, `HttpStatusCode` (not `ResultCode_1` or `TargetResourceProvider` as originally assumed)
- `AzdErrorCategory` has 26 distinct values. The big ones: `arm` (61%), `unknown` (14%), `bicep` (10%), `pwsh` (5.5%), `other` (3.6%), `terraform` (3.2%)
- "Other" and "No template" in TemplateName represent unregistered repos and custom code respectively — together they account for ~56K failures
- The `calcAzd*` functions are pre-built helpers but for custom analysis, querying tables directly gives more flexibility

**Key findings:**
- Overall azd provisioning failure rate is ~52% and has been flat for 12 months (range: 49.78%–55.46%)
- ARM deployment failures (`service.arm.deployment.failed` + `service.arm.400`) are 56% of all errors alone
- Bicep compilation failures (8,778) are fully detectable by static analysis — highest ROI for Template Doctor
- HTTP 429 (throttling) is a major error source across AppConfig, Storage, NSG, and Web resources — this needs AZD retry logic
- AI/ML templates have the worst failure rates (azure-gpt-rag at 80%, deploy-your-ai-application-in-production at 77%)
- CognitiveServices CapabilityHosts return HTTP 500 (1,993 traces) — likely unstable preview APIs

**Query patterns established:**
- Workbench file at `.ai-team/agents/frodo/azd-provisioning-error-analysis.kqlx` with 12 reusable query sections
- Always use `columnsConflict='replace_source'` in `bag_unpack` to avoid column name collisions
- For ARM trace analysis, filter to `toint(HttpStatusCode) >= 400` to isolate actual failures from successful operations in failed deployments
- `dcount(TemplateName)` gives affected-template breadth; high dcount = systemic issue, low dcount = template-specific bug

