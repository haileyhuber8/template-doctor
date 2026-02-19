# Work Routing

How to decide who handles what.

## Routing Table

| Work Type | Route To | Examples |
|-----------|----------|----------|
| Architecture, scope, decisions, check design | Gandalf | What checks to build, trade-offs, system design, prioritization |
| Kusto queries, telemetry analysis, error patterns | Frodo | Query provisioning data, identify common failures, analyze error distributions |
| Analyzer-core logic, rule engine, server routes | Gimli | Implement new checks in run-analyzer.ts, add server endpoints, database operations |
| Azure provisioning, Bicep, quotas, security checks | Legolas | Azure-specific validation rules, Bicep analysis, quota pre-flight, infra patterns |
| Testing, edge cases, validation | Eowyn | Write Playwright/Vitest tests, verify checks against real templates, find edge cases |
| GitHub issue filing, remediation docs, comms | Sam | Draft issue bodies, write fix guidance, documentation, template-facing communication |
| Code review | Gandalf | Review PRs, check quality, suggest improvements |
| Risk analysis (mandatory) | Sauron | Every change gets a risk assessment — always invoked after work completes |
| Session logging | Scribe | Automatic — never needs routing |

## Issue Routing

| Label | Action | Who |
|-------|--------|-----|
| `squad` | Triage: analyze issue, assign `squad:{member}` label | Gandalf |
| `squad:gandalf` | Architecture and decision work | Gandalf |
| `squad:frodo` | Data analysis and Kusto queries | Frodo |
| `squad:gimli` | Backend implementation | Gimli |
| `squad:legolas` | Azure/infra analysis | Legolas |
| `squad:eowyn` | Testing and validation | Eowyn |
| `squad:sam` | Issue filing and DevRel | Sam |
| `squad:sauron` | Risk analysis and review | Sauron |

## Rules

1. **Eager by default** — spawn all agents who could usefully start work, including anticipatory downstream work.
2. **Scribe always runs** after substantial work, always as `mode: "background"`. Never blocks.
3. **Quick facts → coordinator answers directly.** Don't spawn an agent for "what port does the server run on?"
4. **When two agents could handle it**, pick the one whose domain is the primary concern.
5. **Kusto analysis → Frodo.** Any request involving telemetry data, error pattern discovery, or provisioning metrics goes to Frodo.
6. **New check implementation → Gimli + Legolas.** Gimli writes the analyzer logic, Legolas provides Azure domain expertise for the check's rules.
7. **Issue filing → Sam.** Once a check identifies problems, Sam drafts and files the GitHub issues with remediation guidance.
8. **Risk review → Sauron (mandatory).** After ANY batch of agent work completes, Sauron is ALWAYS spawned to review the output for risks, regressions, and failure modes. No work is considered complete until Sauron has assessed it.
