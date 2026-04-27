# ADO Output Modes

How to write triage results back to ADO. Loaded only when user invokes `--apply` or asks to create a meta bug.

---

## Output modes

| Mode | What it does | When to use |
|---|---|---|
| **local** (default) | Markdown file in `triage-reports/` only | Default — never auto-write to ADO |
| **meta-bug** | Create 1 new bug summarizing the cluster, link sources as `Related` | User wants a single trackable artifact for the whole batch |
| **enrich-1to1** | N source bugs → N enriched bugs, one sub-agent per source | User wants individually actionable enriched bugs (see [enrichment-agent-prompt.md](./enrichment-agent-prompt.md)) |
| **update-source** | Modify the source bug fields in place (severity / state / assignee / repro) | Triage hygiene — fix wrong assignee, downgrade severity |

**Removed in v0.3.0**: `comment` mode and `both` mode. Reason: owners ignore comments on autobug-style bugs (they're already drowning in autobug noise). Either create a new tracked artifact (`enrich-1to1` / `meta-bug`) or update the source in place.

Always run the **mandatory 2-step intake** (defined in SKILL.md Routing section) before writing.

---

## Guided intake — ask before assuming

When the user says anything like "输出到 ADO" / "post to ADO" / "create a bug" / "apply" without specifying details, **proactively ask 2–4 questions via `AskUserQuestion`** instead of guessing. Skip questions you already know the answer to.

**Required clarifications (ask any that aren't already answered):**

Each question MUST present a **recommended default** as the first option, marked `(推荐)` / `(Recommended)`. The recommendation is inferred from context — same project as source bugs, current user as assignee, max severity of source bugs, etc. Goal: user can press Enter on every question and get a sensible result.

1. **Output mode** — recommended: `meta-bug` (most common ask)
2. **Assignee** — recommended: current user (from MCP `whoami` / prior context)
3. **Project** — recommended: same project as source bugs
4. **Severity / Priority** — recommended: match the highest severity among source bugs

**Optional follow-ups (only if relevant):**
- Iteration / Area path overrides (default: project root)
- Additional tags beyond `AI-Triage; bug-triage-skill`
- Whether to set `State` to something other than `New`

**Don't over-ask** — if user already said "create meta bug assigned to me in ODSP-Web", you have everything; just confirm assignee email if not visible from MCP context, then proceed.

**Sample question set** (use `AskUserQuestion`, always recommended-first; v0.3.0 — 2-step intake, no comment option):
```
Q1: 你想输出到哪里？
    - 本地 markdown 报告（推荐 — 默认，不动 ADO）
    - 创建新 ADO bug (enriched, 1:1 或 meta) — 写新 bug，源 bug 不变
    - 直接更新源 bug 字段 — 改 severity/state/assignee/repro
    - 啥也不做，只看摘要 — exit

Q2 (only if Q1 is "创建新 ADO bug" or "更新源 bug"): Assign 给哪个邮箱？
    - {last-used-or-current-user-email}（推荐）
    - 其他 email（下一条告诉我）

Q3 (only if multi-bug AND Q1 is "创建新 ADO bug"): 1:1 enrich 还是 meta-bug？
    - 1:1 enrich (N 个新 bug) — 推荐，每个源 bug 独立可 act
    - meta-bug (1 个新 bug) — 适合 cluster 已收敛、想要单一 tracker

Q4 (only if writing to ADO): Severity / Priority？
    - Sev {max-of-sources} / Pri {derived}（推荐 — 跟随源 bug 最高等级）
    - 我手动指定
```

**Default summary line** before executing: surface the inferred recommendations even if user didn't answer all questions, so they have a chance to redirect:
> "我会创建 1 个 meta bug → ODSP-Web 项目，assign 给 v-ziqyang@microsoft.com，Sev 3。OK 就回 'go'，要改哪项告诉我。"

After the user answers, briefly confirm the plan in one sentence ("好，创建 meta bug 在 ODSP-Web 项目，assign 给 v-ziqyang@microsoft.com") then execute.

---

## Custom mode — user-driven overrides

The guided questions cover 80% of cases. For the other 20%, **let the user fully customize** anything about the output. Treat the dashboard as a draft, not a fixed template.

**What users can override (accept any of these without pushback):**

| Override | Example user request |
|---|---|
| **Title** | "标题改成 [P0] xxx" / "title it 'Test infra: cross-origin failures'" |
| **Body content** | "去掉 root-cause hypothesis 那段" / "只要 TL;DR 表" / "加一段 owner FAQ" |
| **Field placement** | "放到 Acceptance Criteria 而不是 ReproSteps" / "也复制一份到 Description" |
| **Work item type** | "用 Task 不要 Bug" / "create as User Story" / "Epic" |
| **Tags** | "加 tag 'TestInfra-Followup'" / "去掉 Demo tag" |
| **State / Reason** | "直接设成 Active" / "open as Resolved with reason 'By Design'" |
| **Iteration / Area** | "iteration 设到 ODSP-Web\\Sprint 89" / "area path: ODSP-Web\\TestInfra" |
| **Linking style** | "用 Parent/Child 不要 Related" / "link as 'Duplicate of' 而不是 Related" |
| **Multiple meta bugs** | "拆成 2 个 bug，sub-cluster A 一个，B 一个" |
| **Reviewers / @-mentions** | "在 description 里 @joanthon" |
| **Custom fields** | "Custom.RiskLevel 设成 High" |
| **Skip steps** | "不要 link source bugs" / "只创建 bug 不要 comment" |
| **Dry-run** | "先给我看看 HTML 长啥样，别真创建" |

**How to handle custom requests:**

1. **Accept the override** without arguing for the default — user knows their workflow better
2. If the override conflicts with a known constraint (e.g. invalid state transition), **flag it once** and ask before changing course
3. If the user gives a partial override ("change the title"), keep everything else the same — don't reset other choices
4. After applying overrides, **show what you're about to write** before executing if the change is large (new title, removed sections, different work item type)
5. For dry-run requests, render the HTML to a local file (e.g. `triage-reports/preview-{date}.html`) so the user can open it in a browser

**Iterative refinement is normal**: user creates bug → "the table is too wide" → you update via `wit_update_work_item` → repeat. Don't treat the first write as final.

**Anti-pattern**: refusing customization because "the template says X". The template is a starting point; the user owns the output.

---

## ⚠️ Critical: ADO fields are HTML, not Markdown

**ADO does NOT render Markdown** in `System.Description` or `Microsoft.VSTS.TCM.ReproSteps`. Both fields are **HTML**.

If you write `## Heading` and `| col |` tables, they show as literal text — owner sees `## Heading` not a heading. The dashboard becomes unreadable.

**You must convert Markdown → HTML before writing.** Use these mappings:

| Markdown | HTML |
|---|---|
| `## H` | `<h2>H</h2>` |
| `### H` | `<h3>H</h3>` |
| `**bold**` | `<b>bold</b>` |
| `` `code` `` | `<code>code</code>` |
| `- item` | `<ul><li>item</li></ul>` |
| `1. item` | `<ol><li>item</li></ol>` |
| `> quote` | `<blockquote>quote</blockquote>` |
| `---` | `<hr>` |
| `[text](url)` | `<a href="url">text</a>` |
| Markdown table | `<table border='1' cellpadding='6' cellspacing='0'><thead>…</thead><tbody>…</tbody></table>` |

Bug links inside the HTML body should be full URLs: `https://dev.azure.com/{org}/{project}/_workitems/edit/{id}`.

---

## ⚠️ Critical: pick the right field per project

Different ADO projects display different fields. **Do not assume `System.Description` is visible.**

Known examples:
- **ODSP-Web**: Bug template hides `System.Description`. Main content must go in `Microsoft.VSTS.TCM.ReproSteps`.
- Many other projects: `System.Description` is the visible main field.

**Before writing**, fetch one existing bug from the same project and check which field has content:
```
wit_get_work_item(id={any-existing-bug}, fields=["System.Description", "Microsoft.VSTS.TCM.ReproSteps"])
```
- If `ReproSteps` has the substantive content → write to `Microsoft.VSTS.TCM.ReproSteps`
- Otherwise → write to `System.Description`

When in doubt, **write to BOTH fields** — redundant but guaranteed visible.

---

## meta-bug mode — recipe

1. **Confirm with user**: project, assignee email, optional severity/priority/area path, output mode
2. **Probe field convention**: read one existing bug in target project to determine `Description` vs `ReproSteps`
3. **Convert dashboard markdown → HTML** (or build HTML directly)
4. **Create the bug** via `wit_create_work_item`:
   - `System.Title`: `"[Triage Summary] {short cluster description}"`
   - `System.AssignedTo`: user-confirmed email
   - `System.Tags`: `"AI-Triage; bug-triage-skill"` (so it's filterable, distinguishable from real bugs)
   - Severity: match the highest severity of source bugs (or Sev 4 for demo / non-product)
   - Body field (per project convention): full HTML dashboard
5. **Link source bugs** via `wit_work_items_link` with `linkType: "related"`, one call per source bug
6. **Verify** with `wit_get_work_item` — confirm body field actually rendered (look for `<h2>` etc, not raw `##`)
7. **Report URL to user**: `https://dev.azure.com/{org}/{project}/_workitems/edit/{newId}`

---

## comment mode — REMOVED in v0.3.0

This mode was removed because comments on autobug-style bugs get ignored — owners already filter them out. Use `enrich-1to1` (creates a tracked new bug) or `update-source` (modifies the source itself) instead.

---

## Anti-patterns

- ❌ Writing markdown to `System.Description` / `ReproSteps` — it won't render
- ❌ Assuming `Description` is the visible field without checking
- ❌ Forgetting the `AI-Triage` tag — meta bugs get mistaken for real bugs in queries
- ❌ Linking source bugs without comments explaining why (use the `comment` param of `wit_work_items_link`)
- ❌ Writing the meta bug without first confirming assignee — it lands in someone's queue uninvited

---

## Validation checklist before reporting "done"

- [ ] Bug created and ID returned
- [ ] Assignee field populated correctly (`uniqueName` matches what user gave)
- [ ] Body field (Description OR ReproSteps) contains `<h` HTML tags, not literal `##`
- [ ] All source bugs linked as `Related`
- [ ] Tags include `AI-Triage`
- [ ] Reported URL to user

If any item fails, fix before declaring done — don't make user discover a broken render.
