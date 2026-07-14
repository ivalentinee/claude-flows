# Implementation Flow — Instructions for Claude

## Denote Metadata System

Read and apply `DENOTE.md` (in this directory) alongside this flow.
DENOTE.md specifies: front matter schema, naming conventions, status
transitions, convergence gate, section heading standards, and the
`denote-query` script interface. DENOTE.md naming rules supersede
naming patterns in this flow file. Denote behavior is mandatory
unless the project's CLAUDE.md contains `denote: disabled`.

---

A structured process for implementing features after the Design flow
is finalized. Every feature goes through the same steps: implement,
self-review, fix, then iterate with the user via review-plan and
review-notes.

---

## Prerequisites

- A finalized design exists at `designs/<feature>.org` (produced by
  the Design flow).
- If no finalized design is found, or the target feature is ambiguous,
  suggest starting the Design flow first or ask the user to specify
  which feature to implement.
- The last feature designed is assumed to be the target. If unsure,
  ask the user.

---

## Files

All implementation files live in `designs/<feature>-design/` (the
same working directory used by the Design flow — recreated if it
was cleaned up during design finalization).

| File                       | Purpose                                                          |
|----------------------------|------------------------------------------------------------------|
| `impl-plan.org`            | Two-section implementation plan: module relationships + build order. Temporary — deleted at finalize. |
| `review-plan.org`          | Emacs org-mode file listing changed files grouped by concern     |
| `review-notes.org`         | User's review notes — first-level headings per file              |
| `review-notes-resolved.org`| Archive of resolved review notes with resolution explanations    |
| `review.org`               | Self-review output (persists for user reference)                 |

**File format:** All working files use **Emacs Org mode** (`.org`), not Markdown. Use Org syntax: `*` / `**` / `***` for headings, `*bold*`, `/italic/`, `=verbatim=`, `~code~`, `- ` for lists, `- [ ]` for checkboxes. For code blocks use `#+begin_src lang` / `#+end_src`. Do NOT slip into Markdown syntax (`##` headings, `**bold**`, triple backticks). The flow instruction files themselves (this file) remain Markdown — only the per-feature working files are Org. All org artifacts include `#+STARTUP: overview` so only top-level headings are visible by default in org-mode editors.

---

## The Steps

### 1. Plan — automatic first step of `implement <feature-name>`

Before writing code, produce `impl-plan.org` using two
sequential subagents. This file has two sections.

**Step 1a — Module Relationship subagent.** Read the finalized design
and the existing codebase (supervision trees, module boundaries,
behaviours, protocols, data flow). Write section 1 of
`impl-plan.org`:

```org
* Module Relationships

** New Modules
(Each new module with its single responsibility.)

** Modified Modules
(Existing modules being touched and what changes in each.)

** Dependencies
(How modules relate: new→new, new→existing. Who calls whom,
who subscribes to what, who supervises whom.)

** Interfaces & Contracts
(Behaviours, protocols, struct shapes, and formal specs — JSON
Schema, OpenAPI, AsyncAPI — to define before implementation.
These are the integration seams.)

** Supervision Tree Placement
(Where new processes sit in the OTP supervision tree.)

** Data Flow
(Which process sends what to whom, through which mechanism —
direct call, PubSub, message queue, ETS.)
```

The Module Relationship subagent does NOT sequence work — it
describes the topology.

**Step 1b — Plan subagent.** Read the module relationships (output of
1a) and the finalized design. Write section 2 of
`impl-plan.org`:

```org
* Implementation Plan

** Step Order
(Numbered steps, each referencing modules from section 1.
Dependencies between steps noted — what blocks what.)

** Parallel Opportunities
(Which steps can be built simultaneously.)

** Integration Checkpoints
(After which steps the modules should be wirable end-to-end.
E.g. "after steps 1–3, the data layer and PubSub are connected
and can be tested together.")

** Formal Specs to Produce First
(Which JSON Schema / OpenAPI / AsyncAPI / behaviour definitions
should be written before any implementation code, because other
modules depend on them.)
```

### 2. Implement — automatic after step 1

**Style calibration:** Before writing code, sample the user's recent
commits (`git log --author` + read 1-2 touched files) to calibrate
naming, structure, and idiom choices. The user's code is the
authoritative style reference. Do this silently.

Follow the plan from `impl-plan.org`. For each step in order:
- Define interfaces and formal specs first (as identified in the plan)
- Write the implementation
- Write tests for the new code's key behaviors and edge cases — do not
  rely solely on existing tests to validate new code

**Test strategy:** prefer e2e tests that exercise real infrastructure
(DB, message queues, blob storage) over unit tests with mocks. Only
stub services that cannot be replicated locally (third-party APIs,
payment providers, external SaaS). If a dependency can run in
docker-compose or an emulator (e.g. Azurite for Azure Blob), test
against the real thing.

Before writing code, also consider:
- Whether infrastructure/exploitation info should be read from or
  updated in the project's exploitation doc

Run the project's test/lint suite.

**Trailing Abstraction Minimalist check.** After implementation is
complete but before the full review, run a lightweight Abstraction
Minimalist subagent that scans only the newly written/modified
functions. It reports inline to the main agent (no file output).
Fix any clear tier violations immediately — this is cheaper than
catching them in the full review after more code has adapted to
the mixed-level functions. Only flag violations where the tier
mismatch is unambiguous; borderline cases are left for the full
reviewer in step 3.

### 3. Parallel self-review — automatic after step 2

Launch specialized reviewer subagents **in parallel**, each with an
**adversarial stance** — assume the code is wrong and try to prove it.
Each reviewer is a different "person" than the author; same model, so
actively compensate for shared blind spots by checking every assumption
against the actual code, not memory.

| # | Role | Focus |
|---|------|-------|
| 1 | **Design Fidelity Reviewer** | Re-read the finalized design and diff intent vs. implementation. Flag any drift, missing pieces, undocumented deviations, or design decisions that were silently ignored. Verify preserved invariants from the intake are maintained |
| 2 | **Correctness Reviewer** | Logic errors, off-by-one mistakes, wrong assumptions, null/undefined handling, incorrect return types |
| 3 | **Edge Case Reviewer** | Race conditions, failure modes, error handling paths, boundary conditions, concurrent access |
| 4 | **Test Reviewer** | Are there behaviors or branches not covered by the new tests? Are tests hitting real infrastructure where possible, or unnecessarily mocking things that can run locally? Suggest missing test cases |
| 5 | **Spec Reviewer** | Are formal specifications (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migrations) implemented correctly? Do runtime validations match the declared schemas? |
| 6 | **Style Reviewer** | Naming consistency, pattern consistency with the existing codebase, code organization, module boundaries |
| 7 | **Principles Reviewer** | Check code against Elixir-adapted SOLID/GRASP principles (see checklist below). Flag violations as smells, not mandates — the goal is to catch problems, not enforce maximal adherence |

**Principles checklist** (for reviewer #7 — smell detector, not design mandate):
- **SRP** — does each module have one reason to change? Flag modules that mix concerns (e.g. data access + business logic + serialization)
- **DIP** — does code depend on behaviours/protocols where it should, or is it coupled to concrete modules that could be swapped?
- **Information Expert** (GRASP) — is behavior placed where the data lives, or is data being extracted and processed elsewhere?
- **Low Coupling** (GRASP) — are modules connected through narrow, well-defined interfaces? Flag modules that reach into each other's internals
- **High Cohesion** (GRASP) — do module contents belong together, or is the module a grab-bag of loosely related functions?
- **Creator** (GRASP) — does the module that has initialization data create the struct/process, or is creation responsibility misplaced?

Skip LSP (inheritance-oriented), ISP (Elixir behaviours are already minimal), OCP (leads to premature extension points). Do NOT recommend abstractions that don't serve an immediate need — three similar lines of code is better than a premature abstraction.

| 8 | **Abstraction Minimalist** | Check that abstraction levels are consistent within each function and module (see checklist below). Hunt for module split candidates: modules over ~150 lines with ≥2 tiers of ≥3 functions each. The goal is context minimization: reading a function should not require holding details from a different abstraction tier. Flag violations as extraction opportunities, not style complaints |

**Abstraction Minimalist checklist** (for reviewer #8):

*Function level — are all statements at the same tier?*
- A function that calls `Repo.get_worker/1` should not also do
  `String.split(raw, ",") |> Enum.map(&String.trim/1)` inline —
  the parsing operates at a lower tier than the orchestration
- A function handling HTTP concerns should not write to DB directly
- Complex loop/comprehension bodies that operate at a lower tier
  than the surrounding code should be extracted into a named function
  — but short, clear inline expressions at the right tier are fine
  (do NOT extract for extraction's sake)

*Module level — does the public API sit at one coherent tier?*
- A module exporting both `sync_worker_state/1` (coordination) and
  `parse_amqp_timestamp/1` (utility) has a leaky abstraction surface
- Public functions should read like a coherent API at one level;
  lower-tier helpers should be private

*Module split — can the module be decomposed by abstraction tier?*
- For modules over ~150 lines, actively look for a tier split:
  assign each function (public and private) a tier label (e.g.,
  "orchestration", "data transformation", "I/O", "formatting",
  "tracing/observability"). If two or more tiers each have ≥3
  functions AND ≥30 lines, the module is a split candidate.
- Group the lower-tier functions by micro-domain (e.g., tracing,
  result processing, registry/tracking, formatting). Each group
  with a coherent purpose becomes a sub-module.
- The high-tier module should read as pure orchestration after the
  split: its private functions call into sub-modules, never doing
  low-tier work inline.
- Do NOT split when: the module is under ~150 lines, the "low tier"
  is just 2-3 small helpers, or the split would create single-caller
  modules with no independent testability or reuse value.

*Cross-module — are callers forced to know implementation details?*
- If a caller must understand the internal data layout, parsing
  format, or storage mechanism of another module to use it, the
  boundary is at the wrong tier

Do NOT recommend extraction when the inline code is short, clear,
and at the same tier as its surroundings. The goal is consistent
tiers, not maximum decomposition.

Each subagent reads the diff from step 2 and all relevant source files
but makes **no edits**. Each produces a structured report with severity
ratings (Critical / High / Medium / Low / Info).

Claude collates and deduplicates the reports into
`review.org` in the feature documentation directory. This file
persists — the user may consult it to understand what was auto-fixed.

### 4. Fix — automatic after step 3

Read `review.org`. For each issue:
- Fixable → apply the fix
- Blocking (design ambiguity, out of scope) → raise for discussion

**Trailing Abstraction Minimalist check.** After fixes are applied,
run the lightweight Abstraction Minimalist on the changed functions
only. Reviewer fixes can introduce tier mismatches (e.g. inlining
detail to fix a correctness issue). Fix clear violations before
proceeding.

Re-run the test/lint suite.

### 5. Review plan + review notes — automatic after step 4

Create `review-plan.org` in the feature documentation directory.

**Format:**
- First-level headings (`*`) for each semantically coherent group of
  changes (e.g. "Cache storage spans", "TS wrapper changes", "Tests").
- Each changed file is a `- [ ]` checkbox list item with an org-mode
  file link (`[[file:path][path]]`). Review points for that file are
  listed as sub-items (indented `- ` under the checkbox).
- Deleted files are listed without checkboxes (nothing to review).
- Unrelated / pre-existing changes get their own section marked
  "do not review", without checkboxes.
- Group files by logical concern, not by directory.

Then create `review-notes.org` with first-level headings for each file
from `review-plan.org`, listed in the same order. Each heading uses
the full path from the project root, wrapped in `=verbatim=` — e.g.
`** =path/to/file.ex=`. The user fills in notes under each heading.

### 6. Iterate — `loop`

**Step 6a — Apply fixes.** Read `review-notes.org`. For each note:
- Apply the fix or change requested
- Move the note to `review-notes-resolved.org`, appending a
  `**Resolved:**` subsection explaining what was done

**Trailing Abstraction Minimalist check.** After applying fixes, run
the lightweight Abstraction Minimalist on changed functions only.
User-requested changes can introduce tier mismatches. Fix clear
violations before validation.

**Step 6b — Fix Validator subagent.** After applying all fixes, launch
a subagent that reads the diff of changes made in this iteration,
`review-notes.org`, and the finalized design. It checks:
- Did each fix actually address the review note it claims to resolve?
- Did any fix introduce new issues or regress previously working code?
- Do the changes still match the finalized design?

Issues found are reported to the user alongside the review-plan update.

**Step 6c — Update review plan.** Update `review-plan.org`:
- If a file was modified by this iteration: uncheck its `- [ ]` item
  and update the review description to reflect only the latest change
  (so the user reviews only what changed, not the whole file again).
- If a file was not modified: keep its existing check state.
- If new files were created: add them to the appropriate section.
- If files were removed: move them to a "deleted" list (no checkbox).

Re-run the test/lint suite.

If `review-notes.org` still has unresolved notes, suggest **`loop`**.
If all notes are resolved, suggest **`finalize`**.

### 7. Finalize — `finalize`

**Step 7a — Completeness Verifier subagent.** Before deleting files,
launch a subagent that reads the finalized design and the current
implementation. It checks:
- Does the implementation cover every aspect of the finalized design?
- Are there design requirements that were not implemented or were only
  partially implemented?
- Are all formal specs (JSON Schema, OpenAPI, AsyncAPI) from the design
  present in the codebase?
- Do all tests pass and cover the key behaviors described in the design?

If the verifier finds gaps, report them to the user before proceeding.
The user can choose to address them now (back to step 6) or defer them.

**Step 7b — Clean up.** Delete the entire working directory
(`designs/<feature>-design/`). The design doc remains as the
permanent record.

Suggest **`commit`**.

### 8. Commit — `commit`

Create a commit following the project's existing commit convention.

### 9. Retrospective (optional) — `retro`

Quick self-assessment after completing a full cycle. Report in 3–5
bullet points:
- What went smoothly vs. what caused friction
- How many loop iterations were needed and why
- Whether the design was sufficient or had gaps that surfaced during
  implementation
- Anything that should change in the design or implementation process

This is informational — no files are created. The user can save
relevant observations to memory if they want to refine the workflow.

---

## Mid-Flow Communication

These conventions apply throughout the flow.

### Effort Forecast

Before entering an autonomous phase, emit a one-line effort
estimate:

- **Light** — a single step, one or two subagents. "Stay here."
- **Moderate** — multiple steps or parallel subagents, a few
  minutes. "Check back shortly."
- **Heavy** — full multi-phase autonomous run (implement + 
  8-reviewer self-review + fix). "Good time to multitask."

### Context Re-establishment

After a **moderate** or **heavy** autonomous phase, open the next
user-facing message with a 2-3 line recap:

- What was being done
- What happened (key outcomes: issues found, fixes applied,
  tests passing/failing)
- What comes next

After a **light** pass, skip the recap.

### Step-Boundary Steering

Between flow phases, check if the user typed anything in chat.
Treat messages as inline amendments (additional context, scope
adjustments, corrections). Acknowledge briefly and incorporate
into the next step.

---

## Resuming Interrupted Work — `resume`

If the user returns to a partially completed implementation (e.g. new
session, interrupted work):

Report state in **human-readable terms** — what was done and
what happens next, never step numbers. The user should not need
to know the flow's internal structure.

Example: "Implementation of token search is complete. Self-review
found 3 issues (2 fixed, 1 needs your input). review-plan.org is
ready for your review."

1. Read all implementation files in `designs/<feature>-design/`
2. Check git status/diff to see what code changes exist
3. Report current state: which step was last completed, what remains
4. If `review-notes.org` has unprocessed notes → suggest **`loop`**
5. If implementation is done but no review files exist → run step 3
   (self-review) onward
6. If `impl-plan.org` exists but no code changes → resume
   from step 2 (implement)
7. If no files exist → start from step 1 (plan)

---

## When NOT to Use This Process

Skip the implementation flow for:
- **One-file changes** — a single function addition or bug fix
- **Config-only changes** — environment variables, feature flags
- **Changes already reviewed** — hotfixes where the user wrote and
  reviewed the code themselves

Use it when the implementation spans multiple files, involves
non-trivial logic, or when you want the structured review cycle.

---

## Guidelines (apply throughout all steps)

1. **Think before writing.** Before coding, consider what properties
   (correctness, cohesiveness, consistency) the code should have.
   Then write. Then verify those properties hold.

2. **Don't assume — ask.** If requirements are unclear, ask the user
   rather than guessing.

3. **Prefer documentation-based approaches.** When unsure, ask the
   user whether JSON Schema validation, generated types, or
   OpenAPI/AsyncAPI definitions are appropriate.

4. **Exploitation doc.** Read from and update the project's
   exploitation/infrastructure doc for any deployment-relevant
   changes. If no exploitation doc exists, suggest creating one.

---

## Standalone: Abstraction Check — `abstraction-check <target>`

A standalone command that runs the Abstraction Minimalist analysis on
any module or function outside of the Implementation Flow. Useful when
writing code manually and wanting an outsider perspective on
abstraction tier consistency.

**Input:** A target — a file path, a module name, or a
`Module.function/arity` reference. Multiple targets can be
space-separated.

**Process:** Launch an Abstraction Minimalist subagent that reads the
target code and its immediate callers/callees (to understand the
tier the code sits at in context). Applies the same analysis as
during the Implementation Flow — function-level tier consistency,
module-level API coherence, and cross-module tier leaks.

**Output:** A short inline report (no file created) with:
- **Tier map** — a brief description of what abstraction tier each
  public function operates at (e.g. "orchestration", "data access",
  "parsing/formatting", "transport")
- **Violations** — specific locations where the tier is inconsistent,
  with a suggested extraction or restructuring
- **Assessment** — overall: clean / minor issues / needs restructuring

This is informational and makes no edits. The user decides what (if
anything) to act on.

Examples:
- `abstraction-check lib/idunn_monitor/environment/amqp.ex`
- `abstraction-check IdunnMonitor.Environment.CRUD`
- `abstraction-check IdunnMonitor.Environment.CRUD.update_state/2`

---

## Trigger Phrases

- "Let's start the implementation flow"
- "Let's start the implementation"
- Typically follows finalization of the Design flow

---

## How to Use This Process

### Starting implementation

After finalizing a design: **`implement <feature-name>`**

Steps 1–5 run automatically: plan → implement → self-review → fix →
create review-plan.org + review-notes.org.

### User fills in review-notes.org, then

Prompt: **`loop`** — Claude applies fixes, updates review-plan.org,
archives resolved notes

### When all notes resolved

Prompt: **`finalize`** — Claude deletes review files

### After finalize

Prompt: **`commit`**
