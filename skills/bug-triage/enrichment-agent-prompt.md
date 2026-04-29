# Enrichment Agent Prompt — the high-value asset

When triaging N bugs into N enriched bugs (1:1 mode), spawn one sub-agent per source bug. **The prompt you give the agent is what determines whether the output is gold or filler.** This file is the prompt template.

Why a sub-agent: it has its own context window, can run 3–5 deep searches without polluting the main thread, and returns only a tiny structured result. Main thread stays small even at N=50.

---

## How to use

For each source bug, spawn an `Explore` sub-agent (or `general-purpose` if write access needed) with the prompt below. Substitute `{SOURCE_BUG_ID}` and `{PROJECT}`.

The agent does the full enrichment pipeline AND creates the new ADO bug itself, returning only `{src_id, new_id, cluster_tag, recommended_action}`.

---

## The prompt template

```
You are enriching ADO bug #{SOURCE_BUG_ID} in project "{PROJECT}" so a feature owner
can act on it in 30 seconds. You will create ONE new ADO bug containing the enrichment,
assigned to {ASSIGNEE_EMAIL}, then return a one-line summary.

═══════════════════════════════════════════════════════════════════════
PHASE 1 — EXTRACT ALL SIGNAL FROM THE SOURCE BUG (don't skip anything)
═══════════════════════════════════════════════════════════════════════

Call wit_get_work_item(id={SOURCE_BUG_ID}, project="{PROJECT}", expand="All") and
capture every field. Then ALWAYS call wit_list_work_item_comments(top=10) — even if
Description looks complete. For autobug-filed items the real story is in comments.

From the source, harvest these specific signals (NOT a summary — the raw artifacts):
  • Test name + framework (e.g. "@guid/playwright" / Spartan / Jest)
  • Full stack trace — identify the SINGLE throw site (file:line)
  • Error message verbatim (the exact string, in quotes)
  • Build / pipeline ID + branch + run URL
  • Linked PRs, linked IcMs, linked work items
  • Author / reporter (often the right person to ask, not the assignee)
  • Test owner — for autobugs filed by "Project Collection Build Service",
    look in Description / comments for a test-owner email or @mention. This
    is the human owner; the autobug reporter is just the pipeline. Surface
    this to TL;DR — it's who the dedup ping should go to.
  • CreatedDate (for dedup window)
  • Any quoted UI strings, resx keys, error codes, regex patterns
  • Comments that contradict or extend the description

If a field is empty, write "—" (dash). Do not invent.

── HONESTY GATE (read before Phase 2) ───────────────────────────────

Before going further, classify the source bug's evidence level. There
are THREE buckets, not two. Default to the middle bucket — most bugs
belong there.

  • REPRODUCIBLE — has stack trace, error string, UA, build link, OR
    screenshots that pin down the failing surface. Diagnosis is direct.

  • INVESTIGABLE — no stack/UA/screenshot, BUT one or more of:
      - linked IcM with content
      - linked PR / commit
      - related or parent work items with diagnoses
      - feature/area tag that maps to a known subsystem (ThemesV2,
        BrandCenter, Sharing, etc.)
      - quoted UI string, error code, regex, resx key, identifier
      - reporter is a known engineer who can be pinged
    This is the COMMON case. You should write a real diagnosis here,
    grounded in what you find by digging into IcM, related bugs, code,
    commits, wiki — but mark inferences as inferences (see Phase 2).

  • NO-SIGNAL — symptom is one short sentence AND every dig in the
    "Mandatory context dig" below returned nothing useful. Only after
    you've exhausted the dig may you fall back to the NO-REPRO template.
    Estimate: <10% of bugs. If you find yourself reaching for this
    bucket more than 1 in 10, you're under-investigating.

Mandatory context dig (run this BEFORE classifying as NO-SIGNAL):
  1. If the bug links an IcM, fetch the IcM summary — IcMs almost always
     contain the customer's actual repro and diagnosis the autobug
     stripped out.
  2. Read EVERY related and parent work item (wit_get_work_items_batch).
     States, titles, ReproSteps. Sibling bugs in the same cluster often
     contain root-cause analysis that applies here.
  3. Search wiki for the feature/tag (e.g. "ThemesV2 mobile", "BrandCenter
     rollout"). Architecture docs and rollout plans explain why a behavior
     differs across hosts/audiences.
  4. Code-search the most specific identifier from the title or tag (NOT
     stop words). 2-3 queries with bluebird `search_code`.
  5. Recent commits touching the suspected area (code_history with the
     feature name, last 90 days). A recent commit may already address it
     or reveal the responsible owner.
  6. Assignee's recent PRs/commits (code_history author=<assignee alias>,
     last ~15). Two strong signals to read off:
       - Are they actively working on this area? → they own it, route
         confidence high.
       - Are ALL their recent PRs in a different surface (e.g. all web,
         while bug is mobile)? → bug may be MIS-ROUTED, or the area has
         no owner. Surface this in TL;DR.
  7. Creator/Reporter's recent PRs + other bugs they filed (code_history
     author=<creator>, search_workitem created-by). Tells you:
       - Is creator a domain expert (their bug carries weight) or filing
         out of their lane (likely a drive-by report)?
       - Did they file sibling bugs in the same batch? → cluster signal.
  8. Bug ID / title cited in PR description or commit message
     (code_history query="<bug-id>" or distinctive title phrase). If a PR
     references this bug, the fix may already be in flight — check its
     state before recommending more work.

If steps 1-8 produced ANY substantive finding (IcM has detail / related
bug has root cause / wiki explains the architecture / code search hits a
specific gate / assignee or creator history reveals ownership or routing
signal / a PR already references the bug), you are INVESTIGABLE — write
a real diagnosis. Do not default to NO-SIGNAL just because the bug body
is one sentence.

If steps 1-8 all returned nothing useful, THEN classify as NO-SIGNAL and
use the NO-REPRO template.

For INVESTIGABLE and REPRODUCIBLE: write a real diagnosis using the
standard Phase 5 skeleton. Mark each anchor honestly as either:
  - VERIFIED (came from stack / error string / IcM / verified linked PR)
  - INFERRED (came from feature-tag mapping / code search / sibling bug
    pattern matching) — these are still useful, but say so

Inferences are allowed and expected — fabricated certainty is not. Words
like "likely", "appears to", "suggests" are honest. "The bug is in X"
without a stack pointing to X is fabricated certainty.

═══════════════════════════════════════════════════════════════════════
PHASE 2 — CODE ANCHOR (this is the highest-value enrichment)
═══════════════════════════════════════════════════════════════════════

Use bluebird `search_code` (call _get_started first if you haven't this session).
Budget: 3 targeted searches max. Stop early if you get a clear hit.

  1. Search the THROW SITE: take file:line from the stack and resolve it.
     Use search_file_paths to confirm the file still exists, then
     get_file_content(path, start_line=line-15, end_line=line+15) to read
     ±15 lines around the throw. Capture: function name, what it throws,
     surrounding control flow.

  2. Search for OTHER CALL SITES of the throwing function — sometimes the
     throw is generic and the real owner is the caller. method:functionName
     or class:className with bluebird syntax.

  3. If the bug is about a UI string / resx key / error message, search
     the literal string in `**/*.resx` or `**/*.tsx`.

If all three searches return nothing, write `no code anchor found — needs manual
investigation` and move on. DO NOT fabricate paths or guess.

IMPORTANT — distinguish anchor from hypothesis:
  • A VERIFIED anchor is a file:line that you have direct evidence for
    (stack trace, error string match, throw site, IcM-confirmed location,
    PR diff). State the evidence inline.
  • An INFERRED anchor is a file:line you reached by following the
    feature tag, sibling-bug pattern, or code search for related
    identifiers. Useful, but say "inferred from <evidence>" — never
    present an inference as a stack-confirmed fact.
  • Hedging words ("likely", "appears to", "suggests") are required
    when writing inferences. Removing them turns honest investigation
    into fabricated certainty.
  • If the source bug has no stack and no error string AND your
    context dig found nothing usable, you have ZERO anchors —
    label them as candidates and use the NO-REPRO template.

═══════════════════════════════════════════════════════════════════════
PHASE 3 — DEDUP / CLUSTER SIGNAL (find what owner can't easily find)
═══════════════════════════════════════════════════════════════════════

These are the cross-bug insights owners can't get without a search engine:

  a. Same throw site recently? Search ADO for other bugs touching the same
     file:line in the last 60 days:
       search_workitem(searchText="<filename> <line>", project="{PROJECT}")
     If you find duplicates / siblings, list their IDs.

  b. Recent fix attempted? code_history(method="commits", query="<filename>",
     start_time=60-days-ago) — if a recent commit touched this exact line,
     mention the commit SHA + PR. The bug may already be addressed.

  c. Linked PR closed? If the source bug links a PR, fetch its state. A merged
     PR + still-open bug = quick close candidate.

  d. Stale autobug? If CreatedBy is "Project Collection Build Service" and the
     test now passes (check pipeline history if accessible), flag as stale.

These four checks are what justifies the agent's existence. Skip them and the
output is just a reformat of the source bug.

═══════════════════════════════════════════════════════════════════════
PHASE 4 — CLASSIFY (one tag, one action, one effort)
═══════════════════════════════════════════════════════════════════════

Pick ONE recommended_action from this CLOSED set (no other values allowed —
if none fit, pick the closest and explain in "Recommended next step"):
  • FIX            — real bug, code anchor found, owner should patch
  • DEDUP          — duplicate of #N (cite the bug)
  • CLOSE-STALE    — autobug whose underlying test now passes
  • CLOSE-FIXED    — fix already merged in PR #N
  • NEEDS-REPRO    — author needs to clarify, not actionable yet
  • NEEDS-DESIGN   — cross-team / unresolved question, schedule meeting
  • INVESTIGATE    — flaky / intermittent, no clear DEDUP target, owner
                     should reproduce or instrument before deciding
DO NOT invent variants like "INVESTIGATE-FLAKY" or "INVESTIGATE-FLAKE" —
just "INVESTIGATE". Use exactly one of the 7 values above, all caps,
hyphenated as shown.

Pick ONE effort tag from this CLOSED set (emoji REQUIRED, no Latin letters):
  • 🟢  — <1 day, mechanical change, no design needed
  • 🟡  — 1–3 days, requires understanding but not architecture changes
  • 🔴  — needs design / cross-team coordination
DO NOT use "S" / "M" / "L" / "XS" — those are different scales. Use the
emoji exactly. If you really cannot tell, default to 🟡 and say so.

Pick ONE cluster_tag. Format: `Cluster<Letter>-<ShortName>`.
  • Letters MUST be globally unique across the batch — if the prompt told
    you a known cluster (e.g. ClusterA-CrossOrigin) and your bug matches
    it, REUSE that exact tag.
  • If your bug does NOT match the known cluster, you MUST pick a letter
    that has not yet been used. The prompt will tell you which letters
    are taken; pick the next free one (B, C, D, ...) and give it a
    descriptive ShortName.
  • DO NOT default to "ClusterB" just because it's "not A". If two
    different sub-agents each face an unmatched bug, they will collide.
    The orchestrator passes a `--next-free-letter` hint; honor it.

═══════════════════════════════════════════════════════════════════════
PHASE 5 — WRITE THE ENRICHED BUG (HTML, not markdown)
═══════════════════════════════════════════════════════════════════════

If you classified the bug as NO-SIGNAL in the Honesty Gate (i.e. the
mandatory context dig produced nothing), use THIS skeleton instead of
the standard one — and STOP HERE for the body. For REPRODUCIBLE and
INVESTIGABLE bugs, skip this section and use the standard skeleton
below.

  <h2>TL;DR</h2>
  <p><b>Recommended action:</b> NEEDS-REPRO (🟢 just for repro+triage)<br/>
  <b>Source bug:</b> <a href="{SOURCE_URL}">#{SOURCE_BUG_ID}</a></p>

  <h2>Symptom (verbatim from source)</h2>
  <blockquote>"{quote the source bug's repro/description verbatim}"</blockquote>

  <h2>Status: NO REPRO YET</h2>
  <p>The source bug has no stack trace, no UA, no screenshots, no telemetry
  pointer. Without a repro we don't know which surface is wrong, which
  platform/browser, or which code path is involved.</p>

  <h2>Repro plan (do this first)</h2>
  <ol>
    <li>{concrete step to set up the precondition}</li>
    <li>{concrete step to trigger the symptom}</li>
    <li>{concrete step to capture diagnostic data — UA, console, network}</li>
    <li>{diff against working case if applicable}</li>
  </ol>

  <h2>Possible starting points (UNVERIFIED)</h2>
  <p><i>Only relevant if the repro shows the bug is in this surface.</i></p>
  <ul>
    <li>{anchor 1 with one-line description of why it MIGHT be relevant}</li>
    <li>{anchor 2 — same rule}</li>
  </ul>
  <p>These are candidate code to look at after the repro narrows scope —
  not a diagnosis. The fix may be in an entirely different repo / layer.</p>

  <h2>Dedup check (verified)</h2>
  <ul>{list of related/parent bugs WITH their states — only verified facts}</ul>

Otherwise (REPRODUCIBLE bugs), build the HTML body using EXACTLY this
skeleton (keep it scannable, not prose):

  <h2>TL;DR</h2>
  <p><b>Recommended action:</b> {ACTION} ({EFFORT}){if DEDUP/CLOSE-FIXED: into <a href="...">#N</a>}<br/>
  <b>Cluster:</b> {CLUSTER_TAG}<br/>
  <b>Test owner:</b> {test_owner email or "—"} (ping for dedup confirmation)<br/>
  <b>Source bug:</b> <a href="{SOURCE_URL}">#{SOURCE_BUG_ID}</a> — {one-line title}</p>
  <p><i>Effort legend: 🟢 &lt;1 day · 🟡 1–3 days · 🔴 needs design / cross-team</i></p>

  <h2>How to repro</h2>
  <p><i>Owner's #1 question is "how do I reproduce this". Answer it
  immediately, even if the source bug had a stack — owners want concrete
  steps, not just a throw site.</i></p>
  <ol>
    <li>{precondition — env / tenant / flight / theme / role}</li>
    <li>{exact navigation — URL or click path}</li>
    <li>{trigger action — what to do that makes the bug appear}</li>
    <li>{what to observe — expected vs actual; include screenshot reference if source has one}</li>
  </ol>
  <p>If the source bug only has a screenshot, describe what the screenshot
  shows here in words AND link the attachment. If repro requires data the
  source didn't include (specific UA / tenant / flight), say so explicitly:
  <i>"need: UA string, tenant ring"</i> — don't fabricate steps.</p>

  <h2>Failure signature</h2>
  <table border='1' cellpadding='6' cellspacing='0'>
    <tr><td><b>Test</b></td><td>{test name + framework}</td></tr>
    <tr><td><b>Throw site</b></td><td><code>{file:line}</code></td></tr>
    <tr><td><b>Error</b></td><td><code>{verbatim error string}</code></td></tr>
    <tr><td><b>Build</b></td><td>{build id + branch + link}</td></tr>
  </table>

  <h2>Code anchor</h2>
  <pre><code>{15 lines around throw site, with ▶ marking the throw line}</code></pre>
  <p>Function: <code>{fn name}</code> in <a href="{file URL if available}"><code>{file:line}</code></a></p>

  <h2>Cross-bug signal</h2>
  <ul>
    <li><b>Same throw site in 60d</b> ({N} total in ADO search):
      <ul>
        <li>{date}: <a href="...">#A</a>, <a href="...">#B</a></li>
        <li>{date}: <a href="...">#C</a>, <a href="...">#D</a>, <a href="...">#E</a></li>
        <li>(group by createdDate; one row per day; 3–5 IDs per row max)</li>
      </ul>
    </li>
    <li><b>Recent commits touching this line:</b> {SHA + PR or "none"}</li>
    <li><b>Linked PR state:</b> {open/merged/abandoned or "no linked PR"}</li>
    <li><b>Test currently:</b> {passing/failing/unknown}</li>
  </ul>

  <h2>Recommended next step</h2>
  <p>{2–3 sentences. Concrete: "Patch {file:line} to handle {case}. Reuse
  pattern from {sibling bug or commit}. Estimated {effort}."}</p>

  <hr/>
  <p><i>Auto-generated by bug-triage skill. Source: #{SOURCE_BUG_ID}.</i></p>

CRITICAL conversion rules (the #1 thing that breaks):
  • This is HTML. Never write `## heading` or `**bold**` — write `<h2>` and `<b>`.
  • Tables MUST use <table border='1' cellpadding='6' cellspacing='0'>. Markdown
    tables show as raw `|` characters.
  • Bug links MUST be full URLs:
    https://dev.azure.com/onedrive/{PROJECT}/_workitems/edit/{id}

═══════════════════════════════════════════════════════════════════════
PHASE 6 — CREATE THE BUG IN ADO
═══════════════════════════════════════════════════════════════════════

For project ODSP-Web specifically: write the body to
`Microsoft.VSTS.TCM.ReproSteps`, NOT `System.Description`. ODSP-Web's bug
template hides Description. For other projects, probe one existing bug first
(wit_get_work_item with both fields, see which one has substantive content).

Call wit_create_work_item with:
  project: "{PROJECT}"
  workItemType: "Bug"
  fields:
    - System.Title: "[Triage-{CLUSTER_TAG}] {failure_mode_short} (src #{SOURCE_BUG_ID})"
      MUST be ≤80 chars total. Drop framework/test family if needed; cluster
      tag + failure mode + src id is enough. Owner sees this in ADO list view.

      The {failure_mode_short} MUST describe WHAT broke, not WHAT YOU DID.
      ✅ good: "SpartanList AddColumn cross-origin", "cmdbar timeout",
               "Sync icon waiter flake", "_spPageContextInfo undefined"
      ❌ bad:  "DEDUP autobug" (says nothing about the bug)
      ❌ bad:  "Verified in Grid mode" (sounds like a test pass, not a failure)
      ❌ bad:  "dedup-flaky" (action verb, not failure)
      Concrete rule: if you removed your failure_mode_short, could the
      reader still tell the bug apart from siblings? If no, rewrite.
    - System.AssignedTo: "{ASSIGNEE_EMAIL}"
    - System.Tags: "AI-Triage; bug-triage-skill; {CLUSTER_TAG}"
    - Microsoft.VSTS.Common.Severity: "{match source severity, default '4 - Low' for triage}"
    - Microsoft.VSTS.Common.Priority: "3"
    - Microsoft.VSTS.TCM.ReproSteps: "{the HTML body from Phase 5}"
      (format: "Html")

Then verify: wit_get_work_item(new_id) and confirm ReproSteps contains "<h2>"
(not literal "## "). If it shows raw markdown, you wrote to the wrong field
or didn't convert — fix before returning.

═══════════════════════════════════════════════════════════════════════
PHASE 7 — RETURN (tiny structured result, NOT a report)
═══════════════════════════════════════════════════════════════════════

Return ONLY this one-line JSON object — no prose, no summary, no headers:

  {"src_id": {SOURCE_BUG_ID}, "new_id": <new bug id>, "cluster_tag": "<tag>",
   "action": "<FIX|DEDUP|CLOSE-STALE|CLOSE-FIXED|NEEDS-REPRO|NEEDS-DESIGN>",
   "effort": "<🟢|🟡|🔴>", "anchor": "<file:line or 'none'>"}

The main agent stitches these into a final dashboard. If you write paragraphs of
explanation, you defeat the entire point of using a sub-agent — you blow up the
main context with redundant prose.
```

---

## Why each phase exists (so you can defend the structure under pressure)

| Phase | Without it | What you'd lose |
|---|---|---|
| 1. Extract | Skip comments → miss real story for autobugs | Owners get a reformat of empty Description |
| 2. Code anchor | Skip → owner spends 30 min finding the file | The single highest-value enrichment |
| 3. Dedup signal | Skip → owner closes 1 bug, ignores 4 siblings | Cross-bug insight is the agent's unique edge |
| 4. Classify | Skip → owner has to re-decide for each bug | Decision fatigue, 15 separate judgments |
| 5. HTML | Skip → output renders as `## raw text` | Bug becomes unreadable, owner ignores it |
| 6. Field probe | Skip → write to invisible field | User can't see the body (we hit this exact bug already) |
| 7. Tiny return | Skip → main context overflows at N≥10 | The original problem this file exists to solve |

---

## Anti-patterns specific to enrichment agents

- ❌ Returning the full HTML body in the result — main agent doesn't need it, it's already in ADO
- ❌ Re-summarizing what the source bug already says — only ADD, never RESTATE
- ❌ Searching natural-language phrases instead of identifiers from the stack
- ❌ Skipping the verify step in Phase 6 — half the broken renders are caught here
- ❌ Doing >5 search calls — if you can't find the anchor in 3, mark "none" and move on
- ❌ Inventing a `Cluster` tag per bug — coordinate naming with the main agent's batch plan

---

## When to NOT use 1:1 enrichment

This 1:1-with-sub-agent pattern is for batches where each bug genuinely needs its own enrichment. Don't use it when:

- Bugs are clearly duplicates of one root cause → use **meta-bug mode** (one new bug, link sources as Related)
- Batch is <5 bugs → main agent can do it inline without sub-agents
- Source bugs are already high-quality (manual bugs with full context) → just generate a local dashboard, no need to write back to ADO
