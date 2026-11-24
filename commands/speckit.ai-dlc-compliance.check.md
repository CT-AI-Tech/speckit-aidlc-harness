---
description: "Load only the AI-DLC harness guardrails relevant to the current vertical and lifecycle phase, and surface binding/advisory findings"
---

# Compliance Guardrail Check

This command is invoked as a hook before/after core Spec Kit commands. It loads
guardrails from the AI-DLC compliance harness — never the whole harness, only the
handful of entries that match this project's vertical and the current lifecycle
phase — and tells the user/agent what applies before they continue.

## Step 1 — Load config, auto-fetch the harness if missing, degrade gracefully if absent

Read `.specify/extensions/ai-dlc-compliance/compliance-config.yml`.

- If it does not exist: skip silently with a one-line note
  (`[ai-dlc-compliance] no compliance-config.yml found, skipping guardrail check`).
  Do not block the calling command.
- If it exists, read `harness_path`, `vertical`, and `harness_repo_url` (defaults to
  `https://github.com/CT-AI-Tech/ai-dlc-harness.git` when the field is absent from the
  config — only set it by hand to point at a fork, mirror, or private checkout).

**Resolve `harness_path`:**

1. If `<harness_path>/registry/guardrail-index.yaml` already exists, use it as-is.
   Never touch an existing checkout — it may be a fork, a pinned commit, or hand-edited.
2. If `<harness_path>` does not exist at all, fetch it once, automatically, before
   doing anything else:
   - Try `git clone --depth 1 <harness_repo_url> <harness_path>`.
   - If `git` isn't available or the clone fails (a network that resets
     `github.com`/`raw.githubusercontent.com` while still allowing
     `codeload.github.com` has been observed in practice), fall back to a zip
     download: rewrite the URL to its codeload form —
     `https://codeload.github.com/<owner>/<repo>/zip/refs/heads/<branch>` — download
     it, extract it, and move the extracted folder's contents into `<harness_path>`.
   - On success, report once, visibly (this is the first time the project touched the
     network on its own behalf, so say so rather than doing it silently):
     `[ai-dlc-compliance] auto-fetched the harness to <harness_path> from
     <harness_repo_url>`. Drop a marker file
     `<harness_path>/.ai-dlc-harness-autofetched` (fetch date + source URL) so it's
     obvious later that this folder was machine-fetched, not placed there by a human.
   - If both the clone and the zip fallback fail (offline, URL unreachable, or a
     private repo with no usable credentials), warn once and skip — same as
     "harness_path unresolvable" below. Never fail the calling command for this.
3. If `harness_path` now resolves (pre-existing or just auto-fetched) but
   `registry/guardrail-index.yaml` is still missing or unparsable, warn once and skip.

## Step 2 — Determine the lifecycle phase from the hook event

Map the hook event that triggered this command to a harness `lifecycle_phase`:

| Hook event | Harness phase | Config gate |
|---|---|---|
| `before_constitution` | `constitution` | `enabled_phases.constitution` |
| `after_plan` | `development` | `enabled_phases.development` |
| `after_tasks` | `development` | `enabled_phases.development` |
| `after_implement` | `review` | `enabled_phases.review` |

If the resolved phase's `enabled_phases.<phase>` is `false`, skip silently.

## Step 3 — Query the index, not the guardrail content

Load `<harness_path>/registry/guardrail-index.yaml` (small and flat — this is the
entire point of the index; never open every guardrail file to decide relevance).
Filter entries where:

1. `vertical == <this project's configured vertical>` OR `vertical == "cross-cutting"`
2. `lifecycle_phase == <resolved phase>` OR `lifecycle_phase == "cross-cutting"`

Derive a handful of task tags from the artifact the hook fired on (the constitution
draft, the plan, the task list, or the implemented diff) — feature keywords like
"connected", "network", "PHI", "patient data", "diagnostic", "research export", etc.
Prefer more tags over fewer; intersect them against each entry's
`applicability_tags`. If no tags clearly apply but the vertical+phase filter already
narrowed the set to a handful of entries, surface all of them — let the human judge
relevance from the `summary` field rather than silently dropping guardrails.

## Step 4 — Load only the matched guardrail files

Open only the `path` values of entries that survived Step 3 — typically 1-4 files,
never the whole vertical. This is the only point where full guardrail prose enters
context.

## Step 5 — Apply binding_tier

- **`binding`**: treat as a hard constraint on the artifact being checked. If the
  constitution/plan/tasks/implementation conflicts with it, or the guardrail's
  "Agent Guidance" `MUST` items aren't satisfied, report it as a blocking finding.
  If `block_on_binding: true` in the config, stop and ask the user to resolve each
  blocking finding before the calling command's result is treated as final.
- **`advisory`**: report it as a flagged note. Never block on these.

## Step 6 — Record evidence

`<feature-or-project-slug>` below is **exactly** the basename of `.specify/feature.json`'s
`feature_directory` field (e.g. `feature_directory: "specs/001-cgm-companion-app"` →
slug `001-cgm-companion-app`) — never a human-readable title or a slug derived from the
feature's name. Tooling that reads this evidence (e.g. a visual journey view) keys off
that same file, so a different slug here means the evidence silently never gets found.

Append one entry per matched guardrail to
`<evidence_dir>/<feature-or-project-slug>/guardrails-applied.md` (create the file with
a one-line header if it doesn't exist), capturing: guardrail id, hook event, date,
binding_tier, and whether it passed / was flagged / blocked. This is what turns "the
harness was available" into an actual audit trail — see the harness's
`governance/CONTENT-OWNERS.md` for why this matters.

Also write/update a machine-readable sidecar at the same path with a `.json`
extension — `<evidence_dir>/<feature-or-project-slug>/guardrails-applied.json` — a
JSON array, one object per matched guardrail, same information as the markdown
entry: `{"id", "title", "phase", "binding_tier", "applicability_tags", "finding",
"status": "passed"|"flagged"|"blocked", "timestamp"}`. Replace the whole array each
run with the current set of matched guardrails for this phase+slug (don't append
duplicates across runs for the same phase). This is what a visual tooling surface
reads instead of parsing the prose file — keep both in sync, but the JSON file is
the one that must stay strictly valid (no trailing commas, no comments).

**`finding` must say something specific to this feature, this run — never a
restatement of the guardrail's own `summary`.** "SaMD risk category must be
determined" is the guardrail's generic rule; "Classified as Class II SaMD per
IMDRF N12, recorded in constitution.md v1.1.0" is a finding. A visual tooling
surface renders this text as the direct answer to "why is this addressed/
flagged/blocked" — a generic finding gives a reviewer nothing beyond what the
status dot already told them.

Three more fields on each entry, all optional, all filled in only when genuinely
known — never fabricated or backfilled by guessing, same as everything else here:

- `"related_tasks"`: an array of task ids from `tasks.md` (e.g. `["T012", "T013"]`)
  that this guardrail's finding bears on. Populate this whenever `tasks.md` exists
  (so at `after_tasks` and `after_implement`, not `before_constitution`/`after_plan`
  — there's no task list yet at those hooks). A guardrail that genuinely applies to
  the whole feature rather than specific tasks (e.g. a constitution-phase guardrail
  like SaMD classification) should leave this empty, not forced onto some task.
- `"related_files"`: an array of file paths the finding bears on. Only populate
  this at `after_implement` — real files only exist by then; never guess a future
  file path at `after_tasks`.
- `"evidence_refs"`: an array of file paths (workspace-relative) or URLs that
  back `finding` beyond your own prose — the specific file/section that
  satisfies the guardrail, a test-run log you actually observed, a PR. This is
  what turns a finding from an assertion into something a reviewer can check
  themselves with one click. Only cite something that genuinely exists and
  genuinely supports the finding — never invent a plausible-looking path.

A visual tooling surface uses `related_tasks`/`related_files` to show "what
applied to this file/task" to a developer, instead of only "what applied to
this feature" to an architect, and renders `finding` + `evidence_refs`
directly on the card so a reviewer sees *why* a status was assigned without a
second click. Leave any of these as `[]` when the guardrail is feature-wide,
the attribution isn't knowable yet, or there's genuinely no artifact beyond
your own judgment to cite — an empty array is the honest answer far more often
than not.

## Step 6a — Emit the design plan graph (`after_plan` hook only)

Only when the hook event that fired this command is **`after_plan`**, and only when
Step 5 did **not** leave this run blocked on an unresolved binding guardrail (if
`plan.md` is a blocked-gate stub rather than a real plan, there is no real design to
graph yet — skip this step entirely and don't write anything).

Otherwise, write `<feature_directory>/flight-plan.json` (same `feature_directory` as
`.specify/feature.json`, already resolved in Step 6) — directly, from the plan you
just produced. This is **not** a call to any external tool or service: you already
did the reasoning to write `plan.md`; this just re-expresses it in a structured
format a visual tooling surface can render as a graph. If a `flight-plan.json`
already exists here (e.g. from an earlier run, or someone's own Flight Plan export),
overwrite it — `plan.md` is the current source of truth once `/speckit-plan` has run.

Shape (exactly, this is read by name-matched fields elsewhere):

```json
{
  "epic": { "code": "<short code>", "title": "<feature title>", "summary": "<1-2 sentences>", "approach": "<a short paragraph on sequencing/strategy>" },
  "brief": "<the feature's requirement text, e.g. spec.md's own summary>",
  "flags": { "phi": true|false, "regulations": ["<e.g. HIPAA, CMS-0057-F>"] },
  "requirements": [
    { "id": "R1", "text": "<requirement text>", "type": "functional|integration|compliance|non-functional", "ambiguity": 0.0, "sources": [] }
  ],
  "questions": [],
  "success": ["<measurable outcome, from plan.md/spec.md success criteria>"],
  "nodes": [
    {
      "id": "D1",
      "type": "domain_model|architecture|data_migration|backend|integration|ui|workflow|ai_ml|infra|reporting",
      "name": "<short component name>",
      "size": "S|M|L|XL",
      "risk": "low|med|high",
      "deps": ["<ids of nodes this depends on>"],
      "reqs": ["<requirement ids this node addresses>"],
      "inputs": ["<what it needs>"],
      "outputs": ["<what it produces>"],
      "checks": ["<how it's verified>"],
      "done": "<one-line done-when statement>",
      "task": "<one-paragraph description of the work>",
      "sources": []
    }
  ]
}
```

Only use the `node.type` values listed above (the design/build component types) —
not `intent`/`test_strategy`/`test_runner`/`security`/`compliance`/`performance`/
`release`/`evidence`, which are scaffolding Flight Plan's own tooling adds around
these, not something you author. `deps` must only reference earlier node ids (no
cycles). This step never fails the calling command — if the plan genuinely doesn't
have enough structure yet to produce a sensible graph, skip it with a one-line note
rather than emitting a thin or fabricated one.

## Step 6b — Record test verification status (`after_implement` hook only)

Only when the hook event that fired this command is **`after_implement`**. Write
`<evidence_dir>/<feature-or-project-slug>/test-results.json` (same slug rule as
Step 6) — a JSON array, one entry per Acceptance Scenario found in `spec.md`:

```json
{
  "user_story": "<the exact User Story heading text, e.g. \"User Story 1 - View glucose trend and get threshold alerts\">",
  "scenario_index": 2,
  "scenario_text": "<the scenario's own Given/When/Then text, verbatim>",
  "status": "passed" | "failed" | "not_verified" | "not_implemented",
  "note": "<why you gave it this status>",
  "evidence_refs": ["<optional: a test-run log path, a screenshot, a PR — whatever actually backs this status>"],
  "timestamp": "<ISO date>"
}
```

`evidence_refs` is optional, same field and same rule as Step 6's: only cite
something that genuinely exists and genuinely backs the status, never a
plausible-looking placeholder. It's what lets a reviewer check a `passed`
claim themselves instead of taking `note`'s word for it.

Reference scenarios by `user_story` + `scenario_index` + their own text — don't
invent scenario IDs. Speckit's own spec template doesn't number them with stable
ids, and this isn't the place to start a parallel numbering scheme Speckit itself
doesn't use; a human (or a view built on this file) can match scenario text back
to `spec.md` directly.

**Be conservative about the status — this is the one place in this harness where
overclaiming is worse than underclaiming:**

- **`passed`/`failed`**: use these ONLY when you directly observed a real test
  execution (a test suite run, an explicit manual verification reported to you)
  confirm or refute the scenario. Never infer these from reading the implementation
  and judging it "looks right."
- **`not_verified`**: the honest default when you reviewed the implementation and
  believe it satisfies the scenario, but no actual test execution happened or was
  reported to you. Say so plainly in `note` (e.g. "implementation reviewed and
  appears to satisfy this; no automated test run observed").
- **`not_implemented`**: the scenario's functionality doesn't appear to exist in
  the diff/codebase yet.

Never fail the calling command over this step — if `spec.md` has no parseable
Acceptance Scenarios, skip with a one-line note and write nothing.

## Step 7 — Report to the user

Summarize in this shape, then continue (or stop, per Step 5):

```
## Compliance guardrails checked — <phase> phase, vertical: <vertical>

**Binding** (must resolve before continuing):
- [GR-XXX-001] <title> — <one-line finding>

**Advisory** (flagged, not blocking):
- [GR-XXX-002] <title> — <one-line finding>

Evidence recorded: <evidence_dir>/<slug>/guardrails-applied.md
```

If nothing matched, say so in one line and continue — do not pad the output with
irrelevant guardrails.

## Graceful degradation

- No config file → skip silently (see Step 1).
- `harness_path` missing → auto-fetch it from `harness_repo_url` (see Step 1); only if
  that also fails do we warn once and skip, never failing the calling command.
- Harness `registry/guardrail-index.yaml` missing or unparsable even after a
  successful auto-fetch → warn once with the parse error, skip the check, never fail
  the calling command.
- This command only ever blocks on a `binding` finding when
  `block_on_binding: true` — it never fails the calling command for its own
  infrastructure problems.
- No `test-results.json` yet (nothing has reached `after_implement`, or `spec.md`
  had no parseable Acceptance Scenarios) → a QA-facing view simply shows nothing
  to check yet, same as `guardrails-applied.json` being absent — never a failure.
