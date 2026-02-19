# Ceremonies

> Team meetings that happen before or after work. Each squad configures their own.

## Design Review

| Field | Value |
|-------|-------|
| **Trigger** | auto |
| **When** | before |
| **Condition** | multi-agent task involving 2+ agents modifying shared systems |
| **Facilitator** | Gandalf |
| **Participants** | all-relevant |
| **Time budget** | focused |
| **Enabled** | ✅ yes |

**Agenda:**
1. Review the task and requirements
2. Agree on check interfaces and contracts between analyzer-core, server, and frontend
3. Identify risks and edge cases for new validation rules
4. Assign action items

---

## Retrospective

| Field | Value |
|-------|-------|
| **Trigger** | auto |
| **When** | after |
| **Condition** | build failure, test failure, or reviewer rejection |
| **Facilitator** | Gandalf |
| **Participants** | all-involved |
| **Time budget** | focused |
| **Enabled** | ✅ yes |

**Agenda:**
1. What happened? (facts only)
2. Root cause analysis
3. What should change?
4. Action items for next iteration

---

## Risk Review

| Field | Value |
|-------|-------|
| **Trigger** | auto |
| **When** | after |
| **Condition** | any agent produces work output (code, designs, proposals, checks) |
| **Facilitator** | Sauron |
| **Participants** | Sauron |
| **Time budget** | focused |
| **Enabled** | ✅ yes |

**Agenda:**
1. Review all changes produced in this batch
2. Identify failure modes, regression risks, and dependency impacts
3. Categorize each risk (CRITICAL / HIGH / MEDIUM / LOW)
4. Recommend mitigations or flag blockers
5. Assess false positive risk for any new analyzer checks
