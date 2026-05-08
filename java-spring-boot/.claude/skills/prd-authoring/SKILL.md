---
name: prd-authoring
description: >-
  PRD format conventions, boundary rules, and template references.
  Load when writing or reviewing product requirements.
compatibility:
  - claude-code
  - opencode
  - github-copilot
metadata:
  version: "1.0"
  author: team
---

## PRD Location

The PRD lives at `docs/prd.md`.

## PRD Boundary Rule

The PRD describes *what* the system does. It must not contain *how*.

**Litmus test:** If it would change when switching from Java to another language, it belongs in `docs/system-design.md`, not the PRD.

When the PRD needs to reference implementation details:
```markdown
**Implementation:** See [system-design.md#section](system-design.md#section)
```

## Prohibited Patterns in PRD

| Pattern | Severity | Fix |
|---|---|---|
| Java code blocks (` ```java `) | Critical | Move to system-design.md, link from PRD |
| Java-specific constructs (annotations, streams, lambdas, Spring APIs) | Critical | Describe behavior, not mechanism |
| Internal code references (class names, method names, variable names) | High | Use behavioral language |
| Algorithm formulas or pseudocode | High | State behavioral constraints, move formulas to system-design.md |
| Regex patterns | High | Describe behavior, not mechanism |
| Hardcoded constant values | Medium | Reference system-design.md#constants |

## Requirement Format

Use the "Parseable Section Templates" requirement format in `docs/documentation.md`.

## Feature Handoff Record (PRE → SDE)

When a feature is approved, append one record to `.scratch/handoff.jsonl` describing the scope. The record is the structured contract that system-design-expert consumes; the markdown PRD entry in `docs/prd.md` remains the human-authored source of truth.

**File:** `.scratch/handoff.jsonl` (append-only; one JSON object per line, terminated by `\n`).

**Schema:** [`schemas/scratch/prd-entry.schema.json`](../../../schemas/scratch/prd-entry.schema.json). The pipeline-coordinator validates each record against the schema at the PRE→SDE transition; malformed records bounce back to you without consuming an SDE dispatch.

**Required fields:**

| Field | Type | Notes |
|---|---|---|
| `type` | `"prd-entry"` | Discriminator. |
| `req_id` | string `^REQ-[A-Z]+-[0-9]{3}$` | Matches the PRD heading. |
| `ts` | ISO 8601 string | Timestamp at append. |
| `author` | `"product-requirements-expert"` | Pinned. |
| `title` | string | Short requirement title. |
| `summary` | string ≤ 400 chars | One- or two-sentence statement. No implementation details. |
| `acceptance_criteria` | array of strings | Testable conditions. At least one. |
| `file_targets` | array of strings | Paths likely to be touched. At least one. Best-effort; SDE may revise. |
| `test_names` | array of strings matching `^[a-z][A-Za-z0-9_]*$` | JUnit test methods expected to exist (camelCase, e.g. `theResultShouldContainNewItems`). At least one. |

**Optional fields:** `non_goals`, `dependencies` (other req_ids), `notes`.

**Append-only discipline:** Read the file first if it exists. Preserve every prior line verbatim. Append your new record as the last line. Never edit, reorder, or delete prior records. If a prior record has a mistake, append a new record that supersedes it.

**Why JSONL, not markdown:** see [`docs/adr/2026-05-08-append-only-jsonl-handoffs.md`](../../../docs/adr/2026-05-08-append-only-jsonl-handoffs.md). The structural-rejection gate this enables converts SDE retries into cheap upstream bounces before a Sonnet/Opus dispatch is consumed.

### Example Record

```json
{"type":"prd-entry","req_id":"REQ-XX-099","ts":"2026-05-08T12:00:00Z","author":"product-requirements-expert","title":"Cache miss diagnostics","summary":"Surface per-component cache miss rate so operators can spot reinjection hotspots.","acceptance_criteria":["report renders cacheMissRate per component","value matches cacheCreation/(cacheCreation+cacheRead)"],"file_targets":["src/main/java/com/example/reference/report/SummaryReport.java","src/test/java/com/example/reference/report/SummaryReportTest.java"],"test_names":["theSummaryShouldRenderCacheMissRate"],"non_goals":["historical trend"]}
```

## Writing Standards

Follow the Writing Standards section in `docs/documentation.md`.
