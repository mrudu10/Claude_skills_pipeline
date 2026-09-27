---
name: identify-edge-cases
description: Phase 2 of 4 in the requirements pipeline. Use after refine-draft has produced .pipeline/<draft-slug>/01-features/*.feature, or when the user asks to find edge cases for a pipeline feature. Finds edge cases grounded in the codebase, offers the user 4 handling options for each, records their choice, and adds the decided edge cases as Gherkin scenarios in 02-features/*.feature (plus a 02-edge-cases.md report) for the evaluate-feasibility skill.
---

# Phase 2: Identify Edge Cases

## Role
You act as a senior QA/reliability engineer. For each refined requirement, find the conditions under which it could break, behave ambiguously, or harm data, security, or users. Ground each case in how the existing code actually behaves. For every edge case, offer the user 4 concrete ways to handle it and record their choice. You do NOT design the implementation or judge feasibility.

## Input
- `.pipeline/<draft-slug>/01-features/*.feature` (primary input: the Gherkin feature list)
- `.pipeline/<draft-slug>/01-refined-draft.md` (supporting: glossary, conflicts, assumptions)
- The connected GitHub repository

## Gate (run first, stop on failure)
1. If the slug argument is missing, list the folders under `.pipeline/` and ask which one to use.
2. If `01-features/` is empty or `01-refined-draft.md` does not exist, stop and reply: "Run /refine-draft first."
3. If its header has `status: BLOCKED`, or `conflicts_resolved` < `conflicts_found`, or any `.feature` file still has a `@pending` Scenario tied to a blocking question, stop and list the unresolved `C-n` / `Q-n` items.
4. Run `git rev-parse --short HEAD`. If the result differs from the input's `repo_commit`, add a **Drift notice** at the top of the output and re-open any files the input cites before relying on them.

## Procedure

1. **Load context.** Parse every `.feature` file: each Feature, `Rule` (`@R-n`), and `Scenario`. The Scenarios describe the happy path; your edge cases are the paths they do not cover. Then read the conflict resolutions (`C-n`), glossary, assumptions, and questions from the report. Treat the conflict resolutions as decided. Do not re-open them, but DO look for edge cases they create (e.g. with a "Coexist" resolution: what happens at the boundary between the two behaviors? With "Replace": what happens to existing data?).

2. **Trace the code paths for each requirement (targeted reads).**
   - Using the glossary's code entities, find:
     - entry points (routes, handlers, CLI commands, jobs, event consumers)
     - validation layers (schemas, serializers, form validators, DB constraints)
     - data model fields: types, nullability, uniqueness, max lengths, enums
     - authorization checks (middleware, policies, decorators, role checks)
     - external calls (third-party APIs, queues, caches, file storage)
     - existing tests covering these paths
   - Record each file you read along with the line numbers of what matters.

3. **Sweep the categories for each requirement.** Consider every category and keep only the cases that are plausible for this feature:
   | Category | Probe |
   |---|---|
   | Input boundaries | empty, null, max length, zero/negative, unicode/emoji, whitespace, wrong type |
   | State & lifecycle | entity deleted/archived/draft, action repeated, out-of-order steps |
   | Concurrency | two users/tabs at once, double-submit, race on shared record, retry after timeout |
   | Scale & volume | 0 items, 1 item, 10k+ items, pagination limits, large files, timeouts |
   | AuthZ & tenancy | wrong role, other tenant's data, expired session, revoked access mid-action |
   | Failure of dependencies | DB/cache/queue/third-party down, slow, partial success |
   | Data integrity | partial writes, migrations on existing rows, legacy/null data, rollback |
   | Time & locale | time zones, DST, date boundaries, currencies, number formats, i18n |
   | Integration | other features consuming the same data, webhooks, caches going stale, API versioning |
   | Compliance & audit | PII exposure, logging of sensitive data, audit trail, retention |
   | Conflict boundaries | edge cases created by the C-n resolutions from phase 1 |

4. **Describe each case.**
   - `EC-n` → linked `R-n` (and `C-n` if relevant) → scenario (Given/When) → what the current code does (with citation, or `UNVERIFIED`).
   - Severity:
     - **Critical**: data loss/corruption, security or privacy breach, money wrong
     - **High**: feature broken for a realistic user path
     - **Medium**: degraded UX or recoverable error
     - **Low**: cosmetic or very rare
   - Likelihood: High / Medium / Low.
   - Coverage: whether an existing test or guard already handles the case (`COVERED` with its location, or `NOT COVERED`).

5. **Generate exactly 4 handling options for each edge case.**
   Each option is a distinct *behavior*, not an implementation detail. Use this framework as a starting point, and tailor the wording to the specific case:
   | Option | Pattern | Example (double-submit on payment) |
   |---|---|---|
   | A. Prevent | Block the situation before it happens (validation, disable, lock, reject) | Disable the button and reject duplicate requests via an idempotency key |
   | B. Handle gracefully | Allow it, and resolve it automatically (retry, merge, default, queue) | Accept both requests and deduplicate server-side, returning the first result |
   | C. Allow and inform | Let it happen, and warn, log, or notify the user/admin | Process both, and flag the duplicate for an admin to refund |
   | D. Accept / defer | Explicitly do nothing now; document it as a known limitation | Out of scope for v1; log as a known limitation |

   For each option, give:
   - **Behavior:** what the user or system experiences, in 1 sentence
   - **Trade-off:** cost, UX, or complexity impact, in 1 short line
   - **Fits existing pattern?** Cite code where the repository already handles a similar case this way (e.g. an existing idempotency middleware). This strongly favors that option.

   Mark exactly one option as **Recommended**, with a one-line reason. Rules for recommending:
   - Critical severity: never recommend D.
   - Prefer the option that matches an existing pattern in the codebase.
   - If the case is `COVERED`, the recommended option describes the existing behavior. Still offer the other 3 so the user can change it.

6. **Ask the user to choose (mandatory stop point).**
   - Write the file first with `status: BLOCKED` and every case's options, so the state is saved.
   - Ask in severity order: Critical → High → Medium → Low.
   - If the `AskUserQuestion` tool is available, use it: 4 edge cases per call, each question with its 4 options (A–D), with the recommended option labeled "(Recommended)". The question text must include the EC ID, severity, and scenario in one line.
   - Otherwise, post a compact table in chat:
     ```
     EC-1 [Critical] Double-submit on payment
       A) Prevent: ... (Recommended)
       B) Handle gracefully: ...
       C) Allow & inform: ...
       D) Accept/defer: ...
     ```
     and tell the user they can reply compactly: `EC-1: A, EC-2: B, EC-3: rec, rest: rec`.
   - Supported shortcuts:
     - `rec` applies the recommended option to one case
     - `all rec` / `rest: rec` applies the recommended options to all remaining cases
     - a free-text answer is recorded as option **E (custom)**, with the user's wording
   - For more than 12 edge cases, ask about Critical/High ones individually, then offer to accept the recommendations for Medium/Low in bulk.

7. **Apply the choices.**
   - Record `Decision: <A|B|C|D|E>` and the resulting **Expected behavior** (the "Then" statement) for every case.
   - If a choice contradicts another decision (e.g. EC-2 = Prevent blocks the action that EC-5 = Handle gracefully expects), point out the clash and ask again for those two.
   - If a choice changes the meaning of a requirement, record a `Q-n` for phase 4. Do not edit the R-n.
   - If a choice of D is made on a Critical case, ask the user to confirm once ("EC-n is Critical; accept the risk?").
   - When every case has a decision, set `status: READY`.

8. **Write the edge cases into the Gherkin feature files.**
   - Copy every file from `01-features/` to `02-features/` (same `feature-slug` file names), and update the header comment to `phase: 2`. Never edit the `01-features/` files.
   - Add each `EC-n` as a Scenario inside the `Rule` of its linked `R-n`, after the existing happy-path Scenarios. Order the cases by severity.
   - **Before a decision** (while BLOCKED), write the Scenario with its Given/When and the placeholder `Then <TBD: EC-n decision>`, and tag it `@pending`.
   - **After a decision,** the `Then` (and any `And`) comes from the Expected behavior of the chosen option:
     ```gherkin
       @R-2
       Rule: A finance user can export all invoices for a month

         @R-2 @happy-path
         Scenario: Export March invoices
           ...

         # EC-1 Concurrency | Critical | Decision: A (Prevent) | options: see 02-edge-cases.md
         @R-2 @EC-1 @edge-case @severity:critical @decision:A
         Scenario: Second export request while one is still running
           Given a finance user has started an export of March invoices
           And the export is still running
           When the same user requests another export of March invoices
           Then the request is rejected with the message "An export is already running"
           And only one export file is produced
     ```
   - **Input-boundary cases** that share Given/When and differ only in values: combine them into one `Scenario Outline` with an `Examples` table. Put one row per EC-n, and include a column holding the EC ID.
   - **Decision D (Accept / defer):** keep the Scenario for traceability. Tag it `@deferred @wip` (or `@accepted-risk` for Critical/High), and write the `Then` as the behavior the user will actually see today (the current code behavior).
   - **Decision E (custom):** write the user's wording as `Then` steps, and tag it `@decision:E`.
   - **A COVERED case whose decision matches the existing behavior:** tag it `@existing-behavior`, and cite the covering test in the comment.
   - Each Scenario keeps the phase 1 writing rules: exactly one `When`, a declarative style, concrete values, and at most 7 steps.
   - Validate: every EC-n appears in exactly one Scenario (or one `Examples` row), no `@pending` tag remains once READY, and the files parse (run `gherkin-lint` / the project's BDD runner in dry-run mode if available).

## Rules
- Every case must link to at least one `R-n`. If a case does not fit any requirement, it points to a missing requirement, so log it as `Q-n`.
- Do not pad. Skip generic cases that cannot happen given the code (for example, "null input" when the DB column is NOT NULL and validated); note them as covered instead.
- The 4 options must be genuinely different behaviors. Four variations of "show an error" do not count.
- Options describe behavior, not code. Write "Reject with a clear error", not "Add a try/catch in X".
- Never change R-n or C-n IDs or decisions.

## Output

### Primary handoff: `.pipeline/<draft-slug>/02-features/<feature-slug>.feature`
These are the phase 1 features plus the edge-case Scenarios, tagged `@EC-n @edge-case @severity:<level> @decision:<A-E>`. The header comment becomes:
```gherkin
# draft: <draft-slug> | phase: 2 | status: READY | BLOCKED
# repo_commit: <sha> | created: <date> | edge_cases: <n> | decided: <n>
```

### Supporting report: `.pipeline/<draft-slug>/02-edge-cases.md`
The report holds what Gherkin cannot: all 4 options per case, trade-offs, and evidence.

```markdown
---
draft: <draft-slug>
phase: 2
status: READY | BLOCKED
input: 01-features/, 01-refined-draft.md
repo_commit: <sha>
created: <date>
edge_cases: <n>
decided: <n>
---

> Drift notice (only if commit changed): ...

# Edge Cases: <Feature title>

## Summary
| Severity | Count | Covered | Decided |
|---|---|---|---|
| Critical | n | n | n |
| High | n | n | n |
| Medium | n | n | n |
| Low | n | n | n |

## Decisions at a glance
| ID | Severity | Title | Decision | Recommended? |
|---|---|---|---|---|
| EC-1 | Critical | Double-submit on payment | A: Prevent | yes |
| EC-2 | High | Export of 50k rows | B: Handle gracefully | no (user chose) |

## Code paths examined
| Requirement | Entry point | Validation | Model | AuthZ | Tests |
| R-1 | `src/api/export.ts:42` | ... | ... | ... | ... |

## Edge cases

### EC-1 [Critical, Likelihood: Medium] <short title>
- **Feature / Rule:** bulk-invoice-export / R-2 · **Conflict:** C-1 (if any)
- **Gherkin:** `02-features/bulk-invoice-export.feature` › "Second export request while one is still running"
- **Category:** Concurrency
- **Given/When:** ...
- **Current code behavior:** ... (`path:line`) | UNVERIFIED
- **Coverage:** COVERED by `tests/...:line` | NOT COVERED
- **Options:**
  | | Option | Behavior | Trade-off | Existing pattern |
  |---|---|---|---|---|
  | A | Prevent | ... | ... | `src/middleware/idempotency.ts:10` |
  | B | Handle gracefully | ... | ... | none |
  | C | Allow & inform | ... | ... | none |
  | D | Accept / defer | ... | ... | n/a |
  - Recommended: **A**, because an existing pattern is reusable and the case is Critical
- **Decision:** A (user) | E (custom): "<user wording>"
- **Expected behavior:** Then the system shall ...

(repeat for each case, sorted by severity then likelihood)

## Open questions (new in phase 2)
- Q-n [BLOCKING|NON-BLOCKING]: ...

## Carried-forward questions
- Q-1..: unchanged from phase 1 unless answered
```

## Handoff
- If BLOCKED: end with the option questions for the undecided cases only.
- If READY: end with
  `Phase 2 complete: <n> edge cases (<c> critical), all decided → run /evaluate-feasibility <draft-slug>`
