# claude-plugin-bug-triage

A Claude Code plugin that batch-triages ADO bugs for feature owners — clusters them by root cause, enriches each with a code anchor + cross-bug signal, and produces an actionable plan.

Built for the workflow where a feature owner opens ADO Monday morning and finds 15+ autobugs they don't have time to read one by one.

## What it does

Given N source bugs (an ADO query, a list of IDs, or a filter):

1. **Bulk-fetches** title / description / repro steps / comments — fetches comments only for autobug-style bugs where the real story isn't in the description.
2. **Clusters** by created-date proximity, throw-site overlap, area path.
3. **Code anchors** each bug — resolves the throw-site `file:line` from the stack, reads ±15 lines around it, identifies the function.
4. **Cross-bug signal** — finds sibling bugs hitting the same throw site in the last 60 days, recent commits touching that line, linked PR state.
5. **Classifies** each bug with a closed action set: `FIX | DEDUP | CLOSE-STALE | CLOSE-FIXED | NEEDS-REPRO | NEEDS-DESIGN | INVESTIGATE`.
6. **Outputs** either a local markdown dashboard (default) or — with `--apply` — writes enriched bugs back to ADO via fan-out sub-agents (1:1, meta-bug, or comment modes).

The high-value mode is `enrich-1to1`: each source bug spawns one sub-agent that does the full pipeline and creates a new ADO bug with a structured HTML body the owner can scan in 30 seconds. Main thread stays small even at N=50 because each sub-agent returns only a one-line JSON.

## Install

```
/plugin marketplace add Userrober/claude-plugin-bug-triage
/plugin install bug-triage@bug-triage-marketplace
```

## Use

The skill triggers automatically on phrases like:

- "triage 这一批 bug"
- "process these bugs: 3000924, 3000926, 3000928"
- "我有 22 个 bug 不知道从哪开始"
- "post bug bash 怎么处理"

Or invoke directly: `/bug-triage:bug-triage`.

Default behavior is **local-only** — generates a markdown report in `triage-reports/`. To write back to ADO, pass `--apply` and the skill will ask you about output mode (1:1 enrichment / meta-bug / comments) before writing anything.

## Requirements

- Claude Code with the Azure DevOps MCP server enabled (the skill calls `wit_get_work_item`, `wit_create_work_item`, `search_workitem`, etc.)
- For code anchors: a code-search MCP (e.g. bluebird's `search_code` / `code_history`) or fall back to `Grep` on a local checkout.

## Layout

```
skills/bug-triage/
├── SKILL.md                       Main entry point (autoloaded)
├── methodology.md                 Clustering rules
├── source-handlers.md             How to handle BugBash / autobug / IcM / etc
├── code-anchor-strategies.md      How to find file:line by intent
├── output-template.md             Dashboard markdown shape
├── ado-output.md                  ADO write-back recipes (--apply modes)
└── enrichment-agent-prompt.md     The sub-agent prompt for 1:1 mode (the asset)
```

`SKILL.md` is the entry point; the others are lazy-loaded only when the relevant phase runs.

## Status

Validated end-to-end on 15 ODSP-Web playwright autobugs. The 1:1 enrichment pipeline produces ADO bugs with HTML bodies that render correctly (h2/table/code/ul). Known noise patterns are documented in `enrichment-agent-prompt.md` Phase 4.

This plugin is hand-built and unsupported. PRs welcome.
