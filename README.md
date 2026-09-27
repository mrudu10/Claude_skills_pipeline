# Requirements Pipeline: 4 Chained Skills (Gherkin handoff)

This pipeline turns a raw feature draft into a final, codebase-grounded **Gherkin feature list**. Every phase hands over `.feature` files, which are the primary artifact, plus a markdown report holding the details Gherkin can't express (options, evidence, estimates).

```
raw draft ──▶ [1] refine-draft ─────────▶ 01-features/*.feature  + 01-refined-draft.md
                                                │  (Features, Rules = R-n, happy-path Scenarios, conflicts resolved)
              [2] identify-edge-cases ◀─────────┘ ──▶ 02-features/*.feature + 02-edge-cases.md
                                                         │  (+ edge-case Scenarios using the user's chosen option)
              [3] evaluate-feasibility ◀─────────────────┘ ──▶ 03-features/*.feature + 03-feasibility.md
                                                                  │  (+ @F-n @verdict @effort tags; steps unchanged)
              [4] final-gherkin-feature-list ◀──────────────────┘ ──▶ 04-features/*.feature + 04-requirements.md
                                                                        (final FR Rules, AC Scenarios, NFRs: the handover)
```

## Handoff contract (shared by all four skills)

- **Workspace:** `.pipeline/<draft-slug>/` at the repo root. A single draft can yield several Gherkin Features, each written as `<feature-slug>.feature`.
- **Each phase writes a new folder** (`01-features/` → `04-features/`) and never edits an earlier one, so you can `diff` any two phases.
- **Feature file header:**
  ```gherkin
  # draft: <draft-slug> | phase: <1-4> | status: READY | BLOCKED
  # repo_commit: <sha> | created: <ISO date>
  ```
- **Report header (YAML):** `draft`, `phase`, `status`, `input`, `repo_commit`, `created`, plus phase-specific counters.
- **Gate:** a phase refuses to run if the previous output is missing, has `status: BLOCKED`, or contains `@pending` Scenarios.
- **User decision points:**
  - Phase 1 asks how to resolve each conflict with existing functionality (Replace / Keep existing / Coexist / Other), plus blocking questions and the Feature grouping.
  - Phase 2 asks you to pick 1 of 4 options (Prevent / Handle gracefully / Allow & inform / Accept-defer) for each edge case.
  - Phase 4 asks again only if phase 3 found a chosen option infeasible (`@needs-review`).
- **Gherkin writing rules:** one `Rule` per requirement, exactly one `When` per Scenario, declarative steps with concrete values, at most 7 steps, observable `Then`s, and code paths in comments only. The conventions of any `.feature` files already in the repo take precedence.
- **Tags as traceability:**
  | Tag | Meaning | Added in |
  |---|---|---|
  | `@R-n` | refined requirement (Rule) | 1 |
  | `@C-n` | conflict with existing functionality | 1 |
  | `@happy-path` / `@alternative-flow` / `@migration` | scenario type | 1, 4 |
  | `@pending` + `<TBD: Q-n>` | awaiting user answer | 1, 2 (must be gone before handoff) |
  | `@EC-n @edge-case @severity:x @decision:A-E` | edge case with user's choice | 2 |
  | `@deferred @wip` / `@accepted-risk` | decision D | 2 |
  | `@F-n @verdict:x @effort:S-XL` / `@needs-review` | feasibility finding | 3 |
  | `@FR-n @AC-n.n @NFR-n @negative` | final requirement / acceptance criterion | 4 |
- **Evidence rule:** any claim about the code cites `path/to/file.ext:line` or a symbol, or is marked `UNVERIFIED`.
- **Drift check:** if `repo_commit` changed since the previous phase, the current phase notes it and re-checks the cited files.

## Installation

Copy each folder (`refine-draft/`, `identify-edge-cases/`, `evaluate-feasibility/`, `final-gherkin-feature-list/`) into your skills directory. For Claude Code, that is `.claude/skills/`. For claude.ai, upload them under Settings → Capabilities → Skills.

## Usage

```
/refine-draft                <paste raw draft>
/identify-edge-cases         <draft-slug>
/evaluate-feasibility        <draft-slug>
/final-gherkin-feature-list  <draft-slug>
```
