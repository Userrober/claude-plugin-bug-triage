# Bug Triage Methodology

The 6 core lessons that drive batch bug triage. Each is distilled from real triage runs.

---

## 1. Bugs are not isolated — cluster first

A list of 22 bugs is overwhelming. The same 22 bugs grouped into 4 clusters is a tractable plan.

**Clustering signals (in priority order):**

| Signal | What it suggests |
|---|---|
| Same creation date (±1 day) | Same event (bug bash, IcM storm, sprint kickoff) |
| Title keyword overlap (≥2 words) | Same feature / area |
| Same `AreaPath` | Same team's responsibility |
| Same tag (e.g. `fileBug`, `Autobugged`, `BugBash-Q2`) | Same source pipeline |
| Repeated names in description (Sam, Zishan, Dave...) | Same conversation thread |

**Within a cluster, sub-cluster by intent:**
- 文案/terminology → resx changes
- Data lifecycle / cleanup → may need design decision
- Real code bug → actual fix
- Tech debt / orphaned code → mechanical cleanup

**Output value**: showing "11 bugs are one epic" is more useful than 11 independent enrichments.

### 1a. Duplicate detection — within-cluster, look for same root cause

Once clustered, scan each cluster for **duplicates** (not just related bugs). Strong duplicate signals:

| Signal | Confidence |
|---|---|
| Same throw site (file:line) in stack trace | 🔴 Almost certainly same root cause |
| Same error message + same test family + dates within 2 days | 🔴 Very likely duplicate |
| Same UI string / resx key referenced | 🟡 Likely same fix |
| Same component name + same symptom | 🟡 Worth checking |
| Same PR referenced in multiple bugs | 🔴 Definitely related, possibly duplicate |

**Action when duplicates detected:**
- Surface in dashboard's **Quick Wins** section
- Recommend marking as duplicate of the lowest-numbered (or highest-quality) bug
- Or link as `Related` if they're variants rather than identical
- **Pick one to fix**, the others auto-resolve

**Why this matters**: auto-bug filers (TestResultsAutobugger, etc.) often create N work items for what is one underlying issue. Owners get charged for N when the work is 1. The Aakanksha test run found 4 bugs → 1 root cause; this is the norm for auto-filed bugs, not the exception.

---

## 2. Different sources need different handling

Don't apply the same processing to every bug. See [source-handlers.md](./source-handlers.md) for per-source playbooks. Key insight: a bug auto-filed from a PR comment doesn't need code-anchor search — the PR link IS the anchor.

---

## 3. Find "already-done" bugs first — they save the most owner time

Before deep enrichment, scan for bugs that can be **closed in 30 seconds**:

- Description references a fix PR ("addressed in PR #218841")
- Description has a commit hash or branch name (`user/ahah/SPACFlightCleanup`)
- Description says "duplicate of #N"
- Auto-bug whose underlying issue is already resolved
- Bug filed for a feature that was scrapped / deprioritized

**Why this matters**: owners want to clear their queue. A bug they can close beats a bug they have to fix.

---

## 4. Code anchor is the highest-value enrichment

For owners working in a large codebase, the time sink is **finding where to change**, not understanding the bug. AI is uniquely good at this because it has full code search and doesn't get tired.

**Strategy by intent** (see [code-anchor-strategies.md](./code-anchor-strategies.md)):
- Terminology bug → resx string key, then usage sites
- Validation bug → validator file, regex constant
- UI behavior → component file
- Lifecycle/data → reducer / data provider / API class
- Performance → flame graph hint or perf-marked code

**Stop after 2-3 targeted queries per bug.** Over-searching dilutes the report. If no hit, **say so** — don't fabricate.

---

## 5. Recognize bugs that AI can't solve — and route them differently

Some bugs aren't "find-and-fix" tasks. Categorize them upfront so owner knows what tool to apply:

| Signal | Routing |
|---|---|
| "design discussion needed" / "let's sync up" / unresolved questions in description | Schedule design meeting |
| Backend vs frontend conflict (e.g. UI regex vs platform support) | Cross-team confirmation needed |
| Question marks in description / author uncertainty | Author needs to clarify, not owner to investigate |
| "Out of scope for this PR" / "later PR" | Tech debt — batch with similar |
| Empty or near-empty description | Needs author follow-up before triage possible |

---

## 6. Output dashboards, not essays

Owners scan, they don't read. Optimize for scan-ability:

**Do:**
- Tables for cluster summaries
- Bullet lists with file:line code anchors
- Time-ordered action plan (this week / next week / waiting)
- Effort tags (🟢 1 day · 🟡 2 days · 🔴 needs design)

**Don't:**
- Long paragraphs explaining each bug
- Bug-by-bug walkthrough in ID order
- Generic advice without specific code references
- Repeat what the bug already says — only add new information

**Test**: an owner should be able to skim the dashboard in 60 seconds and know what to do this week.

---

## Meta-insight: high-quality bugs need different enrichment than low-quality ones

If a bug already has meeting context, repro, expected/actual, **don't pad it with redundant info**. The valuable enrichment is **code anchor + cross-bug pattern recognition** (this bug is part of a larger cluster, this bug already has a fix in branch X, etc).

If a bug is one-line low-quality, the valuable enrichment is **context reconstruction** (find related code, infer repro from title, suggest what to ask author).

Apply the right enrichment to the right bug — don't run one template over everything.
