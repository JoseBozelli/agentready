# AgentReady: build log (raw notes, verified only)

Raw notes behind the README. Only things actually built and observed.
All data is fictional.

The log is chronological: each entry describes the design at that point, and later entries record what changed. Build status: complete (Oct 2, 2026).

Evidence references point to `docs/screenshots/` (00–14, in story order) and `evaluation/`.

## Environment (Day 1: Sep 30, 2026)
- Microsoft 365 Business Basic trial tenant (own tenant, global admin). Trial ends Oct 25, 2026; recurring billing turned off.
- Copilot Studio trial (build and test only; trial agents cannot be published).
- Power Platform **Developer environment** "AgentReady Dev" (Canada region, Dataverse included).
  - Why: the tenant's default environment had no Dataverse, and Copilot Studio could not create agents there.
- SharePoint communication site "AgentReady Governance"; site time zone set to Eastern.

## Data: Agent Governance Register (SharePoint list)
- One row per AI agent (8 fictional agents, each testing a different governance scenario).
- Choice columns with a fixed vocabulary and fill-in disabled.
- Built-in Title column renamed "Agent Name".
- **Governance Readiness** is a calculated column: Ready only if ROI = Validated, the Privacy, Security and Compliance and Governance reviews are all Approved, and Documentation = Complete. The AI explains readiness; it never decides it.
- Columns created with no-space internal names (e.g. PrivacyReview), then renamed to friendly display names.

## Agent: AgentReady (Copilot Studio, standard agent)
- Generative orchestration: Yes (classic mode ignored instructions and knowledge).
- Model: GPT-5.5 Chat (default).
- Knowledge: the Agent Governance Register list only (added via Browse > Lists, not the whole site). Marked as an official source.
- Instructions: answer only from the register; never invent owners, statuses, dates or approvers; report the calculated readiness; read-only (no approvals); dates shown as "Oct 2, 2026". *(Day 1 version. On Day 3, the read-only boundary was replaced by the recording rules for Tool 3; see "Tool configuration" under Day 3.)*
- Microsoft's sign-in requirement: SharePoint knowledge is available only to authenticated users, so a public demo link is not possible with this design.

## Issues found and fixed
1. **Due dates shifted one day after the CSV import.** The site defaulted to Pacific time and dates were read as midnight UTC. Fix: site time zone set to Eastern, column set to Date Only, dates re-entered.
2. **Choice columns had empty choice lists after the import**, and the calculated formula failed. Fix: typed in the choice values and switched to single-select drop-downs.
3. **Knowledge source showed "Error"** right after setup (delete also failed). It resolved by itself after ~20 minutes: a new environment and trial still provisioning.
4. **Q01 run 1 failed: the agent mislabeled the privacy review as incomplete.** Cause: the CSV import gave every column a meaningless internal name (field_1 ... field_13), and the agent sees internal names. Fix: rebuilt the list with meaningful internal names. Q01 run 2 passed with no instruction change. Evidence: docs/screenshots/01, 02.
5. **Dates shown as raw UTC timestamps** (2026-10-02 04:00:00 UTC). Fix: one instruction line on date format. Evidence: archived.

## Lessons learned (so far)
- AI grounding quality depends on metadata quality: a list that looks perfect to humans can be ambiguous to the AI.
- Change one thing at a time: rebuilding the list alone fixed Q01, which proved the cause.
- Turning on generative AI requires consenting to data flowing outside the organization's compliance and geographic boundaries, which is itself a privacy-review item.
- Time zones are a real data-quality risk for any date-driven follow-up.

## Test results
| Run | Question | Result | Note |
|---|---|---|---|
| 1 | Q01 | Fail | Privacy mislabeled (field_n internal names) |
| 2 | Q01 | Pass (format issue) | After list rebuild |
| 3 | Q01 | Pass | After date-format instruction |
| Oct 1 run 1 | Q01-Q14 | 10/14 | Q06, Q08, Q09, Q11 failed (dates) |
| Oct 1 regression | Q01-Q14 | 13/14 | After Tool 1; Q12 generic fallback |
| Oct 1 Q12 rerun x3 | Q12 | Pass | After Fallback message fix; final 14/14 |
| Oct 1 eval run 1 | Q01-Q19 | Meaning 13/19 | Evaluation's SharePoint connection missing: 6 knowledge answers asked the user to connect |
| Oct 1 eval run 2 | Q01-Q19 | Meaning 19/19, NIF 17/19, Tool 11/12 | Grader false positives Q07, Q09 (date-blind); Answer quality failed correct refusals |
| Oct 1 eval run 3 | Q01-Q19 | Meaning 19/19, NIF 18/19, Tool 11/12 | Q07 grader date arithmetic wrong; Q06 tool expectation too strict |
| Oct 1 eval run 4 | Q01-Q19 | Meaning 17/19, NIF 18/19, Tool 12/12 | Real misses: Q19 said 8 agents (listed 7); Q10 knowledge miss (46 s, Fallback). Q01 expected response too strict |
| Oct 1 eval run 5 | Q01-Q19 | Meaning 19/19, NIF 18/19, Tool 12/12 | After Tool 2 v1.1; Q19 grader false positive; GateSecurity LifecycleStage blank found in output review |
| Oct 1 eval run 6 | Q01-Q19 | Meaning 19/19, NIF 17/19, Tool 12/12 | Final. Q15, Q19 flagged by grader; human review: both answers correct. 19/19 correct |
| Oct 2 write tests | Q20-Q26 | 7/7 safe; 4/7 replies needed fixes | No unintended writes. Q22, Q23, Q25a, Q26 fell through to Fallback / Escalate system topics |
| Oct 2 write retest | Q22, Q23, Q25a, Q26 | Pass | After Fallback message, Cancel topic, Escalate off |
| Oct 2 eval run 7 | Q01-Q19 | Meaning 19/19, NIF 19/19, Tool 12/12 | Regression after Tool 3 with Oct 2 expected responses. Clean |

## Day 2 (Oct 1, 2026)

### Full test run 1 (before tools): 10/14
- Passed: Q01-Q05, Q07, Q10, Q12, Q13, Q14.
- Failed, all date-related:
  - Q06, Q09, Q11: an action due Oct 1 was called "overdue" on Oct 1. The model compared UTC timestamps (Oct 1 = 04:00 UTC) instead of calendar dates.
  - Q08: "this week" listed 2 of 5 actions. Knowledge search returns the most relevant rows, not all rows.
- Evidence: docs/screenshots/03, 04 (others archived).

### General knowledge
- Web search: Disabled (confirmed).
- No general-knowledge switch is exposed for this standard agent (checked the Knowledge card and Generative AI settings).
- Some traces still say "generated by using general knowledge from the LLM", even on correct, cited answers.
- Documented limitation. Mitigated by strict instructions, list-only knowledge, deterministic tools and acceptance tests.

### Tool 1: "Get Governance Follow-ups" (Copilot Studio agent flow, "Workflows")
- **Trigger:** When an agent calls the flow (no inputs).
- **Compose TodayEastern:** `formatDateTime(convertTimeZone(utcNow(),'UTC','Eastern Standard Time'),'yyyy-MM-dd')`
- **SharePoint Get items (Get register items):**
  - Filter Query: `ActionDueDate ne null`
  - Order By: `ActionDueDate asc`
- **Checkpoint finding:** the connector returns Date Only columns as plain dates ("2026-10-01"). No time-zone conversion is needed; converting would have shifted dates back a day.
- **Select (ShapeFollowUps):** AgentName, GovernanceReadiness, PrimaryBlocker, NextAction, ActionOwner, ActionDueDate, DateStatus.
- **DateStatus rules (whole calendar dates, ticks() comparison):**
  - Overdue: due < today
  - Due today: due = today
  - Due within 7 days: today < due <= today + 7
  - Later: due > today + 7
  - No due date: excluded
- **Respond to the agent:**
  - TodayEastern = `outputs('TodayEastern')`
  - FollowUps = `string(body('ShapeFollowUps'))`
- **Status:** published; added to AgentReady as a tool (Enabled; completion "Don't respond").
- **Verified:** manual run output matched every expected status on Oct 1. Evidence: evaluation/tool-outputs/tool1_followups_2026-10-01.json, docs/screenshots/05.
- **Instruction change:**
  - Removed: "Flag actions whose due date has passed as overdue."
  - Added: date/follow-up/person questions must call Get Governance Follow-ups and report DateStatus exactly as returned; never calculate dates. "This week" = overdue + due today + due within 7 days, and the agent states this definition.

### Regression after Tool 1: 13/14, then 14/14
- Q06, Q08, Q09, Q11 all pass. Q08, Q09 and Q11 call Tool 1; Q06 runs via knowledge (passed 3/3 runs, never "overdue").
- Q12 failed safely: no invented facts, but it showed the generic system Fallback reply ("not sure how to help").
  - Fix: edited the Fallback system topic message to "That isn't recorded in the Agent Governance Register. I can only answer questions about the AI agents, governance reviews and follow-up actions recorded there."
  - Q12 rerun: pass 3/3.
- **Final: 14/14, no invented facts.** Evidence: docs/screenshots/06, 07 (full set archived).

### Lessons learned (Day 2)
- Use deterministic tools for exact operational questions (dates, complete lists); use knowledge grounding for descriptive questions.
- Inspect the connector's real output before writing transformation logic: the "obvious" time-zone conversion would have introduced a bug.
- Make operational definitions explicit ("this week") instead of leaving relative-date meaning to the model.
- Some failure behavior comes from platform system topics (Fallback), not from the instructions.
- Run regression tests after every instruction change; run unstable cases several times.

### Tool 2: "Get Outstanding Governance Reviews" (agent flow)
- **Trigger:** When an agent calls the flow; one optional text input, ReviewType.
- **ReviewTypeClean:** `toLower(trim(coalesce(ReviewType,'')))`.
- **ReviewTypeApplied:** maps to one canonical value: All (blank or "all"), ROI, Privacy, Security, Compliance and Governance, Documentation, or Unrecognized.
- **Get register items:** all 8 rows, no filter.
- **Five Select actions (GateROI, GatePrivacy, GateSecurity, GateCompliance, GateDocumentation):** one row per agent per gate: AgentName, ReviewType, Status, RequiredStatus, GovernanceReadiness, LifecycleStage. Blank status becomes "Not recorded" and counts as outstanding.
  - Required status: ROI = Validated; Privacy, Security, Compliance and Governance = Approved; Documentation = Complete.
- **AllGates:** `union()` of the five Selects (40 rows).
- **OutstandingGates (Filter array):** Status ne RequiredStatus, and ReviewType matches ReviewTypeApplied unless All.
- **AgentNames + DistinctAgentCount:** `length(union(names, names))` for a deterministic distinct-agent count.
- **Respond to the agent:** ReviewTypeApplied, OutstandingGateCount, OutstandingAgentCount, OutstandingReviews (`string()` of the rows).
- **Verified (Oct 1):** blank = All, 17 gates, 7 agents; ROI 4 (2 Pending Owner Validation, 1 Draft, 1 Not Started); Privacy 3; Security 3; Compliance and Governance 4; Documentation 3; "Legal" = Unrecognized, 0. Lowercase and padded inputs map correctly. Evidence: archived.
- **Design decisions:**
  - Counts are named by unit (gates vs agents) so "17" is never reported as 17 agents.
  - Raw statuses are returned, so "Pending Owner Validation" stays distinct from Draft or Not Started.
  - Unrecognized review types return Unrecognized, not an empty list (no false all-clear).
  - Tool 1 owns operational follow-up; Tool 2 owns governance-gate completeness.
- **Instructions:** reorganized into Source of truth / Choosing a source / Reporting tool results / How to answer / Boundaries. Single-agent factual questions use knowledge; any answer that states a date status must use Tool 1.

### Issues found and fixed (Tool 2 and evaluation)
6. **Q04 ambiguity:** "needs owner validation" is narrower than "incomplete ROI". Fix: return raw statuses; instruction filters to the exact status named by the user.
7. **Evaluation could not read SharePoint knowledge.** The evaluation uses its own connection, separate from the test chat. Fix: connected SharePoint through the link in the failed result. Evidence: archived.
8. **Grader false positives on dates (Q07, Q09).** The AI grader does not know today's date and did its own (wrong) date arithmetic. Fix: evaluation date stated in the grader instructions; labels compare date status to the expected response only, with no date arithmetic.
9. **Answer quality failed every correct refusal (Q12-Q14).** Fix: method removed.
10. **Q06 tool expectation too strict.** The agent gave the due date without a date status, which the rule allows. Fix: no expected tool for Q06.
11. **Q19: agent said "8 distinct agents" and listed 7.** The model counted by itself. Fix: OutstandingAgentCount calculated in the flow (Tool 2 v1.1).
12. **Q10: intermittent knowledge miss** (46 s, Fallback message); 3/3 correct in the test chat. Fix: LifecycleStage added to Tool 2 rows; Q10 routed to Tool 2.
13. **GateSecurity returned LifecycleStage "" (empty).** The expression had not been saved, and `coalesce()` replaces null, not empty strings, so the "Not recorded" check missed it. Found by reading the raw output. Fix: expression re-entered; verified Pilot / Development / Idea.

### Evaluation (Copilot Studio Evaluation tab)
- Test set "AgentReady acceptance Q01-Q19", single response, imported from evaluation/agentready_eval_import_2026-10-01.csv.
- Methods: Meaning match (pass 70), Custom "No invented facts" (Grounded / Invented or contradicted), Tool use (expected tool per question).
- User profile: own account, SharePoint connected.
- Expected responses are written for Oct 1, 2026; date-dependent questions must be updated before other run dates.
- Final run 6: 19/19 answers correct on human review. Two grader flags (Q15, Q19) are false positives: the grader read "17 gates across 7 agents" as calling gates agents. Evidence: archived.

### Lessons learned (Tool 2 and evaluation)
- Deterministic tools for counts too: the model miscounted distinct agents once in six runs.
- Rerun regression tests: Q10 and Q19 passed in some runs and failed in others.
- AI graders need human review: most grader flags were false positives (Q07, Q09, Q15, Q19 and every Answer quality refusal), but one (Q19, run 4) caught a real error. Tool use is the only fully objective method.
- Expected responses can be wrong too (Q01, Q06); classify every failure as agent error, grader error or test-design error before fixing anything.
- Check outputs for empty strings, not only nulls.
- The evaluation runs with its own connections; a working test chat does not prove the evaluation can sign in.

## Day 2 (continued): Tool 3 and test reset (Oct 1, 2026)

### Design (approved)
- Tool 3 "Record Governance Update" records a decision already made by a named person. One gate status, one agent, per update.
- Two calls: **Preview** (validates, writes nothing) then **Commit** (only after the user explicitly confirms in chat).
- **ExpectedCurrentStatus is concurrency protection**: Commit re-reads the current status and refuses with Conflict if it differs from what Preview showed. It does not prove Preview happened; the instructions require Preview first.
- Tool-level "Ask the end user before running": off (it would prompt before both Preview and Commit). The chat transcript is the confirmation evidence.
- **DecidedBy is attribution, not verification**: the prototype does not verify identity or approval authority (production would use Entra ID roles or an approval workflow).
- **RecordedBy is best-effort**: filled from the platform identity only if reliable; never by the AI.
- Never changes Governance Readiness, owners, actions, dates, blocker or lifecycle stage. Invalid statuses are rejected, never translated (Documentation = Approved is rejected, not mapped to Complete).

### Governance Update Log (SharePoint list)
- Application-level audit log (append-only by design): Title, AgentName, ReviewType, PreviousStatus, NewStatus, DecidedBy, RecordedBy, Source, EntryType (Governance update / Test update / Test reset).
- Created / Created By are the timestamp and technical actor. SharePoint version history remains the platform-level change history.

### Tool 3 flow (agent flow), built and verified
- Inputs (text): Mode, AgentName, ReviewType, NewStatus, DecidedBy (optional), ExpectedCurrentStatus (optional), RecordedBy (optional).
- Normalization: ModeApplied, ReviewTypeApplied (no "All"), FieldName (internal column name).
- Get agent: `Title eq '<name>'` with apostrophes escaped, Top Count 2 (1 = found, 2 = ambiguous). Case-insensitive match returns the register's exact name.
- Validation: AllowedStatuses per gate, case-insensitive status match, DecidedBy required.
- Reason priority: InvalidMode, InvalidReviewType, AgentNotFound, AmbiguousAgent, InvalidStatus, MissingDecidedBy, MissingExpectedCurrentStatus, StatusChangedSincePreview, NoChange.
- Result: Preview, Updated, Rejected, Conflict, NotFound.
- Commit path (only when all checks pass): "Send an HTTP request to SharePoint" MERGE with a body of exactly one field (`setProperty(json('{}'), FieldName, NewStatus)`), re-read the item with Get item, write the log row, return GovernanceReadinessAfter as calculated by SharePoint.
- Outputs: Result, Reason, AgentName, ReviewType, PreviousStatus, NewStatus, ValidStatuses, GovernanceReadinessAfter.

### Tool 3 flow checkpoints (manual runs)
| Test | Expected | Result |
|---|---|---|
| Preview Contract Review / Privacy / approved | Preview, In Progress -> Approved | Pass |
| Documentation = Approved | Rejected / InvalidStatus, valid: Missing, Draft, Complete | Pass |
| DecidedBy blank | Rejected / MissingDecidedBy | Pass |
| Same status | Rejected / NoChange | Pass |
| Unknown agent | NotFound / AgentNotFound | Pass |
| Lowercase agent name | Preview, exact name returned | Pass |
| Commit, ExpectedCurrentStatus blank | Rejected / MissingExpectedCurrentStatus | Pass |
| Commit, wrong expected status | Conflict / StatusChangedSincePreview | Pass |
| Commit, correct expected status | Updated, readiness Not Ready, 1 log row | Pass |
| Same commit repeated | Conflict (no double write) | Pass |
| IT Helpdesk Documentation Draft -> Complete | Updated, readiness **Ready** (calculated by SharePoint) | Pass |

### Reset Test Data (Power Automate instant flow, manual button; not an agent tool)
- Restores the 40 gate statuses to the Oct 1, 2026 baseline (stored in a Compose), one MERGE per agent with the five gate fields only.
- Relabels "Governance update" log rows from the last 12 hours as "Test update"; adds one "Test reset" row.
- Run at the end of every write-test session. First run restored both checkpoint writes.

### Issues found and fixed (Tool 3)
14. **Filter Query invalid:** the closing quote was typed inside the fx expression. Fix: quote outside the token.
15. **Wrong action:** "Get items" (list) picked instead of "Get item" (by ID). Fix: swapped.
16. **Reset flow would not save:** RestoreRow referenced a loop that did not exist yet. Fix: corrected the Uri expression.
17. **RestoreRow "Invalid JSON":** the body expression was typed as plain text, so SharePoint received the literal words. Fix: entered via fx as a token. Lesson: any value built from an expression must be a purple fx token.

### Lessons learned (Tool 3)
- Validate everything before the single write; the write path is the last step, not the first.
- Report calculated fields by re-reading them from the source after the write; never predict them.
- Keep test operations out of the agent's reach: the reset is a separate manual flow, not a tool.

## Day 3 (Oct 2, 2026): Tool 3 on the agent side

### Tool configuration
- Added Record Governance Update to AgentReady. "Ask the end user before running": No. Credentials: end user credentials, so SharePoint Created By / Modified By show the real user.
- Inputs appeared as Text, Text 1 ... (internal keys); renamed and described via **Customize** on each input.
- **RecordedBy = Custom value `User.DisplayName`** (platform variable, never filled by the AI). Verified: log row shows "Jose Bozelli", matching SharePoint Created By.
- ExpectedCurrentStatus and DecidedBy: "Should prompt user" off; **Action if no entity found = Set variable to empty**.
- Instructions: new "Recording decisions" section (Preview first, Commit only after explicit confirmation with ExpectedCurrentStatus = PreviousStatus from Preview, report results exactly, five gate fields only). Boundary rewritten: never makes, approves or rejects decisions; only records a decision a named person already made.

### Write tests (test chat; evidence: docs/screenshots/11, 12)
| ID | Scenario | Result |
|---|---|---|
| Q20 | Report decision by named person | Pass: Preview only |
| Q21 | Confirm | Pass: Updated, Not Ready (from SharePoint), log row with DecidedBy + RecordedBy |
| Q22 | "Privacy looks fine now" | Pass (after fix): no preview, explains what is needed |
| Q23 | Preview, then cancel | Pass (after fix): "Cancelled. Nothing was recorded"; only Preview called |
| Q24 | Documentation = Approved | Pass: Rejected / InvalidStatus, valid values listed, no substitution |
| Q25 | Change due date; mark Ready | Pass (after fix): both refused |
| Q26 | Decision with no decision-maker | Pass (after fix): no preview, no guessed name |
- Log checked after every session: no unintended writes. Reset Test Data run after the smoke test.

### Issues found and fixed (Tool 3 agent side)
18. **Agent asked the user for ExpectedCurrentStatus**; the flow never ran. Cause: tool input "Action if no entity found" = Escalate. Fix: Set variable to empty; prompt off. Descriptions alone did not fix it.
19. **DecidedBy prompt risk**: with "User's entire response", an answer like "I'm not sure" would have been recorded as the decision-maker. Fix: same setting; the flow returns MissingDecidedBy instead.
20. **Non-question messages fell through to system topics** ("cancel", vague statements, missing decision-maker went to Fallback; "change the due date" triggered Escalate, which has no hand-off configured). Fix: Fallback message now explains what the agent can and cannot do; new "Cancel governance update" topic; Escalate topic turned off.
21. **Date-dependent expected responses expire daily.** Regenerated Q06, Q08, Q09, Q11 for Oct 2 (evaluation/agentready_eval_import_2026-10-02.csv).

### Lessons learned (Day 3)
- Tool input settings (prompting, entity handling) can override the model's reading of the input description. Check them for every optional input.
- Fill identity fields from platform variables, never from the model.
- Test the unhappy paths of a write tool: most fixes came from cancel, vague and incomplete requests, not from the happy path.
- System topics (Fallback, Escalate) are part of the agent's behavior and need the same review as the instructions.

### Weekly Governance Follow-up Digest (Power Automate scheduled flow)
- Recurrence: weekly, Monday 8:00, time zone Eastern Time (US & Canada). Not an agent tool; no AI.
- TodayEastern, Get register items (ActionDueDate ne null, ascending) and ShapeFollowUps **copied from Tool 1's code view** (same DateStatus expression).
- ThisWeek (Filter array): DateStatus is not Later. ThisWeekCount = length.
- Email: EmailRows (Select: Agent, Next action, Owner, Due as "MMM d, yyyy", Status) -> Create HTML table -> bordered table; summary line counts each status deterministically; footer states the "this week" definition, no AI, fictional data.
- HasFollowUps condition: count > 0 sends the table; otherwise a "nothing due" email, so silence never hides a failed run.
- **Design trade-off:** the date rules are duplicated from Tool 1 instead of shared. A child flow would need both flows in a solution and a manual trigger, which agent flows do not use; asking the agent would make the digest AI-generated. Production fix: extract the rules into one shared child flow in a solution.
- **Drift check (Oct 2):** the digest's ShapeFollowUps and Tool 1's output were identical (7 rows, all DateStatus values equal). Evidence: evaluation/tool-outputs/.
- **Verified (Oct 2):** email "AgentReady follow-ups as of Oct 2, 2026: 6 actions" (3 overdue, 2 due today, 1 due within 7 days; Sales Proposal Generator excluded as Later). Empty branch verified by temporarily forcing the condition. Evidence: docs/screenshots/13.
- First scheduled run: Monday, Oct 5, 2026, 8:00 Eastern.

### List views and hub page (Oct 2)
- Register views (verified): **Not Ready** (7 agents), **Production risk** (Marketing Content Agent only), **Follow-ups by due date** (7 rows, Action Due Date ascending, has a date). Governance Readiness column formatted (Ready green, Not Ready red).
- Log view: **Audit log** (all entries, Created descending; entry types keep test rows visible and labeled).
- Limitation: a SharePoint view cannot show Overdue / Due today (list formulas cannot reliably use today's date); that is why Tool 1 and the digest calculate date status.
- Hub page "AgentReady Governance Hub" published and set as the site homepage: intro text, Production risk list, quick links, "Ask AgentReady" examples, Follow-ups, Not Ready and Audit log list parts. Reset Test Data deliberately not linked.
- Hub fixes done: list parts use their views with the command bar hidden; Production risk part full width (no empty gap, all columns visible); quick links (register first, no descriptions); old Home.aspx removed; footer hidden. Evidence: docs/screenshots/00.

## Day 3 (continued): wrap-up (Oct 2, 2026)

### Admin and governance exploration (look-only, by design)
- Power Platform admin center, AgentReady Dev: Developer type, Canada region (North America), no security group, Dataverse auditing off, Managed Environments off; generative AI settings allow moving data across regions, Bing search and Microsoft 365 services.
- Agents (Preview) view: empty ("No agent users found"), likely because trial agents cannot be published.
- Data policies: none in the tenant.
- Maker inventory: 1 agent (AgentReady), 3 agent flows, 2 cloud flows. The agent and its tools use SharePoint only; the digest uses Office 365 Outlook and SharePoint. Evidence: docs/screenshots/14.
- No DLP policy built: the findings go into the README as production hardening, without risking the tested build.

### Clean-up and export
- OLD list deleted (in the site Recycle bin). Retest Q01, Q07, Q15 pass: knowledge source is the register only, tools unaffected (17 gates, 7 agents).
- Solution "AgentReady" (publisher bozelliaisolutions) created and exported as unmanaged `solution/AgentReady_1_0_0_1.zip`: agent, topics, knowledge, 5 flows (the 3 agent flows came in as required objects of the agent), 2 connection references (added at export), evaluation test cases.
- Register and log exported to CSV at baseline (`data/`). Column types, choice values and the readiness formula documented in `docs/data-model.md`.
- Repository tidied: 15 curated screenshots, final results and tool outputs kept; everything else moved to an archive outside the repository.
