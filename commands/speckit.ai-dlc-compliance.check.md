---
description: "Load only the AI-DLC harness guardrails relevant to the current vertical and lifecycle phase, and surface binding/advisory findings"
---

# Compliance Guardrail Check

This command is invoked as a hook before/after core Spec Kit commands. It loads
guardrails from the AI-DLC compliance harness — never the whole harness, only the
handful of entries that match this project's vertical and the current lifecycle
phase — and tells the user/agent what applies before they continue.

## Step 1 — Load config, degrade gracefully if absent

Read `.specify/extensions/ai-dlc-compliance/compliance-config.yml`.

- If it does not exist: skip silently with a one-line note
  (`[ai-dlc-compliance] no compliance-config.yml found, skipping guardrail check`).
  Do not block the calling command.
- If it exists but `harness_path` does not resolve to a real directory: warn once
  (don't fail the calling command) and skip.

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

Append one entry per matched guardrail to
`<evidence_dir>/<feature-or-project-slug>/guardrails-applied.md` (create the file with
a one-line header if it doesn't exist), capturing: guardrail id, hook event, date,
binding_tier, and whether it passed / was flagged / blocked. This is what turns "the
harness was available" into an actual audit trail — see the harness's
`governance/CONTENT-OWNERS.md` for why this matters.

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
- `harness_path` unresolvable → warn once, skip, never fail the calling command.
- Harness `registry/guardrail-index.yaml` missing or unparsable → warn once with the
  parse error, skip the check, never fail the calling command.
- This command only ever blocks on a `binding` finding when
  `block_on_binding: true` — it never fails the calling command for its own
  infrastructure problems.
