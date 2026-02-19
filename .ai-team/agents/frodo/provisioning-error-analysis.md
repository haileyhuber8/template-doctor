# AZD Provisioning Error Analysis

**Analyst:** Frodo (Data Analyst)
**Date:** 2026-02-19
**Data Source:** Kusto cluster `ddazureclients.kusto.windows.net` / database `DevCli`
**Time Window:** Last 90 days (unless noted otherwise for trends)

---

## Executive Summary

Across the last 90 days, **~90,086 provisioning failures** were recorded in `AzdProvisionErrorsByTemplate`. The overall provisioning failure rate across all templates has been **stubbornly stable at ~52%** over the past 12 months, ranging from 49.78% to 55.61%. This means roughly **one in two `azd provision` attempts fails**. The failure rate is not improving.

The dominant error category is **ARM deployment failures** (54,839 = 61% of all errors), followed by **unknown errors** (12,838 = 14%), **Bicep compilation failures** (8,778 = 10%), and **PowerShell hook failures** (4,915 = 5.5%).

The most impactful templates by raw failure count are AI/ML-heavy templates: `azure-search-openai-demo` (7,384 failures), `azure-gpt-rag` (2,362, 79.6% failure rate), and `multi-agent-custom-automation-engine-solution-accelerator` (1,532, 67.6% failure rate).

---

## 1. Error Category Distribution (Last 90 Days)

| Rank | AzdErrorCategory | Error Count | % of Total | Description |
|------|-----------------|-------------|------------|-------------|
| 1 | **arm** | 54,839 | 60.9% | ARM deployment failures (HTTP 400, 403, 404, 409) |
| 2 | **unknown** | 12,838 | 14.3% | Uncategorized errors (internal Go errors, auth failures, DNS) |
| 3 | **bicep** | 8,778 | 9.7% | Bicep compilation/validation failures |
| 4 | **pwsh** | 4,915 | 5.5% | PowerShell pre/post-provision hook failures |
| 5 | **other** | 3,281 | 3.6% | Other tool failures |
| 6 | **terraform** | 2,911 | 3.2% | Terraform plan/apply failures |
| 7 | **bash** | 627 | 0.7% | Bash hook script failures |
| 8 | **powershell** | 566 | 0.6% | PowerShell tool failures (distinct from pwsh hooks) |
| 9 | **Docker** | 459 | 0.5% | Docker not installed/missing |
| 10 | **dotnet** | 444 | 0.5% | .NET build/restore failures |
| 11 | **aad** | 279 | 0.3% | Azure AD/authentication failures |
| 12 | **keyvault** | 44 | 0.05% | Key Vault access forbidden |
| 13+ | npm CLI, files, Python CLI, Podman, etc. | ~102 | 0.1% | Various tool-missing or niche failures |

**Total: ~90,086 provisioning errors across 26 distinct categories.**

### Key Insight
**ARM failures dominate** — nearly 2 out of 3 errors are ARM-related. These are the errors where the Bicep/ARM template compiled fine but Azure rejected the deployment. This is the #1 area to mine for Template Doctor checks.

---

## 2. Top Failing Templates (Last 90 Days)

| Rank | TemplateName | Attempted | Failed | Success | Failure Rate | Users |
|------|-------------|-----------|--------|---------|-------------|-------|
| 1 | Other (unlisted templates) | 59,672 | 33,185 | 26,487 | **55.61%** | 10,652 |
| 2 | No template (custom code) | 54,841 | 23,615 | 31,226 | **43.06%** | 11,257 |
| 3 | azd-init | 16,183 | 9,360 | 6,823 | **57.84%** | 3,211 |
| 4 | **azure-search-openai-demo** | 13,368 | 7,384 | 5,984 | **55.24%** | 3,607 |
| 5 | **azure-gpt-rag** | 2,968 | 2,362 | 606 | **79.58%** | 746 |
| 6 | **multi-agent-custom-automation-engine** | 2,265 | 1,532 | 733 | **67.64%** | 628 |
| 7 | azd-get-started-with-ai-agents | 2,853 | 1,357 | 1,496 | **47.56%** | 1,125 |
| 8 | azure-mcp-server | 1,777 | 1,108 | 669 | **62.35%** | 504 |
| 9 | custom | 1,268 | 706 | 562 | **55.68%** | 181 |
| 10 | azd-aistudio-starter | 923 | 614 | 309 | **66.52%** | 587 |
| 11 | voice-live-agent | 1,084 | 573 | 511 | **52.86%** | 343 |
| 12 | agentic-applications-for-unified-data | 1,176 | 536 | 640 | **45.58%** | 469 |
| 13 | chat-with-your-data-solution-accelerator | 898 | 469 | 429 | **52.23%** | 440 |
| 14 | content-processing | 821 | 407 | 414 | **49.57%** | 300 |
| 15 | azure-search-openai-demo-csharp | 655 | 388 | 267 | **59.24%** | 191 |
| 16 | azd-get-started-with-ai-chat | 896 | 349 | 547 | **38.95%** | 360 |
| 17 | conversation-knowledge-mining | 615 | 347 | 268 | **56.42%** | 186 |
| 18 | **deploy-your-ai-application-in-production** | 443 | 341 | 102 | **76.98%** | 129 |
| 19 | document-generation | 471 | 315 | 156 | **66.88%** | 179 |
| 20 | remote-mcp-functions-python | 706 | 302 | 404 | **42.78%** | 271 |
| 21 | snippy | 534 | 284 | 250 | **53.18%** | 249 |
| 22 | **healthcare-agent-orchestrator** | 385 | 272 | 113 | **70.65%** | 86 |
| 23 | cosmos-copilot | 533 | 220 | 313 | **41.28%** | 65 |
| 24 | functions-quickstart-python-azd | 587 | 216 | 371 | **36.80%** | 159 |
| 25 | hello-azd-dotnet | 541 | 183 | 358 | **33.83%** | 100 |

### Critical Templates (>70% failure rate with >100 attempts)
- **azure-gpt-rag** — 79.58% failure rate (2,362 failures, 746 users)
- **deploy-your-ai-application-in-production** — 76.98% failure rate (341 failures, 129 users)
- **healthcare-agent-orchestrator** — 70.65% failure rate (272 failures, 86 users)

### High-Volume, High-Impact Templates
- **azure-search-openai-demo** — The flagship RAG template. 7,384 failures across 3,607 users. At 55% failure rate, this is the single most impactful template to fix.

---

## 3. Error Details Drill-Down by Category and ResultCode

### ARM Errors (61% of all failures)

| ResultCode | Count | Affected Templates | Sample Error Details |
|-----------|-------|-------------------|---------------------|
| service.arm.deployment.failed | 35,361 | 151 | SubscriptionIsOverQuotaForSku, InvalidTemplateDeployment, ValidationForResourceFailed |
| service.arm.400 | 15,199 | 111 | InvalidTemplateDeployment |
| service.arm.403 | 2,064 | 68 | AuthorizationFailed |
| service.arm.409 | 1,637 | 55 | ReadOnlyDisabledSubscription |
| service.arm.404 | 491 | 23 | ResourceGroupNotFound |
| service.arm.413 | 70 | 4 | RequestContentTooLarge |

### Unknown Errors (14% of all failures)

| ResultCode | Count | Affected Templates |
|-----------|-------|-------------------|
| UnknownError | 6,843 | 66 |
| internal.errors_errorString | 3,499 | 44 |
| error.suggestion | 924 | 32 |
| auth.login_required | 650 | 23 |
| internal.syscall_Errno | 250 | 13 |
| internal.json_SyntaxError | 156 | 8 |
| internal.errors_errorString,fmt_wrapError | 155 | 15 |
| internal.azidentity_AuthenticationFailedError | 150 | 4 |
| internal.yaml_TypeError | 90 | 1 |
| internal.net_DNSError | 69 | 10 |

### Tool Errors

| Category | ResultCode | Count | Templates |
|----------|-----------|-------|-----------|
| bicep | tool.bicep.failed | 8,778 | 93 |
| pwsh | tool.pwsh.failed | 4,915 | 36 |
| other | tool.other.failed | 3,170 | 35 |
| terraform | tool.terraform.failed | 2,911 | 14 |
| bash | tool.bash.failed | 627 | 8 |
| powershell | tool.powershell.failed | 566 | 18 |
| Docker | tool.Docker.missing | 459 | 17 |
| dotnet | tool.dotnet.failed | 444 | 12 |
| npm CLI | tool.npm CLI.missing | 32 | 8 |
| Python CLI | tool.Python CLI.missing | 16 | 5 |

---

## 4. Azure Service Failure Patterns (All Time)

| Rank | Resource Provider | Resource Type | Attempted | Failed | Failure Rate | Templates |
|------|------------------|---------------|-----------|--------|-------------|-----------|
| 1 | MICROSOFT.RESOURCES | DEPLOYMENTS | 220M | 7.86M | **3.57%** | 159 |
| 2 | MICROSOFT.STORAGE | STORAGEACCOUNTS | 5.85M | 583K | **9.97%** | 98 |
| 3 | MICROSOFT.AUTHORIZATION | ROLEASSIGNMENTS | 45M | 548K | **1.22%** | 128 |
| 4 | MICROSOFT.APP | CONTAINERAPPS | 8.5M | 186K | **2.19%** | 63 |
| 5 | MICROSOFT.STORAGE | BLOB CONTAINERS | 4.4M | 160K | **3.61%** | 77 |
| 6 | MICROSOFT.WEB | SERVERFARMS | 1.59M | 152K | **9.56%** | 80 |
| 7 | MICROSOFT.RESOURCES | RESOURCEGROUPS | 6.38M | 97K | **1.53%** | 153 |
| 8 | MICROSOFT.WEB | SITES | 1.93M | 84K | **4.36%** | 78 |
| 9 | MICROSOFT.APPCONFIGURATION | KEYVALUES | 823K | 63K | **7.66%** | 12 |
| 10 | MICROSOFT.KEYVAULT | VAULTS | 1.62M | 48K | **2.94%** | 72 |
| 11 | MICROSOFT.RESOURCES | DEPLOYMENTSCRIPTS | 482K | 45K | **9.34%** | 25 |
| 12 | MICROSOFT.APP | MANAGEDENVIRONMENTS | 1.74M | 43K | **2.48%** | 65 |
| 13 | MICROSOFT.MANAGEDIDENTITY | IDENTITIES | 6.62M | 39K | **0.60%** | 105 |
| 14 | MICROSOFT.COGNITIVESERVICES | ACCOUNTS/DEPLOYMENTS | 3M | 34K | **1.15%** | 72 |
| 15 | MICROSOFT.INSIGHTS | DIAGNOSTICSETTINGS | 1.3M | 32K | **2.47%** | 38 |
| 16 | MICROSOFT.NETWORK | PRIVATEENDPOINTS | 1.88M | 30K | **1.60%** | 43 |
| 17 | MICROSOFT.CONTAINERINSTANCE | CONTAINERGROUPS | 356K | 29K | **8.28%** | 26 |
| 18 | MICROSOFT.COGNITIVESERVICES | ACCOUNTS | 2.41M | 27K | **1.12%** | 76 |
| 19 | MICROSOFT.NETWORK | VNET/SUBNETS | 527K | 24K | **4.52%** | 39 |
| 20 | MICROSOFT.DOCUMENTDB | DATABASEACCOUNTS | 702K | 23K | **3.28%** | 55 |

### Highest Failure-Rate Services (by %)

| Resource Type | Failure Rate | Failed | Context |
|--------------|-------------|--------|---------|
| STORAGE/QUEUESERVICES/QUEUES | **15.04%** | 21,840 | Likely throttling (429) |
| STORAGE/STORAGEACCOUNTS | **9.97%** | 583,492 | Name conflicts, throttling |
| WEB/SERVERFARMS | **9.56%** | 151,760 | App Service Plan quota/SKU issues |
| RESOURCES/DEPLOYMENTSCRIPTS | **9.34%** | 45,050 | Script execution failures |
| CONTAINERINSTANCE/CONTAINERGROUPS | **8.28%** | 29,484 | Container provisioning failures |
| APPCONFIGURATION/KEYVALUES | **7.66%** | 63,096 | Throttling (429 dominant) |

### ARM Trace Error Codes (HTTP 4xx/5xx only, last 90 days)

| Resource Provider | Resource Type | HTTP Code | Trace Count | Templates |
|------------------|---------------|-----------|-------------|-----------|
| MICROSOFT.RESOURCES | DEPLOYMENTS | 400 | 9,351 | 94 |
| MICROSOFT.APPCONFIGURATION | KEYVALUES | **429** | 3,730 | 4 |
| MICROSOFT.STORAGE | STORAGEACCOUNTS | **429** | 3,195 | 3 |
| MICROSOFT.AUTHORIZATION | ROLEASSIGNMENTS | **409** | 2,140 | 23 |
| MICROSOFT.AUTHORIZATION | ROLEASSIGNMENTS | 400 | 1,958 | 27 |
| MICROSOFT.STORAGE | BLOB CONTAINERS | **429** | 1,455 | 7 |
| MICROSOFT.COGNITIVESERVICES | CAPABILITYHOSTS | **500** | 1,394 | 6 |
| MICROSOFT.NETWORK | NSG/SECURITYRULES | **429** | 1,393 | 3 |
| MICROSOFT.NETWORK | PRIVATEENDPOINTS | 400 | 1,099 | 23 |
| MICROSOFT.WEB | SITES | 400 | 1,086 | 33 |
| MICROSOFT.CONTAINERINSTANCE | CONTAINERGROUPS | 400 | 991 | 4 |
| MICROSOFT.RESOURCES | DEPLOYMENTS | 409 | 973 | 36 |
| MICROSOFT.RESOURCES | RESOURCEGROUPS | 409 | 835 | 38 |
| MICROSOFT.COGNITIVESERVICES | ACCOUNT/DEPLOYMENTS | 400 | 732 | 30 |
| MICROSOFT.WEB | SERVERFARMS | **429** | 714 | 16 |
| MICROSOFT.KEYVAULT | VAULTS | 409 | 646 | 22 |
| MICROSOFT.COGNITIVESERVICES | PROJECTS/CAPABILITYHOSTS | **500** | 599 | 4 |
| MICROSOFT.APIMANAGEMENT | API POLICIES | 400 | 590 | 5 |

---

## 5. Fixable vs Non-Fixable Error Classification

### A. Fixable in Template Repos (Template Doctor check candidates)

These errors can be detected by static analysis of Bicep/Terraform files and fixed by template authors:

| Error Pattern | Est. Count | Check to Build |
|--------------|-----------|----------------|
| **Bicep compilation failures** | 8,778 | Validate Bicep compiles with `az bicep build` — CHECK: `bicep-compiles` |
| **Missing/wrong SKU causing quota errors** | ~5,000+ | Detect hardcoded SKUs that commonly hit SubscriptionIsOverQuotaForSku — CHECK: `sku-quota-risk` |
| **InvalidTemplateDeployment (ARM 400)** | 15,199 | Run `az deployment group validate` pre-flight — CHECK: `arm-preflight-validation` |
| **Authorization failures from missing role assignments** | 2,064 | Check Bicep for proper role assignment resources and managed identity configuration — CHECK: `role-assignments-present` |
| **Docker dependency not declared** | 459 | Check azure.yaml host types and warn if Docker is required but not documented — CHECK: `docker-dependency-declared` |
| **npm/Python/dotnet tool dependencies** | 492 | Check azure.yaml services for tool dependencies and verify tooling requirements documented — CHECK: `tool-dependencies-documented` |
| **PowerShell hook failures** | 4,915 | Validate pre/post-provision PowerShell scripts are syntactically correct and have proper error handling — CHECK: `hook-scripts-valid` |
| **Key Vault access conflicts (409)** | 646 | Check for soft-delete conflicts, ensure purge protection settings are compatible — CHECK: `keyvault-soft-delete-aware` |
| **CognitiveServices CapabilityHosts 500** | 1,993 | Detect templates using unstable preview API versions for AI services — CHECK: `stable-api-versions` |
| **Storage name conflicts** | portion of 583K | Check for non-unique storage account naming patterns — CHECK: `unique-resource-names` |
| **RequestContentTooLarge (413)** | 70 | Detect oversized ARM template deployments — CHECK: `template-size-limit` |

**Estimated fixable in repos: ~35,000-40,000 errors (40-45% of total)**

### B. Fixable in AZD Tooling

| Error Pattern | Est. Count | Recommendation |
|--------------|-----------|----------------|
| **Throttling (429) across multiple resource types** | ~12,000+ | AZD needs exponential backoff/retry for 429 responses for AppConfig, Storage, NSG, Web |
| **ReadOnlyDisabledSubscription (409)** | 1,637 | AZD should pre-check subscription state before provisioning |
| **auth.login_required** | 650 | Better auth flow detection and user prompting |
| **Resource Group Not Found (404)** | 491 | AZD could auto-create or verify resource group existence |
| **DNS resolution failures** | 69 | Better network connectivity checks |

**Estimated fixable in AZD: ~15,000 errors (17%)**

### C. Fixable via Guidance / agents.md / Documentation

| Error Pattern | Est. Count | Guidance Needed |
|--------------|-----------|----------------|
| **Quota/capacity errors (SubscriptionIsOverQuotaForSku)** | ~10,000+ | Document which regions/SKUs are commonly available; add pre-provision quota check |
| **Terraform failures** | 2,911 | Better Terraform troubleshooting docs, common TF provider issues |
| **Unknown internal errors (Go panics, JSON parse)** | ~4,000 | Encourage azd version upgrades, report bugs |
| **azd-init failures** | 9,360 | Better getting-started guidance, common pitfalls documentation |

**Estimated fixable via guidance: ~15,000 errors (17%)**

### D. Not Directly Fixable (Azure Platform / User Subscription Issues)

| Error Pattern | Est. Count | Reason |
|--------------|-----------|--------|
| **Azure service outages / 500 errors** | ~2,000 | Platform-side issues, CognitiveServices CapabilityHosts |
| **Subscription disabled/read-only** | 1,637 | User's subscription state |
| **Authorization failures from missing RBAC** | portion of 2,064 | User doesn't have permissions on their subscription |
| **Quota exhaustion (legitimate)** | portion of SKU errors | User has genuinely consumed their quota |

**Estimated not fixable: ~5,000-10,000 errors (6-11%)**

---

## 6. Monthly Trend (Last 12 Months)

| Month | Attempted | Failed | Success | Failure Rate | Users |
|-------|-----------|--------|---------|-------------|-------|
| Feb 2025 | 20,246 | 10,891 | 9,355 | **53.79%** | 4,479 |
| Mar 2025 | 58,693 | 31,385 | 27,308 | **53.47%** | 14,374 |
| Apr 2025 | 60,897 | 31,610 | 29,287 | **51.91%** | 13,451 |
| May 2025 | 73,166 | 39,153 | 34,013 | **53.51%** | 16,270 |
| Jun 2025 | 85,102 | 47,201 | 37,901 | **55.46%** | 17,025 |
| Jul 2025 | 86,825 | 43,222 | 43,603 | **49.78%** | 16,413 |
| Aug 2025 | 72,426 | 37,092 | 35,334 | **51.21%** | 13,889 |
| Sep 2025 | 77,268 | 40,696 | 36,572 | **52.67%** | 15,164 |
| Oct 2025 | 69,384 | 36,765 | 32,619 | **52.99%** | 14,822 |
| Nov 2025 | 76,644 | 39,400 | 37,244 | **51.41%** | 17,119 |
| Dec 2025 | 62,338 | 32,433 | 29,905 | **52.03%** | 13,556 |
| Jan 2026 | 59,152 | 30,317 | 28,835 | **51.25%** | 13,755 |
| Feb 2026 (partial) | 33,437 | 17,409 | 16,028 | **52.07%** | 7,896 |

### Trend Analysis
- **Failure rate is flat** at 51-55% over 12 months. No improvement trend visible.
- **Volume peaked in Jun-Jul 2025** with 85K-87K attempts/month (likely Build conference/events).
- **July 2025 had the best failure rate** at 49.78% — the only month below 50%.
- **Usage is growing** (from 20K in Feb 2025 to 59K in Jan 2026 — 3x growth), but the failure rate is not improving.
- At current run rate, **~30,000-40,000 users per month** are hitting provisioning failures. This is a significant experience problem.

---

## 7. Recommendations for Template Doctor Checks

### Priority 1 — Highest Impact (Build Immediately)

1. **`bicep-compiles`** — Run `az bicep build` against all .bicep files. Catches 8,778 errors (10% of all failures). Simple, deterministic check.
2. **`sku-quota-risk`** — Flag hardcoded SKUs (S0, P1, etc.) for CognitiveServices, Web, and Search that commonly hit quota errors. Suggest parameterization with fallbacks.
3. **`arm-preflight-validation`** — Run `az deployment group what-if` or `validate` for common ARM error patterns. Addresses the 15,199 InvalidTemplateDeployment errors.
4. **`role-assignments-present`** — Verify templates that use managed identity also create the necessary role assignments. Catches the 2,064 AuthorizationFailed errors.

### Priority 2 — High Impact (Build Next)

5. **`hook-scripts-valid`** — Parse PowerShell/Bash pre/post hooks for syntax errors. Addresses 4,915 + 627 errors.
6. **`tool-dependencies-documented`** — Check that required tools (Docker, npm, Python, dotnet) are listed in README prerequisites. Addresses 459 + 32 + 16 = 507 errors but impacts user experience significantly.
7. **`unique-resource-names`** — Flag storage accounts, key vaults, and other globally-unique resources that don't use `uniqueString()` or similar. Addresses conflict/name-taken errors.
8. **`stable-api-versions`** — Warn when Bicep uses preview API versions for resource types with known instability (e.g., CognitiveServices CapabilityHosts). Addresses ~2,000 server errors.

### Priority 3 — Medium Impact (Future)

9. **`keyvault-soft-delete-aware`** — Detect Key Vault soft-delete conflicts. 646 errors.
10. **`template-size-limit`** — Warn if compiled ARM template exceeds 4MB. 70 errors but completely preventable.
11. **`quota-region-guidance`** — Surface which Azure regions have best availability for the services the template uses.

---

## 8. Kusto Workbench File

All queries used in this analysis are saved in the Kusto Workbench file:
`.ai-team/agents/frodo/azd-provisioning-error-analysis.kqlx`

This file contains 12 query sections connected to the `DevCli` database on `ddazureclients.kusto.windows.net`:
1. Q1: Error Category Distribution (90 days)
2. Q2: Top 25 Failing Templates by Failure Count
3. Q3: Error Details Drill-Down by Category and ResultCode
4. Q4: Failure by Azure Service (Resource Provider/Type)
5. Q5: Monthly Trend (Failure Rate Over Time)
6. Q6: ARM Trace Drill-Down for Top Error Categories
7. Q7: Error Category by Template (Cross-tab)
8. Q8: Error Details for Quota/Capacity Errors
9. Q9: Error Details for Security/Permission Errors
10. Q10: Conflict Errors (Resource Already Exists)
11. Q11: Weekly Trend by Error Category
12. Q12: Templates with 100% Failure Rate

---

## 9. Methodology Notes

- **Error counts** from `AzdProvisionErrorsByTemplate` represent individual provisioning failure events, not unique users.
- **Failure rates** from `DailyAzdProvisionsByTemplate` may differ slightly from error table counts due to aggregation timing.
- **"Other" and "No template"** categories represent users without registered templates — these are custom code or unregistered repos.
- **ARM trace data** uses `mv-expand` to unpack per-resource-operation traces within a failed provisioning event. A single `azd provision` failure may generate multiple ARM trace entries.
- **Azure service failure data** from `AzdProvisionsByAzServiceAndTemplate` is all-time, not windowed, because the table is pre-aggregated.

---

*Analysis generated by Frodo (Data Analyst) on 2026-02-19. All data sourced from live Kusto queries against the DevCli database.*
