---
name: final-gherkin-feature-list
description: Phase 4 of 4 in the requirements pipeline. Use after evaluate-feasibility has produced .pipeline/{draft-slug}/03-features/*.feature, or when the user asks for the final Gherkin feature list or final requirements of a pipeline feature. Rewrites the annotated Gherkin into a final, clean, testable Gherkin feature list (04-features/*.feature) plus a 04-requirements.md summary, ready to hand over to the implementing team.
---

# Phase 4: Final Gherkin Feature List

## Role
You act as a senior engineer writing the spec that the implementing team will build and test against. Merge the refined intent (phase 1), the decided edge cases (phase 2), and the feasibility findings (phase 3) into a final Gherkin feature list that is unambiguous, testable, and traceable. The `.feature` files ARE the requirements handed over. Every Scenario must be something a developer can implement and a tester can automate.

## Input
- `.pipeline/{draft-slug}/03-features/*.feature` (primary input)
- `.pipeline/{draft-slug}/03-feasibility.md` (required, must be READY)
- `.pipeline/{draft-slug}/02-edge-cases.md` and `01-refined-draft.md` (supporting)
- The connected GitHub repository (for verifying names, paths, and existing step definitions)

## Gate (run first, stop on failure)
1. If `03-features/` or `03-feasibility.md` is missing, stop and reply: "Run /evaluate-feasibility first."
2. If the report has `status: BLOCKED`, stop and list its blocking questions.
3. If `overall_verdict: NO-GO`, do not write the spec. Summarize the reasons and ask the user whether to re-scope (go back to phase 1 with a changed draft) or stop.
4. Drift check against `repo_commit`. If the commit changed, re-verify every file path you intend to cite.

## Procedure

1. **Resolve `@needs-review` items first (mandatory stop point if any exist).**
   - For each Scenario that phase 3 tagged `@needs-review` (the user's chosen option is CONFLICT or XL), ask the user whether to keep their original choice or switch to the alternative named in the comment.
   - Use `AskUserQuestion` if available (4 per call, with "Keep original" and "Switch to {option}" as choices, plus the effort and risk in the question text); otherwise post a numbered list in chat.
   - Never switch silently. Record each answer in the Change log.

2. **Apply the feasibility decisions.**
   - For each `R-n`, follow the "Recommended requirement changes" in the phase 3 report: keep, modify, split, descope, or defer.
   - Record every change in the Change log with its reason (the F-n, Q-n, or user answer that caused it).

3. **Rewrite each Feature into its final form.**
   Write the files to `04-features/{feature-slug}.feature`. Final structure:
   ```gherkin
   # draft: {draft-slug} | phase: 4 | status: READY | repo_commit: {sha} | created: {date}
   @feature:{feature-slug} @effort:{range}
   Feature: {Feature title}
     As a {actor}
     I want {capability}
     So that {benefit}

     {2-4 line free-text description: scope and key constraints.
      Out of scope: ... (see 04-requirements.md)}

     Background:
       Given {shared preconditions}

     # Implements via: F-1 | touches: src/invoices/export.ts, src/api/routes.ts | pattern: src/reports/csv-export.ts
     @FR-1 @R-1 @F-1
     Rule: FR-1 A finance user can export all invoices for a selected month as CSV

       @FR-1 @AC-1.1 @happy-path
       Scenario: Export all invoices for March
         Given 3 invoices dated between 2026-03-01 and 2026-03-31
         When the finance user exports invoices for March 2026
         Then a CSV file with 3 invoice rows is downloaded

       @FR-1 @AC-1.2 @EC-1 @negative
       Scenario: Second export request while one is still running
         Given an export of March 2026 invoices is running for the finance user
         When the finance user requests another export of March 2026 invoices
         Then the request is rejected with the message "An export is already running"
   ```
   **Transformation rules:**
   - **Rules → FRs:** each surviving `Rule` becomes `FR-n` (Rule title prefixed `FR-n`). Keep the `@R-n` tag for traceability. A split R-n becomes several FRs; a merged pair becomes one FR carrying both `@R-` tags.
   - **Scenarios → acceptance criteria:** each Scenario gets `@AC-{fr}.{n}`. The happy path comes first, then the negative and edge cases in severity order.
   - **Edge cases**, from their `@decision` tag:
     - A / B / C / E → keep as a Scenario under its Rule, tagged `@EC-n @negative` (or `@alternative-flow`)
     - D (deferred) → remove from `04-features/` and list under "Deferred edge cases" in the report
     - D on Critical/High (`@accepted-risk`) → keep the Scenario tagged `@accepted-risk @wip`, so the gap stays visible in the test suite, and list it under "Accepted risks"
   - **Conflicts**, from the `C-n` resolution:
     - Replace → the FR for the new behavior plus a `@migration` Scenario for existing data and callers
     - Keep existing → the Rule is reworded or removed (logged in the Change log)
     - Coexist → the scoping condition goes in `Given`, with Scenarios on both sides of the boundary
   - **NFRs:** write them as their own Rules in `04-features/non-functional.feature` (or inside the relevant Feature if they apply to only one Feature), tagged `@NFR-n @nfr:{performance|security|reliability|observability|compatibility|accessibility}`, with measurable `Then` steps (e.g. `Then the export completes within 5 seconds for 10,000 invoices`). Tie each one to an EC-n or F-n in a comment.
   - **Phase annotations:** move `@verdict:` and `@effort:` tags and the F-n comments into one `# Implements via:` comment per Rule. Keep `@F-n`; drop `@verdict:`, `@severity:`, `@decision:`, `@needs-review`, `@edge-case`, and `@pending`. Those details live in the report, and the final files stay clean for the delivery team.

4. **Polish the steps for automation.**
   - **Reuse existing step definitions:** if the repository has BDD step definitions, reuse their exact phrasing wherever the meaning matches. For each step, record in the report whether it is `EXISTING` (with the step definition's `path:line`) or `NEW`.
   - **Consistent phrasing:** the same action is worded identically everywhere (one wording for "the finance user exports invoices for {month}"). Build a step glossary.
   - Exactly one `When` per Scenario. `Then` describes observable outcomes only.
   - Use a declarative style with concrete values, and at most 7 steps per Scenario. Use a `Scenario Outline` with `Examples` for data variations.
   - **Banned words in steps:** should, may, might, fast, easy, user-friendly, appropriate, properly, correctly, etc., and/or. Replace each with a specific, measurable outcome.
   - No UI mechanics ("clicks", "types into field") unless the requirement itself is about the UI.

5. **Check traceability (mandatory self-check before writing).**
   - Every original R-n is accounted for: it appears as an FR Rule, or is listed as modified, descoped, or deferred in the Change log.
   - Every EC-n appears as a Scenario matching its Decision, or is listed as deferred or accepted.
   - Every C-n resolution is reflected in a Rule or Scenario, or in the Change log.
   - Every F-n with verdict EXTEND/NEW-CAPABILITY maps to at least one FR or NFR (`@F-n` tag).
   - Every Rule has at least one `@happy-path` Scenario, and every FR with a linked EC has at least one `@negative` Scenario.
   - No `@pending`, `{TBD`, or `@needs-review` remains.
   - The files parse: run `gherkin-lint` or the project's BDD runner in dry-run mode if available (e.g. `npx cucumber-js --dry-run`, `behave --dry-run`, `pytest --collect-only`), and report any undefined steps as `NEW` steps rather than as errors.
   - Every cited path exists at the current commit.
   If any check fails, fix it before writing. Do not hand over a spec that has gaps.

6. **Resolve open questions.** List every remaining Q-n with an owner role (Product / Eng / Design / Security) and state its impact if unanswered. Blocking questions should not exist at this point; if one does, set `status: BLOCKED`.

## Rules
- Do not introduce new scope. Anything new must trace to an EC-n, C-n, or F-n.
- Keep FR/NFR/AC IDs stable if the skill is re-run. Append new IDs; never renumber.
- The `.feature` files must be readable by someone who has not seen phases 1–3. Keep comments short, and put detail in the report.
- Code references go only in comments, and must be exact (`path:line` or symbol) and verified.
- Match the conventions of any `.feature` files already in the repository (language, folder, tag style) over these defaults.

## Output

### Primary handoff: `.pipeline/{draft-slug}/04-features/`
- One `{feature-slug}.feature` file per Feature, in the final form from step 3
- `non-functional.feature` if NFRs span several Features
- These files are the deliverable. They can be copied into the repository's features folder (e.g. `features/` or `tests/features/`) as-is.

### Supporting report: `.pipeline/{draft-slug}/04-requirements.md`

```markdown
---
draft: {draft-slug}
phase: 4
status: READY | BLOCKED
input: 03-features/, 03-feasibility.md
repo_commit: {sha}
created: {date}
overall_verdict: {from phase 3}
features: [{feature-slug}, ...]
---

# {Draft title}: Requirements Handover

## 1. Overview
{Problem, goal, and who benefits, in 3–5 sentences}

## 2. Feature list
| Feature | File | FRs | Scenarios | Effort | New steps |
|---|---|---|---|---|---|
| Bulk invoice export | 04-features/bulk-invoice-export.feature | FR-1..FR-3 | 9 | M–L | 4 |

## 3. Scope
**In scope:** ...
**Out of scope / deferred:** ... (with reason)

## 4. Non-functional requirements (summary)
- NFR-1 (Performance): ... · in `non-functional.feature` · From: EC-7, F-3

## 5. Dependencies and sequencing
{from phase 3, updated}

## 6. Rollout
Feature flag, migration order, backfill, monitoring/alerts to add

## 7. Step glossary
| Step phrase | Status | Step definition |
|---|---|---|
| `Given {int} invoices dated between {date} and {date}` | EXISTING | `features/steps/invoice.steps.ts:14` |
| `When the finance user exports invoices for {month}` | NEW | n/a |

## 8. Accepted risks
- EC-n: why it was accepted, and who accepted it (kept as a `@accepted-risk @wip` Scenario)

## 9. Deferred edge cases
- EC-n: ...

## 10. Open questions
| ID | Question | Owner | Impact if unanswered |

## 11. Change log (vs. phase 1 feature list)
| Original | Change | Reason |
| R-4 | Descoped | F-5 CONFLICT: ... |

## 12. Traceability matrix
| R-n | C-n (resolution) | EC-n (decision) | F-n | FR/NFR | Scenarios (AC) | Status |
```

## Handoff
End your reply with a 5-line summary: the number of Features, FRs, NFRs, and Scenarios; what was descoped; the top risk; the number of NEW steps to implement; and the path of `04-features/`. Offer to (a) copy the feature files into the repository's features folder on a new branch and open a PR, or (b) create one GitHub issue per FR with its Scenarios in the body.
