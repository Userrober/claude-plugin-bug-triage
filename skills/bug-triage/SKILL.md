---
name: bug-triage
description: |
  Triage ADO bugs for a feature owner — works for ANY bug-related ask, from
  a single bug ("what should I do with #3013580") to a 50-bug post-bug-bash
  batch. Always uses guided intake (asks the user how to handle the bug)
  before any write. Never auto-writes to ADO.

  TRIGGER when the user mentions an ADO bug ID and any verb that implies
  "decide what to do with it" — triage, handle, process, review, dedup,
  close, fix, look at, work on, etc. The trigger is intentionally broad
  because the value of this skill is the GUIDED INTAKE (asking the user
  what they want to do) before any destructive action — not just clustering.

  TRIGGER specifically when user:
  - says "triage" / "handle" / "process" / "review" + a bug ID or query
  - provides 1+ bug IDs and asks what to do with them
  - mentions "batch process bugs", "review my bugs", "post-bug-bash cleanup"
  - asks "what should I work on this week" with bugs in scope
  - asks to process all bugs assigned to a person / area / tag
  - says they're a feature owner facing too many bugs
  - mentions dedup, close, resolve a specific bug

  SKIP for:
  - read-only single-bug fetch with no action verb ("show me bug 3000924",
    "what's bug 3000924 about") → use ado MCP directly, no skill needed
  - general ADO questions like "how do I create a work item"
  - bug creation from scratch (this skill only triages existing bugs)
  - work items that are not bugs (Tasks, User Stories, Features)

  Examples that SHOULD trigger:
  - "triage bug 3013580"
  - "帮我处理 bug 3013580"
  - "I want to dedup #3013580 against #3013053"
  - "帮我 triage Aharon 的 active bugs"
  - "我有 22 个 bug 不知道从哪开始"
  - "process these bugs: 3000924, 3000926, 3000928"
  - "post bug bash 怎么处理这一堆"

  Examples that should NOT trigger:
  - "show me bug 3000924" (just a read, no action)
  - "what's the status of PR #123" (PR not bug)
  - "create a new bug for X" (creation from scratch)

  Default behavior: local markdown report only. NEVER writes to ADO unless
  user explicitly confirms via guided intake.
---

# Bug Triage Skill

Process ADO bugs for a feature owner — anywhere from 1 bug ("what do I do with this autobug?") to 50 ("post-bug-bash, help"). The unifying value is **guided intake**: this skill ALWAYS asks the user what to do with each bug before any write. No silent ADO modifications, ever.

## When to use

- A feature owner has 1+ bugs and needs to decide what to do (triage, dedup, close, fix-plan)
- A bug list is too noisy to read top-to-bottom (5+ bugs)
- An owner needs to know "what should I do this week vs next week"

## Routing — pick the path based on bug count

The skill has two modes; pick automatically based on input size:

### Single-bug mode (N=1)
The user said something like "triage bug 3013580". Don't run cluster analysis (you only have one bug). Instead:

1. Fetch the bug (`wit_get_work_item` + comments if autobug-style)
2. Show a 1-screen summary (title / state / severity / repro / stack trace / linked items)
3. **Guided intake — ask what to do** via `AskUserQuestion`. Recommended options (4 max, recommend the most likely first based on the bug):
   - **Dedup into existing bug** — if you found a likely duplicate, recommend this
   - **Add a comment** — if user just wants to leave context, no state change
   - **Update fields** (severity / assignee / state / repro) — for triage hygiene
   - **Create enriched bug** — only if there's no existing target; rarely the right answer for 1 bug
   - **Nothing — just show me** — read-only summary, exit
4. Execute only after user confirms. NEVER write to ADO without an explicit "Yes, do it" answer.

### Batch mode (N≥3)
Run the full pipeline below.

(N=2 is a judgment call — usually run batch mode if they look related, single-bug mode twice if they don't.)

## Inputs (any of)

- Explicit bug IDs: `/bug-triage 3000924, 3000926, 3000928`
- Single bug: `triage bug 3013580` or `帮我处理 bug 3013580`
- ADO **shared** query GUID (NOT temp query): `/bug-triage query:abc-123-...`
- Filter: `/bug-triage assignee:ahah@microsoft.com state:New`
- Bug list pasted from ADO

## Workflow (batch mode)

Execute these phases **in order**. Use TaskCreate to track each phase as you go.
- Bug list pasted from ADO

## Workflow

Execute these phases **in order**. Use TaskCreate to track each phase as you go.

### Phase 1 — Bulk Fetch (parallel)

Use `wit_get_work_items_batch_by_ids` with these fields to control payload size:
```
["System.Id", "System.Title", "System.State", "System.Tags",
 "System.AreaPath", "System.CreatedDate", "System.CreatedBy",
 "System.AssignedTo", "System.Description",
 "Microsoft.VSTS.TCM.ReproSteps",
 "Microsoft.VSTS.Common.Severity", "Microsoft.VSTS.Common.Priority"]
```

If batch result exceeds context (the ADO MCP saves it to a file), use `jq` to extract structured summary first, then deep-read individual bugs as needed.

**Phase 1.5 — Conditional comment fetch.** For any bug where:
- `System.Description` is empty AND `Microsoft.VSTS.TCM.ReproSteps` is empty/trivial, OR
- `System.CreatedBy` is `Project Collection Build Service` / a bot account, OR
- Tags include `Autobugged`, `TestResultsAutobugger`, `fileBug`

…the **real content lives in comments**, not fields. Call `wit_list_work_item_comments` (top 5 is enough — newest first) for those bugs in parallel. Skip this for high-quality bugs (BugBash with full meeting context, manual bugs with description) — comments add noise there.

### Phase 2 — Classify by Source

Apply rules from [source-handlers.md](./source-handlers.md) to tag each bug:
- BugBash · PRComment · IcM · TestFailure · Manual · Unknown

This determines downstream processing.

### Phase 3 — Cluster

Apply rules from [methodology.md](./methodology.md) §1 (clustering):
- By creation date proximity (±1 day = likely same event)
- By title keyword overlap
- By area path
- Sub-cluster within each by intent (terminology / data lifecycle / real bug / etc)

### Phase 4 — Identify Quick Wins

Scan descriptions for:
- "addressed in PR #N" / "fixed by PR #N" → **close-able after PR merges**
- Embedded commit hash / branch like `user/xxx/fix-yyy` → **fix may already exist**
- "duplicate of #N" → **dedup**
- Source-tagged auto-bugs that have been resolved → **likely stale**

Flag these — owner reviews each in 30 seconds.

### Phase 5 — Code Anchor (parallel per bug)

Apply [code-anchor-strategies.md](./code-anchor-strategies.md) by intent:
- Terminology bug → search resx files for string keys
- Validation bug → search validator files / regex constants
- UI behavior bug → search component name
- Lifecycle/data bug → search reducer / data provider / API class
- PR comment bug → use PR link directly, skip search

Use `bluebird` MCP `search_code` if available; fall back to `Grep`/`Glob` on local files.
**Do not over-search**: 2-3 targeted queries per bug max. If no hit, mark "no code anchor — needs manual investigation".

### Phase 6 — Generate Dashboard

Use [output-template.md](./output-template.md) for the report structure:
- Top-level cluster summary table
- Per-cluster details with sub-clusters
- Time-ordered action plan (this week / next week / waiting / needs meeting)
- Meta-insights specific to this batch

**Output is a local markdown file** at `triage-reports/{owner}-{date}.md` unless user says otherwise.

## Behaviors

### Default: local-only
Never write to ADO. Show enrichment as suggestions in the dashboard.

### `--apply` flag — ADO write modes
Multiple output modes available — see [ado-output.md](./ado-output.md) for full recipes:

- **comment mode** — add triage comment to each source bug (markdown supported in comments)
- **meta-bug mode** — create one new bug summarizing the cluster, link sources as `Related`
- **enrich-1to1 mode** — N source bugs → N enriched bugs, one sub-agent per source (see [enrichment-agent-prompt.md](./enrichment-agent-prompt.md) — this is the highest-value mode and the prompt is the asset)
- **both** — meta bug + back-reference comments on sources

Always confirm mode + assignee + project with user before writing.

**For batches ≥5 in `enrich-1to1` mode**: spawn one sub-agent per source bug using the prompt in `enrichment-agent-prompt.md`. Sub-agent does the full pipeline (extract → code anchor → dedup signal → classify → write HTML → create bug → verify) and returns a single-line JSON. Main thread stays small even at N=50. Run a 1-bug preview first, get user OK, then fan out.

**Cluster letter coordination (orchestrator responsibility):**
Sub-agents can't see each other, so two unrelated bugs both not-matching ClusterA will each pick "ClusterB" and collide. To prevent this:
1. After the 1-bug preview, the orchestrator records the known clusters (e.g. `{A: CrossOrigin}`).
2. Before fanning out, **assign each remaining sub-agent a `--next-free-letter` hint in its prompt** based on the order they're spawned: bug #2 gets "B if mismatch", bug #3 gets "C if mismatch", etc. Most will reuse A; the few that don't get a unique pre-assigned letter.
3. Alternative cheap approach: fan out anyway, then in the dashboard generation phase, post-process collisions by reading all new bug tags and renaming duplicates (B → C, B → D). Lower upfront cost, requires the post-process step.

**Effort + action vocabularies are CLOSED sets** (see Phase 4 of enrichment-agent-prompt.md). Sub-agents tend to invent variants ("INVESTIGATE-FLAKY", effort="M"). The prompt now forbids this explicitly, but if you see violations in the JSON returns, normalize before writing the dashboard.

**⚠️ Two gotchas that bite every time:**
1. ADO `Description` / `ReproSteps` fields are **HTML, not Markdown** — must convert before writing or owner sees raw `## headings`
2. Different projects display different fields (e.g. ODSP-Web hides `Description`, uses `ReproSteps`) — probe an existing bug first to pick the right one

### Stop conditions
- 0 results from initial fetch → ask user to refine filter
- > 50 bugs → warn user about cost, ask to confirm or narrow
- Permission denied on a project → degrade gracefully, report which bugs were skipped

## Anti-patterns (don't do these)

- ❌ **Don't act on an ADO bug without guided intake.** Even if the user asked you to "triage" or "handle" or "dedup" a bug, ALWAYS pause and ask via `AskUserQuestion` what specific action they want — `dedup` / `comment` / `update` / `enrich` / `nothing`. The user's natural-language request is intent, not authorization.
- ❌ Don't enrich bugs that are already high-quality with redundant info — focus on what's missing (usually code anchor)
- ❌ Don't auto-write to ADO without explicit per-bug confirmation
- ❌ Don't generate essay-style reports — owners want dashboards, not narratives
- ❌ Don't cluster by ID order — cluster by semantic relationship
- ❌ Don't make up code anchors when search returns nothing — say "no anchor found"
- ❌ **Don't dump a polished analysis as a final answer when there's a follow-up action implied.** "Triage bug X" is a request that ends in an ASK, not a markdown report. Show summary briefly, then ASK.

## Reference

See `bug-triage/` directory for detailed sub-files. Read them lazily — only load the ones relevant to the current phase.
