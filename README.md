# claude-plugin-bug-triage

A Claude Code plugin that batch-triages Azure DevOps bugs for feature owners — clusters them by root cause, enriches each with a code anchor + cross-bug signal, and produces an actionable plan you can scan in 60 seconds.

Built for the workflow where a feature owner opens ADO Monday morning and finds 15+ autobugs they don't have time to read one by one.

---

## Why

If you own a feature in a large repo, your bug queue is mostly noise:

- 5 bugs are the same playwright cross-origin failure with different file paths
- 3 are PR-comment follow-ups already promised in the PR thread
- 2 are killswitched tests that should be closed
- 1 is the actual bug that needs you

Reading them top-to-bottom takes 45 minutes. Most of that time is **finding the file:line in the stack trace**, **searching for sibling bugs**, and **deciding which ones can be closed without thought**. AI is unreasonably good at all three.

This plugin does that work in parallel sub-agents and gives you a dashboard.

---

## Install

In Claude Code (the CLI), at the `❯` prompt — **not in bash**:

```
/plugin marketplace add Userrober/claude-plugin-bug-triage
/plugin install bug-triage@bug-triage-marketplace
```

Then `/reload-plugins` (or restart Claude Code) so the skill is picked up.

---

## What it does

Given **N source bugs** (1, 6, 50 — doesn't matter), in 1–2 minutes wall time:

1. **Bulk fetches** title / state / repro / comments — fetches comments only for autobug-style bugs where the real story isn't in the description.
2. **Clusters** by created-date proximity, throw-site overlap, area path, and tag.
3. **Code anchors** each bug — resolves the throw site `file:line` from the stack, reads ±15 lines around it, identifies the function.
4. **Cross-bug signal** — finds sibling bugs hitting the same throw site in the last 60 days, recent commits touching that line, linked PR state, whether the test currently passes.
5. **Classifies** each bug with a closed action set — exactly one of:
   - `FIX` — real bug, anchor found, owner should patch
   - `DEDUP` — duplicate of #N
   - `CLOSE-STALE` — autobug whose test now passes
   - `CLOSE-FIXED` — fix already merged in PR #N
   - `NEEDS-REPRO` — author needs to clarify
   - `NEEDS-DESIGN` — cross-team / unresolved question
   - `INVESTIGATE` — flaky, owner should reproduce or instrument
6. **Effort tag** — 🟢 (<1 day) · 🟡 (1–3 days) · 🔴 (needs design)
7. **Outputs** a local markdown dashboard by default, or — with your confirmation — writes back to ADO.

---

## How to use

### The skill triggers automatically

You don't need to remember a command. Just say what you want:

- `triage bug 3013580`
- `帮我处理 bug 3013580`
- `process these bugs: 3000924, 3000926, 3000928`
- `enrich https://dev.azure.com/.../_queries/query/{guid}/`
- `triage Aimee 这周收到的所有 bug`
- `我有 22 个 bug 不知道从哪开始`
- `post bug bash 怎么处理这一堆`

The skill will:
1. Fetch the bug(s)
2. Show a one-screen summary
3. **Always pause and ask you 2 questions** (this is mandatory — see below)
4. Execute whatever you picked

You can also invoke explicitly: `/bug-triage:bug-triage`.

### The mandatory 2-step intake

Every invocation — even a single bug, even when you said "just fix it" — pauses for two questions:

**Q1: Where to output?**
- 本地报告 (local markdown only) — default, doesn't touch ADO
- 创建新 ADO bug (enriched) — writes one or more new bugs, source unchanged
- 直接更新源 bug 字段 — modifies severity/state/assignee/repro in place
- 啥也不做，只看摘要 — exit

**Q2 (only if Q1 writes to ADO): Assign to which email?**

This isn't bureaucracy — it's the safety net. The skill **never** auto-writes to ADO without you picking these two answers. If you want to read the analysis without committing to anything, pick "本地报告" and you'll get a markdown file in `triage-reports/`.

### Parallel by default

The moment you have **2+ bugs**, the skill fans out parallel sub-agents — one per source bug, all running concurrently. Each sub-agent:

- Has its own context window (no cross-pollution)
- Does the full enrichment pipeline (extract → code anchor → dedup signal → classify → write HTML → create new bug → verify)
- Returns a one-line JSON `{src_id, new_id, cluster_tag, action, effort, anchor}`

The main thread stays small even at N=50 because sub-agents only return tiny structured results.

For N≥3, the skill runs a **1-bug preview first** — you verify the rendering looks right, then it fans out the remaining N-1 in parallel. At N=2 it skips the preview (re-running 1 bad bug is cheap; re-running 14 isn't).

Wall time at N=15: ~2 minutes (validated on real ODSP-Web batches). Sequentially that would be ~30 minutes.

---

## Output modes

| Mode | What it does | When |
|---|---|---|
| **local** (default) | Markdown file in `triage-reports/` only | Default — never auto-write to ADO |
| **enrich-1to1** | N source bugs → N new enriched bugs (one sub-agent each) | Highest-value mode — each new bug is independently actionable, source unchanged |
| **meta-bug** | One new bug summarizing the whole cluster, sources linked as `Related` | When you want a single tracker for the batch |
| **update-source** | Modify severity / state / assignee / repro on the source bug | Triage hygiene only — fix wrong assignee, downgrade severity |

The high-value mode is **enrich-1to1**. Each new bug has an HTML body with:

- TL;DR (action, effort, cluster, test owner, source link, **effort legend**)
- Failure signature (test name, throw site, verbatim error, build link)
- Code anchor (15 lines around the throw, function name)
- Cross-bug signal (siblings in 60d, recent commits, linked PR state, current test status)
- Recommended next step (2–3 concrete sentences)

Tagged `AI-Triage; bug-triage-skill; Cluster<Letter>-<Name>` so they're filterable and distinguishable from real bugs.

---

## How a single bug gets enriched (the 7-phase pipeline)

Every sub-agent runs the same 7 phases on its assigned source bug. This is the asset — the prompt template lives in `skills/bug-triage/enrichment-agent-prompt.md` and is what determines whether the output is gold or filler.

### Phase 1 — Extract
`wit_get_work_item` + `wit_list_work_item_comments`. For autobug-filed items the real story is in **comments**, not Description (which is usually empty). Harvest: test name + framework, full stack trace + single throw site (`file:line`), verbatim error string, build/pipeline ID + branch + URL, linked PRs/IcMs, author, test owner email/@mention, CreatedDate, any quoted UI strings or resx keys.

Empty fields are written as `—`. Never invented.

### Phase 2 — Code anchor (the highest-value enrichment)
Take `file:line` from the stack, use bluebird `search_file_paths` to confirm the file exists, then `get_file_content(start_line=line-15, end_line=line+15)` to read ±15 lines around the throw. Capture function name, what it throws, surrounding control flow.

Hard budget: **3 targeted searches per bug**. If nothing found → `no code anchor found — needs manual investigation`. Fabricating a path is forbidden.

This is what saves the owner the most time. Resolving a stack trace to a code window by hand is 5–10 min per bug; the anchor lets them skip straight to the diff.

### Phase 3 — Cross-bug signal (the agent's unique edge)
Four checks an owner can't easily do without a search engine:

- **(a) Sibling bugs** — `search_workitem` for other bugs touching the same `file:line` in the last 60 days. Lists IDs grouped by date. This is what powers DEDUP recommendations.
- **(b) Recent fix attempts** — `code_history(method="commits", query="<filename>", start_time=60-days-ago)`. If a commit touched this exact line recently, mention SHA + PR. The bug may already be addressed.
- **(c) Linked PR state** — if the source bug links a PR, fetch its state. Merged PR + still-open bug = CLOSE-FIXED candidate.
- **(d) Stale autobug** — if the test now passes (per pipeline history), flag as CLOSE-STALE.

Skip these and the output is just a reformat of the source bug.

### Phase 4 — Classify (closed sets, no drift)
Three tags, each from a fixed enum:

- **action** ∈ `{FIX, DEDUP, CLOSE-STALE, CLOSE-FIXED, NEEDS-REPRO, NEEDS-DESIGN, INVESTIGATE}` — exactly one. The prompt explicitly forbids invented variants like `INVESTIGATE-FLAKY`.
- **effort** ∈ `{🟢, 🟡, 🔴}` — emoji only. The prompt forbids `S` / `M` / `L` (different scale, different meaning).
- **cluster_tag** = `Cluster<Letter>-<ShortName>` — letter must be **globally unique across the batch**. The orchestrator pre-assigns each sub-agent a `--next-free-letter` hint to prevent collisions (sub-agents can't see each other).

### Phase 5 — Render HTML (not Markdown)
ADO does **not** render Markdown in `Description` or `ReproSteps`. Both fields are HTML. Writing `## heading` shows as literal text and the bug becomes unreadable.

The body is built from a fixed skeleton: TL;DR (with effort legend) → Failure signature table → Code anchor `<pre><code>` block with ▶ marking the throw line → Cross-bug signal `<ul>` → Recommended next step (2–3 concrete sentences).

Tables must use `<table border='1' cellpadding='6' cellspacing='0'>`. Bug links must be full URLs. Both rules exist because earlier versions hit exactly these failure modes.

### Phase 6 — Create + verify
`wit_create_work_item` writes the new bug. For ODSP-Web specifically, the body goes to `Microsoft.VSTS.TCM.ReproSteps`, **not** `System.Description` — ODSP-Web's bug template hides Description, so writing there means the owner sees an empty body. For other projects, the skill probes one existing bug first to pick the right field.

Title format: `[Triage-{CLUSTER_TAG}] {failure_mode_short} (src #{SOURCE_ID})`, ≤80 chars. The `failure_mode_short` must describe **what broke** — `SpartanList AddColumn cross-origin`, not `DEDUP autobug`.

Then immediately verify: `wit_get_work_item(new_id)` and confirm `ReproSteps` contains `<h2>`, not raw `## `. Half of all broken renders get caught here, not later.

### Phase 7 — Return
The sub-agent returns **one line of JSON**:

```
{"src_id": <int>, "new_id": <int>, "cluster_tag": "<tag>", "action": "<one of 7>", "effort": "<🟢|🟡|🔴>", "anchor": "<file:line or 'none'>"}
```

No prose, no summary, no headers. The main thread stitches these into the final dashboard.

This is what makes N=50 feasible. If sub-agents returned paragraphs, the main thread would overflow at N≈10 and the whole architecture collapses.

### Why sub-agents instead of one big main-thread loop

| Trait | Sub-agent fan-out | Single-thread loop |
|---|---|---|
| Per-bug context | Own 200k window | Shared with all other bugs |
| Wall time at N=15 | ~2 min (parallel) | ~30 min (serial) |
| Context cost at N=50 | 50 × 1 line of JSON = trivial | Overflows at N≈10 |
| Search budget per bug | 3–5 searches without polluting others | Each search costs main-thread tokens |

Cost: sub-agents can't see each other, so the orchestrator has to coordinate things they'd otherwise share — most importantly the cluster-letter assignment.

---

## What makes this not just a wrapper around `wit_get_work_item`

Three things you'd otherwise do by hand:

1. **Code anchor** — for each stack trace, the skill resolves `file:line`, reads ±15 lines around the throw, names the function, and links it. No more `Ctrl+F` through 3 repos.
2. **Cross-bug pattern** — searches ADO for other bugs hitting the same throw site in the last 60 days, finds recent commits touching that line, checks linked PR state. This is what lets you say "5 of these 6 are the same bug, dedup them" without reading any of them.
3. **Cluster letter coordination** — sub-agents can't see each other, so the orchestrator pre-assigns each one a `--next-free-letter` hint. No two unrelated bugs both get tagged `ClusterB`.

---

## Requirements

- **Claude Code** (CLI, desktop, or web)
- **Azure DevOps MCP server** enabled — the skill calls `wit_get_work_item`, `wit_create_work_item`, `wit_get_query_results_by_id`, `wit_list_work_item_comments`, `search_workitem`, etc.
- **Code-search MCP** for code anchors — bluebird's `search_code` / `code_history` / `get_file_content` is what we tested with. Falls back to `Grep`/`Glob` on a local checkout if no MCP is available (lower-quality anchors but still works).

ODSP-Web specifics are baked in (bugs use `Microsoft.VSTS.TCM.ReproSteps`, not `System.Description`, because the bug template hides Description). For other projects the skill probes one existing bug to pick the right field.

---

## Layout

```
.claude-plugin/
├── plugin.json                    Plugin manifest
└── marketplace.json               Marketplace manifest

skills/bug-triage/
├── SKILL.md                       Entry point, autoloaded — triggers + 2-step intake + routing
├── methodology.md                 Clustering rules, dedup signals, scan-don't-read principle
├── source-handlers.md             How to handle BugBash / TestFailure / PRComment / IcM / etc
├── code-anchor-strategies.md      How to find file:line by intent (terminology / validation / UI / lifecycle)
├── output-template.md             Local markdown dashboard shape
├── ado-output.md                  ADO write-back recipes (HTML conversion, field probing, intake)
└── enrichment-agent-prompt.md     The sub-agent prompt for 1:1 mode — the asset
```

`SKILL.md` is the entry point; the others are lazy-loaded only when the relevant phase runs.

---

## Anti-patterns the skill protects against

These are mistakes the prompt has explicit guards for, learned from real triage runs:

- **Don't write Markdown to ADO** — `Description` and `ReproSteps` are HTML; `## heading` shows as literal text. The skill always converts.
- **Don't write to the wrong field** — different projects display different fields. ODSP-Web hides Description. The skill probes before writing.
- **Don't auto-default the assignee** — even if you mentioned an email earlier in the conversation, the skill re-asks. Bugs land in the wrong inbox otherwise.
- **Don't act without intake** — even on `triage bug X`, the skill pauses and asks Q1+Q2 first. Your natural-language ask is intent, not authorization.
- **Don't make up anchors** — if 3 searches find nothing, the bug gets `no code anchor found — needs manual investigation`, not a fabricated path.
- **Don't collide cluster letters** — orchestrator pre-assigns letters so two blind sub-agents don't both pick `ClusterB`.
- **Don't drift from closed sets** — `action` and `effort` are fixed enums. Sub-agents tend to invent variants like `INVESTIGATE-FLAKY` or `effort=M`; the prompt forbids this and the orchestrator normalizes if it slips through.
- **Don't write essays** — owners scan, they don't read. Output is a dashboard with tables and bullets, not paragraphs.

---

## Validated end-to-end

- 15 ODSP-Web playwright autobugs in ~2 min wall time, 1 preview + 14 parallel sub-agents
- 6-bug query (5 cross-origin DEDUP + 1 killswitch CLOSE-STALE) in ~100s
- HTML rendering verified — all bugs show `<h2>` / `<table>` / `<code>`, not raw markdown

---

## Status

Hand-built and unsupported. PRs welcome.

If something doesn't trigger or writes to the wrong place, file an issue with the conversation snippet — the skill's value is in its prompt, and most fixes are 5-line edits to `SKILL.md` or `enrichment-agent-prompt.md`.
