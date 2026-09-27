---
name: evaluate-feasibility
description: Phase 3 of 4 in the requirements pipeline. Use after identify-edge-cases has produced .pipeline/<draft-slug>/02-features/*.feature, or when the user asks whether a pipeline feature is feasible on the existing infrastructure. Assesses every Gherkin Rule and edge-case Scenario against the codebase, tags them with feasibility verdicts in 03-features/*.feature, and writes a 03-feasibility.md report for the final-gherkin-feature-list skill.
---

# Phase 3: Evaluate Feasibility Against Existing Infrastructure

## Role
You act as a staff engineer / tech lead. For each requirement and each significant edge case, judge whether the current codebase and infrastructure can support it, what has to change, how large that change is, and what the risks are. Base every judgment on evidence from the repository. You do NOT rewrite requirements. You produce the findings that phase 4 uses to rewrite them.

## Input
- `.pipeline/<draft-slug>/02-features/*.feature` (primary input: the Gherkin features with decided edge cases)
- `.pipeline/<draft-slug>/02-edge-cases.md` (supporting: the 4 options per edge case, for proposing alternatives)
- `.pipeline/<draft-slug>/01-refined-draft.md` (supporting: glossary, conflicts)
- The connected GitHub repository

## Gate (run first, stop on failure)
1. If `02-features/` or `02-edge-cases.md` is missing, stop and reply: "Run /identify-edge-cases first."
2. If the report has `status: BLOCKED`, or `decided` < `edge_cases`, or any Scenario in `02-features/` is still tagged `@pending`, stop and list the undecided `EC-n` items.
3. Drift check against `repo_commit`, as in phase 2.

## Decisions are inputs, not suggestions
- The **conflict resolutions** (`C-n`, from phase 1) and the **edge-case decisions** (`EC-n` Decision A–E, from phase 2) were made by the user. Assess the feasibility of the option the user CHOSE, not the one you would prefer.
- If a chosen option is `CONFLICT` or `XL`, say so in the finding and name the cheaper alternative among the other 3 options already listed for that EC. Do not switch the decision yourself; put the alternative under "Recommended requirement changes" for the user to reconsider in phase 4.
- Every `C-n` resolved as "Replace" or "Coexist" needs its own finding (migration, compatibility, flag wiring).

## Procedure

1. **Inventory the infrastructure (build once, cite everything).** Scan the relevant files and fill in this table:
   | Layer | Where to look |
   |---|---|
   | Runtime & frameworks | manifests, lockfiles, `Dockerfile`, `.tool-versions`/`.nvmrc` |
   | Data stores | ORM models, `migrations/`, `schema.*`, DB config, cache config |
   | Messaging / jobs | queue libraries, worker definitions, cron/scheduler config |
   | External services | SDK clients, `env.example`/config keys, API wrappers |
   | Auth | auth middleware, policy/permission modules, session/token handling |
   | Deployment & CI | `.github/workflows/`, `infra/`, `terraform/`, `k8s/`, `helm/`, `serverless.*` |
   | Observability | logging, metrics, tracing, error reporting setup |
   | Feature flags / config | flag library, config loaders |
   | Conventions | similar existing features to reuse as a pattern (the strongest evidence of feasibility) |
   Record anything that is absent as well, since a missing capability is a finding in its own right.

2. **Find prior art.** For each requirement, search for the closest existing feature that does something similar (e.g. an existing CSV export when the request is for a PDF export). Prior art is the best predictor of effort and of the pattern to follow.

3. **Assess each Gherkin `Rule` (`R-n`), each resolved conflict (`C-n`), and each edge-case Scenario's `Then` (`EC-n` with `@severity:critical`/`@severity:high`, plus any Medium/Low case tagged `@decision:A` or `@decision:B`).** The Scenario steps are the behavior you are checking feasibility for. Read each `Then` literally and ask: can the system produce this observable outcome, and what is missing? Produce a finding `F-n` containing:
   - **Verdict:**
     - `SUPPORTED`: works with existing components; config or small wiring only
     - `EXTEND`: existing components need moderate changes (new fields, endpoints, jobs)
     - `NEW-CAPABILITY`: needs a new component, service, library, or infrastructure
     - `CONFLICT`: contradicts the current architecture, a constraint, or another requirement
     - `UNKNOWN`: cannot be determined from the repository; needs a spike or human input
   - **Evidence:** file:line citations for what exists and what is missing
   - **Required changes:** the concrete list of touch points (files/modules, migrations, new dependencies, infrastructure resources)
   - **Effort:** S (≤1 day) / M (2–5 days) / L (1–2 weeks) / XL (>2 weeks), with one line of justification
   - **Risks:** migration on large tables, breaking API contracts, performance hotspots, security surface, vendor limits
   - **Options** (only for NEW-CAPABILITY / CONFLICT): 2–3 alternatives with trade-offs, plus a recommended option

4. **Check cross-cutting concerns** across the whole feature:
   - Backwards compatibility (public APIs, DB schema, clients)
   - Performance budget versus the current patterns (N+1 queries, sync work that should be async)
   - Security and privacy impact
   - Test strategy: which test layers already exist and what can be reused
   - Rollout: is a feature flag available? Does it need a migration sequence?

5. **Record dependencies and sequencing.** Note which F-n must happen before others (e.g. schema migration → API → UI).

6. **Give an overall verdict.**
   - `GO`: everything is SUPPORTED/EXTEND and there are no unresolved CONFLICTs
   - `GO WITH CHANGES`: some NEW-CAPABILITY, each with a recommended option
   - `RE-SCOPE`: CONFLICTs or XL items where cutting or changing a requirement is recommended (say which R-n to change)
   - `NO-GO`: fundamentally incompatible; explain why
   Set `status: BLOCKED` if the verdict depends on an `UNKNOWN` that only a human can resolve, and ask the user. Otherwise set READY.

7. **Annotate the Gherkin feature files.**
   - Copy `02-features/` to `03-features/` and update the header comment to `phase: 3`. Never edit earlier folders.
   - Do NOT change any step text. Phase 3 adds tags and comments only.
   - On each `Rule` and each assessed Scenario, add the tags `@F-n @verdict:<supported|extend|new-capability|conflict|unknown> @effort:<S|M|L|XL>`.
   - Add a comment directly above each tagged item: `# F-n: <one-line summary> | touches: <main paths> | risk: <top risk>`.
   - If a Scenario's `Then` is `CONFLICT` or `XL`, also tag it `@needs-review` and add `# Alternative: EC-n option <B|C|…> (<one-line behavior>)`, so phase 4 can ask the user.
   - At the Feature level, add `@effort:<total range>` and a comment giving the overall verdict for that Feature.
   - Validate: every Rule has an `@F-n` tag, the step text is byte-identical to `02-features/` (check with `diff` while ignoring tag and comment lines), and the files parse.

## Rules
- Every verdict needs evidence. "Probably fine" is not allowed; use `UNKNOWN` instead.
- Estimates are ranges for a single engineer familiar with the codebase. State that assumption.
- Do not write implementation code. Pseudocode of at most 5 lines is allowed only to clarify an option.
- Do not alter R-n or EC-n. If a requirement should change, recommend the change in the finding; phase 4 applies it.
- Prefer reusing existing patterns over introducing new libraries. If you recommend a new dependency, name it, its version, and the reason the existing tools are insufficient.

## Output

### Primary handoff: `.pipeline/<draft-slug>/03-features/<feature-slug>.feature`
These are the phase 2 features with feasibility tags and comments added. The step text is unchanged. The header comment becomes:
```gherkin
# draft: <draft-slug> | phase: 3 | status: READY | BLOCKED | verdict: <overall>
# repo_commit: <sha> | created: <date>
```
Example:
```gherkin
  # F-2: no background job runner; needs a queue worker | touches: src/jobs/, package.json | risk: new infra
  @R-2 @F-2 @verdict:new-capability @effort:L
  Rule: A finance user can export all invoices for a month
```

### Supporting report: `.pipeline/<draft-slug>/03-feasibility.md`

```markdown
---
draft: <draft-slug>
phase: 3
status: READY | BLOCKED
input: 02-features/, 02-edge-cases.md
repo_commit: <sha>
created: <date>
overall_verdict: GO | GO WITH CHANGES | RE-SCOPE | NO-GO
---

# Feasibility: <Feature title>

## Verdict
**<overall verdict>**: <2–3 sentence rationale>
Total effort: <range>, assuming one engineer familiar with the codebase.

## Infrastructure inventory
| Layer | What exists | Location | Relevant? |

## Prior art
- <existing feature> (`path/`): reusable for R-1, R-3

## Findings
| ID | Feature | Covers | Verdict | Effort | Top risk |
|---|---|---|---|---|---|
| F-1 | bulk-invoice-export | R-1 | SUPPORTED | S | ... |
| F-2 | bulk-invoice-export | R-2, EC-3 | NEW-CAPABILITY | L | ... |

### F-2: <title>
- **Covers:** R-2, EC-3
- **Verdict:** NEW-CAPABILITY
- **Evidence:** `src/jobs/` has no queue worker (`package.json:12` has no queue lib) ...
- **Required changes:** ...
- **Effort:** L, because ...
- **Risks:** ...
- **Options:**
  1. ... (recommended)
  2. ...

## Cross-cutting concerns
- Backwards compatibility: ...
- Performance: ...
- Security/privacy: ...
- Testing: ...
- Rollout: ...

## Sequencing
F-3 → F-1 → F-2 ...

## Recommended requirement changes (for phase 4)
- R-4: descope to ... because F-5 CONFLICT
- New NFR needed: ... from EC-2 / F-2

## Open questions
- Q-n [BLOCKING|NON-BLOCKING]: ...
```

## Handoff
End your reply with:
`Phase 3 complete: <verdict>, <effort range> → run /final-gherkin-feature-list <draft-slug>`
