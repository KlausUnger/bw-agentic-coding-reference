---
name: pipeline-handoff
description: >-
  Pipeline routing rules and handoff conditions between specialist agents.
  Load when coordinating feature delivery, checking pipeline state,
  or determining which agent to invoke next.
compatibility:
  - claude-code
  - opencode
  - github-copilot
metadata:
  version: "1.0"
  author: team
---

## Agent Selection

| User Request | Agent | Shortcut Allowed |
|---|---|---|
| New feature or enhancement | product-requirements-expert | No — full pipeline required |
| Discuss or explore feature idea | product-requirements-expert | Yes — single agent |
| Requirement clarification | product-requirements-expert | Yes — single agent |
| Architecture question | system-design-expert | Yes — single agent |
| Bug fix (known cause) | feature-implementer | Yes — skip PRD/design |
| Code review request | All four reviewers | Yes — parallel invocation |

**Skip agents for:** git operations, answering questions, running commands, reviewing already-completed changes.

## Handoff Conditions

All transitions are gated on the latest record per `(req_id, type)` in `.scratch/handoff.jsonl`. The Validation Gates section below defines each gate's structural checks.

| Current Agent | Trigger | Next Agent |
|---|---|---|
| product-requirements-expert | latest `prd-entry` record passes the Validation Gate | system-design-expert |
| system-design-expert | latest `design-block` record has `verdict: "approved"` or `"revised"` and passes the Validation Gate | feature-implementer |
| feature-implementer | latest `build-pass` record exists and post-dates any `build-failure` for the same `req_id` | All reviewers (parallel) |
| feature-implementer | latest `build-failure` record has `retry < 3` | feature-implementer (retry with error context) |
| feature-implementer | latest `build-failure` record has `retry == 3` | system-design-expert (escalation) |
| All four reviewers | each reviewer's latest `review-feedback` record has `verdict: "approved"` | Feature complete |
| Any reviewer | latest `review-feedback` record has `verdict: "changes_requested"` or `"blocked"` with non-empty findings | feature-implementer (process findings) |

## Validation Gates

Each agent transition validates the inbound record(s) against a schema before dispatching the next specialist. Malformed or missing records bounce back to the upstream agent without consuming a Sonnet/Opus dispatch — see ADR [`2026-05-08-append-only-jsonl-handoffs`](../../../docs/adr/2026-05-08-append-only-jsonl-handoffs.md) for rationale.

### Common Procedure

1. **Discover.** Run `Glob .scratch/**/*` to enumerate state files. Then `Read .scratch/handoff.jsonl` only if it appears in the Glob result. Do not `Read` directories.
2. **Gate.** If `handoff.jsonl` is missing or empty when a record is required, the gate fails — route back to the upstream agent.
3. **Filter.** Records by `req_id` and `type`. The **latest** record for each `(req_id, type)` is the active state.
4. **Check.** Required fields, types, and pattern constraints per the schema.
5. **Decide.** If every check passes: dispatch the next agent. If any check fails: route back upstream with a `Blocked` recommendation naming the specific failed check.

### Gate 1: PRE → SDE (`prd-entry`)

Schema: [`schemas/scratch/prd-entry.schema.json`](../../../schemas/scratch/prd-entry.schema.json). Required checks:

- `type == "prd-entry"`, `author == "product-requirements-expert"`.
- `req_id` matches `^REQ-[A-Z]+-[0-9]{3}$`. `ts` is a non-empty ISO 8601 string.
- `title`, `summary` are non-empty strings.
- `acceptance_criteria`, `file_targets`, `test_names` are non-empty arrays of non-empty strings.
- Each `test_names` entry matches `^Test[A-Z]`.

### Gate 2: SDE → implementer (`design-block`)

Schema: [`schemas/scratch/design-block.schema.json`](../../../schemas/scratch/design-block.schema.json). Required checks:

- `type == "design-block"`, `author == "system-design-expert"`, valid `req_id` and `ts`.
- `verdict` is one of: `approved`, `needs_changes`, `blocked`, `revised`, `escalated`.
- `architectural_fit` is non-empty; `primary_paths` is a non-empty array of non-empty strings.
- When `verdict == "escalated"`: `escalations` array is present and non-empty.
- When `verdict == "revised"`: `supersedes_record_at` is present and points to a prior `design-block` record line in the file.

Routing:

- `approved` or `revised` → dispatch feature-implementer. (`revised` resets the build-failure retry counter for that `req_id`.)
- `needs_changes` or `blocked` → bounce back; PRE may need to refine, or SDE may need to re-run.
- `escalated` → stop the pipeline; human decides.

### Gate 3: implementer → reviewers (`build-pass`)

Schema: [`schemas/scratch/build-pass.schema.json`](../../../schemas/scratch/build-pass.schema.json). Required checks:

- The latest `build-*` record for `req_id` is `type == "build-pass"`.
- `author == "feature-implementer"`, valid `req_id` and `ts`.

If the latest is a `build-failure`, apply Build-Failure Recovery instead.

### Gate 4: reviewers → next step (`review-feedback`)

Schema: [`schemas/scratch/review-feedback.schema.json`](../../../schemas/scratch/review-feedback.schema.json). For each of the four reviewers, the latest `review-feedback` record (filtered by `req_id` and `author`) must:

- Have `type == "review-feedback"`, valid `req_id` and `ts`.
- `author` is one of: `code-quality-reviewer`, `test-reviewer`, `security-reviewer`, `doc-reviewer`.
- `verdict` is one of: `approved`, `changes_requested`, `blocked`.
- `findings` is an array; when `verdict != "approved"`, it should be non-empty (warn but do not hard-fail; an empty findings list with a non-approved verdict means the reviewer did not produce actionable output and should be re-dispatched).
- Each finding has `tag`, `location`, `description`. When `tag == "clarify"`, `clarify_target` is required.

Routing:

- All four `verdict == "approved"` → feature complete; load `feature-eval` skill.
- Any `verdict == "changes_requested"` or `"blocked"` → split the union of findings by artifact owner (see `review-checklist` § Artifact Ownership), then dispatch each owner agent with the relevant slice. **Exception:** `tag == "autofix"` findings whose `location` is a design-doc path (`docs/system-design.md` or `docs/adr/*.md`) are applied by root directly per `review-checklist` § Root-Applied Autofix on Design Docs — they do NOT redispatch system-design-expert. Every other finding on those paths still routes to SDE.
- Any `tag == "escalate"` finding → also append an entry to `.scratch/escalations.md`.

### What the gates do NOT check

- Content quality (are the acceptance criteria *good*? are the findings *correct*?). That is the consuming agent's judgement.
- Cross-record consistency beyond `req_id` linkage (e.g. whether `design-block.primary_paths` overlaps `prd-entry.file_targets`). Consumers may surface mismatches as findings; gates do not.

The gates are structural: required fields present, types correct, patterns match. Every check must catch deterministically.

## Blocking

If any gate fails, or any record carries `verdict: "blocked"` or `"escalated"`, stop the pipeline and resolve before continuing.

## Build-Failure Recovery

When the feature-implementer runs the quality gate and it fails (build error, test failure, lint failure), the implementer appends a `build-failure` record to `.scratch/handoff.jsonl` with the error output and retry count, then exits. Schema: [`schemas/scratch/build-failure.schema.json`](../../../schemas/scratch/build-failure.schema.json).

### Coordinator retry logic

1. Read `.scratch/handoff.jsonl`. Take the latest `build-failure` record for the active `req_id`.
2. If `retry < 3`, route back to feature-implementer with this prompt context:
   - The latest `build-failure` record (the error output).
   - The latest `design-block` record (the original design).
   - `.scratch/implementation-plan.md` (what was planned).
   - Instruction: "Fix the build failure described in the latest `build-failure` record. This is retry N of 3."
3. If `retry == 3`, escalate to system-design-expert with this prompt context:
   - All `build-failure` records for the active `req_id` since the latest `design-block` (the failure trail).
   - The latest `design-block` record.
   - Instruction: "The implementer failed 3 times. Review whether the design needs revision."
   - The design expert appends a new `design-block` record with `verdict: "revised"` (and `supersedes_record_at`) or `verdict: "escalated"`.
4. After a design revision (`verdict: "revised"`), the retry counter resets — the next `build-failure` record starts at `retry: 1`.

### Retry rules

- The implementer increments `retry` in each new `build-failure` record (1, 2, 3). Compute the next value by counting `build-failure` records for the active `req_id` appended *after* the latest `design-block` line, then setting `retry = count + 1`. The first failure after a fresh `design-block` (whether `verdict: "approved"` or `"revised"`) is `retry: 1`. Append-only — never edit a prior record.
- On success, the implementer appends a `build-pass` record. Prior `build-failure` records remain in the file as the diagnostic retry trail.
- The coordinator never modifies records — it only reads them for routing decisions.
- Maximum 3 retries per design cycle. A new `design-block` with `verdict: "revised"` starts a fresh cycle.

## Mid-Implementation Feedback

The feature-implementer may invoke product-requirements-expert or system-design-expert during TDD cycles when tests uncover requirement gaps or design needs. These are not handoffs. The implementer continues with other cycles while waiting.

## Review Feedback Actions

See the `review-checklist` skill for feedback tag definitions and the review process.

## State Files

| File | Created By | Consumed By |
|---|---|---|
| `.scratch/handoff.jsonl` | product-requirements-expert, system-design-expert, feature-implementer, four reviewers, root (all append-only) | coordinator (validation gates), all consumer agents |
| `.scratch/implementation-plan.md` | feature-implementer | feature-implementer (self-tracking) |
| `.scratch/escalations.md` | feature-implementer | Human |
| `.scratch/eval-*.md` | coordinator (via feature-eval skill) | Human |

`.scratch/handoff.jsonl` is the append-only structured handoff log; one JSON object per line, each carrying a `type` discriminator. Record types:

| Record `type` | Producer | Purpose |
|---|---|---|
| `prd-entry` | product-requirements-expert | Active feature scope for SDE and implementer. |
| `design-block` | system-design-expert | Architectural fit and implementation guidance. |
| `review-feedback` | each of the four reviewer agents | Per-reviewer verdict and findings. |
| `build-failure` | feature-implementer | Quality-gate failure with error context and retry counter. |
| `build-pass` | feature-implementer | Quality-gate success marker. |
| `design-doc-autofix` | root (coordinator) | Audit trail for root-applied autofixes on design-doc paths (see `review-checklist` § Root-Applied Autofix on Design Docs). |

## Human Checkpoints

The human approves at these points:

1. **After PRD update** — Confirm requirement captures intent.
2. **After design notes** — Confirm architectural approach.
3. **After escalations** — Decide on `[ESCALATE]` items.
4. **After feature complete** — Final approval before merge.

## Coordinator Output Format

The pipeline coordinator responds with a structured recommendation:

```
## Pipeline State
[Current state based on .scratch/ files]

## Recommendation
**Action:** Invoke [agent-name]
**Prompt:** "[suggested prompt for the agent]"
**Shortcut:** Yes/No
**Reason:** [why this agent is next]
```

If blocked:
```
## Pipeline State
[Current state]

## Blocked
**Blocker:** [description]
**Resolution:** [what needs to happen]
```

## Coordinator Rules

1. Never skip pipeline stages for new features.
2. Shortcuts are allowed only per the agent selection table above.
3. If `.scratch/` contains stale state from a previous feature, recommend clearing it first.
4. Report all `verdict: "escalated"` records and `tag: "escalate"` findings.
5. If the latest `build-*` record for the active `req_id` is a `build-failure`, apply the retry logic in the "Build-Failure Recovery" section.
6. After all four reviewers' latest `review-feedback` verdicts are `"approved"`, load the `feature-eval` skill and write the evaluation scorecard.

## Pipeline Flow

```
User Request
    |
    v
Pipeline Coordinator (classifies request, validates latest handoff.jsonl records)
    |
    +--- New feature ------> product-requirements-expert
    |                              | (appends prd-entry record)
    |                              v
    |                        system-design-expert
    |                              | (appends design-block, verdict: approved | revised)
    |                              v
    |                        feature-implementer
    |                              | (appends build-failure or build-pass)
    |                     +--------+--------+
    |                     |                 |
    |                     v (build-pass)    v (build-failure, retry < 3)
    |               All reviewers      feature-implementer
    |                  (parallel)       (retry with error context)
    |                     |                 |
    |                     |                 v (build-failure, retry == 3)
    |                     |           system-design-expert
    |                     |           (verdict: revised or escalated)
    |                     |                 |
    |                     |                 v (verdict: revised, retry reset)
    |                     |           feature-implementer
    |                     |
    |                     v (all four review-feedback verdicts: approved)
    |               Feature eval → .scratch/eval-<name>.md
    |                     |
    |                     v
    |               Feature complete
    |
    +--- Bug fix (known) --> feature-implementer (shortcut)
    +--- Architecture Q ---> system-design-expert (single agent)
    +--- Code review ------> All reviewers (parallel)
```
