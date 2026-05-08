---
name: design-validation
description: >-
  Architectural validation checklist for feature approval.
  Load when validating that features fit into the existing architecture.
compatibility:
  - claude-code
  - opencode
  - github-copilot
metadata:
  version: "1.0"
  author: team
---

## Input Contract

The active feature scope arrives as a `type: "prd-entry"` record in `.scratch/handoff.jsonl` (one JSON object per line, append-only). Schema: [`schemas/scratch/prd-entry.schema.json`](../../../schemas/scratch/prd-entry.schema.json).

**Read discipline:**

1. Read `.scratch/handoff.jsonl`. Take the last `type: "prd-entry"` record as the active scope.
2. The pipeline-coordinator validates the record against the schema before dispatching you; you may assume the required fields are present and well-typed. If the record is missing or fails a sanity check (e.g. `req_id` does not match the PRD), append a `design-block` record with `verdict: "escalated"` rather than papering over malformed input.
3. Use `acceptance_criteria`, `file_targets`, and `test_names` from the record verbatim when producing your `design-block` record. Do not re-derive them; that is the rework loop the JSONL handoff exists to break.
4. If a required structural field is missing or contradicts the PRD, the right action is to bounce back via `verdict: "needs_changes"` (or `"escalated"` for unresolvable conflicts), not to fill in guesses.

**Forbidden:** re-reading `docs/prd.md` to reconstruct scope when the prd-entry record is present. The record is the contract.

## Output Contract

Append one `design-block` record to `.scratch/handoff.jsonl` per dispatch. Schema: [`schemas/scratch/design-block.schema.json`](../../../schemas/scratch/design-block.schema.json).

**Required fields:**

| Field | Type | Notes |
|---|---|---|
| `type` | `"design-block"` | Discriminator. |
| `req_id` | string `^REQ-[A-Z]+-[0-9]{3}$` | Same as the prd-entry being implemented. |
| `ts` | ISO 8601 string | Timestamp at append. |
| `author` | `"system-design-expert"` | Pinned. |
| `verdict` | enum | `approved` (initial design ready), `needs_changes` (PRE/SDE rework needed), `blocked` (cannot proceed), `revised` (design updated after build-failure escalation), `escalated` (human decision needed). |
| `architectural_fit` | string | How the feature integrates with `docs/system-design.md`. |
| `primary_paths` | array of paths | At least one. The starting target set for the implementer. |

**Optional fields:** `supporting_paths`, `integration_points`, `patterns` (each `{ref, description}`), `risks` (each `{risk, mitigation}`), `escalations` (required when `verdict == "escalated"`), `supersedes_record_at` (line number of the prior design-block this revision supersedes; required when `verdict == "revised"`), `notes`.

**Append-only discipline:** Read `.scratch/handoff.jsonl` first. Preserve every prior line verbatim. Append your record as the last line, terminated by `\n`. Never edit, reorder, or delete prior records — `supersedes_record_at` is how you correct a prior decision.

### Example Record

```json
{"type":"design-block","req_id":"REQ-XX-099","ts":"2026-05-08T14:00:00Z","author":"system-design-expert","verdict":"approved","architectural_fit":"Cache miss diagnostics live in the report layer alongside existing per-agent rates; no new package, extends internal/report/summary.go.","primary_paths":["internal/report/summary.go","internal/report/summary_test.go"],"supporting_paths":["internal/cache/measure.go"],"integration_points":["summary report row gains a cache_miss_rate column derived from internal/cache/measure"],"patterns":[{"ref":"internal/report/summary.go:120","description":"existing per-agent rate computation pattern"}],"risks":[{"risk":"divisor zero when cache_eligible_token_count is 0","mitigation":"emit null with insufficient_data flag"}]}
```

## Design Principles

Apply these principles when evaluating features:

1. **Security and reliability are emergent** — must be designed in, not retrofitted.
2. **Consistency over novelty** — match existing patterns unless there is a compelling reason.
3. **Explicit dependencies** — every integration point documented.
4. **Layer respect** — features belong in appropriate architectural layers.
5. **Minimal surface** — prefer internal packages.
6. **Understandable systems** — if it cannot be reasoned about, it cannot be secured.
7. **Fail secure** — errors leave the system in a safe state.

## Validation Checklist

Before approving a feature for implementation:

### Architectural Fit
- [ ] Feature aligns with project goals
- [ ] Feature not in Non-Goals or Out of Scope
- [ ] Package placement follows existing `internal/` structure
- [ ] Error handling matches `fmt.Errorf("context: %w", err)` pattern
- [ ] New types follow existing naming conventions
- [ ] No circular dependencies between packages
- [ ] Integration points identified
- [ ] New dependencies from approved sources (see `docs/system-design.md`); ADR required for exceptions

### DDD Alignment

See `docs/ddd-principles.md` for full principles.

- [ ] Value objects are immutable with no framework dependencies
- [ ] Aggregates enforce their own invariants
- [ ] Data mappers are stateless and pure at all boundaries
- [ ] One aggregate per package
- [ ] Dependencies flow inward (infrastructure → service → domain)
- [ ] `make deps-check` passes

### Security by Design
- [ ] Credentials handled per existing patterns (config, not hardcoded)
- [ ] Input validation specified
- [ ] Error messages don't leak sensitive data
- [ ] Logging follows redaction patterns
- [ ] Network operations use TLS

### Reliability by Design
- [ ] Failure modes enumerated
- [ ] Timeouts specified for all blocking operations
- [ ] Resource limits defined (buffers, connections)
- [ ] Graceful shutdown behavior specified
- [ ] Context cancellation propagated

### Understandability
- [ ] Component can be understood in isolation
- [ ] State changes are explicit
- [ ] Interfaces are minimal and typed
- [ ] No implicit dependencies
