# Full Flow — Instructions for Claude

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
   rationale in `-resolved.org` so the user can audit Claude's
   reasoning at any point.

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

## Files

All files live in the feature's documentation directory.

### Design phase files

| File | Purpose |
|------|---------|
| `<feature>.org` | Design doc — grows richer each iteration |
| `<feature>-questions.org` | Open questions needing resolution |
| `<feature>-criticism.org` | Open criticisms and concerns |
| `<feature>-resolved.org` | Archive of resolved questions and criticisms with rationale |
| `<feature>-critic-context.org` | Persistent Critic memory across iterations |
| `<feature>-boundary-context.org` | Persistent Boundary Analyst memory |
| `<feature>-journal.org` | Shared append-only design journal |

No `-answers.org` file — Claude resolves non-critical items directly.
When Critical items exist and are escalated to the user, the user
writes answers in `<feature>-answers.org` (created on demand with
only the Critical items listed).

### Implementation phase files

| File | Purpose |
|------|---------|
| `<feature>-impl-plan.org` | Module relationships + build order |
| `<feature>-review.org` | Self-review output (persists for user reference) |
| `review-plan.org` | Changed files grouped by concern |
| `review-notes.org` | User's review notes per file |
| `review-notes-resolved.org` | Archive of resolved review notes |

**File format:** All working files use **Emacs Org mode** (`.org`).
Use Org syntax: `*` / `**` / `***` for headings, `*bold*`, `/italic/`,
`=verbatim=`, `~code~`, `- ` for lists, `- [ ]` for checkboxes. For
code blocks use `#+begin_src lang` / `#+end_src`. Do NOT use Markdown
syntax in working files.

---

## Phase 1: Design

### Step 1 — Author

Launch an Author subagent. Read the relevant source files. Write:

- `<feature>.org` — what the feature does and a proposed design.
  **Documentation-first:** define behavior through formal specs
  (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migration
  schemas) where applicable. Prefer declarative, machine-readable
  definitions over prose.
- `<feature>-questions.org` — genuine unknowns. **Tag each item**
  with `[Critical]` or `[Auto]` per the Criticality Classification.

Initialize `<feature>-journal.org` with an "Iteration 1 — Author"
entry.

### Step 2 — Critic + Boundary Analyst (parallel)

Launch both subagents in parallel. Both read the design doc,
questions, and the same source files.

**Critic subagent:**
- Write `<feature>-criticism.org` — concerns, gaps, risks. **Tag
  each item** `[Critical]` or `[Auto]`.
- Write `<feature>-critic-context.org` — initialize active concerns.
- May add items to `-questions.org` (tagged).
- Append journal entry.

**Boundary Analyst subagent:**
- Write `<feature>-boundary-context.org` — subsystem boundaries,
  interfaces, formalization recommendations, distillation
  opportunities, independence assessments.
- May add items to `-questions.org` (tagged).
- Append journal entry.

### Step 3 — Auto-resolve

Process all `[Auto]`-tagged items in `-questions.org` and
`-criticism.org`:

For each item:
1. Decide on a resolution based on codebase context, design
   constraints, and established patterns
2. Update `<feature>.org` to reflect the decision
3. Move the item to `-resolved.org` with `*Rationale:*` and
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

1. Create `<feature>-answers.org` with the three-section structure
   (Considerations, Questions, Criticism), pre-populated with only
   the `[Critical]` items
2. Present a summary to the user: list each Critical item with a
   one-line description and why it needs user input
3. **Wait for user** to fill in `-answers.org`

If zero Critical items remain, skip directly to Step 6.

### Step 5a — Process user answers

When the user provides answers:

1. **Apply answers** — update design doc, move resolved items to
   `-resolved.org`
2. **Validator subagent** — check answers for vagueness,
   contradictions, implicit new questions
3. **Critic + Boundary Analyst re-evaluation** (parallel)
4. Auto-resolve any new `[Auto]` items
5. If new `[Critical]` items emerged, return to Step 5
6. If converged, proceed to Step 6

### Step 6 — Design review (MANDATORY)

This step is **not optional**. Launch up to 7 specialized reviewer
subagents in parallel. Before reporting on any file, each subagent
verifies the file exists at the referenced path.

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
  immediately, documents in `-resolved.org`
- `[Critical]`-level findings: escalate to user (return to Step 5)
- If review introduced design changes: run one more Critic +
  Boundary Analyst re-evaluation, then re-check convergence

After the review is clean (no Critical findings, all High+ addressed),
proceed to Step 7.

### Step 7 — Finalize design

1. **Consolidate** — merge `<feature>.org` into the main design
   document
2. **Finalization Verifier subagent** — diff consolidated section
   against `<feature>.org` + `-resolved.org` + `-journal.org`.
   Check all resolved decisions, formal specs, criticism resolutions,
   boundary definitions, and journal insights are represented. Patch
   gaps before proceeding.
3. **Clean up** — delete all design phase files except the main
   design document

**Do NOT wait for user confirmation.** Proceed directly to Phase 2.

---

## Phase 2: Implementation

### Step 8 — Plan

Two sequential subagents produce `<feature>-impl-plan.org`:

**Step 8a — Module Relationship subagent.** Read finalized design +
existing codebase. Write section 1:
- New modules, modified modules, dependencies, interfaces &
  contracts, supervision tree placement, data flow

**Step 8b — Plan subagent.** Read module relationships + finalized
  design. Write section 2:
- Step order, parallel opportunities, integration checkpoints,
  formal specs to produce first

### Step 9 — Implement

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

This step is **not optional**. Launch all 8 specialized reviewer
subagents in parallel, each with an adversarial stance:

| # | Role | Focus |
|---|------|-------|
| 1 | **Design Fidelity Reviewer** | Diff intent vs. implementation — flag drift, missing pieces, silent deviations |
| 2 | **Correctness Reviewer** | Logic errors, off-by-one, wrong assumptions, null handling, incorrect return types |
| 3 | **Edge Case Reviewer** | Race conditions, failure modes, error handling, boundary conditions, concurrency |
| 4 | **Test Reviewer** | Uncovered behaviors/branches, unnecessary mocking, missing test cases |
| 5 | **Spec Reviewer** | Schema correctness, runtime validation matching declared schemas |
| 6 | **Style Reviewer** | Naming consistency, pattern consistency with existing codebase, organization |
| 7 | **Principles Reviewer** | SRP, DIP, Information Expert, Low Coupling, High Cohesion, Creator (smell detection, not mandates) |
| 8 | **Abstraction Minimalist** | Function-level tier consistency, module-level API coherence, cross-module tier leaks |

Each produces a structured report (Critical / High / Medium / Low /
Info). Claude collates and deduplicates into `<feature>-review.org`.

### Step 11 — Fix

Read `<feature>-review.org`. For each issue:
- Fixable: apply the fix
- Blocking (design ambiguity, out of scope): escalate to user

**Trailing Abstraction Minimalist check** on changed functions.

Re-run `mix paranoid`.

### Step 12 — Review plan + review notes

Create `review-plan.org` and `review-notes.org` in the feature
documentation directory.

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

**Step 14b — Clean up.** Delete all implementation flow files:
- `<feature>-impl-plan.org`
- `review-plan.org`
- `review-notes.org`
- `review-notes-resolved.org`
- `<feature>-review.org`

### Step 15 — Commit

Create a commit following the project's existing commit convention.

---

## Commands

| Command | Action |
|---------|--------|
| `full <feature-name>` | Start from Step 1. Runs through all phases automatically, pausing only for Critical escalations (Step 5) and user review (Step 12). |
| `loop` | In design phase: process `-answers.org` (Step 5a). In implementation phase: process `review-notes.org` (Step 13). |
| `finalize` | In design phase: run Step 7. In implementation phase: run Step 14. |
| `commit` | Run Step 15. |
| `resume` | Detect current state from existing files + git status, report where we are, and continue from the appropriate step. |
| `retro` | Post-completion retrospective (3-5 bullets, no files created). |

The flow **does not expose** `review` as a standalone command — the
design review (Step 6) and implementation self-review (Step 10)
always run as part of the flow.

---

## Session Resumption — `resume`

All state lives in the files. When resuming:

1. Read all existing files for the feature
2. Check git status/diff for code changes
3. Determine phase and step:
   - Design files exist, no impl files → Design phase
   - Impl plan exists, no code changes → Step 9 (implement)
   - Code changes exist, no review files → Step 10 (self-review)
   - `review-notes.org` has content → Step 13 (user review loop)
   - `<feature>-answers.org` has content → Step 5a (process answers)
4. Report current state and continue

---

## Flow Diagram

```
Phase 1: Design
  Step 1: Author ──────────────────────────────────┐
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
  Step 7: Finalize design
       │
       ▼ (automatic — no pause)

Phase 2: Implementation
  Step 8: Plan (module relationships + build order)
  Step 9: Implement + tests + mix paranoid
  Step 10: Self-review (MANDATORY, 8 reviewers)
  Step 11: Fix + mix paranoid
  Step 12: Review plan + notes ──► wait for user
  Step 13: User review loop ◄──── loop until clean
  Step 14: Finalize implementation
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
