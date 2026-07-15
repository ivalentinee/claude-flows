# Full Flow — Instructions for Claude

## Denote Metadata System

Read and apply `DENOTE.md` (in this directory) alongside this flow.
DENOTE.md specifies: front matter schema, naming conventions, status
transitions, convergence gate, section heading standards, and the
`denote-query` script interface. DENOTE.md naming rules supersede
naming patterns in this flow file. Denote behavior is mandatory
unless the project's CLAUDE.md contains `denote: disabled`.

---

A combined design-through-implementation flow that maximizes Claude
autonomy while enforcing every quality gate. Claude resolves design
questions autonomously unless they are critical; every review step is
mandatory and cannot be skipped.

---

## Principles

1. **Claude-autonomous design.** Claude acts as both Author and
   Decision-Maker during the design phase. Questions and criticisms
   are self-resolved unless they meet the Critical escalation
   criteria (see below). The user is consulted only when Claude
   genuinely cannot decide.

2. **Every gate is mandatory.** No review step is optional. The
   design review, implementation self-review, and user review all
   run unconditionally. The word "suggest" is replaced by "proceed
   to" — the flow advances through every step.

3. **Escalation, not permission.** Claude does not ask "should I
   continue?" between steps. It proceeds automatically and escalates
   only when blocked by a Critical item or a failing quality gate.

4. **Traceability.** Every autonomous decision is recorded with
   rationale in `resolved.org` so the user can audit Claude's
   reasoning at any point.

5. **Communicate effort.** Before entering an autonomous phase,
   emit an effort estimate (light / moderate / heavy) so the user
   knows whether to stay or multitask. After a moderate or heavy
   phase, re-establish context with a 2-3 line recap — assume the
   user was away and does not remember the last interaction.

---

## Criticality Classification

Every question, criticism, and finding is tagged with a severity that
determines whether Claude resolves it autonomously or escalates.

### Critical — escalate to user

Claude MUST escalate when the item involves:

- **Product/scope decisions** — what the feature should or should not
  do, which use cases to support, what is in or out of scope
- **UX/UI choices** — user-facing behavior, interaction patterns,
  visual design, wording
- **External contracts** — API shapes, message formats, or data
  schemas that other systems or teams depend on
- **Irreversible architectural choices** — decisions that would be
  expensive to change later (new database tables, new external
  dependencies, new public API surfaces)
- **Contradictions** — any proposed decision that conflicts with
  existing documented decisions, CLAUDE.md, or user-stated preferences
- **Ambiguous requirements** — when the design doc or user input is
  genuinely ambiguous and multiple interpretations lead to
  meaningfully different outcomes
- **Risk trade-offs** — when the choice involves accepting a known
  risk (data loss scenarios, security implications, performance
  degradation under specific conditions)

### Non-critical — Claude resolves autonomously

Claude resolves items that are:

- **Technical implementation details** — data structures, algorithms,
  internal module organization, function signatures for internal APIs
- **Code structure** — where to place modules, how to split files,
  naming of internal functions and variables
- **Error handling strategy** — within established project patterns
- **Test approach** — which test cases to write, how to structure
  test modules, what fixtures to create
- **Performance choices** — caching strategy, query optimization,
  data structure selection — when the trade-offs are technical, not
  product-level
- **Pattern selection** — choosing between equivalent Elixir/OTP
  patterns (GenServer vs Agent, ETS vs process state, etc.) when
  both are viable and the choice doesn't affect the public interface

When auto-resolving, Claude writes a `*Rationale:*` entry explaining
the decision and why it was not escalated. If in doubt about whether
an item is Critical, escalate — false escalations cost a question,
false auto-resolutions cost rework.

---

## Directory Structure

All design and implementation files live under the project's
`designs/` directory, organized by feature.

```
designs/
  <feature>.org                     # Root design doc (intake + design)
  <feature>-design/                 # Working files for this feature
    questions.org
    criticism.org
    answers.org                     # Created on demand for Critical items
    resolved.org
    critic-context.org
    boundary-context.org
    journal.org
    impl-plan.org                   # Implementation phase
    review.org
    review-plan.org
    review-notes.org
    review-notes-resolved.org
  <feature>/                        # Sub-features (if any)
    <sub-feature>.org               # Sub-feature design doc
    <sub-feature>-design/           # Sub-feature working files
      questions.org
      criticism.org
      ...                           # Same file set as parent
```

**Key rules:**

- **Root design docs** live directly in `designs/`.
- **Working files** (questions, criticism, journal, impl-plan, etc.)
  live in `designs/<feature>-design/`. File names are short — the
  directory provides the namespace (e.g. `questions.org`, not
  `<feature>-questions.org`).
- **Sub-feature docs** live in `designs/<feature>/`, each with its
  own `-design/` subdirectory for working files.
- **Sub-features are referenced, not merged.** When a sub-feature
  is finalized, the parent design doc gets a reference link — not
  the full content inlined. This keeps context small when working
  on a specific sub-feature.

---

## Intake Template

The `init` command creates `designs/<feature>.org`. It derives a
kebab-case filename from the name or description: strip filler
words, extract noun phrases, max 3-4 words. The user fills in what
they know; empty sections are fine.

```org
#+STARTUP: overview
* <Feature Name>

** Goal
(What this feature should accomplish and why — the user problem
it solves or the technical need it addresses.)

# The following sections are optional. If missing, the Author
# subagent infers from Goal and codebase context. Fill in what
# you know; leave empty what you don't.

** Constraints
(Non-goals, scope limits, things to avoid, compatibility
requirements. Explicit constraints reduce false auto-resolutions.)

** Preserved Invariants
# Optional. Existing behaviors, APIs, or data formats that must
# NOT break. The Critic and reviewers verify these are maintained.

** References
(Files, URLs, external docs, related features Claude should read
before designing. Pointers, not content.)

** Acceptance Criteria
(Specific, testable conditions. When is this feature "done"?
These give the Completeness Verifier concrete checkpoints.)

** Design
(Empty at init. Filled by the Author in Step 1.)

** Sub-features
(Empty at init. Populated with reference links as sub-features
are designed and finalized.)

** Known Deferred Work
(Empty at init. Items deferred during the design loop.)
```

The Goal and Constraints fields directly shape Claude's autonomous
decisions — they define what is in scope and what the feature should
NOT do, reducing the need for Critical escalations.

---

## Files

Working files live in `designs/<feature>-design/` (or
`designs/<feature>/<sub-feature>-design/` for sub-features).

### Design phase files

| File | Purpose |
|------|---------|
| `questions.org` | Open questions needing resolution |
| `criticism.org` | Open criticisms and concerns |
| `resolved.org` | Archive of resolved questions and criticisms with rationale |
| `critic-context.org` | Persistent Critic memory across iterations |
| `boundary-context.org` | Persistent Boundary Analyst memory |
| `journal.org` | Shared append-only design journal |

No `answers.org` file by default — Claude resolves non-critical items
directly. When Critical items exist and are escalated to the user,
the user writes answers in `answers.org` (created on demand with only
the Critical items listed).

### Implementation phase files

| File | Purpose |
|------|---------|
| `impl-plan.org` | Module relationships + build order |
| `review.org` | Self-review output (persists for user reference) |
| `review-plan.org` | Changed files grouped by concern |
| `review-notes.org` | User's review notes per file |
| `review-notes-resolved.org` | Archive of resolved review notes |

**File format:** All working files use **Emacs Org mode** (`.org`).
Use Org syntax: `*` / `**` / `***` for headings, `*bold*`, `/italic/`,
`=verbatim=`, `~code~`, `- ` for lists, `- [ ]` for checkboxes. For
code blocks use `#+begin_src lang` / `#+end_src`. Do NOT use Markdown
syntax in working files. All org artifacts include `#+STARTUP: overview`
so only top-level headings are visible by default in org-mode editors.

---

## Phase 1: Design

### Step 1 — Author

Launch an Author subagent. Read `designs/<feature>.org` (the intake
doc — Goal, Constraints, References, Acceptance Criteria) and the
relevant source files (including any files listed in References).

Write into the **Design** section of `designs/<feature>.org` — the
proposed design. **Documentation-first:** define behavior through
formal specs (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB
migration schemas) where applicable. Prefer declarative,
machine-readable definitions over prose.

**Eager decomposition:** Proactively look for opportunities to split
the feature into independently implementable sub-features. If the
feature spans multiple modules or has multiple independently
verifiable outcomes, propose an overarching design with sub-feature
references rather than a monolithic design. Each sub-feature gets
its own `designs/<feature>/<sub-feature>.org` and goes through its
own design → implement cycle. Do not decompose trivially small
features — decompose when it makes reviews smaller and acceptance
criteria more checkable.

Write `designs/<feature>-design/questions.org` — genuine unknowns.
**Tag each item** with `[Critical]` or `[Auto]` per the Criticality
Classification. Items already answered by the intake's Goal,
Constraints, or Acceptance Criteria should not become questions.

Initialize `designs/<feature>-design/journal.org` with an
"Iteration 1 — Author" entry.

### Step 2 — Critic + Boundary Analyst (parallel)

Launch both subagents in parallel. Both read the design doc,
questions, and the same source files.

**Critic subagent:**
- Write `criticism.org` in the working directory — concerns, gaps,
  risks. **Tag each item** `[Critical]` or `[Auto]`.
- Write `critic-context.org` — initialize active concerns.
- **Challenge monolithic designs:** if the feature could be split
  into independently deliverable sub-features but wasn't, flag
  this as a concern. Decomposition makes reviews smaller and
  acceptance criteria more checkable.
- **Verify preserved invariants:** if the intake lists invariants,
  check whether the proposed design could violate them.
- May add items to `questions.org` (tagged).
- Append journal entry.

**Boundary Analyst subagent:**
- Write `boundary-context.org` in the working directory — subsystem
  boundaries, interfaces, formalization recommendations, distillation
  opportunities, independence assessments.
- May add items to `questions.org` (tagged).
- Append journal entry.

### Step 3 — Auto-resolve

Process all `[Auto]`-tagged items in `questions.org` and
`criticism.org`:

For each item:
1. Decide on a resolution based on codebase context, design
   constraints, and established patterns
2. Update the design doc to reflect the decision
3. Move the item to `resolved.org` with `*Rationale:*` and
   `*Resolved by:* Claude (auto)` annotation
4. If resolving one item creates a new question or concern, add it
   (tagged) to the appropriate file

After processing, report the net change: "Auto-resolved N items.
M Critical items remain for user input." (or "0 Critical items —
design is self-consistent.")

### Step 4 — Auto-loop (up to 3 iterations)

After auto-resolving, launch Critic + Boundary Analyst re-evaluation
(parallel) — same as the original Design flow step 3c:

**Critic re-evaluation:** reads its context, updated design,
resolved archive, journal. Moves genuinely resolved concerns out,
adds new concerns (tagged), updates patterns, appends journal entry.
Escalates blocking concerns unaddressed for 2+ iterations.

**Boundary Analyst re-evaluation:** reads its context, updated
design, resolved archive, journal. Updates subsystem boundaries,
logs changes, updates tensions, appends journal entry.

Then auto-resolve any new `[Auto]` items. Repeat until either:
- No `[Auto]` items remain (converged), or
- 3 auto-resolve iterations have passed (circuit breaker — escalate
  remaining `[Auto]` items to `[Critical]` to prevent infinite loops)

**Convergence check:** report net change per iteration. If an
iteration produces more new items than it resolves, flag this — the
feature may need splitting.

### Step 5 — Escalate (if Critical items exist)

If any `[Critical]` items remain:

1. Create `answers.org` in the working directory with the
   three-section structure (Considerations, Questions, Criticism),
   pre-populated with only the `[Critical]` items
2. Present a summary to the user: list each Critical item with a
   one-line description and why it needs user input
3. **Wait for user** to fill in `answers.org`

If zero Critical items remain, skip directly to Step 6.

### Step 5a — Process user answers

When the user provides answers:

1. **Apply answers** — update design doc, move resolved items to
   `resolved.org`
2. **Validator subagent** — check answers for vagueness,
   contradictions, implicit new questions
3. **Critic + Boundary Analyst re-evaluation** (parallel)
4. Auto-resolve any new `[Auto]` items
5. If new `[Critical]` items emerged, return to Step 5
6. If converged, proceed to Step 6

### Step 6 — Design review (MANDATORY)

This step is **not optional**, even when the user has asked to skip
user review. It must be orchestrated from the **main conversation**,
not delegated inside a single subagent. Launch up to 7 specialized
reviewer subagents in parallel. Before reporting on any file, each
subagent verifies the file exists at the referenced path.

| # | Role | Focus |
|---|------|-------|
| 1 | **Correctness Reviewer** | Verify claims against actual code, find missing operations or mismatched signatures. Flag contracts that could be schemas but aren't |
| 2 | **Edge Case Reviewer** | Concurrent access, race conditions, failure modes, missing error handling |
| 3 | **Integration Reviewer** | Caller/callee contracts, ETS access patterns, cross-module consistency |
| 4 | **API Reviewer** | Naming consistency, function signatures, module boundaries |
| 5 | **Test Reviewer** | Test sufficiency, untested code paths, e2e preference, specific test cases |
| 6 | **Spec Reviewer** | Formal spec completeness and validity, prose-only contracts that should be specs |
| 7 | **Boundary Reviewer** | Subsystem boundaries, interface narrowness, formalization levels, cross-boundary leaks |

Additionally, include the **UX Reachability & Visual Quality Reviewer**
(per CLAUDE.md) when the feature has UI components.

Each reviewer produces a structured report with severity ratings
(Critical / High / Medium / Low / Info).

**Processing review findings:**
- `[Auto]`-level findings (High and below): Claude fixes the design
  immediately, documents in `resolved.org`
- `[Critical]`-level findings: escalate to user (return to Step 5)
- If review introduced design changes: run one more Critic +
  Boundary Analyst re-evaluation, then re-check convergence

After the review is clean (no Critical findings, all High+ addressed),
proceed to Step 7.

### Step 7 — Finalize design

**For root features:**

1. **Consolidate** — ensure the Design section of
   `designs/<feature>.org` is complete and self-contained. All
   resolved decisions, boundary definitions, and formal specs are
   represented in the design doc itself.
2. **Finalization Verifier subagent** — diff the design doc against
   `resolved.org` + `journal.org` in the working directory. Check
   all resolved decisions and their rationales are represented,
   formal specs are accounted for, criticism resolutions are
   reflected, and boundary definitions are clear. Patch gaps.
3. **Clean up** — delete `designs/<feature>-design/` (the entire
   working directory). The design doc `designs/<feature>.org`
   remains as the permanent record.

**For sub-features:**

1. **Consolidate** — ensure the sub-feature design doc
   `designs/<feature>/<sub-feature>.org` is complete.
2. **Reference in parent** — add a reference link in the parent
   design doc's **Sub-features** section:
   `[[file:<feature>/<sub-feature>.org][<Sub-feature name>]]`
   with a one-line summary. Do NOT inline the sub-feature content
   into the parent — the link is the interface.
3. **Verifier + clean up** — same as root features, but for the
   sub-feature's working directory.

**Step 7d — Context Maintenance subagent.** Check whether design
decisions affect project context documented in CLAUDE.md:
- New modules or directories → update Key Directories
- Changed data flow or architecture → update relevant sections
- New conventions specific to this project → update Conventions

The subagent updates project context only — NOT process instructions.
Process lives in flows (generic, portable across projects). CLAUDE.md
provides project-specific context and flow amendments. If the subagent
detects process instructions in CLAUDE.md that belong in a flow file,
flag this for the user.

**Do NOT wait for user confirmation.** Proceed directly to Phase 2.

---

## Phase 2: Implementation

### Step 8 — Plan

Two sequential subagents produce `impl-plan.org` in the working
directory:

**Step 8a — Module Relationship subagent.** Read finalized design +
existing codebase. Write section 1:
- New modules, modified modules, dependencies, interfaces &
  contracts, supervision tree placement, data flow

**Step 8b — Plan subagent.** Read module relationships + finalized
  design. Write section 2:
- Step order, parallel opportunities, integration checkpoints,
  formal specs to produce first

### Step 9 — Implement

**Style calibration:** Before writing code, sample the user's recent
commits (`git log --author` + read 1-2 touched files) to calibrate
naming, structure, and idiom choices. The user's code is the
authoritative style reference. Do this silently.

Follow the plan. For each step:
1. Define interfaces and formal specs first
2. Write the implementation
3. Write tests (prefer e2e with real infrastructure; only stub
   non-replicable external services)

Run the project's test/lint suite (`mix paranoid`).

**Trailing Abstraction Minimalist check.** After implementation,
run a lightweight Abstraction Minimalist subagent on newly
written/modified functions. Fix clear tier violations immediately.

### Step 10 — Self-review (MANDATORY)

This step is **not optional**, even when the user has asked to skip
user review (Steps 12-13). It must be orchestrated from the **main
conversation** — a single delegated subagent cannot launch 8 parallel
reviewers. When implementation is delegated to a subagent, return
control to the main conversation before running this step.

Launch all 9 specialized reviewer subagents in parallel, each with
an adversarial stance:

| # | Role | Focus |
|---|------|-------|
| 1 | **Design Fidelity Reviewer** | Diff intent vs. implementation — flag drift, missing pieces, silent deviations |
| 2 | **Correctness Reviewer** | Logic errors, off-by-one, wrong assumptions, null handling, incorrect return types |
| 3 | **Edge Case Reviewer** | Race conditions, failure modes, error handling, boundary conditions, concurrency |
| 4 | **Test Reviewer** | Uncovered behaviors/branches, unnecessary mocking, missing test cases |
| 5 | **Spec Reviewer** | Schema correctness, runtime validation matching declared schemas |
| 6 | **Style Reviewer** | Naming consistency, pattern consistency with existing codebase, organization |
| 7 | **Principles Reviewer** | SRP, DIP, Information Expert, Low Coupling, High Cohesion, Creator (smell detection, not mandates) |
| 8 | **Abstraction Minimalist** | Function-level tier consistency, missing intermediate tiers (Composed Method — ≥6 detail calls without grouping), module-level API coherence, cross-module tier leaks, module split candidates (hunt for ≥2 tiers with ≥3 functions each in modules over ~150 lines — propose sub-module decomposition by micro-domain) |
| 9 | **Code Clarity Reviewer** | Self-documenting quality: naming reveals intent (not just consistent), types express constraints, comments explain "why" not "what", control flow is self-evident. Check against CODE-QUALITY.md "Self-Documenting Code" checklist |

Each produces a structured report (Critical / High / Medium / Low /
Info). Claude collates and deduplicates into `review.org` in the
working directory.

### Step 11 — Fix

Read `review.org`. For each issue:
- Fixable: apply the fix
- Blocking (design ambiguity, out of scope): escalate to user

**Trailing Abstraction Minimalist check** on changed functions.

Re-run `mix paranoid`.

### Step 12 — Review plan + review notes

Create `review-plan.org` and `review-notes.org` in the working
directory.

**`review-plan.org` format:**
- First-level headings for each semantic group of changes
- Each changed file is a `- [ ]` checkbox with org-mode file link
  and review points as sub-items
- Deleted files listed without checkboxes
- Unrelated/pre-existing changes in a "do not review" section

**`review-notes.org` format:**
- First-level headings for each file from `review-plan.org`, in
  the same order, using `=verbatim=` paths

Present both files to the user. **Wait for user** to fill in
`review-notes.org`.

### Step 13 — User review loop

**Step 13a — Apply fixes.** For each note in `review-notes.org`:
- Apply the fix
- Move to `review-notes-resolved.org` with `**Resolved:**` subsection

**Trailing Abstraction Minimalist check** on changed functions.

**Step 13b — Fix Validator subagent.** Check:
- Did each fix address its review note?
- Did any fix introduce new issues or regressions?
- Do changes still match the finalized design?

**Step 13c — Update review plan.** Update `review-plan.org`:
- Modified files: uncheck, update description to latest change only
- Unmodified files: keep existing check state
- New files: add to appropriate section
- Removed files: move to deleted list

Re-run `mix paranoid`.

If `review-notes.org` has unresolved notes, prompt **`loop`**.
If all resolved, proceed to Step 14.

### Step 14 — Finalize implementation

**Step 14a — Completeness Verifier subagent.** Check:
- Does implementation cover every design aspect?
- Are there partially implemented requirements?
- Are all formal specs present in the codebase?
- Do all tests pass and cover key behaviors?

If gaps found: report to user, return to Step 13 if user wants to
address them now, or document as deferred.

**Step 14b — Context Maintenance subagent.** Check whether the
implementation changed anything documented in CLAUDE.md:
- New modules or directories → update Key Directories
- New data flow patterns → update Data Flow section
- New environment variables → update Environment Variables
- New project-specific conventions → update Conventions

Same rules as Step 7d: update project context only, not process
instructions. CLAUDE.md changes appear in review-plan.org if the
user hasn't reviewed yet, or are visible in the commit diff.

**Step 14c — Clean up.** Delete the entire working directory
(`designs/<feature>-design/` or the sub-feature equivalent). The
design doc remains as the permanent record.

### Step 15 — Commit

Create a commit following the project's existing commit convention.

---

## Commands

| Command | Action |
|---------|--------|
| `init <name-or-description>` | Create `designs/<feature>.org` with the intake template and `designs/<feature>-design/` directory. Derives kebab-case filename (strip filler words, noun phrases, max 3-4 words). Does not start the flow — the user fills in the intake fields first. |
| `full [<feature-name>]` | Start from Step 1. Reads the intake doc, then runs through all phases automatically, pausing only for Critical escalations (Step 5) and user review (Step 12). If `<feature-name>` is omitted, infer from the most recently initialized or active feature. |
| `loop` | In design phase: process `answers.org` (Step 5a). In implementation phase: process `review-notes.org` (Step 13). |
| `finalize` | In design phase: run Step 7. In implementation phase: run Step 14. |
| `commit` | Run Step 15. |
| `resume` | Detect current state from existing files + git status, report where we are, and continue from the appropriate step. |
| `retro` | Post-completion retrospective (3-5 bullets, no files created). |

The flow **does not expose** `review` as a standalone command — the
design review (Step 6) and implementation self-review (Step 10)
always run as part of the flow.

---

## Mid-Flow Communication

These conventions apply throughout the flow.

### Effort Forecast

Before entering an autonomous phase, emit a one-line effort
estimate:

- **Light** — a single step, one or two subagents. "Stay here."
- **Moderate** — multiple steps or parallel subagents, a few
  minutes. "Check back shortly."
- **Heavy** — full multi-phase autonomous run. "Good time to
  multitask."

### Context Re-establishment

After a **moderate** or **heavy** autonomous phase, open the next
user-facing message with a 2-3 line recap:

- What was being done
- What happened (key outcomes)
- What comes next

After a **light** pass, skip the recap.

### Step-Boundary Steering

Between flow phases, check if the user typed anything in chat.
Treat messages as inline amendments (additional context, scope
adjustments, corrections). Acknowledge briefly and incorporate
into the next step.

---

## Session Resumption — `resume`

All state lives in the files. When resuming:

Report state in **human-readable terms** — what was found and
what happens next, never step numbers. The user should not need
to know the flow's internal structure.

1. Scan `designs/` for feature docs and working directories
2. Read all existing files for the feature
3. Check git status/diff for code changes
4. Determine phase and step:
   - Design doc exists, no `-design/` directory → just initialized,
     user may still be filling intake. Ask or run `full`.
   - `-design/` exists with design files, no impl files → Design phase
   - `impl-plan.org` exists, no code changes → Step 9 (implement)
   - Code changes exist, no review files → Step 10 (self-review)
   - `review-notes.org` has content → Step 13 (user review loop)
   - `answers.org` has content → Step 5a (process answers)
5. Report current state and continue

---

## Flow Diagram

```
  init ──► designs/<feature>.org (intake template)
       ──► designs/<feature>-design/ (working dir)
       │
       ▼ (user fills intake, then runs `full`)

Phase 1: Design
  Step 1: Author (reads intake) ───────────────────┐
  Step 2: Critic + Boundary Analyst (parallel) ─────┤
  Step 3: Auto-resolve [Auto] items ────────────────┤
  Step 4: Auto-loop (up to 3x) ◄───────────────────┘
       │
       ├─ 0 Critical items ─────────────────────────┐
       │                                             │
       └─ Critical items exist                       │
            Step 5: Escalate to user                 │
            Step 5a: Process answers ──► loop back   │
                     if new Critical items           │
       │                                             │
       ◄─────────────────────────────────────────────┘
       │
  Step 6: Design review (MANDATORY, 7 reviewers)
       │
       ├─ Critical findings ──► Step 5 (escalate)
       └─ Clean
       │
  Step 7: Finalize design (delete -design/, keep .org)
       │
       ▼ (automatic — no pause, recreates -design/ for impl)

Phase 2: Implementation
  Step 8: Plan (module relationships + build order)
  Step 9: Implement + tests + mix paranoid
  Step 10: Self-review (MANDATORY, 8 reviewers)
  Step 11: Fix + mix paranoid
  Step 12: Review plan + notes ──► wait for user
  Step 13: User review loop ◄──── loop until clean
  Step 14: Finalize implementation (delete -design/)
  Step 15: Commit
```

---

## Standalone Commands (unchanged)

These commands from the original flows remain available independently:

- **`boundary-audit [scope]`** — scan existing code for implicit
  subsystem boundaries. Produces a ranked report.
- **`abstraction-check <target>`** — run Abstraction Minimalist on
  any module or function. Informational, no edits.

---

## When NOT to Use This Flow

Skip the full flow for:
- **Trivial changes** — renaming, typos, config with obvious values
- **Fully specified changes** — exact requirements, no ambiguity
- **Bug fixes** — correct behavior is already defined
- **One-file changes** — single function addition
- **Config-only changes** — environment variables, feature flags

Use the full flow when there are genuine design decisions, the
implementation spans multiple files, or the feature touches multiple
subsystems.

---

## Guidelines (apply throughout)

1. **Think before writing.** Before coding, consider what properties
   (correctness, cohesiveness, consistency) the code should have.

2. **Don't assume — escalate.** If requirements are genuinely
   ambiguous and multiple interpretations lead to different outcomes,
   escalate rather than guess.

3. **Documentation-first.** Prefer formal specs (JSON Schema,
   OpenAPI, AsyncAPI) over prose descriptions of contracts.

4. **Test strategy.** Prefer e2e tests with real infrastructure.
   Only stub non-replicable external services.

5. **Exploitation doc.** Read from and update the project's
   exploitation/infrastructure doc for deployment-relevant changes.

6. **No silent skips.** If a step cannot run (e.g. no formal specs
   to review), the reviewer still runs and reports "N/A — no formal
   specs in scope" rather than being silently omitted.
