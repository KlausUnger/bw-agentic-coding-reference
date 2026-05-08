# Agent Team

Agent definitions for reference. Each agent has a specific role in the feature development pipeline.

## Architecture

**Agents own behavior.** Each agent is a thin wrapper: persona, tool permissions, model selection. Domain expertise stays in the agent definition.

**Skills own knowledge.** Portable workflow logic lives in `.claude/skills/`. All three tools (Claude Code, OpenCode, GitHub Copilot) read skills from this location.

**Project docs own truth.** Requirements (`docs/prd.md`), architecture (`docs/system-design.md`), and decisions (`docs/adr/`) are the authoritative sources.

## Agents

| Agent | Role | Model | Outputs |
|-------|------|-------|---------|
| **pipeline-coordinator** | Classify requests, check state, route to agents | Sonnet | Routing recommendations |
| **product-requirements-expert** | Define and clarify feature requirements | Opus | `docs/prd.md`, `.scratch/handoff.jsonl` (`prd-entry` record) |
| **system-design-expert** | Validate architectural fit | Opus | `docs/system-design.md`, `.scratch/handoff.jsonl` (`design-block` record) |
| **feature-implementer** | TDD/DDD implementation | Opus | Code, tests, `cmd/config.example.yaml`, `.scratch/handoff.jsonl` (`build-failure` / `build-pass` records), `.scratch/implementation-plan.md` |
| **code-quality-reviewer** | Readability, Go style guide | Sonnet | `.scratch/handoff.jsonl` (`review-feedback` record, `author: "code-quality-reviewer"`) |
| **test-reviewer** | Test pyramid, coverage | Sonnet | `.scratch/handoff.jsonl` (`review-feedback` record, `author: "test-reviewer"`) |
| **security-reviewer** | OWASP, vulnerabilities | Sonnet | `.scratch/handoff.jsonl` (`review-feedback` record, `author: "security-reviewer"`) |
| **doc-reviewer** | Doc coherence, structure, writing | Sonnet | `.scratch/handoff.jsonl` (`review-feedback` record, `author: "doc-reviewer"`) |

## Skills

Pipeline routing, quality gates, and templates live in portable skills:

| Skill | Purpose | Used By |
|-------|---------|---------|
| `pipeline-handoff` | Routing table, handoff conditions, blocking rules | pipeline-coordinator |
| `prd-authoring` | PRD format, boundary rules, requirement template | product-requirements-expert |
| `tdd-workflow` | TDD cycle process, design-check decision tree, document ownership | feature-implementer |
| `code-quality-gate` | Build/test/lint requirements, completion criteria | feature-implementer, reviewers |
| `review-checklist` | Feedback tags, issue classification, review output format, review process | All reviewers, feature-implementer |
| `code-quality-review` | Go code quality checklist (Google Go Style Guide) | code-quality-reviewer |
| `test-review` | Test quality checklist, security testing, dynamic analysis | test-reviewer |
| `security-review` | Security checklists, threat model, severity, supply chain verification | security-reviewer |
| `design-validation` | Architectural validation checklist for feature approval | system-design-expert |
| `adr-template` | ADR format, naming conventions | system-design-expert |
| `new-feature` | Clear scratch directory, start fresh context | pipeline-coordinator |
| `audit-agents` | Audit agent config for consistency, coherence, cross-tool parity | Human / any agent |
| `feature-eval` | Score completed features: tests, reviews, retry count | pipeline-coordinator |
| `doc-review` | Documentation review checklist, validation categories, review process | doc-reviewer |
| `doc-sync` | Synchronize documentation with codebase after implementation | Human / any agent |
| `seed` | Push template into a downstream project (init + upgrade modes) | Human / any agent |
| `harvest` | Pull generalizable improvements from a downstream project back into the template | Human / any agent |
| `lint-docs` | On-demand documentation validation | Human / any agent |
| `ship` | Commit staged changes and push to remote in one step | Human / any agent |
| `next` | Reset scratch, recommend next PRD requirement to implement | Human / any agent |

## When to Use Each Agent

| Scenario | Agent | Why |
|----------|-------|-----|
| "Add user authentication" | **pipeline-coordinator** | New feature needs full pipeline |
| "Does REQ-XX-003 cover edge cases?" | **product-requirements-expert** | Requirement clarification (shortcut) |
| "Where should the retry logic live?" | **system-design-expert** | Architectural decision (shortcut) |
| "Implement REQ-XX-001" | **feature-implementer** | Clear requirement, ready to build |
| "Fix the connection timeout bug" | **feature-implementer** | Bug with known location (shortcut) |
| "Review my PR" | All four reviewers | Parallel review invocation |

For the full routing table, see the `pipeline-handoff` skill.

## Cross-Tool Compatibility

This workflow targets three tools: Claude Code (primary), OpenCode (experimental), GitHub Copilot.

### Rules

1. **No `AGENTS.md` file.** `CLAUDE.md` is the single rules file. All three tools read it. An `AGENTS.md` causes OpenCode to stop reading `CLAUDE.md`.
2. **Skills in `.claude/skills/` only.** This is the only location all three tools discover. Do not create `.opencode/skills/` or `.github/skills/`.
3. **Agent definitions are tool-specific.** Claude Code agents live in `.claude/agents/`. OpenCode equivalents go in `.opencode/agents/`. Copilot equivalents go in `.github/agents/`. Do not try to make agent files portable.
4. **Project docs are tool-agnostic.** `docs/` is read by all tools with no special discovery. Keep requirements, architecture, and ADRs here.
5. **Pipeline state is tool-agnostic.** `.scratch/handoff.jsonl` is append-only JSONL (one JSON record per line) and other `.scratch/` markdown helpers use plain text. Any tool can read and write them.

### What Each Tool Reads

| Location | Claude Code | OpenCode | Copilot |
|----------|:-----------:|:--------:|:-------:|
| `CLAUDE.md` | Yes | Yes (if no AGENTS.md) | Yes |
| `.claude/skills/*/SKILL.md` | Yes | Yes | Yes |
| `.claude/agents/*.md` | Yes | No | No |
| `.opencode/agents/*.md` | No | Yes | No |
| `.github/agents/*.agent.md` | No | No | Yes |
| `docs/` | Yes | Yes | Yes |
| `.scratch/` | Yes | Yes | Yes |

### Adding OpenCode Support

1. Create `.opencode/agents/` with translated agent definitions (different YAML format).
2. Do not create `AGENTS.md`.
3. Skills and docs work without changes.

### Adding Copilot Support

1. Create `.github/agents/` with `.agent.md` files (different YAML format).
2. Skills and docs work without changes.
3. Do not create `.github/copilot-instructions.md` — Copilot CLI reads `CLAUDE.md` natively.

## Maturity Levels

The pipeline follows a maturity progression. Each level builds on the previous.

| Level | Name | Status | How It Works |
|-------|------|--------|-------------|
| 1 | Manual Pipeline | Superseded | User invokes each agent, checks `.scratch/`, triggers next agent manually |
| 2 | Coordinator + Skills | **Current** | Coordinator agent reads state and routes. Skills carry workflow logic. User reviews between stages |
| 3 | Parallel Reviewers | Available | Coordinator spawns all four reviewers as parallel subagents. Each appends a `review-feedback` record to `.scratch/handoff.jsonl` independently |
| 4 | Agent Teams | Experimental | Reviewers run as an Agent Team with peer-to-peer messaging. Claude Code only, Opus model, ~5x token cost |
| 5 | Full Team Orchestration | Future | Entire pipeline runs as coordinated team. Blocked by: experimental status, single-model constraint, no cross-tool support |

### Progression Guidance

- **Stay at Level 2-3** until Agent Teams exits experimental status.
- The file-based state machine (`.scratch/`) is more portable, transparent, and reliable than Agent Teams for sequential pipelines.
- When ready for Level 4: enable Agent Teams for the review phase first (lowest risk, highest value from cross-referencing findings).
- Keep the file-based handoff system as the coordination backbone at all levels.

## Scratch Directory

The `.scratch/` directory holds temporary files for the current feature cycle. It is git-ignored. Delete all files after feature merge.

### Structure

```
.scratch/
├── handoff.jsonl             # Append-only structured handoff log (all agents)
├── implementation-plan.md    # TDD cycle plan (from feature-implementer)
├── escalations.md            # Items requiring human decision
├── eval-<req-id>.md          # Feature evaluation scorecard
└── tmp/                      # Intermediate computation files (auto-cleaned)
```

`handoff.jsonl` carries five record types, one JSON object per line:

| Record `type` | Producer | Schema |
|---|---|---|
| `prd-entry` | product-requirements-expert | `schemas/scratch/prd-entry.schema.json` |
| `design-block` | system-design-expert | `schemas/scratch/design-block.schema.json` |
| `build-failure` | feature-implementer | `schemas/scratch/build-failure.schema.json` |
| `build-pass` | feature-implementer | `schemas/scratch/build-pass.schema.json` |
| `review-feedback` | each reviewer | `schemas/scratch/review-feedback.schema.json` |

### File Lifecycle

See the `pipeline-handoff` skill for which agent appends each record type and how the coordinator validates them at agent transitions.

### File Templates

Templates for human-read markdown files are in `.claude/templates/`:

| Template | Used By | When |
|----------|---------|------|
| `implementation-plan.md` | feature-implementer | Before coding |
| `escalations.md` | feature-implementer | When `tag: "escalate"` findings or `verdict: "escalated"` records exist |

JSONL records do not use markdown templates — they are validated against the JSON Schemas in `schemas/scratch/`.

### Rules

1. **One feature at a time** — Clear scratch before starting new feature.
2. **Agents own their record types** — Each agent appends only the record types listed above.
3. **Read before write** — Agents read upstream records before appending their own.
4. **Append-only** — Never edit, reorder, or delete prior records in `handoff.jsonl`. Use `supersedes_record_at` (where supported) to correct a prior decision.
5. **Traceability** — Every record references the requirement ID (`req_id` matching `^REQ-[A-Z]+-[0-9]{3}$`).
6. **No system /tmp** — Use `.scratch/tmp/` for intermediate computation files.
