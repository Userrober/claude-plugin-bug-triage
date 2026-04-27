# Output Template

The standard structure for a triage dashboard. Owners should be able to scan it in 60 seconds.

---

## Filename convention

`triage-reports/{owner-shortname}-{YYYY-MM-DD}.md`

Example: `triage-reports/aharon-2026-04-23.md`

---

## Required sections (in order)

### 1. Header
```
# 📊 {Owner Name} — {N} Bug Triage Dashboard
**{Source/Context} · No ADO writes** · Generated {date}
```

### 2. Top-level cluster summary
A table grouping all bugs into 3-5 clusters, max 1 line per cluster:

| Cluster | Count | Recommended action |
|---|---|---|
| 🌟 {ClusterName} | {N} | {one-sentence action} |

**Rule**: If you can't fit a cluster onto one row of action, the cluster is too vague — split it.

### 3. Per-cluster details

For each cluster:

```
## 🌟 Cluster N: {Cluster Title} ({M} bugs)

> **Meta-insight**: {one-sentence why these are grouped, why it matters}

### 📦 Sub-cluster A: {sub-name} ({K} bugs)

| Bug | Issue | Code Anchor | Effort |
|---|---|---|---|
| **#XXXXX** | short summary | `file:line` | 🟢 1 day |
```

Effort tags:
- 🟢 < 1 day (mechanical, anchor is clear)
- 🟡 1-3 days (needs investigation)
- 🔴 needs design / cross-team / design meeting

### 4. Quick wins section (if any)

Bugs that can be closed/resolved with minimal action:

```
## ⚡ Quick Wins ({N} bugs)

| Bug | Why it's quick | Action |
|---|---|---|
| **#XXXXX** | Has fix in PR #YYY | Wait for merge, close |
| **#XXXXX** | Copilot already wrote fix in branch X | Review + merge |
```

### 5. Action plan (time-ordered)

```
## 💡 Suggested Action Plan

### This week (quick wins)
- [ ] {time estimate}: {specific action with bug ID}

### Next week (focused work)
- [ ] {time estimate}: {specific action with bug ID}

### Needs scheduling
- [ ] **Design meeting**: {topic} — needed for #X, #Y, #Z

### Waiting on others
- [ ] **PR review**: #X waiting on {person}
- [ ] **Backend confirmation**: #Y needs {team}
```

### 6. Meta-insights for this batch

```
## 🤖 What this batch revealed

- {insight 1 - e.g. "11 of 22 bugs are from same bug bash, suggest one epic"}
- {insight 2 - e.g. "4 bugs all touch CustomPropertiesDrawer.resx — bundle into 1 PR + 1 loc cycle"}
- {insight 3 - e.g. "2 bugs are stale auto-bugs that auto-resolved"}
```

These are the cross-bug observations that make the dashboard worth more than 22 individual reports.

### 7. Footer

```
---

**No ADO writes performed.** To apply enrichments to ADO comments, re-run with `--apply` flag.

Source files for further investigation:
- {list any non-trivial files referenced}
```

---

## Style rules

- Use emojis sparingly as visual anchors (🌟 cluster, 📦 sub-cluster, ⚡ quick win, 🟢🟡🔴 effort)
- Bug IDs always **bold** so they stand out
- File paths always in `code` style with line numbers
- Person names in plain text (don't @-mention — this is local markdown, not ADO)
- Tables > bullets for comparisons; bullets > tables for action items

---

## Length budget

- Total dashboard: ~300-500 lines for 20 bugs
- Per-cluster: ~30-60 lines
- Per-bug row in tables: 1 line
- Per-bug detail (when needed): 5-10 lines max

If exceeding budget, the report is too verbose — trim.

---

## Apply mode (`--apply`)

When user passes `--apply`, after generating the dashboard:

1. Show the user a list of bugs with proposed comments
2. Ask **per bug** whether to apply (default: no)
3. Use `wit_add_work_item_comment` only for confirmed bugs
4. **Never** modify `System.Description` or `Microsoft.VSTS.TCM.ReproSteps` — comments only
5. Comment format starts with `## Triage Note (added by AI analysis)` so it's clear what's AI-generated

After applying, append a section to the dashboard listing what was written.
