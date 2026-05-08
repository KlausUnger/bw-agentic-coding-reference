---
name: review-checklist
description: >-
  Review process overview, feedback tag definitions, and output format.
  Load when conducting or processing code reviews.
compatibility:
  - claude-code
  - opencode
  - github-copilot
metadata:
  version: "1.0"
  author: team
---

## Review Phase

After the feature-implementer appends a `build-pass` record to `.scratch/handoff.jsonl`, invoke all four reviewers in parallel:

| Reviewer | `author` value | Focus |
|---|---|---|
| code-quality-reviewer | `"code-quality-reviewer"` | Readability, Java style |
| test-reviewer | `"test-reviewer"` | Test pyramid, coverage, edge cases |
| security-reviewer | `"security-reviewer"` | OWASP, vulnerabilities |
| doc-reviewer | `"doc-reviewer"` | Documentation coherence, structure |

Each reviewer appends one `review-feedback` record. Schema: [`schemas/scratch/review-feedback.schema.json`](../../../schemas/scratch/review-feedback.schema.json).

## Output Protocol (Reviewers)

Your sole deliverable is the appended `review-feedback` record. The pipeline cannot proceed without it.

1. **Read** `.scratch/handoff.jsonl` first. If the file does not exist, the implementer has not signalled gate-pass — abort and report the missing precondition.
2. **Append one line** to `.scratch/handoff.jsonl`: a single JSON object conforming to the `review-feedback` schema. Required fields: `type` (`"review-feedback"`), `req_id`, `ts`, `author` (your reviewer name), `verdict` (`approved` | `changes_requested` | `blocked`), `findings` (array, possibly empty when `verdict: "approved"`).
3. Each finding requires `tag`, `location`, `description`. Add `fix` for `tag: "autofix"`. Add `clarify_target` for `tag: "clarify"`. Severity is optional.
4. **Preserve every prior line verbatim** when appending. Append-only is non-negotiable.
5. **Verify** the append succeeded by reading the file back and confirming your record is the last line.
6. Your reply to the caller MUST be exactly one line: `Appended review-feedback (<verdict>) for <req_id>`.
7. Do NOT include review content, summaries, or analysis in your reply. The caller reads the record.

**Why:** when review content lands in the reply instead of the file, the dispatcher cannot route fixes, artifact-owner agents cannot read findings, and the audit trail is lost. Stopping before the append forces the user to re-run the review — this is a recurring reviewer failure mode.

### Example Record

```json
{"type":"review-feedback","req_id":"REQ-XX-099","ts":"2026-05-08T16:30:00Z","author":"code-quality-reviewer","verdict":"changes_requested","findings":[{"tag":"autofix","location":"src/main/java/com/example/reference/report/SummaryReport.java:142","description":"Variable name `r` shadows the field `r`.","fix":"Rename loop variable to `row`."},{"tag":"blocked","location":"src/main/java/com/example/reference/report/SummaryReport.java:160","description":"Possible ArithmeticException: divide by zero when cacheEligibleTokenCount is 0.","severity":"critical"}],"approved_aspects":["Test naming follows project convention","Exceptions wrapped with cause"]}
```

## Feedback Tags

| Tag | Meaning | Action |
|---|---|---|
| `autofix` | Clear fix, no decision needed | Route to artifact owner |
| `blocked` | Critical issue, must fix before merge | Route to artifact owner; escalate if unclear |
| `escalate` | Needs human decision | Append to `.scratch/escalations.md` |
| `clarify` (with `clarify_target`) | Requirement, design, or review question | Route to the named agent |

## Artifact Ownership

Review feedback targets the artifact, not a fixed agent. Route fixes to the owning agent:

| Artifact | Owner Agent |
|---|---|
| `docs/prd.md` | product-requirements-expert |
| `docs/system-design.md`, `docs/adr/*.md` | system-design-expert |
| `src/main/java/**/*.java` | feature-implementer |
| `src/test/java/**/*.java` | feature-implementer |
| Resource files, templates | feature-implementer |

Do not bundle doc fixes into a feature-implementer call. Do not send code fixes to doc agents.

## Issue Classification

| Checklist Category | Default Severity | Tag |
|--------------------|-----------------|-----|
| Cross-document coherence | Critical | `blocked` |
| PRD boundary violations (Java code, class references, internal code names) | Critical | `blocked` |
| Security vulnerabilities (CRITICAL/HIGH per `security-review` skill) | Critical | `blocked` |
| Structural issues (missing anchors, broken links) | Fixable | `autofix` |
| Writing standards | Fixable | `autofix` |

## Processing Reviews

After all reviewers complete:

0. Verify each of the four reviewers has appended a `review-feedback` record for the current `req_id` since the latest `build-pass`. For each missing record, re-dispatch the corresponding reviewer ONCE with this prompt: `"Your previous run returned without appending a review-feedback record to .scratch/handoff.jsonl. Run the review now. Your only deliverable is that record — see Output Protocol in review-checklist."` If a record is still missing after the retry, append an entry to `.scratch/escalations.md` naming the reviewer and stop — do not proceed to step 1.
1. feature-implementer reads all four `review-feedback` records (latest per reviewer for the active `req_id`).
2. `tag: "autofix"` findings: fix immediately using the `fix` field.
3. `tag: "blocked"` findings: fix immediately; escalate if fix is unclear.
4. `tag: "escalate"` findings: append the description to `.scratch/escalations.md`.
5. `tag: "clarify"` findings: request clarification from the agent named in `clarify_target`.
6. (No consolidated summary file needed; the four `review-feedback` records are the canonical record.)
7. If all four `verdict` values are `"approved"`, feature is complete.
8. If any `verdict` is `"changes_requested"` or `"blocked"`, re-run the quality gate (append fresh `build-failure`/`build-pass` records) and re-invoke reviewers.
