---
name: bug-triage
description: |
  Batch triage ADO bugs for a feature owner — cluster bugs, enrich with code
  anchors, identify quick wins, and generate a time-ordered action plan.

  TRIGGER when user:
  - mentions "triage", "batch process bugs", "review my bugs", "post-bug-bash cleanup"
  - asks "what should I work on this week" with bugs in scope
  - provides 3+ bug IDs at once and wants analysis
  - asks to process all bugs assigned to a person / area / tag
  - says they're a feature owner facing too many bugs

  SKIP for:
  - single-bug analysis (just answer directly with ado MCP tools)
  - general ADO questions like "how do I create a work item"
  - bug creation / writing a new bug (this skill only triages existing ones)
  - work items that are not bugs (Tasks, User Stories, Features)

  Examples that should trigger:
  - "帮我 triage Aharon 的 active bugs"
  - "我有 22 个 bug 不知道从哪开始"
  - "process these bugs: 3000924, 3000926, 3000928"
  - "what should this feature owner work on?"
  - "post bug bash 怎么处理这一堆"

  Examples that should NOT trigger:
  - "show me bug 3000924" → use ado MCP directly
  - "create a new bug for X" → use ado MCP wit_create_work_item directly
  - "what's the status of PR #123" → use ado MCP repo tools

  Default behavior: local markdown report only. Never writes to ADO unless user
  explicitly passes --apply or asks to update ADO.
---

# Bug Triage Skill

Batch-process N ADO bugs into a structured triage dashboard with code anchors, clusters, quick wins, and an actionable plan. Default behavior is **local-only** — never writes to ADO unless `--apply` is passed.

## When to use

- A feature owner has 5+ bugs to triage (post bug bash, sprint planning)
- A bug list is too noisy to read top-to-bottom
- An owner needs to know "what should I do this week vs next week"

## Inputs (any of)

- Explicit bug IDs: `/bug-triage 3000924, 3000926, 3000928`
- ADO **shared** query GUID (NOT temp query): `/bug-triage query:abc-123-...`
- Filter: `/bug-triage assignee:ahah@microsoft.com state:New`
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

- ❌ Don't enrich bugs that are already high-quality with redundant info — focus on what's missing (usually code anchor)
- ❌ Don't auto-write to ADO without `--apply` and explicit per-bug confirmation
- ❌ Don't generate essay-style reports — owners want dashboards, not narratives
- ❌ Don't cluster by ID order — cluster by semantic relationship
- ❌ Don't make up code anchors when search returns nothing — say "no anchor found"

## Reference

See `bug-triage/` directory for detailed sub-files. Read them lazily — only load the ones relevant to the current phase.
