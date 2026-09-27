---
name: refine-draft
description: Phase 1 of 4 in the requirements pipeline. Use when the user gives a raw feature idea, ticket, or rough draft and wants it refined into a structured, codebase-grounded Gherkin feature list. Detects requirements that contradict existing functionality, asks the user to clarify, then writes .pipeline/{draft-slug}/01-features/*.feature plus 01-refined-draft.md for the identify-edge-cases skill.
---

# Phase 1: Refine Raw Draft into a Gherkin Feature List

## Role
You act as a senior engineer doing requirements intake. Your job is to turn a messy draft into a list of Gherkin features that uses the codebase's real vocabulary. You must find every place where the draft contradicts what the system already does, and you must get the user's decision on each one before handing off. You do NOT design solutions, find edge cases, or judge feasibility here. Those are later phases.

## Input
- The raw draft the user provides (text, ticket, pasted notes, or a linked issue).
- The connected GitHub repository.

## Procedure

1. **Create the workspace.**
   - Derive a `draft-slug` from the draft as a whole: kebab-case, 2–5 words (e.g. `invoice-improvements`). It names the workspace folder `.pipeline/{draft-slug}/`.
   - Save the untouched draft to `.pipeline/{draft-slug}/00-raw-draft.md` so later phases can diff against it.
   - Feature slugs are derived later, in step 5: one per Gherkin Feature.

2. **Orient in the codebase (light scan).**
   - Read `README*`, top-level docs, and the directory tree (2 levels deep).
   - Identify the stack: languages, frameworks, package manifests (`package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, etc.).
   - For each noun in the draft (e.g. "user", "order", "export"), grep for matching models, tables, routes, or services. Record the actual names used in code.
   - **Find existing BDD assets:** search for `*.feature` files and step definitions (Cucumber, SpecFlow, Behave, pytest-bdd, Godog, Karate, etc.). If any exist, record their folder, language (`# language:`), tag conventions, and step phrasing. The Gherkin you write must match them.

3. **Decompose the draft.**
   - Separate the draft into: problem, goal, actors, requested behaviors, and constraints stated by the user.
   - Split compound sentences so that each requirement expresses exactly one behavior.
   - Write each behavior as `R-n: {Actor} can/must {action} {object} [condition]`. These become Gherkin `Rule`s in step 5.
   - Replace vague words ("fast", "easy", "some", "etc.", "handle") with a measurable phrasing, or turn them into an open question.

4. **Map terms to code.**
   - Build a glossary: draft term → code entity (file:line) → note. If a term has no code counterpart, mark it `NEW`. If it maps to several entities, mark it `AMBIGUOUS` and raise a question.
   - Use the code names from the glossary (or the domain names already used in existing `.feature` files) in all Gherkin steps.

5. **Transform into a Gherkin feature list.**
   - **Group requirements into Features.** A Feature is one user-facing capability that an actor would recognize (e.g. "Bulk invoice export", "Invoice reminders"). A draft usually yields 1–5 Features. Each R-n belongs to exactly one Feature.
   - **Feature slug:** for each Feature, derive a `feature-slug` in kebab-case, 2–5 words (e.g. `bulk-invoice-export`). Write the Feature to `.pipeline/{draft-slug}/01-features/{feature-slug}.feature`.
   - **Structure of each file:**
     ```gherkin
     @draft:{draft-slug} @feature:{feature-slug} @phase:1
     Feature: {Feature title}
       As a {actor from glossary}
       I want {capability}
       So that {benefit from the problem statement}

       # Context: {1-2 lines of problem statement}
       # Code: {main module paths from the glossary}

       Background:
         Given {shared precondition, only if every scenario needs it}

       @R-1
       Rule: {R-1 text, in one sentence}

         @R-1 @happy-path
         Scenario: {concrete example of the rule working}
           Given {state, using code entity names}
           When {single actor action}
           Then {observable outcome}
     ```
   - **Writing rules** (these apply in every phase):
     - Use one `Rule` per R-n, and at least one happy-path `Scenario` per Rule, plus a second Scenario only where the draft states an explicit alternative flow.
     - Use exactly one `When` per Scenario. `Given` describes state, `When` is one action, and `Then` is an observable outcome (UI, API response, stored data, event emitted), never an internal implementation detail.
     - Write declaratively ("When the finance user exports March invoices"), not imperatively ("When I click the Export button").
     - Use concrete example values ("3 invoices dated 2026-03-01..03-31"), not placeholders, unless you are writing a `Scenario Outline` with `Examples`.
     - Keep each Scenario to at most 7 steps and use `And`/`But` for continuation. Use a data table for more than 3 fields.
     - Tags: `@R-n` on each Rule and its Scenarios, `@happy-path` or `@alternative-flow`, and `@C-n` where a conflict applies.
     - Ambiguous or unanswered details: write the step with a placeholder `{TBD: Q-n}` and tag the Scenario `@pending`.
   - Follow the conventions of any existing `.feature` files (language, tag style, step phrasing) over these defaults, and note any deviation.

6. **Detect conflicts with existing functionality (mandatory; do not skip).**
   For EVERY Rule (`R-n`) and its Scenarios, search the codebase for existing behavior that the requirement would contradict, change, or duplicate. Check each of these sources:
   | Source | What to look for |
   |---|---|
   | Business logic | services, domain models, and rules enforcing the opposite (e.g. the draft says "users can delete orders" but `OrderService.delete` rejects orders with status `SHIPPED`) |
   | Validation & constraints | schema validators, DB constraints (UNIQUE, NOT NULL, CHECK, FK), enum values, max lengths |
   | Permissions | role checks, policies, middleware that restrict who can do the action |
   | API contracts | existing endpoints, response shapes, versioned APIs, GraphQL schemas, and their consumers |
   | UI flows | existing screens or flows that implement the action differently |
   | Config & feature flags | flags or settings that currently disable or alter this behavior |
   | Tests & existing `.feature` files | tests or scenarios that assert the opposite of a new Scenario's `Then` (strong evidence of intended current behavior) |
   | Docs / ADRs | `docs/`, ADRs, and READMEs stating a decision the draft reverses |
   | Jobs & integrations | scheduled jobs, webhooks, and events that assume the current behavior |

   Log each hit as a conflict `C-n` with one of these types:
   - `CONTRADICTS`: the requirement is the direct opposite of the current behavior
   - `OVERRIDES`: the requirement changes an existing behavior that other parts of the system rely on
   - `BREAKS-CONTRACT`: the requirement changes an API, schema, or event that has consumers
   - `DUPLICATES`: the functionality already exists, fully or partially
   - `INCONSISTENT`: the draft contradicts itself (R-a vs R-b)

   For each conflict, record the linked `R-n`, the current behavior, and the evidence (`path:line`, or the test/scenario name that asserts it). A conflict without evidence is not a conflict; log it as a `Q-n` instead.
   In the `.feature` file, tag the affected Rule `@C-n` and add a comment directly above it: `# C-n CONTRADICTS: {current behavior} (path:line). Resolution: PENDING`.
   If you find no conflicts, state explicitly: "No conflicts found. Checked: {list of areas searched}."

7. **Record assumptions and gaps.**
   - Record every inference you made that the draft did not state as `A-n` (an assumption).
   - Record every gap you cannot resolve from the draft or the code as `Q-n`, tagged `[BLOCKING]` or `[NON-BLOCKING]`. A question is blocking if the answer changes *what* is being built.

8. **Ask for clarification (mandatory stop point).**
   Before writing a READY file, you MUST ask the user and wait for their answers. Ask in this order:
   1. **Every conflict `C-n`.** Offer these options, reworded to fit the specific case:
      - **(1) Replace:** the new behavior replaces the existing one, and existing data and callers are migrated
      - **(2) Keep existing:** the current behavior stays, and the requirement is dropped or modified to fit it
      - **(3) Coexist:** both behaviors exist, scoped by role, flag, tenant, or configuration
      - **(4) Other:** the user describes their own resolution
      Mark the option you recommend and give a one-line reason.
   2. **Every `[BLOCKING]` question.**
   3. **Every `AMBIGUOUS` glossary term.**
   4. **The Feature grouping and the `A-n` assumptions,** as a single confirm/correct item (e.g. "I split the draft into 3 features: A, B, C. OK?").

   How to ask:
   - If the `AskUserQuestion` tool is available, use it: at most 4 questions per call with 2–4 options each, and several calls if needed. Put the conflict ID, the evidence, and the affected Scenario name in each question text, and put the recommended option first, labeled "(Recommended)".
   - Otherwise, post a numbered list in chat and tell the user they can reply compactly (e.g. `C-1: 3, C-2: 1, Q-1: yes, A: ok`).
   - Write all files with `status: BLOCKED` and the pending questions before asking, so the state is saved if the session ends.

9. **Apply the answers to the Gherkin.**
   - Record each decision in the `Resolution` field of its `C-n` or `Q-n` in the report, and update the comment in the `.feature` file (`Resolution: (3) Coexist, admins only`).
   - Update the affected Rule and Scenarios to match the decision:
     - **Replace:** keep the new Scenario, and add a Scenario covering existing data/callers (tag `@migration`)
     - **Keep existing:** reword the Rule/Scenario to fit current behavior, or remove it and list the R-n as `DROPPED (C-n)` in the report
     - **Coexist:** put the scoping condition into `Given` (e.g. `Given the user has the "admin" role`), and add a Scenario showing the other side of the boundary
   - Replace every `{TBD: Q-n}` that now has an answer, and remove `@pending`.
   - If an answer creates a new conflict or question, ask again (repeat step 8, only for the new items).
   - When every conflict and blocking question is resolved and no `@pending` tag remains on a blocking item, set `status: READY` (in the report and in the `@phase` header comment of each file).
   - If the user says "use your recommendation" for any or all items, apply your recommended option and mark the item `Resolution: recommended option accepted by user`.

10. **Validate the Gherkin before handing off.**
    - Every R-n appears as exactly one `Rule` tagged `@R-n` (or is listed as DROPPED).
    - Every Rule has at least one Scenario, and every Scenario has exactly one `When`.
    - The files parse. If `@cucumber/gherkin`, `gherkin-lint`, or the project's own BDD runner is available, run it in dry-run mode; otherwise check the keywords and indentation manually.

## Rules
- Do not add features the draft does not ask for. You may note opportunities under "Out of scope (suggested)".
- Do not decide a conflict on the user's behalf, even when the answer seems obvious. Recommend, then ask.
- Cite file:line for every code reference: in the report, and in `# Code:` comments in the feature files.
- Gherkin steps use the domain language; code paths go only in comments.

## Output

### Primary handoff: `.pipeline/{draft-slug}/01-features/{feature-slug}.feature` (one file per Feature)
Every file starts with this header comment block, followed by the Feature as described in step 5:
```gherkin
# draft: {draft-slug} | phase: 1 | status: READY | BLOCKED
# repo_commit: {sha} | created: {date}
```

### Supporting report: `.pipeline/{draft-slug}/01-refined-draft.md`

```markdown
---
draft: {draft-slug}
phase: 1
status: READY | BLOCKED
input: 00-raw-draft.md
features: [{feature-slug}, ...]
repo_commit: {sha}
created: {date}
conflicts_found: {n}
conflicts_resolved: {n}
---

# {Draft title}

## Problem statement
{2–4 sentences: who has what problem today}

## Goal
{1–2 sentences: what success looks like}

## Feature list
| Feature slug | Feature title | Rules (R-n) | Scenarios | File |
|---|---|---|---|---|
| bulk-invoice-export | Bulk invoice export | R-1, R-2 | 3 | 01-features/bulk-invoice-export.feature |

## Actors
| Actor | Code entity | Notes |

## Requirements
- R-1 (bulk-invoice-export): ...
- R-2 (bulk-invoice-export): ... (modified per C-1)
- R-3: DROPPED (C-2)

## Conflicts with existing functionality
| ID | Type | Requirement / Scenario | Current behavior | Evidence | Resolution |
|---|---|---|---|---|---|
| C-1 | CONTRADICTS | R-2 / "Delete a shipped order" | Shipped orders cannot be deleted | `src/orders/service.ts:88`, test `orders.spec.ts › rejects delete when shipped` | (3) Coexist: admins only |

(or: "No conflicts found. Checked: business logic, validation, permissions, API, UI, config, tests/features, docs, jobs.")

## Constraints (stated by user)
- ...

## Glossary (draft term → code)
| Term | Code entity | Location | Status (MAPPED/NEW/AMBIGUOUS) |

## Codebase context
- Stack: ...
- Existing BDD setup: {runner, feature folder, step definition folder} | none
- Relevant modules: `path/`, and why each matters

## Assumptions
- A-1: ... (confirmed | corrected: ...)

## Open questions
- Q-1 [BLOCKING|NON-BLOCKING]: ... → Resolution: ...

## Out of scope (stated / suggested)
- ...
```

## Handoff
- If BLOCKED: end with the clarification questions only.
- If READY: end with
  `Phase 1 complete: {f} features, {n} rules, {s} scenarios, {c} conflicts resolved → run /identify-edge-cases {draft-slug}`
