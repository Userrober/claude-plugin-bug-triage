# Code Anchor Strategies

How to find the relevant code for a bug efficiently. Each strategy is keyed to bug **intent**, not bug source.

**Universal rule**: 2-3 targeted searches per bug, max. If no hit, mark `no code anchor` and move on. Do NOT fabricate file paths.

---

## Intent → Strategy table

| Bug intent | Primary search | Secondary search | What to capture |
|---|---|---|---|
| Terminology / wording | resx string key (the literal label, e.g. `"Pending verification"`) | usage of the resx key in `.tsx` | resx file:line + usage site:line |
| Validation rejecting valid input | regex constant or validator function name | call sites of validator | regex constant + validator + call site |
| Validation accepting invalid input | same as above | + check unit tests for missing cases | + missing test case suggestion |
| UI element missing/broken | component name from title | parent that renders it | component file + parent mount site |
| Wrong data shown | data provider / API method | reducer / state slice | API call + state derivation |
| Lifecycle/cleanup (orphaned data) | delete / cleanup function | what creates the data | create site + delete site (asymmetry = bug) |
| Performance | function from stack/perf trace | callers (often called too many times) | hot function + caller count |
| Flight gating missing | `Flight*` / `is*Enabled` / ECS flight | API class missing the check | API entry + flight definition |
| Test failure | test file from stack | product code under test | test file:line + product code |
| Cleanup orphaned code | function/component from description | callers (should be 0) | confirm 0 callers, list dead files |

---

## Search tactics

### Use the most specific identifier from the bug

❌ Generic: `search_code(query="thumbnail rendering")` → noise
✅ Specific: `search_code(query="PageDetailsThumbnail")` → exact hit

The bug description usually contains the specific identifier — extract it.

### Quoted strings in description = goldmine

If the bug quotes a UI label ("Pending verification") or error message, **search the literal string in resx files first**. This is almost always the fastest path to the code.

### Path filtering when the codebase is large

If the bug mentions "Catalog Management" or "Site Management", scope searches to that path:
```
search_code(query="<term>", path="**/catalogManagement/**")
```

### Don't search on stop words

Avoid: "render", "settings", "page", "show", "click" — these dilute results.
Prefer: PascalCase identifiers, exact strings, file extensions.

---

## When code search fails

Reasons it might fail:
1. Identifier doesn't match codebase naming (renamed since)
2. Bug is in a different repo (codebase index doesn't cover it)
3. Bug is about a config / data file, not code
4. Bug is about a third-party integration

**What to do**:
- Try ONE alternate spelling or related term
- Check if there's a wiki page for the feature instead
- Mark `no code anchor — needs manual investigation` and move on
- Flag in the dashboard so owner knows to look themselves

---

## Special: PR-linked bugs

If the bug description contains a PR URL, **the PR is the anchor** — don't search.

What to extract from the PR:
- Files changed
- Reviewer comments (if not already in bug description)
- Merge status (open / merged / abandoned)
- Linked work items (might dedupe)

Use `repo_get_pull_request_by_id` for this.

---

## Special: IcM-linked bugs

If the bug references an IcM, the IcM usually has the root-cause analysis. Don't re-search the code from scratch.

What to extract:
- Linked gist/fix PR
- Affected services (from IcM title)
- Deployment status

---

## Output format for anchors

In the dashboard, format anchors as:
```
📁 `path/to/file.tsx:123` — short description of what's there
```

Keep paths relative, line numbers exact, descriptions short. Owners click these and read the code themselves.

---

## Anti-patterns

- ❌ Searching natural language phrases ("button doesn't work")
- ❌ Listing 10 search results when 1 is the right one — pick the best, drop the rest
- ❌ Inventing file paths to fill the gap
- ❌ Re-searching after a clear hit — stop early, don't optimize for completeness
- ❌ Quoting large code blocks — owners can read the file, just give the anchor
