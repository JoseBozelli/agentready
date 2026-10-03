# AgentReady: data model

AgentReady uses two SharePoint lists on the site **AgentReady Governance**. The **Agent Governance Register** is the source of truth. The **Governance Update Log** is an append-only, application-level audit log. All data is fictional.

Baseline exports (Oct 2, 2026): `data/agent_governance_register_baseline.csv` and `data/governance_update_log_baseline.csv`.

## 1. Agent Governance Register

One row per AI agent: 8 fictional agents, each testing a different governance scenario.

| Column | Type | Notes |
|---|---|---|
| Agent Name | Single line of text | Built-in Title column, renamed |
| Purpose | Multiple lines of text | What the agent does |
| Business Owner | Single line of text | Accountable business person |
| Technical Owner | Single line of text | Accountable builder or operator |
| Lifecycle Stage | Choice | Where the agent is in its lifecycle |
| ROI Assessment | Choice | Gate 1. Required: **Validated** |
| Privacy Review | Choice | Gate 2. Required: **Approved** |
| Security Review | Choice | Gate 3. Required: **Approved** |
| Compliance and Governance Review | Choice | Gate 4. Required: **Approved** |
| Documentation | Choice | Gate 5. Required: **Complete** |
| Primary Blocker | Multiple lines of text | Main reason the agent is not ready |
| Next Action | Multiple lines of text | The single next follow-up action |
| Action Owner | Single line of text | Person responsible for the next action |
| Action Due Date | Date and Time (Date Only) | Site time zone: Eastern |
| Governance Readiness | Calculated | Ready / Not Ready (formula below) |

The Choice columns use a fixed vocabulary, with fill-in values disabled. Their values:

| Column | Allowed values |
|---|---|
| Lifecycle Stage | Idea, Development, Pilot, Production |
| ROI Assessment | Not Started, Draft, Pending Owner Validation, Validated |
| Privacy Review | Not Started, In Progress, Changes Requested, Approved |
| Security Review | Not Started, In Progress, Changes Requested, Approved |
| Compliance and Governance Review | Not Started, In Progress, Changes Requested, Approved |
| Documentation | Missing, Draft, Complete |

Tool 3 also checks every requested status against an allowed-status list for that gate. A value that isn't allowed is rejected (InvalidStatus), never translated. For example, "Documentation = Approved" is rejected, not mapped to Complete.

### Readiness rule (calculated by SharePoint, never by AI)

```
=IF(AND([ROI Assessment]="Validated",[Privacy Review]="Approved",[Security Review]="Approved",[Compliance and Governance Review]="Approved",Documentation="Complete"),"Ready","Not Ready")
```

An agent is **Ready** only when all five gates meet their required status. AgentReady explains readiness; it never decides or predicts it. After a recorded update, Tool 3 re-reads the item and reports the readiness SharePoint calculated.

### Baseline state

At baseline, one agent is Ready (HR Onboarding Assistant) and seven are Not Ready. Together they have **17 incomplete gates**:

| Gate | Incomplete |
|---|---|
| ROI | 4 |
| Privacy | 3 |
| Security | 3 |
| Compliance and Governance | 4 |
| Documentation | 3 |

One agent, Marketing Content Agent, is in **Production** while Not Ready. That is the hub page's "Production risk" view.

### Design notes

- Columns were created with no-space internal names (for example, `PrivacyReview` and `ActionDueDate`) and then renamed to friendly display names. The agent and the connectors see internal names. An early CSV import produced `field_1 ... field_13`, which caused a grounding error, so the list was rebuilt (BUILD_LOG issue 4).
- Action Due Date is Date Only, and the site time zone is Eastern. That prevents the one-day shift seen after the first import (BUILD_LOG issue 1).
- Date status (Overdue, Due today, Due within 7 days, Later) is **not** stored in the list. A SharePoint view cannot reliably use today's date, so Tool 1 and the weekly digest calculate it at run time.

## 2. Governance Update Log

One row per recorded decision or test reset. Rows are written only by the Record Governance Update flow (Tool 3) and by the manual Reset Test Data flow.

| Column | Type | Notes |
|---|---|---|
| Title | Single line of text | "Agent – Review type", or "Test data reset to baseline" |
| Agent Name | Single line of text | Exact agent name from the register; "All" for resets |
| Review Type | Single line of text | ROI, Privacy, Security, Compliance and Governance, Documentation; "All" for resets |
| Previous Status | Single line of text | Gate status before the update |
| New Status | Single line of text | Gate status after the update |
| Decided By | Single line of text | The named person who **made** the decision (required) |
| Recorded By | Single line of text | The user who **recorded** it: platform variable `User.DisplayName`, never filled by the AI |
| Source | Single line of text | AgentReady, or Reset Test Data |
| Entry Type | Choice | Governance update, Test update, Test reset |
| Created, Created By | System | Timestamp and technical actor |

### Design notes

- **Decided By vs Recorded By** separates the decision-maker from the person entering it. AgentReady never makes, approves or infers a decision. It records one a named person already made, after a Preview and explicit confirmation.
- **Test rows are kept and labeled, not deleted.** Reset Test Data relabels recent "Governance update" rows as "Test update" and adds one "Test reset" row, so the log stays append-only.
- The first two rows in the baseline export have a blank Recorded By. They were written before the column was added on Oct 2.
- SharePoint version history remains the platform-level change history. This log is the application-level decision trail.

## 3. How the components use the data

| Component | Reads | Writes |
|---|---|---|
| Agent knowledge | Register (official source) | Nothing |
| Tool 1: Get Governance Follow-ups | Register: rows with an Action Due Date | Nothing |
| Tool 2: Get Outstanding Governance Reviews | Register: all rows, 5 gates each | Nothing |
| Tool 3: Record Governance Update | Register: one item | One gate field (MERGE) and one log row |
| Weekly Governance Follow-up Digest | Register | Nothing (sends email) |
| Reset Test Data (manual, not a tool) | Register and log | Restores baseline; relabels log rows |
