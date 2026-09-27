# Requirements Pipeline

**From rough feature idea to codebase-validated, test-ready requirements.**

The Requirements Pipeline is a system of four Claude skills that transforms an ambiguous feature request into final Gherkin requirements—grounded in your actual codebase.

Instead of jumping from **idea → ticket → development**, the pipeline systematically validates what should be built, what could go wrong, whether it can be supported by the existing system, and how it should be tested.

## The Problem

Feature requests rarely arrive development-ready.

They start as loose ideas with:

- Ambiguous requirements
- Missing edge cases
- Assumptions about existing functionality
- No validation against the current architecture
- Acceptance criteria that leave room for interpretation

Those gaps become **rework, bugs, scope disputes, and engineering surprises** later in the lifecycle.

## The Pipeline

### 01 — Refine the Draft
Turn the raw request into structured requirements.

Claude translates the idea into Gherkin, traces terminology and functionality into the existing codebase, and surfaces contradictions or assumptions.

**You decide how each conflict should be resolved.**

### 02 — Identify Edge Cases
Trace the relevant code paths to uncover realistic failure modes, boundary conditions, and overlooked scenarios.

For every edge case, Claude presents four possible strategies:

**Prevent · Handle Gracefully · Allow & Inform · Defer**

Your decision becomes part of the requirements.

### 03 — Evaluate Feasibility
Validate every scenario against the existing infrastructure and similar implementations already in the codebase.

Each requirement is assessed for:

**Feasibility · Effort (S–XL) · Dependencies · Risks**

This exposes implementation constraints **before development begins.**

### 04 — Generate Final Gherkin
Turn the validated decisions into clean, handover-ready `.feature` files.

The final output includes:

- Numbered functional requirements (**FR-n**)
- Feature / Rule / Scenario structure
- Acceptance criteria
- Non-functional requirements
- Traceability across pipeline stages
- Step language aligned with existing test conventions

## Why It’s Different

**Code-grounded**  
Claude validates claims against the connected GitHub repository instead of reasoning from the feature description alone.

**Human-in-the-loop**  
Claude identifies conflicts and recommends approaches. **You make the product decisions.**

**Fully traceable**  
Every final scenario can be traced back to the original requirement, discovered edge case, and feasibility assessment.

**Test-ready**  
Requirements follow the conventions already used by your codebase, making them directly usable by engineering and QA.

## The Output

You end with a set of **repository-ready `.feature` files**—plus a summary covering:

**Scope · Risks · Rollout · Dependencies · Open Questions**

The result is more than a refined ticket.

**It is a codebase-validated specification that engineering and QA can build from.**
