# Bug Source Handlers

Different bug origins have different schemas, signals, and processing logic. Identify the source first, then apply the right playbook.

---

## How to detect source

Check signals in this order (first match wins):

| Signal | Source |
|---|---|
| Tag contains `Autobugged`, `TestResultsAutobugger`, `LRAutoResolved` | **TestFailure** |
| Tag contains `fileBug`, `Filed via /fileBug command` | **fileBug command** (variant of Manual) |
| Tag contains `S360`, `EcsStaleFlight`, `DebtManagementTracking` | **Compliance/Debt** |
| Title starts with `[Address PR Comments]` or `[... Feedback]` | **PRComment** |
| Description contains "Pull Request =>" with link | **PRComment** (even if title doesn't say so) |
| Description contains `Identified from IcM` or IcM URL | **IcM** |
| Description contains "Meeting Context (Bug Bash" | **BugBash** |
| None of the above | **Manual** |
| Empty description AND empty repro | **Insufficient** (special case) |

---

## Playbook: BugBash

**Profile**: Created in batches on a single day. Has meeting context with named participants. Quoted dialogue. Often well-structured with repro/expected/actual.

**What's usually missing**: Code anchors. Owner needs to translate "we discussed X" into "fix file Y line Z".

**Processing**:
1. Group bugs by creation date — likely all from same bug bash
2. Cluster by intent (terminology / data lifecycle / real bug / design question)
3. Heavy code anchor search — this is where AI adds most value
4. Identify cross-bug patterns (same string file, same component, etc.)
5. Suggest **PR consolidation** — multiple related bugs may be one PR

**Pitfalls**:
- Don't just rephrase the meeting context — owner already read it
- Watch for "design needed" cues from the discussion ("maybe this is a bigger discussion", "let's sync up")

---

## Playbook: PRComment

**Profile**: Auto-generated work item summarizing a PR review comment. Has PR link, file path, reviewer name, and the original comment quoted.

**What's usually missing**: Nothing factually — but it's often **already done or already promised**.

**Processing**:
1. **Check if PR is merged** — if yes, was the comment addressed? May be close-able
2. Look at owner's reply in the quoted thread:
   - "Agreed - I'll fix in next PR" → already promised, just track
   - "Out of scope for this PR" → real follow-up, needs to do
   - GitHub Copilot already replied with a fix commit → review and merge
3. **Use the PR link as the code anchor** — don't waste search budget
4. If reviewer comment is vague ("make this more secure"), flag for clarification

**Pitfalls**:
- These look low-priority but accumulate as tech debt — surface them as a cluster
- Multiple PRComment bugs from same PR → batch them into one cleanup PR

---

## Playbook: IcM (Production Incident)

**Profile**: Description mentions IcM ID, often has detailed technical analysis with code samples, gist PRs, root cause already understood.

**What's usually missing**: Usually nothing — these are high-quality.

**Processing**:
1. Check if linked gist PR / fix PR is merged
2. Verify severity reflects production impact
3. Code anchor only if not already in description
4. Flag if assigned-to is wrong (some IcMs auto-assign to wrong owner)

**Pitfalls**:
- Don't downgrade severity without evidence — these came from real prod issues

---

## Playbook: TestFailure (Autobugged)

**Profile**: Title is a test name + stack trace excerpt. Tag includes `Autobugged`. Often `LRAutoResolved` (auto-closed after test stabilized).

**What's usually missing**: Whether it's still failing. Many auto-resolve.

**⚠️ Critical: real info lives in COMMENTS, not description.**
For auto-filed bugs, `System.Description` is usually empty and `Microsoft.VSTS.TCM.ReproSteps` is just `TotalTestsImpacted: N`. The actual stack trace, test name, owner, and Debt Management Bot status are all in the **comments** (added by `Project Collection Build Service`). **Always fetch comments via `wit_list_work_item_comments` for these bugs** before doing any analysis.

**Processing**:
1. **Fetch comments first** — they contain the stack trace + test link + Debt Bot status
2. Check state — if `Done` / `LRAutoResolved`, **probably stale**
3. **Check for Debt Management Bot comment** — if it onboarded the bug to an initiative (e.g. `FlakyTestFailures`), the work is already being tracked by the initiative driver. Owner just needs to confirm linkage. Don't duplicate effort.
4. If multiple bugs share the same throw site (file:line in stack trace), they're **almost certainly the same root cause** — flag for dedup
5. **Skip code anchor search** — the stack trace IS the anchor. No need to grep for the function. Just extract the throw site (e.g. `runTest.ts:1032`) directly from the comment.
6. Look for related test files with same `@guid` tag
7. **Investigate test infra ownership** — if the throw site is in a shared utility (e.g. `playwright-tab-utilities`, `test-helpers`, etc.), the real owner may be the test infra team, NOT the assignee. Surface this in the dashboard.

**Pitfalls**:
- Many of these clear themselves — don't waste enrichment cycles on auto-resolved ones
- Stack traces in title are usually noise; the test name is the signal
- Don't search code for the function in the stack — the stack IS the anchor, just cite it directly
- Severity inconsistency across same-root-cause bugs is common (one Sev 3, three Sev 4) — call this out as a triage suggestion

---

## Playbook: Compliance/Debt

**Profile**: S360 hygiene, ECS stale flights, security debt. Often has structured fields, deadline-driven.

**Processing**:
1. Check the deadline (`DMTargetDate` tag often has it)
2. These rarely need code anchor — they need a process action (acknowledge, configure, etc.)
3. Just confirm what's required and surface the deadline

---

## Playbook: Manual

**Profile**: Free-form, no obvious schema. Quality varies wildly.

**Processing**:
- If well-structured (has repro + expected/actual): treat like BugBash
- If one-liner: try to infer from title, ping author for repro
- If has screenshot only: flag as needing OCR / multimodal analysis (note in dashboard)

---

## Playbook: Insufficient

**Profile**: Description and repro both empty. Only has a title.

**Processing**:
1. Try to infer expected behavior from title
2. Generate a draft repro for owner to confirm
3. Suggest pinging author with specific questions
4. **Don't fabricate** — clearly mark inferred content as "AI-suggested, please verify"

---

## Cross-source meta-rules

- If 80%+ of the batch is one source type, mention it in the dashboard summary (e.g. "this is mostly a bug bash output")
- If sources are mixed, the dashboard should group BY source first, then BY cluster within each source
- A bug can match multiple sources (e.g. PRComment + Manual). Use the most specific match.
