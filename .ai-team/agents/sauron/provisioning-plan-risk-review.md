# Risk Review: Provisioning Error Analysis & Fix Plan

**Reviewer:** Sauron (Risk Analyst)
**Date:** 2026-02-19
**Deliverables Reviewed:**
- Frodo's Provisioning Error Analysis (332 lines)
- Gandalf's Provisioning Fix Plan (603 lines)
**Code Inspected:**
- `packages/analyzer-core/src/run-analyzer.ts` (528 lines)
- `packages/analyzer-core/src/types.ts` (95 lines)
- `packages/app/src/dashboard/adapt.ts` (196 lines)
- `packages/app/src/scripts/dashboard-renderer.ts` (category tile rendering)
- `packages/app/configs/dod-config.json`, `partner-config.json`
- `schemas/database.schema.json`, `schemas/results.schema.json`

---

## Summary

This review covers both the data analysis underpinning the initiative and the implementation plan. The initiative is well-motivated — a stable 52% provisioning failure rate is unacceptable. However, the plan has several risks ranging from CRITICAL to LOW that must be mitigated before implementation begins.

**Overall Risk Level: HIGH** — primarily driven by frontend/backend synchronization gaps, false positive potential in 4 of the 10 proposed checks, and compliance percentage regression on all existing templates.

---

## 1. Data Quality Risks (Frodo's Analysis)

### 🟡 MEDIUM: Fixability estimates are optimistic — confidence: 70%

Frodo estimates ~35,000–40,000 errors (40–45%) are "fixable in template repos." This figure is derived by summing raw error counts per category. However:

- **ARM 400 errors (15,199)**: Many `InvalidTemplateDeployment` errors are runtime-conditional (subscription policies, region availability). Static analysis cannot detect all of these. Frodo includes the full 15,199 in the "fixable" bucket, but realistically only the subset caused by actual template defects (bad SKU hardcoding, missing dependencies) is addressable — likely 30–50% of that number.
- **Bicep compilation failures (8,778)**: This is the most reliable "fixable" category — these are genuinely broken Bicep. However, Template Doctor does not run `az bicep build` (it does regex analysis), so the proposed check (B1/Frodo's recommendation #1) would require a new runtime dependency or a different approach.
- **Double-counting risk**: A single `azd provision` failure can generate multiple ARM trace entries (Frodo notes this in §9). Error counts may overstate the number of distinct user-impacting failures.

**Mitigation:** Frodo should add a confidence interval to fixability estimates. Lower bound should assume only 25–30% is template-fixable. This doesn't change the initiative's value but prevents overpromising on impact metrics.

### 🟢 LOW: Sampling window is appropriate

90-day window with 12-month trend is methodologically sound. The monthly trend table confirms the 52% rate is stable, giving confidence the data isn't skewed by one-time events.

### ✅ NO RISK: Error category taxonomy

The 26 error categories align with known AZD error classification. Frodo correctly identifies ARM as dominant (61%). The drill-down by ResultCode is actionable.

### 🟡 MEDIUM: "Unknown" errors (14%) are a blind spot

12,838 errors categorized as "unknown" include internal Go errors, auth failures, DNS issues, and JSON parse errors. These are a significant chunk that the plan doesn't address with any check. If the unknown category shifts (e.g., AZD improves error classification), the error distribution changes and check priorities may need adjustment.

**Mitigation:** Frodo should set up a recurring query (monthly) to monitor the unknown category composition. If it shifts significantly, re-prioritize checks.

---

## 2. False Positive Risks (Gandalf's 10 Checks)

This is the highest-impact risk category. A check that flags correct templates erodes template author trust and creates noise that obscures real issues. I assess each check individually:

### 🔴 CRITICAL: B4 (Bicep Parameter Validation) — Estimated FP Rate: 40–60%

The check requires: (a) all `param` declarations have `@description`, (b) SKU parameters have `@allowed`, (c) `location` defaults to `resourceGroup().location`, (d) interactive-prompt parameters are identified.

- **`@description` on every param**: The vast majority of real-world Bicep templates do NOT have `@description` on every parameter. This will flag nearly every template in the gallery. A check with 40%+ false positive rate is worse than no check.
- **`@allowed` on SKU params**: Legitimately good practice, but many templates intentionally allow flexible SKU selection. Flagging all of them treats a design choice as a defect.
- **`location` default**: Most templates already do this, but some intentionally require explicit location for multi-region deployments.

**Mitigation:** Either (a) split into sub-checks with individual enable/disable, or (b) make `severity: info` for missing `@description` and `severity: warning` only for missing defaults on params that will prompt during `azd up`. The "prompt-detection" aspect is the valuable part; the rest is noise.

### 🟠 HIGH: B6 (Quota-Risky SKU Detection) — Estimated FP Rate: 20–35%

SKU availability varies by: subscription type (PAYG vs EA vs MSDN), region, current usage, and Azure capacity. A static list of "risky" SKUs will inevitably flag templates that work fine for most users.

- `Standard_DS2_v2` is quota-constrained in some regions but available in 90%+ of deployments.
- GPU SKUs are legitimately needed for AI templates — flagging them tells the user nothing they don't already know.
- The check can't know the user's subscription type or remaining quota.

**Mitigation:** Restrict this check to truly exotic SKUs (e.g., preview-only GPU families, specific capacity-constrained SKUs like `Standard_NC24ads_A100_v4`). Frame warnings as informational, not errors. Include region guidance.

### 🟠 HIGH: B7 (Resource Dependency Ordering) — Estimated FP Rate: 25–40%

Bicep has **implicit dependency resolution** via symbolic references. If resource B references `resourceA.id`, Bicep automatically creates a dependency — no `dependsOn` needed. Regex-based analysis cannot reliably detect implicit dependencies:

- Flagging "missing `dependsOn`" when Bicep already infers it = false positive.
- Regex cannot trace symbolic references through variables, modules, or bicep expressions.
- This is marked "Large" effort for good reason — it's effectively building a partial Bicep semantic analyzer.

**Mitigation:** Either (a) defer B7 until a Bicep CLI integration is available (parse compiled JSON), or (b) scope B7 to only the most specific pattern: role assignments that use `principalId` from a managed identity resource without an explicit `dependsOn` on the identity. Even this narrow scope will have false positives from implicit deps.

### 🟡 MEDIUM: B2 (Local Authentication Detection) — Estimated FP Rate: 10–20%

Checking for `disableLocalAuth: true` is a good practice, but:

- Many demo/quickstart templates intentionally keep local auth enabled for simplicity.
- Templates targeting environments without Azure AD (e.g., certain dev/test scenarios) may legitimately need local auth.
- Some resource types (e.g., Cosmos DB) have different property names (`disableLocalAuth` vs `localAuthenticationDisabled`). Regex must cover all variants.

**Mitigation:** Severity should be `warning` (never `error`). Message should explain WHY disabling local auth matters, not just that it's missing. Maintain a map of resource type → exact property name/path.

### 🟡 MEDIUM: B8 (Container App Configuration) — Estimated FP Rate: 10–15%

Container App configurations vary significantly by architecture:

- Not all Container Apps need ingress (background workers).
- ACR pull via managed identity is best practice but not required (public images, Docker Hub).
- Regex parsing of Container App Bicep is fragile — these resources have deeply nested schemata.

**Mitigation:** Only flag missing managed environment reference (always required) and missing ingress when service host type in `azure.yaml` suggests a web endpoint. Don't flag ACR config unless template references an ACR resource.

### 🟢 LOW: B3 (Soft-Delete Prone Resources) — Estimated FP Rate: <5%

This is straightforward: detect Key Vault, Cognitive Services, API Management, and App Configuration resources and emit an informational warning about soft-delete. Low FP risk because it's informational, not judging the template as wrong.

### 🟢 LOW: B5 (azure.yaml Deep Validation) — Estimated FP Rate: <5%

Checking that `language`, `host`, and `project` paths exist and resolve is deterministic. If the path doesn't exist, it's a real problem. This check has the best signal-to-noise ratio of all proposed checks.

### 🟢 LOW: B9 (Hook Script Validation) — Estimated FP Rate: <5%

Checking file existence and shebang lines is deterministic. The cross-platform concern (`.ps1` using Linux commands) has some FP risk, but keeping it to shebang checks keeps it clean.

### ✅ NO RISK: B10 (agents.md Presence Check) — FP Rate: ~0%

File existence check. Either the file exists or it doesn't. Severity is `warning`, which is appropriate for a new standard that most templates don't meet yet.

### ✅ NO RISK: B1 (Framework Infrastructure) — N/A

This is scaffolding, not a user-facing check. No FP risk.

---

## 3. Regression Risks

### 🔴 CRITICAL: Frontend category rendering is HARDCODED and will not display "provisioning"

Both `packages/app/src/dashboard/adapt.ts` (line 75-80) and `packages/app/src/scripts/dashboard-renderer.ts` (line 492-499) have **hardcoded category lists** with exactly 6 categories:

```typescript
// adapt.ts
repositoryManagement, functionalRequirements, deployment, security, testing, agents

// dashboard-renderer.ts categoryMap
[
  { key: 'repositoryManagement', label: 'Repository Management', icon: 'fa-folder' },
  { key: 'functionalRequirements', label: 'Functional Requirements', icon: 'fa-tasks' },
  { key: 'deployment', label: 'Deployment', icon: 'fa-cloud-upload-alt' },
  { key: 'security', label: 'Security', icon: 'fa-shield-alt' },
  { key: 'testing', label: 'Testing', icon: 'fa-vial' },
  { key: 'agents', label: 'Agents', icon: 'fa-robot' },
]
```

**Gandalf's plan mentions "Frontend category tiles render automatically for new categories"** — this is incorrect. The frontend will NOT render a provisioning tile. Worse: `adapt.ts` line 86 has a fallback `categoryMap[cat] || 'repositoryManagement'`, meaning provisioning issues will **silently appear under Repository Management**, misleading users.

**Impact:** Every template scanned after the provisioning category is added to the backend will show provisioning issues miscategorized under Repository Management. Users see wrong categories. Dashboard is wrong.

**Mitigation:** MANDATORY — update both `adapt.ts` and `dashboard-renderer.ts` simultaneously with the backend changes. This must be in the same PR as B1. Add `provisioning` to:
1. `adapt.ts` `categoryMap` (add entries: `provisioning: 'provisioning'`, `quota: 'provisioning'`, etc.)
2. `adapt.ts` `categories` initialization (add `provisioning` with `enabled: true`)
3. `dashboard-renderer.ts` `categoryMap` array (add `{ key: 'provisioning', label: 'Provisioning', icon: 'fa-server' }`)

### 🟠 HIGH: Compliance percentage will DROP on all existing templates

The compliance percentage formula is:
```
percentage = compliant.length / (issues.length + compliant.length) * 100
```

Adding 10 new checks where most templates will fail several means:
- Templates currently at 80% compliance may drop to 60-65%.
- Batch scan comparison dashboards will show a sudden drop across all templates.
- Template authors receiving "your compliance dropped" notifications will be confused/alarmed.

**Impact:** Perception of regression when there is no actual change in template quality. Template authors lose trust if their score drops 20 points overnight.

**Mitigation:** Options:
1. **Phase rollout**: Enable provisioning checks as `severity: info` initially (don't count toward compliance percentage). Graduate to `warning`/`error` after template authors have time to fix.
2. **Changelog**: Clearly communicate that a new category was added and existing templates are expected to have new issues.
3. **Category-level percentages only**: Don't let provisioning checks affect the top-level compliance percentage until Phase 3. Calculate provisioning percentage independently.

I recommend option 3 — it lets us ship the checks without penalizing templates and gives authors a migration period.

### 🟡 MEDIUM: Saved results in database lack provisioning category

Existing saved analysis results in MongoDB don't have a `provisioning` category in their `categories` object. When the frontend loads old results:
- The provisioning tile will render as "0 passed / 0 issues" (0%) or not render at all depending on implementation.
- Comparison views between old and new scans will be inconsistent.

**Mitigation:** Frontend should handle missing categories gracefully (already somewhat does with `catData` null check in `dashboard-renderer.ts` line 514). Verify this works for `provisioning` before shipping.

### 🟡 MEDIUM: `AnalyzerConfig` type changes require updating all config JSON files

Adding `ProvisioningChecksConfig` to `AnalyzerConfig` type means all four ruleset configs (`dod`, `partner`, `custom`, `docs`) need updating. Config files that don't include `provisioningChecks` should get default behavior (enabled or disabled?) — this needs to be decided explicitly.

**Mitigation:** Make `provisioningChecks` optional in the interface (Gandalf already proposes this with `?`). Define default: `undefined` = disabled, so existing configs are unaffected until explicitly opted in.

---

## 4. Scope & Timeline Risks

### 🟠 HIGH: 6-week timeline with 10 checks is aggressive — 60% confidence

- **Gimli is implementing 9 of 10 checks** (B2-B10) plus the framework (B1). That's ~1 check every 3 working days including tests.
- **B7 (dependency ordering) is marked Large** and requires pseudo-Bicep-parsing that regex fundamentally cannot do well. This alone could consume a full week and still produce poor results.
- **B4 (parameter validation) requires parsing Bicep `param` declarations** including decorators (`@description`, `@allowed`). Regex for decorators is tricky with multi-line formatting.
- No time allocated for false positive tuning (iterating on thresholds after testing against real templates).

**Mitigation:**
1. Cut B7 from the 6-week scope entirely. It's the highest-risk, highest-effort check with the worst FP profile.
2. Move B4 to Phase 3 pending FP assessment against gallery templates.
3. Revised timeline: B1 + B2 + B3 (P0, weeks 1-2), B5 + B8 + B9 (P1, weeks 3-4), B6 + B10 (P1, weeks 5-6). That's 8 checks in 6 weeks — still aggressive but feasible.
4. Add a week 7 "FP tuning sprint" where checks are run against 50+ real gallery templates and thresholds are adjusted.

### 🟡 MEDIUM: Workstream A (template repo fixes) depends on Workstream B checks existing

PRs to template repos (A1-A7) are contingent on checks detecting the issues. If checks slip, PRs slip. The plan doesn't have explicit slack for this dependency chain.

**Mitigation:** Decouple A1 (disable local auth) and A2 (managed identity) from B2 — these are known patterns that can be filed manually by Sam without waiting for automated detection.

---

## 5. Blast Radius

### 🟠 HIGH: Wrong check damages batch scan credibility

Template Doctor runs batch scans across all gallery templates. If a check has a high false positive rate:
- Every batch scan generates noise issues across dozens/hundreds of templates.
- Sam's auto-filer creates GitHub issues on real template repos with incorrect findings.
- Template owners receive "your template has provisioning issues" when it doesn't — trust erodes.
- Reverting a check doesn't undo already-filed issues.

**Impact:** At current batch scan coverage, one bad check could generate 50-200 false GitHub issues before detection.

**Mitigation:**
1. Every new check MUST be tested against at least 20 real gallery templates before enabling in production.
2. New checks ship as `severity: info` for the first batch scan cycle.
3. Sam's auto-filer should have a "dry run" mode for provisioning checks during the first cycle.
4. Add a per-check "confidence" metric: if a check fires on >50% of templates, flag it for human review before filing issues.

### 🟡 MEDIUM: Rollback complexity

If a provisioning check needs to be disabled post-deployment:
- Backend: Remove from `run-analyzer.ts` or disable in config — straightforward.
- Database: Results already saved with provisioning issues can't be easily cleaned.
- Filed GitHub issues: Cannot be automatically closed.

**Mitigation:** Design check enable/disable at the config level (Gandalf already proposes this). Ensure the disable path zeros out the provisioning category rather than removing it (prevents frontend null reference errors).

---

## 6. Security Risks

### ✅ NO RISK: No new attack surface

The provisioning checks are read-only static analysis of Bicep/YAML files already fetched via the existing GitHub API integration. No new inputs, no new network calls, no new auth flows.

### ✅ NO RISK: B2 (local auth check) improves security posture

Detecting and flagging enabled local auth is security-positive. This check should eventually be `severity: error` for rulesets that enforce security baselines (DoD, partner).

---

## 7. Performance Risks

### 🟢 LOW: 10 new regex checks — negligible scan time impact

Current analyzer runs 15+ regex checks on all files. Adding 10 more regex/string checks adds < 10ms to total scan time. Bicep files are typically < 500 lines. No network calls in any proposed check.

**Caveat:** If B5 (azure.yaml deep validation) requires full YAML parsing, that adds a YAML parser dependency. Current code uses regex for azure.yaml. Adding a proper YAML parse would be better but changes the dependency profile.

### 🟡 MEDIUM: B7 (dependency ordering) could be expensive on large templates

If B7 attempts to trace symbolic references across multiple Bicep files (modules), scan time could increase significantly for complex templates with 20+ Bicep files.

**Mitigation:** Already recommended cutting B7 from initial scope. If implemented later, add timeout/file-count guard.

---

## 8. Dependency Risks

### 🟡 MEDIUM: No `checks/` directory or module system exists yet

The `packages/analyzer-core/src/` directory currently has exactly 3 files: `index.ts`, `run-analyzer.ts`, `types.ts`. There is no existing modular check architecture. B1 (framework) creates this from scratch.

**Risk:** The modular check pattern hasn't been validated. If the function signature `(files, config, issues, compliant) => void` doesn't fit all checks cleanly (e.g., some checks need async operations, some need the full `AnalyzerOptions`), the framework may need redesign mid-implementation.

**Mitigation:** Prototype B1 + B3 (the simplest check) together. Validate the function signature works before committing to it for all 10 checks. Consider `async` signatures from the start.

### 🟡 MEDIUM: AZD team coordination (Workstream C) has no SLA

Gandalf proposes complementary work with 8 existing AZD GitHub issues. Template Doctor can't control the AZD team's timeline. If AZD #6797 (hook error handling) doesn't land, A5 (pre-provision hooks) is blocked.

**Mitigation:** Already handled — Gandalf correctly notes checks are complementary, not dependent. But A5 should be explicitly marked as "blocked until AZD #6797 ships" rather than slotted into weeks 5-6.

### 🟢 LOW: No new npm dependencies required

All proposed checks are regex/string-based against already-fetched file content. No new npm packages needed (unless YAML parsing is added for B5).

---

## 9. Missing from the Plan

### 🟠 HIGH: No staged rollout strategy

The plan goes from "build check" to "ship check" with no intermediate validation stage. Missing:
- Shadow mode: Run checks without affecting scores for 1-2 weeks.
- Canary: Test on a subset of templates before batch scanning all.
- Feedback loop: How do template authors report false positives? What's the SLA to fix them?

**Mitigation:** Add a "shadow mode" phase to each check's lifecycle: Build → Shadow (info only) → Warning → Error.

### 🟡 MEDIUM: No definition of "false positive rate threshold"

Gandalf says "Sauron reviews before merge" but there's no quantitative threshold. What FP rate is acceptable? 5%? 10%? This needs to be defined before implementation starts.

**Recommendation:** Set threshold at 5% FP rate against gallery templates. If a check exceeds 5% on a test batch of 50 templates, it doesn't ship until tuned.

### 🟡 MEDIUM: Success metric "95% azd up success rate" is aspirational

Current rate is ~48% success. Target is 95%. Even if ALL template-fixable errors (Frodo's optimistic 40%) are resolved, that gets to ~70-75% success. Reaching 95% requires AZD tooling fixes, Azure platform improvements, and user subscription conditions — none of which Template Doctor controls.

**Mitigation:** Set an intermediate target: 70% success rate within 6 months (attributable to template fixes). 95% is a north star, not a 6-week deliverable.

---

## Risk Summary Matrix

| # | Risk | Level | Likelihood | Impact | Mitigation Status |
|---|------|-------|-----------|--------|-------------------|
| 1 | Frontend hardcoded categories won't render provisioning | 🔴 CRITICAL | Certain | High | Needs fix in same PR as B1 |
| 2 | B4 (param validation) FP rate 40-60% | 🔴 CRITICAL | High | High | Defer or scope-reduce |
| 3 | Compliance percentage drops on all templates | 🟠 HIGH | Certain | Medium | Phase rollout or decouple from score |
| 4 | B7 (dependency ordering) FP rate + infeasible with regex | 🟠 HIGH | High | Medium | Cut from 6-week scope |
| 5 | B6 (quota SKU) FP rate 20-35% | 🟠 HIGH | Medium | Medium | Restrict to extreme SKUs |
| 6 | Batch scan false positives → wrong GitHub issues filed | 🟠 HIGH | Medium | High | Shadow mode, dry run for auto-filer |
| 7 | No staged rollout strategy | 🟠 HIGH | Certain | Medium | Add shadow mode lifecycle |
| 8 | 6-week timeline aggressive for 10 checks | 🟠 HIGH | Medium | Medium | Cut to 8 checks, add FP tuning week |
| 9 | Fixability estimates optimistic by ~15% | 🟡 MEDIUM | Medium | Low | Add confidence intervals |
| 10 | Unknown error category is blind spot | 🟡 MEDIUM | Low | Low | Monthly monitoring query |
| 11 | B2 (local auth) FP rate 10-20% | 🟡 MEDIUM | Medium | Low | Warn-only, never error |
| 12 | B8 (Container App) FP rate 10-15% | 🟡 MEDIUM | Medium | Low | Scope to required fields only |
| 13 | Saved DB results lack provisioning category | 🟡 MEDIUM | Certain | Low | Frontend null handling |
| 14 | Config file updates needed for all rulesets | 🟡 MEDIUM | Certain | Low | Optional field, undefined = disabled |
| 15 | No FP rate threshold defined | 🟡 MEDIUM | Certain | Medium | Set 5% threshold |
| 16 | Check framework function signature untested | 🟡 MEDIUM | Medium | Low | Prototype B1+B3 together |
| 17 | AZD coordination has no SLA | 🟡 MEDIUM | Medium | Low | Mark A5 as externally blocked |
| 18 | 95% success target aspirational | 🟡 MEDIUM | Certain | Low | Set intermediate 70% target |
| 19 | Performance: regex checks negligible | 🟢 LOW | Low | Low | Monitor if >100ms |
| 20 | B3, B5, B9, B10 low FP risk | 🟢 LOW | Low | Low | Standard testing |
| 21 | No new security attack surface | ✅ NONE | N/A | N/A | N/A |
| 22 | No new npm dependencies | ✅ NONE | N/A | N/A | N/A |
| 23 | Data sampling window appropriate | ✅ NONE | N/A | N/A | N/A |

---

## Recommendations

### MUST DO (blocking)
1. **Fix frontend category hardcoding** in the same PR as B1 — both `adapt.ts` and `dashboard-renderer.ts`.
2. **Scope-reduce B4** to only check for missing defaults on parameters that will cause `azd up` interactive prompts. Drop `@description` requirement entirely for now.
3. **Cut B7 from Phase 1-3 scope.** It requires semantic understanding of Bicep that regex can't provide.
4. **Define FP threshold** at 5% and test every check against 50+ real templates before enabling.

### SHOULD DO (strongly recommended)
5. Implement shadow mode lifecycle for all checks: info → warning → error.
6. Add "dry run" flag to Sam's auto-filer for provisioning checks during first batch cycle.
7. Decouple provisioning percentage from top-level compliance percentage during rollout.
8. Prototype B1 + B3 together to validate the modular check function signature.

### NICE TO HAVE
9. Frodo adds confidence intervals to fixability estimates.
10. Set intermediate success metric target at 70%.
11. Monthly monitoring of "unknown" error category composition.

---

*Review completed by Sauron (Risk Analyst) on 2026-02-19. All risks are assessments based on code inspection and domain analysis. Confidence levels stated where uncertainty exists.*
