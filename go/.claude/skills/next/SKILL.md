---
name: next
description: >-
  Reset feature context and recommend what to work on next based on PRD coverage.
  Load when the user asks "what's next" or invokes /next.
compatibility:
  - claude-code
  - opencode
  - github-copilot
metadata:
  version: "1.0"
  author: team
---

# Next

Clear the scratch directory, survey unimplemented PRD requirements, and recommend the next feature to tackle.

## Prerequisite

A skill cannot invoke `/clear` — slash commands run in the harness, not Claude. If the prior conversation is large or unrelated, ask the user to run `/clear` first, then re-invoke `/next`.

## Instructions

1. Reset scratch state:

   ```bash
   rm -rf .scratch && mkdir -p .scratch/reviews .scratch/tmp
   ```

2. Extract requirement identifiers from the PRD:

   ```bash
   grep -oE 'REQ-XX-[0-9]+' docs/prd.md | sort -u
   ```

3. Extract requirement identifiers already addressed (implemented or withdrawn) from git history:

   ```bash
   git log --pretty=%s%n%b | grep -oiE 'REQ-XX-[0-9]+' | tr a-z A-Z | sort -u
   ```

4. Compute the set difference — requirements present in the PRD but absent from git history. These are candidates.

5. For up to five candidates, read the requirement section from `docs/prd.md` and capture: identifier, title, one-line summary, and any dependency it declares on other requirements.

6. Rank candidates by:
   - **Foundational first**: cross-cutting infrastructure before level-specific requirements.
   - **Dependency order**: a requirement whose dependencies are met outranks one that is blocked.
   - **Smallest viable next step**: prefer single-package requirements over cross-package ones.

7. Present a short recommendation: top pick with rationale, plus 2–3 alternates. Format:

   ```
   Recommended: REQ-XX-NNN — <title>
     Why: <one line>

   Alternates:
     - REQ-XX-NNN — <title>
     - REQ-XX-NNN — <title>
   ```

8. Stop and wait for the user to choose. Do not invoke `pipeline-coordinator` until the user confirms a target.

## Rules

- Never assume an identifier is implemented from grep alone — git history is the authority. A REQ mentioned in a comment or doc does not count as done.
- If the PRD and git history are in sync (no unimplemented requirements), report that and stop.
- If the user asks for the recommendation without resetting scratch (e.g. follow-up in the same conversation), skip step 1.
- Keep the recommendation under 15 lines. The user reads it, decides, then routes through `pipeline-coordinator`.
