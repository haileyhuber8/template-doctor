# Provisioning Fix Plan — Comprehensive Strategy

> **Author:** Gandalf (Lead / Architect)
> **Date:** 2026-02-19
> **Status:** APPROVED — Ready for execution
> **Requested by:** Hailey Victory (haileyhuber8)

---

## Executive Summary

**Goal:** Every user can run `azd up` on any gallery template and succeed with zero errors or stops.

This plan organizes work across four workstreams to systematically eliminate provisioning failures. It is informed by analysis of the current Template Doctor analyzer engine (`packages/analyzer-core/src/run-analyzer.ts`, 528 lines), the four existing ruleset configurations (`dod`, `partner`, `custom`, `docs`), the AZD validation pipeline (`packages/server/src/services/azd-validation.ts`), and the eight AZD issues filed by Shayne Boyer.

### Current Analyzer State

The analyzer today has **7 check domains** (README, LICENSE, azure.yaml, workflow files, .github folder, infra folder, Bicep security) spread across **6 categories** (`repositoryManagement`, `functionalRequirements`, `deployment`, `security`, `testing`, `agents`). There are **zero provisioning-specific checks** — no detection of patterns that cause ARM deployment failures, quota exhaustion, soft-delete conflicts, RBAC issues, or Container App misconfigurations. This is the gap we close.

### Architecture Decision: New "provisioning" Category

We will **add a `provisioning` category** to the existing category system (not a new ruleset). Rationale:
- All four rulesets (`dod`, `partner`, `custom`, `docs`) can opt in via their config JSON
- Checks slot into the existing `categories` map in `AnalysisResult`
- No schema breaking changes
- Frontend category tiles render automatically for new categories
- The `categoryMap` in `run-analyzer.ts` (line ~467) just needs a new mapping entry

### Architecture Decision: Check Modularity

Today all checks live inline in `runAnalyzer()`. For the provisioning checks, we will extract a **modular check system**:
- New file: `packages/analyzer-core/src/checks/provisioning.ts`
- Each check is an exported function with signature: `(files, config, issues, compliant) => void`
- `run-analyzer.ts` imports and calls them conditionally based on config
- This pattern will be retrofitted to existing checks in a follow-up

---

## Workstream A: Template Repo Fixes

> Things we roll out directly to template repositories via PRs (filed by Sam using issue-to-PR workflow).

### A1. Disable Local Authentication on Azure Resources

| Field | Value |
|-------|-------|
| **What** | Ensure Bicep templates set `disableLocalAuth: true` or equivalent on resources that support it (Cosmos DB, Service Bus, Event Hubs, Storage, Azure OpenAI, Cognitive Services, App Configuration, SignalR) |
| **Why** | Local auth (keys/connection strings) is the #1 security audit failure and leads to provisioning issues when org policies enforce "no local auth" via Azure Policy |
| **Who** | Legolas (Bicep), Sam (PR filing) |
| **Effort** | M |
| **Priority** | P0 |
| **Dependencies** | A1-check (B2) detects the issue; Sam files the fix PR |

### A2. Managed Identity for All Authentication

| Field | Value |
|-------|-------|
| **What** | Replace connection string / access key patterns with Managed Identity + RBAC role assignments in Bicep |
| **Why** | Connection strings cause provisioning failures when org policies block key-based auth; MI is the secure default |
| **Who** | Legolas (Bicep), Sam (PR filing) |
| **Effort** | L |
| **Priority** | P0 |
| **Dependencies** | Builds on existing `checkForManagedIdentity()` and `detectAuthenticationMethods()` in analyzer |

### A3. Parameter Validation and Defaults in Bicep

| Field | Value |
|-------|-------|
| **What** | Ensure all Bicep parameters have: (a) `@description` decorator, (b) `@allowed` or `@minLength`/`@maxLength` where appropriate, (c) sensible defaults for region/SKU |
| **Why** | Missing defaults cause interactive prompts during `azd up`; bad SKU values cause ARM deployment failures; missing descriptions confuse users |
| **Who** | Legolas |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | B4 (parameter validation check) detects these |

### A4. Add `agents.md` Files to All Templates

| Field | Value |
|-------|-------|
| **What** | Create `agents.md` in template root that documents: required Azure subscriptions, quota needs, known provisioning pitfalls, retry guidance, soft-delete recovery steps |
| **Why** | Codex/Copilot agents reading the template need machine-readable guidance; human users need troubleshooting help before hitting errors |
| **Who** | Sam (templates), Gandalf (template design) |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | D1 (agents.md template) must be designed first |

### A5. Pre-provision Hook Scripts

| Field | Value |
|-------|-------|
| **What** | Add `hooks/preprovision.sh` and `hooks/preprovision.ps1` to templates that do pre-flight checks: verify quota, check for soft-deleted resources, validate RBAC permissions |
| **Why** | Catches provisioning blockers BEFORE the ARM deployment starts (complementary to AZD issue #6800) |
| **Who** | Legolas (scripts), Gimli (hook integration) |
| **Effort** | L |
| **Priority** | P1 |
| **Dependencies** | Coordinated with AZD team — hooks must exit non-zero for `azd` to abort; AZD #6797 (hook error handling) should land first |

### A6. azure.yaml Service Configuration Hardening

| Field | Value |
|-------|-------|
| **What** | Ensure `azure.yaml` has: (a) explicit `language` per service, (b) `host` type specified, (c) correct `project` paths, (d) `hooks` section when pre-provision scripts exist |
| **Why** | Missing `language` causes deployment failures; wrong `project` paths cause "no files to deploy" errors |
| **Who** | Gimli |
| **Effort** | S |
| **Priority** | P1 |
| **Dependencies** | B5 (azure.yaml deep check) |

### A7. Resource Dependency Ordering in Bicep

| Field | Value |
|-------|-------|
| **What** | Validate that resources with RBAC role assignments have `dependsOn` on the identity resource; Key Vault access policies depend on the identity |
| **Why** | Race conditions are a top cause of transient ARM failures (related to AZD #6793 retry logic) |
| **Who** | Legolas |
| **Effort** | M |
| **Priority** | P2 |
| **Dependencies** | B7 (dependency check) |

---

## Workstream B: Template Doctor Analyzer Checks

> New rules we build in `packages/analyzer-core/src/checks/provisioning.ts` and integrate into the analyzer engine.

### Implementation Architecture

```
packages/analyzer-core/src/
├── run-analyzer.ts          # Main engine (existing, add provisioning check call)
├── types.ts                 # Types (existing, add ProvisioningChecksConfig)
├── checks/
│   └── provisioning.ts      # NEW — all provisioning check functions
│   └── index.ts             # NEW — barrel export
└── index.ts                 # Existing barrel
```

**Type additions to `types.ts`:**
```typescript
export interface ProvisioningChecksConfig {
  checkLocalAuth?: boolean;       // B2
  checkManagedIdentity?: boolean; // (existing, already in securityBestPractices)
  checkParameterDefaults?: boolean; // B4
  checkAzureYamlServices?: boolean; // B5
  checkSoftDeleteProne?: boolean;  // B3
  checkQuotaRiskySkus?: boolean;   // B6
  checkResourceDependencies?: boolean; // B7
  checkContainerAppConfig?: boolean;  // B8
  checkHookScripts?: boolean;     // B9
  checkAgentsMd?: boolean;        // B10
}

// Add to AnalyzerConfig:
export interface AnalyzerConfig {
  // ... existing fields ...
  provisioningChecks?: ProvisioningChecksConfig;
}
```

**Category mapping addition in `run-analyzer.ts`:**
```typescript
const categoryKeys = [
  'repositoryManagement',
  'functionalRequirements',
  'deployment',
  'security',
  'testing',
  'agents',
  'provisioning',  // NEW
];

const categoryMap = {
  // ... existing mappings ...
  provisioning: 'provisioning',
  quota: 'provisioning',
  softdelete: 'provisioning',
  rbac: 'provisioning',
  localauth: 'provisioning',
  hooks: 'provisioning',
};
```

**Ruleset config addition (all configs):**
```json
{
  "provisioningChecks": {
    "checkLocalAuth": true,
    "checkParameterDefaults": true,
    "checkAzureYamlServices": true,
    "checkSoftDeleteProne": true,
    "checkQuotaRiskySkus": true,
    "checkResourceDependencies": true,
    "checkContainerAppConfig": true,
    "checkHookScripts": true,
    "checkAgentsMd": true
  }
}
```

### Individual Check Specifications

#### B1. Provisioning Check Framework (Infrastructure)

| Field | Value |
|-------|-------|
| **What** | Create `packages/analyzer-core/src/checks/provisioning.ts` with the modular check system, add `provisioning` to category keys and mapping, add `ProvisioningChecksConfig` to types |
| **Why** | Foundation for all provisioning checks — must ship first |
| **Who** | Gimli (implementation), Gandalf (review) |
| **Effort** | M |
| **Priority** | P0 |
| **Dependencies** | None |

#### B2. Local Authentication Detection Check

| Field | Value |
|-------|-------|
| **What** | Scan all Bicep files for resources that support `disableLocalAuth` and flag if it's missing or set to `false`. Target resource types: `Microsoft.DocumentDB/databaseAccounts`, `Microsoft.ServiceBus/namespaces`, `Microsoft.EventHub/namespaces`, `Microsoft.Storage/storageAccounts`, `Microsoft.CognitiveServices/accounts`, `Microsoft.AppConfiguration/configurationStores`, `Microsoft.SignalRService/signalR` |
| **Why** | Org policies increasingly enforce no-local-auth; templates that don't set this fail silently or with obscure ARM errors |
| **Who** | Gimli |
| **Effort** | M |
| **Priority** | P0 |
| **Dependencies** | B1 (framework) |
| **Check ID** | `provisioning-local-auth-not-disabled-{file}` |
| **Severity** | `warning` (P0 templates) → `error` (when org-policy enforcement detected) |

#### B3. Soft-Delete Prone Resource Detection

| Field | Value |
|-------|-------|
| **What** | Detect resources that support soft-delete (Key Vault, Cognitive Services, API Management, App Configuration) and flag if the template doesn't include purge protection awareness. Emit a warning with guidance on `az keyvault purge` etc. |
| **Why** | Soft-delete conflicts are a top cause of `azd up` failures — user deploys, tears down, redeploys, and hits "resource already exists in soft-deleted state" (AZD #6794) |
| **Who** | Gimli |
| **Effort** | S |
| **Priority** | P0 |
| **Dependencies** | B1 |
| **Check ID** | `provisioning-soft-delete-risk-{resourceType}` |
| **Severity** | `warning` |

#### B4. Bicep Parameter Validation Check

| Field | Value |
|-------|-------|
| **What** | Verify that: (a) all `param` declarations have `@description`, (b) SKU parameters have `@allowed` values, (c) `location` parameter defaults to `resourceGroup().location`, (d) parameters that will prompt during `azd up` are identified |
| **Why** | Missing defaults cause interactive stops during `azd up`; invalid SKUs cause ARM failures at deploy time |
| **Who** | Gimli |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | B1 |
| **Check ID** | `provisioning-param-no-description-{param}`, `provisioning-param-no-default-{param}`, `provisioning-sku-not-validated-{param}` |
| **Severity** | `warning` |

#### B5. azure.yaml Deep Validation Check

| Field | Value |
|-------|-------|
| **What** | Parse `azure.yaml` and validate: (a) every service has `language` field, (b) every service has `host` field, (c) `project` paths resolve to actual directories in the repo, (d) `hooks` section references scripts that exist |
| **Why** | Misconfigured azure.yaml is the second most common `azd up` failure after ARM errors |
| **Who** | Gimli |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | B1 |
| **Check ID** | `provisioning-azure-yaml-missing-language-{service}`, `provisioning-azure-yaml-missing-host-{service}`, `provisioning-azure-yaml-broken-path-{service}` |
| **Severity** | `error` |

#### B6. Quota-Risky SKU Detection

| Field | Value |
|-------|-------|
| **What** | Flag resources using SKUs known to be quota-constrained in common regions: GPU SKUs for Cognitive Services, `Premium` tiers for Service Bus/Event Hubs in regions with limited capacity, high-tier App Service plans |
| **Why** | Quota exhaustion is a top ARM failure (AZD #6800); we can warn before deploy |
| **Who** | Gimli, Frodo (data on which SKUs are problematic) |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | B1, Frodo's telemetry analysis |
| **Check ID** | `provisioning-quota-risky-sku-{resource}-{sku}` |
| **Severity** | `warning` |

#### B7. Resource Dependency Ordering Check

| Field | Value |
|-------|-------|
| **What** | Detect role assignments that reference identities and verify `dependsOn` is present; detect Key Vault access policies referencing identities without dependency |
| **Why** | Missing dependencies cause race conditions leading to transient ARM failures (related to AZD #6793) |
| **Who** | Gimli, Legolas (Bicep expertise) |
| **Effort** | L |
| **Priority** | P2 |
| **Dependencies** | B1 |
| **Check ID** | `provisioning-missing-dependency-{resource}` |
| **Severity** | `warning` |

#### B8. Container App Configuration Check

| Field | Value |
|-------|-------|
| **What** | For templates using `Microsoft.App/containerApps`: verify container environment exists, ingress is configured, registry credentials are set up, and managed identity is configured for ACR pull |
| **Why** | Container App misconfigurations are a growing cause of failures (AZD #6799) |
| **Who** | Gimli |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | B1 |
| **Check ID** | `provisioning-container-app-missing-{aspect}` |
| **Severity** | `error` |

#### B9. Hook Script Validation Check

| Field | Value |
|-------|-------|
| **What** | If `azure.yaml` references hook scripts: verify files exist, verify `.sh` files have `#!/bin/bash` or `#!/usr/bin/env bash` shebang, verify `.ps1` files don't use Linux-only commands |
| **Why** | Hook failures are confusing to diagnose (AZD #6797); catching bad scripts before deploy saves users time |
| **Who** | Gimli |
| **Effort** | S |
| **Priority** | P2 |
| **Dependencies** | B1, B5 |
| **Check ID** | `provisioning-hook-missing-{hookName}`, `provisioning-hook-no-shebang-{file}` |
| **Severity** | `warning` |

#### B10. agents.md Presence and Quality Check

| Field | Value |
|-------|-------|
| **What** | Verify `agents.md` exists and contains required sections: Prerequisites, Known Issues, Provisioning Guidance |
| **Why** | AI agents need structured guidance to help users provision; this drives the agents.md rollout (A4) |
| **Who** | Gimli |
| **Effort** | S |
| **Priority** | P2 |
| **Dependencies** | B1, D1 (agents.md template) |
| **Check ID** | `provisioning-missing-agents-md`, `provisioning-agents-md-incomplete` |
| **Severity** | `warning` |

### Check Priority Matrix

| Priority | Checks | Rationale |
|----------|--------|-----------|
| **P0** | B1 (framework), B2 (local auth), B3 (soft-delete) | Foundation + highest-impact provisioning failures |
| **P1** | B4 (params), B5 (azure.yaml), B6 (quota SKUs), B8 (Container Apps) | Common failure modes from telemetry |
| **P2** | B7 (dependencies), B9 (hooks), B10 (agents.md) | Less frequent failures or foundational improvements |

### Integration with Issue Filing (Sam's Workflow)

When provisioning checks produce issues:
1. Sam's issue-filing system already reads `AnalysisResult.compliance.issues`
2. Each provisioning issue has a unique `id` with `provisioning-` prefix
3. Sam can use the `id` prefix to apply the `provisioning` label automatically
4. Issue body should include: what the check found, why it matters, how to fix it, and a link to the relevant AZD issue if applicable
5. We will create issue body templates for each check in `packages/app/src/issue/templates/provisioning/`

---

## Workstream C: AZD Tooling Improvements (Coordination)

> Cross-reference with the 8 issues Shayne filed. Template Doctor's role: complement these AZD changes, don't duplicate them.

| AZD Issue | Title | Template Doctor Complement | Coordination Notes |
|-----------|-------|---------------------------|-------------------|
| **#6796** | Improve error classification | Our checks catch patterns before deploy; AZD classifies errors at runtime. Complementary. | Share Frodo's error taxonomy with AZD team |
| **#6793** | Add retry logic for transient ARM failures | B7 (dependency ordering) reduces transient failures at source. AZD retry handles the rest. | Template fixes reduce retry burden |
| **#6794** | Detect soft-delete conflicts | B3 detects soft-delete-prone resources statically. AZD #6794 handles runtime detection. | A5 (pre-provision hooks) can also check at deploy time |
| **#6795** | Surface ARM root cause errors | We can't do this statically — this is purely an AZD improvement. | Frodo's error analysis can inform AZD's error messages |
| **#6800** | Quota/capacity pre-flight checks | B6 warns about risky SKUs statically. AZD #6800 checks actual quota at runtime. | A5 hooks can do real-time quota checking |
| **#6798** | RBAC permission pre-check | B7 detects missing `dependsOn` for role assignments. AZD #6798 checks actual permissions. | Complementary: static + runtime |
| **#6799** | Container App error handling | B8 validates Container App config statically. AZD #6799 provides better runtime errors. | Template Doctor catches config before deploy |
| **#6797** | PowerShell hook failure guidance | B9 validates hooks exist and are well-formed. AZD #6797 improves error messages when they fail. | Complementary |

### Additional AZD Improvements We Should Propose

| # | Proposal | Why |
|---|----------|-----|
| C1 | `azd validate` command — dry-run ARM template validation without deploying | Would eliminate ~30% of provisioning failures by catching errors before resource creation starts |
| C2 | `azd template check` command — invoke Template Doctor checks from CLI | Integrates our checks into the developer workflow; no need to visit the web UI |
| C3 | Structured error output format (JSON) from `azd up` | Enables programmatic error handling and automated issue filing |

---

## Workstream D: Documentation & Guidance

### D1. agents.md Template Library

| Field | Value |
|-------|-------|
| **What** | Create reusable `agents.md` templates for common template patterns: (a) Web App + SQL, (b) Container App + Cosmos DB, (c) AI/ML with Cognitive Services, (d) Serverless Functions, (e) Multi-service microservices |
| **Why** | Codex/Copilot agents reading templates need structured provisioning guidance; standardized templates ensure consistency |
| **Who** | Sam (content), Gandalf (structure design) |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | None |
| **Output** | `docs/templates/agents-md/` directory with 5 template files |

**Proposed agents.md structure:**
```markdown
# {Template Name} — Provisioning Guide

## Prerequisites
- Azure subscription with {specific resources} enabled
- {Quota requirements}
- {RBAC role requirements}

## Known Provisioning Issues
### Soft-Delete Conflicts
{Resource-specific guidance}

### Quota Limitations
{SKU-specific guidance}

### Common Errors and Fixes
| Error | Cause | Fix |
|-------|-------|-----|

## Environment Configuration
{Required environment variables and their meanings}

## Post-Deployment Verification
{Steps to verify the deployment succeeded}
```

### D2. Provisioning Error Troubleshooting Guide

| Field | Value |
|-------|-------|
| **What** | Comprehensive guide mapping common ARM error codes to root causes and fixes |
| **Why** | Users currently search Stack Overflow; we should own the canonical troubleshooting source |
| **Who** | Sam, Frodo (error data) |
| **Effort** | M |
| **Priority** | P1 |
| **Dependencies** | Frodo's error taxonomy |
| **Output** | `docs/guides/provisioning-troubleshooting.md` |

### D3. GitHub Issue Templates for Provisioning Failures

| Field | Value |
|-------|-------|
| **What** | Create issue templates that Sam's auto-filer uses, with structured sections for provisioning failures: error code, region, SKU, subscription type, steps to reproduce |
| **Why** | Structured issue reports make fixes faster; template owners get actionable data |
| **Who** | Sam |
| **Effort** | S |
| **Priority** | P1 |
| **Dependencies** | B1 (check framework provides the issue data) |
| **Output** | `packages/app/src/issue/templates/provisioning/` |

### D4. Ruleset Documentation Update

| Field | Value |
|-------|-------|
| **What** | Update `docs/features/ruleset-validation.md` to document the new `provisioning` category and all provisioning checks |
| **Why** | Users and contributors need to understand what each check does |
| **Who** | Sam |
| **Effort** | S |
| **Priority** | P2 |
| **Dependencies** | B1-B10 (checks must exist to document) |

---

## Execution Timeline

### Phase 1: Foundation (Weeks 1-2)

| Item | Owner | Dependencies |
|------|-------|-------------|
| B1 — Provisioning check framework | Gimli | None |
| B2 — Local auth detection | Gimli | B1 |
| B3 — Soft-delete detection | Gimli | B1 |
| D1 — agents.md template design | Sam, Gandalf | None |
| Frodo — Error telemetry analysis | Frodo | None |

### Phase 2: Core Checks (Weeks 3-4)

| Item | Owner | Dependencies |
|------|-------|-------------|
| B4 — Parameter validation | Gimli | B1 |
| B5 — azure.yaml deep validation | Gimli | B1 |
| B6 — Quota-risky SKU detection | Gimli, Frodo | B1, Frodo data |
| B8 — Container App config | Gimli | B1 |
| A1 — Local auth fixes in templates | Legolas, Sam | B2 (detects issues) |
| A2 — Managed Identity rollout | Legolas, Sam | B2 |

### Phase 3: Polish & Docs (Weeks 5-6)

| Item | Owner | Dependencies |
|------|-------|-------------|
| B7 — Resource dependency check | Gimli, Legolas | B1 |
| B9 — Hook script validation | Gimli | B1, B5 |
| B10 — agents.md check | Gimli | B1, D1 |
| A3 — Parameter hardening | Legolas | B4 |
| A4 — agents.md rollout | Sam | D1 |
| A5 — Pre-provision hooks | Legolas, Gimli | AZD #6797 |
| D2 — Troubleshooting guide | Sam, Frodo | Frodo data |
| D3 — Issue templates | Sam | B1 |
| D4 — Docs update | Sam | B1-B10 |

### Phase 4: Coordination (Ongoing)

| Item | Owner | Dependencies |
|------|-------|-------------|
| A6, A7 — azure.yaml & dependency fixes | Gimli, Legolas | B5, B7 |
| C1-C3 — AZD proposals | Gandalf, Hailey | AZD team engagement |
| Eowyn — Test coverage for all new checks | Eowyn | B1-B10 |
| Sauron — Risk review of new check accuracy | Sauron | B1-B10 |

---

## Testing Strategy

### Unit Tests (Eowyn)

Every check in `provisioning.ts` gets:
- **Positive test:** Correct Bicep/YAML → passes check, produces `CompliantItem`
- **Negative test:** Bad Bicep/YAML → produces correct `AnalysisIssue` with right `id`, `severity`, `message`
- **Edge case tests:** Empty files, malformed YAML, Bicep with unusual formatting
- **No false positive tests:** Each check must demonstrate zero false positives on 10+ real templates

Test location: `packages/analyzer-core/src/checks/__tests__/provisioning.test.ts`

### Integration Tests (Eowyn)

- Run full `runAnalyzer()` with provisioning config enabled against fixture repos
- Verify provisioning issues appear in `result.compliance.categories.provisioning`
- Verify issue count in compliance percentage calculation

### Playwright E2E Tests

- Category tile for "provisioning" renders when issues exist
- Issue details display correctly in the UI
- Sam's issue-filing includes provisioning issues in generated GitHub issue body

---

## Risk Assessment (Sauron Review Required)

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **False positives** — checks flag correct templates | Medium | High | Extensive testing against real gallery; Sauron reviews before merge |
| **Regex fragility** — Bicep regex patterns miss valid syntax | Medium | Medium | Use Bicep parser if available; otherwise test against diverse Bicep samples |
| **Scope creep** — too many checks at once | Low | Medium | Phase execution strictly; no check ships without tests |
| **AZD coordination delays** — AZD issues not resolved in time | Medium | Low | Our checks are complementary, not dependent; hooks can validate independently |
| **Performance** — many new checks slow analysis | Low | Low | Checks are regex/string-based, not network-bound; profile if >100ms added |

---

## Success Metrics

| Metric | Current | Target | Measurement |
|--------|---------|--------|-------------|
| `azd up` success rate (gallery templates) | ~70% (estimated from telemetry) | 95%+ | Frodo's Kusto dashboard |
| Template Doctor checks covering provisioning issues | 0 | 10+ | Count of `provisioning-*` check IDs |
| Templates with local auth disabled | ~20% | 100% | Batch scan results |
| Templates with agents.md | ~5% | 80%+ | Batch scan results |
| Average time to fix provisioning issue (from detection to PR) | Unknown | < 48 hours | Sam's issue-to-PR metrics |

---

## Open Questions

1. **Should we parse Bicep with a real parser (e.g., Bicep CLI JSON output) or continue with regex?** Regex is faster to implement but more fragile. A Bicep compiler integration would be more reliable but adds a runtime dependency. **Recommendation:** Start with regex (Phase 1-2), evaluate Bicep CLI integration as a Phase 4 improvement if false positives are >5%.

2. **Should provisioning checks run on all rulesets by default?** **Recommendation:** Yes, enable by default on all rulesets. Individual checks can be disabled per-config.

3. **How do we handle Terraform templates?** Current checks target Bicep only. **Recommendation:** Scope provisioning checks to Bicep initially. Add Terraform support as a separate workstream if demand warrants it.

4. **Should we integrate with Azure Resource Graph for live validation?** **Recommendation:** Not in this phase. Live validation belongs in the pre-provision hooks (A5) and AZD's own pre-flight checks (#6800). Template Doctor should remain a static analysis tool.

---

## Appendix: Existing Category System Reference

Current categories in `run-analyzer.ts` (lines 452-460):
```
repositoryManagement, functionalRequirements, deployment, security, testing, agents
```

Current category mapping (lines 465-482):
```
file → repositoryManagement
folder → repositoryManagement
missing → repositoryManagement
required → repositoryManagement
readme → functionalRequirements
documentation → functionalRequirements
workflow → deployment
infra → deployment
infrastructure → deployment
azure → deployment
bicep → deployment
bicepFiles → deployment
security → security
auth → security
authentication → security
testing → testing
test → testing
agents → agents
meta → meta
```

New mappings to add:
```
provisioning → provisioning
quota → provisioning
softdelete → provisioning
rbac → provisioning
localauth → provisioning
hooks → provisioning
containerapp → provisioning
paramvalidation → provisioning
```
