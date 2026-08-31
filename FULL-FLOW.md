# Full Flow — Instructions for Claude

## Denote Metadata System

## Grounded Reasoning

Read and apply `GROUNDED-REASONING.md` (in this directory) alongside
this flow. The nine principles apply to all reasoning — research,
design, dialog, and implementation, not just code.

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

** Original Prompt
(Verbatim copy of the user's chat message that triggered this flow.
Preserves raw intent — urgency, scope hints, context — that the
Goal section may formalize away. Used by the Steward for intent
drift detection. Set once at init, never modified.)

** Steering
(Append-only log of user messages during the flow that change
direction, add scope, constrain approach, or redirect focus.
Claude appends timestamped entries as they happen. Not every user
message is steering — "continue", "yes", "finalize" are not.
Only messages that alter the trajectory of the work.
Used by the Steward alongside Original Prompt for drift detection.)

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

*** Decisions
(Empty at init. Filled by the Author in Step 1 — core choices,
approach direction, trade-offs at overview level. No code samples
or detailed architecture. Reviewed and approved by user before
Details are developed.)

*** Details
(Empty at init. Filled by the Author in Step 6 — detailed
architecture expanding approved Decisions. Interfaces, specs,
data structures, code samples.)

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

## Prompt Files

Sometimes you want to capture an idea without filling in the full
intake template. A **prompt file** is a minimal artifact with just
a `** Prompt` section — the lightest-weight entry point into the
flow system.

**Creating a prompt file:**
- `prompt <idea>` command (see Commands)
- The user creates one via Emacs `denote` or manually

**Template:**

```org
#+STARTUP: overview
#+title:      <Idea description>
#+date:       [YYYY-MM-DD Day HH:MM]
#+filetags:   :prompt:
#+identifier: <YYYYMMDDTHHmmss>

* <Idea description>

** Prompt
(Free-form description of the idea. Claude uses this to derive
intake fields when a flow is started.)
```

Prompt files live in `designs/` alongside design and research docs.
They follow denote naming: `ID--slug__prompt.org`.

**Hydration.** When `full`, `design`, or `research` targets a prompt
file (detected by `:prompt:` in `#+filetags:`), Claude transforms it
into a proper intake document before running the flow:

1. Read the `** Prompt` section content
2. Add all missing intake sections (`** Original Prompt`,
   `** Goal`, `** Constraints`, etc. for design; `** Question`,
   `** Context`, etc. for research)
3. Copy prompt content verbatim to `** Original Prompt`
4. Infer and fill Goal/Question, Constraints/Scope, and other
   intake fields from the prompt content + codebase context
5. Update `#+filetags:` from `:prompt:` to `:design:` or
   `:research:`
6. Rename file to reflect new type keyword (denote rename procedure)
7. Run convergence gate and proceed with the flow

The prompt file is transformed in-place — it becomes the intake
document rather than spawning a separate file.

**Detection.** `resume` scans `designs/` for `:prompt:` files and
reports them as ideas awaiting a flow. If `full` or `research` is
run without a feature name and prompt files exist, list them and ask
which to use.

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

## Phase 1A: Design Decisions

### Step 1 — Author (Decisions)

Launch an Author subagent. Read `designs/<feature>.org` (the intake
doc — Goal, Constraints, References, Acceptance Criteria) and the
relevant source files (including any files listed in References).

Write into the `*** Decisions` subsection of `** Design` in
`designs/<feature>.org` — the core choices, approach direction,
patterns, and trade-offs. **Overview perspective only:** no code
samples, no detailed interface definitions, no exhaustive data
structures. Think "which direction and why," not "how exactly."

Decisions should cover:
- Approach selection (which pattern, library, or strategy)
- Boundary placement (which modules are affected, where new ones go)
- Key trade-offs acknowledged (what is prioritized over what)
- Decomposition proposal (sub-features, if applicable)

Write `designs/<feature>-design/questions.org` — genuine unknowns.
**Tag each item** with `[Critical]` or `[Auto]` per the Criticality
Classification. Items already answered by the intake's Goal,
Constraints, or Acceptance Criteria should not become questions.

Initialize `designs/<feature>-design/journal.org` with an
"Iteration 1 — Author (Decisions)" entry.

### Step 2 — Critic + Boundary Analyst + Steward (parallel)

Launch all three subagents in parallel. All read the design doc,
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

**Steward subagent (S1 — intent check):**
- Re-read the intake (Original Prompt, Steering log, Goal,
  Constraints, Acceptance Criteria, Preserved Invariants) and compare
  against the Author's Decisions.
- Check: "If these decisions were carried forward, would the
  resulting design solve the problem described in Goal? Is every
  Constraint honored? Is every Acceptance Criterion addressable?"
- Output one of:
  - *Pass:* design aligns with intake. (No action.)
  - *Drift:* specific divergence added to `criticism.org` tagged
    `[Critical]` with `*Source:* Steward (intent check)`.
  - *Annotation:* note in `journal.org` tracking observed evolution.
- **Self-contradiction check:** before comparing decisions vs intake,
  check whether the Steering log contradicts the Original Prompt.
  If the user's steering overrides their own original intent, flag
  this explicitly so the decision is conscious: "Your steering at
  [time] changes your original direction on [topic]. Confirming
  this is intentional?"
- The Steward does NOT critique design quality, suggest alternatives,
  or make autonomous decisions. It only checks intent alignment.
- Append journal entry.

### Step 3 — Auto-resolve

Process all `[Auto]`-tagged items in `questions.org` and
`criticism.org`:

For each item:
1. Decide on a resolution based on codebase context, design
   constraints, and established patterns
2. Update the Decisions subsection to reflect the decision
3. Move the item to `resolved.org` with `*Rationale:*` and
   `*Resolved by:* Claude (auto)` annotation
4. If resolving one item creates a new question or concern, add it
   (tagged) to the appropriate file

After processing, report the net change: "Auto-resolved N items.
M Critical items remain for user input." (or "0 Critical items —
decisions are self-consistent.")

### Step 4 — Auto-loop (up to 3 iterations)

After auto-resolving, launch Critic + Boundary Analyst re-evaluation
(parallel):

**Critic re-evaluation:** reads its context, updated decisions,
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

**Steward check (S2 — post-auto-resolve intent check).** After the
final auto-resolve iteration, run the Steward: re-read the intake
and compare against the current decisions. This is the
highest-value single intervention point — auto-resolve loops are
where most intent drift accumulates (each resolution is individually
reasonable but the aggregate can shift direction). Output goes to
`criticism.org` as `[Critical]` if drift detected, or `journal.org`
as annotation if evolution is observed but alignment holds.

### Step 5 — Escalate (if Critical items exist)

If any `[Critical]` items remain:

1. Create `answers.org` in the working directory with the
   three-section structure (Considerations, Questions, Criticism),
   pre-populated with only the `[Critical]` items
2. Present a summary to the user: list each Critical item with a
   one-line description and why it needs user input
3. **Wait for user** to fill in `answers.org`

If zero Critical items remain, skip directly to Step 5b.

### Step 5a — Process user answers

When the user provides answers:

1. **Apply answers** — update Decisions subsection, move resolved
   items to `resolved.org`
2. **Validator subagent** — check answers for vagueness,
   contradictions, implicit new questions, AND constraint
   violations. The Validator also reads Original Prompt,
   Constraints, Acceptance Criteria, and Preserved Invariants —
   if any answer contradicts these, flag it: "This answer would
   violate [constraint]. Proceed anyway?"
3. **Critic + Boundary Analyst re-evaluation** (parallel)
4. Auto-resolve any new `[Auto]` items
5. If new `[Critical]` items emerged, return to Step 5
6. If converged, proceed to Step 5b

### Step 5b — Decision review (MANDATORY)

Present the `*** Decisions` section to the user. This is a
**mandatory pause** — even when the user has asked to skip user
review (which applies to implementation Steps 13–14 only).

The review focuses on direction: the user verifies the core choices,
approach, trade-offs, and decomposition are correct before detailed
design work begins. This is intentionally easier to review than the
full design — no code samples, no interface definitions, just the
"what and why" of each choice.

The user may:
- **Approve** decisions as-is → proceed to Phase 1B (Step 6)
- **Amend** specific decisions → apply changes, re-run Critic on
  affected decisions (return to Step 3)
- **Reject and redirect** → Author rewrites decisions (return to
  Step 1)

Approved decisions are **locked** for Phase 1B. The Details phase
cannot change decisions without explicit user escalation.

---

## Phase 1B: Design Details

### Step 6 — Author (Details)

Launch an Author subagent. Read the approved `*** Decisions`, the
intake doc, and the relevant source files.

Write into the `*** Details` subsection of `** Design` — the
detailed architecture that implements the approved decisions.
**Documentation-first:** define behavior through formal specs
(JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migration
schemas) where applicable. Prefer declarative, machine-readable
definitions over prose.

Details should cover:
- Interface definitions (function signatures, type shapes, contracts)
- Data structures and schemas
- Implementation approach per module
- Error handling strategy
- Formal specs where applicable
- Code samples where they clarify intent

**Eager decomposition:** If the Decisions proposed sub-features,
create their design docs now:
`designs/<feature>/<sub-feature>.org` with their own intake
templates. Each sub-feature goes through its own design → implement
cycle. Do not decompose trivially small features.

Write new questions to `questions.org` (tagged). **Decisions are
locked** — questions must be about detail choices, not about
revisiting decisions. Any concern that would require changing an
approved decision is tagged `[Critical]` with
`*Note:* Requires reopening decisions`.

Append journal entry: "Author (Details)".

### Step 6a — Critic + Boundary Analyst (details scope)

Launch Critic + Boundary Analyst in parallel, scoped to details
quality. **Decisions are locked** — critics must not re-litigate
approved decisions.

**Critic subagent:**
- Update `criticism.org` with detail-level concerns (tagged).
- Update `critic-context.org`.
- Concerns that would change an approved decision are tagged
  `[Critical]` with `*Note:* Requires reopening decisions`.
- Append journal entry.

**Boundary Analyst subagent:**
- Update `boundary-context.org` with detail-level observations.
- Append journal entry.

### Step 6b — Auto-resolve on details

Same auto-resolve pattern as Steps 3–4, with lighter touch:
- Max **2** iterations (details should converge quickly with locked
  decisions)
- Items that would change a decision are NOT auto-resolved — they
  remain `[Critical]`

If `[Critical]` items remain:
- Items tagged "Requires reopening decisions" → escalate to user
  with explicit note: "This detail concern affects an approved
  decision. Reopening decisions requires your approval."
  If approved, return to Step 5b with the concern.
- Other `[Critical]` items → create/update `answers.org`, escalate
  to user, process answers (same as Step 5/5a pattern), then
  continue.

### Step 7 — Design review (MANDATORY)

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

**Reviewer context:** Decisions are approved by the user — reviewers
evaluate whether details faithfully implement those decisions and
whether the details are sound. A reviewer may note that a decision
creates a problem, but this is flagged as `[Critical]` with
`*Note:* Requires reopening decisions`, not auto-resolved.

**Processing review findings:**
- `[Auto]`-level findings (High and below): Claude fixes the design
  immediately, documents in `resolved.org`
- `[Critical]`-level findings about details: escalate to user
  (return to Step 6b)
- `[Critical]`-level findings requiring decision changes: escalate
  to user with reopening note (return to Step 5b if approved)
- If review introduced design changes: run one more Critic +
  Boundary Analyst re-evaluation, then re-check convergence

After the review is clean (no Critical findings, all High+ addressed),
run **Steward check (S3)**: did addressing reviewer findings shift
the design's scope away from the original intake? If drift detected,
escalate to user. Otherwise proceed to Step 7a.

### Step 7a — Detail review (MANDATORY)

Present the full `** Design` section (both `*** Decisions` and
`*** Details`) to the user. Since decisions were already approved
in Step 5b, this review focuses on **detail quality**: interfaces,
specs, data structures, and implementation approach.

This is a **mandatory pause** — even when the user has asked to
skip user review (which applies to implementation Steps 13–14 only).

The user may:
- **Approve** details → proceed to Step 8 (finalize)
- **Request amendments** → apply changes, re-run Critic on affected
  details (return to Step 6b)
- **Flag a decision concern** → this reopens decisions (return to
  Step 5b with the concern)

### Step 8 — Finalize design

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

**Step 8d — Context Maintenance subagent.** Check whether design
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

### Step 9 — Plan

Two sequential subagents produce `impl-plan.org` in the working
directory:

**Step 9a — Module Relationship subagent.** Read finalized design +
existing codebase. Write section 1 of `impl-plan.org`:

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
Schema, OpenAPI, AsyncAPI — to define before implementation.)

** Supervision Tree Placement
(Where new processes sit in the OTP supervision tree.)

** Data Flow
(Which process sends what to whom, through which mechanism —
direct call, PubSub, message queue, ETS.)
```

The Module Relationship subagent does NOT sequence work — it
describes the topology.

**Step 9b — Plan subagent.** Read module relationships + finalized
  design. Write section 2:
- Step order, parallel opportunities, integration checkpoints,
  formal specs to produce first

**Step 9c — Commit staging.** Structure the plan into blame-friendly
atomic commits. The atom is one **complete concept**, not one
mechanical operation. First question for every commit boundary:
"will `git blame` in one year show WHY this code exists?"

Core principles:

- **Blame-first atomicity:** each commit encompasses one complete
  idea — the full "why" behind a group of changes. Tests live in the
  same commit as the code they test. A commit message that describes
  a mechanical operation with no conceptual meaning ("extract helper
  functions") signals a wrong boundary — merge it with the conceptual
  commit it supports.
- **Consolidated preparation:** all preparatory refactoring for a
  feature (renames, extractions, scaffolding) goes in ONE prepare
  commit, not one per operation. Skip the prepare commit entirely if
  preparation is minimal. Exception: keep a separate commit when
  moving code AND modifying it (diff tracking limitation).
- **No internal supersession:** within the same feature branch, no
  commit should fix or supersede changes from a previous commit.
  The plan reads as "I knew what I was building from the beginning."
- **Each commit compiles and passes tests.** No intermediate broken
  state.
- **Conciseness test:** if the commit message lists multiple unrelated
  things, split. If it describes something with no standalone
  conceptual meaning, merge.
- **Fine-grained exception:** keep a separate commit only when the
  mechanical change IS a meaningful concept on its own (independent
  refactoring, separable bug fix).

The Plan subagent produces a numbered commit list in `impl-plan.org`
section 3:
```
*** Commit staging
1. Prepare for event-driven dispatch (rename, extract, scaffold)
2. Event-driven dispatch: lifecycle management (impl + tests)
3. Event-driven dispatch: error recovery (impl + tests)
```

### Step 10 — Implement

**Style calibration:** Before writing code, sample the user's recent
commits (`git log --author` + read 1-2 touched files) to calibrate
naming, structure, and idiom choices. The user's code is the
authoritative style reference. Do this silently.

Follow the commit staging plan. For each commit:

1. **Enumerate properties** — list the specific behavioral properties
   this commit should validate (Beck's "test list"). For each property,
   decide: test now (P0-P1), test later (P2), or not tested (with
   reason). This is the specification for what "success" means.
2. **Write tests** targeting the enumerated P0-P1 properties (prefer
   e2e with real infrastructure; only stub non-replicable services)
3. **Run tests** — verify they fail (the behavior doesn't exist yet)
4. **Implement** — define interfaces/specs first, then the minimum
   code to pass the tests
5. **Run tests** — verify they pass

Run the project's test/lint suite (`mix paranoid`) after each commit.

**Trailing Abstraction Minimalist check.** After implementation,
run a lightweight Abstraction Minimalist subagent on newly
written/modified functions. Fix clear tier violations immediately.

### Step 11 — Self-review (MANDATORY)

This step is **not optional**, even when the user has asked to skip
user review (Steps 13–14). It must be orchestrated from the **main
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
Info).

**Evidence requirement:** every finding rated High or Critical must
include a `*Grounds:*` field citing specific evidence — file path +
line number, test output, error reproduction, or benchmark result.
Findings that make empirical claims ("this will cause X") without
evidence are downgraded to Info. Structural observations ("this
module has mixed concerns") do not require empirical evidence.

**Step 11b — Review Collator.** Launch a Review Collator subagent
that reads all 9 reviewer reports and produces a consolidated
`review.org`. The collator:

- **Deduplicates**: merges semantically equivalent findings from
  different reviewers into single entries (noting which reviewers
  flagged them)
- **Resolves severity**: when reviewers disagree on severity for the
  same finding, picks the highest
- **Flags conflicts**: when reviewers make contradictory
  recommendations, lists both with a `*Conflict:*` marker — does
  NOT resolve them
- **Checks evidence**: findings rated High or Critical without a
  `*Grounds:*` field are flagged with `*Missing evidence*`
- **Enforces structure**: consistent format (severity / finding /
  grounds / affected files / recommendation) for every entry

The collator does NOT suppress findings — everything passes through.
It does NOT evaluate quality or make judgment calls. It is a
mechanical post-processor that saves the main conversation from
parsing 9 independent reports.

### Step 12 — Fix

Read `review.org`. For each issue:
- Fixable: apply the fix
- Blocking (design ambiguity, out of scope): escalate to user

**Trailing Abstraction Minimalist check** on changed functions.

Re-run `mix paranoid`.

**Steward check (S4 — pre-user-review intent check).** Does the
implementation, as built, solve the problem described in the original
Goal? Are all Acceptance Criteria met by actual code? This is the
last check before the user reviews — it catches cases where the
Design Fidelity Reviewer validated the implementation against a
design that itself had drifted. If drift detected, report to user
alongside the review plan.

### Step 13 — Review plan + review notes

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

### Step 14 — User review loop

**Step 14a — Apply fixes.** For each note in `review-notes.org`:
- Apply the fix
- Move to `review-notes-resolved.org` with `**Resolved:**` subsection

**Trailing Abstraction Minimalist check** on changed functions.

**Step 14b — Fix Validator subagent.** Check:
- Did each fix address its review note?
- Did any fix introduce new issues or regressions?
- Do changes still match the finalized design?
- Do changes still honor the original intake (Original Prompt,
  Constraints, Acceptance Criteria, Preserved Invariants)? If a
  user-requested fix violates an original constraint, flag it.

**Step 14c — Update review plan.** Update `review-plan.org`:
- Modified files: uncheck, update description to latest change only
- Unmodified files: keep existing check state
- New files: add to appropriate section
- Removed files: move to deleted list

Re-run `mix paranoid`.

If `review-notes.org` has unresolved notes, prompt **`loop`**.
If all resolved, proceed to Step 14d.

**Step 14d — Pattern capture.** Review all resolved corrections in
`review-notes-resolved.org`. For each correction, evaluate:

1. Is this a project convention or a one-off mistake? (one-off → skip)
2. Will this recur in future implementations? (unlikely → skip)
3. Is the pattern already captured in memory or notes? (yes → skip)
4. Is the correction an imperative or a fact?
   - Imperative ("do X, not Y") → save to Claude Code memory
     (project conventions file, e.g. `feedback_code_principles.md`)
   - Fact about the system ("payloads use camelCase") → save to
     `notes/` as a `:fact:` entry (if `notes/` exists and is not
     read-only)

Format for memory entries:
```
**<Pattern title> — <one-line rationale>.**
- `<correct approach>` not `<incorrect approach>`.
- Why: <brief explanation>.
```

This is a lightweight trailing check, not a gate — it does not block
progression. Proceed to Step 15.

### Step 15 — Finalize implementation

**Step 15a — Completeness Verifier subagent.** Check:
- Does implementation cover every design aspect?
- Are there partially implemented requirements?
- Are all formal specs present in the codebase?
- Do all tests pass and cover key behaviors?

If gaps found: report to user, return to Step 14 if user wants to
address them now, or document as deferred.

**Step 15b — Context Maintenance subagent.** Check whether the
implementation changed anything documented in CLAUDE.md:
- New modules or directories → update Key Directories
- New data flow patterns → update Data Flow section
- New environment variables → update Environment Variables
- New project-specific conventions → update Conventions

Same rules as Step 8d: update project context only, not process
instructions. CLAUDE.md changes appear in review-plan.org if the
user hasn't reviewed yet, or are visible in the commit diff.

**Step 15c — Clean up.** Delete the entire working directory
(`designs/<feature>-design/` or the sub-feature equivalent). The
design doc remains as the permanent record.

### Step 16 — Commit

Follow the commit staging plan from Step 9c. Create each commit
as a separate, atomic unit with its own goal. Stage files precisely
per commit — do not batch everything into one commit unless the
staging plan has a single entry. Each commit should compile and pass
tests independently. Use the project's existing commit message
convention.

---

## Commands

| Command | Action |
|---------|--------|
| `prompt <idea>` | Create a prompt-only file in `designs/` (see Prompt Files). If `<idea>` text is provided, pre-populate the Prompt section; otherwise leave it empty for the user to fill in. |
| `init <name-or-description>` | Create `designs/<feature>.org` with the intake template and `designs/<feature>-design/` directory. Derives kebab-case filename (strip filler words, noun phrases, max 3-4 words). Does not start the flow — the user fills in the intake fields first. |
| `full [<feature-name>]` | Start from Step 1. Reads the intake doc, then runs through all phases automatically, pausing for Critical escalations (Step 5), decision review (Step 5b), detail review (Step 7a), and implementation review (Step 13). If targeting a prompt file, hydrate it first (see Prompt Files). If `<feature-name>` is omitted, infer from the most recently initialized or active feature. |
| `design [<feature-name>]` | Run Phase 1 only (Steps 1–8). Stops after design finalization. If targeting a prompt file, hydrate it first. |
| `implement [<feature-name>]` | Run Phase 2 only (Steps 9–16). Requires a finalized design doc. Use when design was done separately or already exists. |
| `loop` | In design phase: process `answers.org` (Step 5a). In implementation phase: process `review-notes.org` (Step 14). **Cumulative drift check:** re-read ALL Steering entries, check aggregate drift against Original Prompt. |
| `finalize` | In design phase: run Step 8. In implementation phase: run Step 15. **Cumulative drift check** before finalizing. |
| `commit` | Run Step 16. |
| `resume` | Detect current state from existing files + git status, report where we are, and continue from the appropriate step. |
| `retro` | Post-completion retrospective (3-5 bullets, no files created). |

`design` and `implement` run the SAME steps as `full` — they are
entry points into specific phases, not separate definitions. All
quality gates, Steward checks, evidence requirements, and mandatory
reviews apply identically regardless of entry point.

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
Before incorporating any steering message:

1. Append to the `** Steering` section with timestamp
2. **Contradiction check:** does this steering contradict the
   Original Prompt, Goal, Constraints, or Acceptance Criteria?
   - If yes: flag explicitly — "This changes your original
     direction: [Original said X, this says Y]. Confirming this
     is intentional?" Record the user's confirmation as an
     administrative decision.
   - If no: incorporate into the next step.
3. If the steering proposes a technical approach (not a preference
   or scope decision): check it against actual code before
   incorporating. If the code shows the user's model is wrong,
   inform before complying (Grounded Informed Consent).

**State-and-Comply default:** when Claude disagrees with user
direction after checking, state the objection in one sentence with
specific evidence, then comply if the user confirms. Record the
disagreement in journal.org. Never silently comply when evidence
suggests the direction is wrong — but also never refuse.

---

## Session Resumption — `resume`

All state lives in the files. When resuming:

Report state in **human-readable terms** — what was found and
what happens next, never step numbers. The user should not need
to know the flow's internal structure.

1. Scan `designs/` for feature docs, working directories, and
   prompt files (`:prompt:` tagged)
2. Read all existing files for the feature
3. Check git status/diff for code changes
4. Determine phase and step:
   - Prompt file exists (`:prompt:` tag) → idea captured, not yet
     started. Suggest `full` or `research` to hydrate and run.
   - Design doc exists, no `-design/` directory → just initialized,
     user may still be filling intake. Ask or run `full`.
   - `-design/` exists, `*** Decisions` empty → Step 1 (author
     decisions)
   - `-design/` exists, `*** Decisions` filled, `*** Details` empty
     → decisions written, check for pending review (Step 5b) or
     ready for details (Step 6)
   - `-design/` exists, `*** Details` filled, no impl files →
     details written, check for pending review (Step 7a) or
     ready for finalize (Step 8)
   - `impl-plan.org` exists, no code changes → Step 10 (implement)
   - Code changes exist, no review files → Step 11 (self-review)
   - `review-notes.org` has content → Step 14 (user review loop)
   - `answers.org` has content → Step 5a (process answers)
5. Report current state and continue

---

## Flow Diagram

```
  prompt ──► designs/<slug>__prompt.org (idea only)
       │
       ▼ (user fills prompt, later runs `full` or `research`)
       │ (hydration: prompt → intake doc, in-place)

  init ──► designs/<feature>.org (intake template)
       ──► designs/<feature>-design/ (working dir)
       │
       ▼ (user fills intake, then runs `full`)

Phase 1A: Design Decisions
  Step 1: Author (Decisions) ──────────────────────┐
  Step 2: Critic + Boundary Analyst + Steward S1 ──┤
  Step 3: Auto-resolve [Auto] items ────────────────┤
  Step 4: Auto-loop (up to 3x) ◄───────────────────┘
       │ + Steward S2 (intent check after final loop)
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
  Step 5b: Decision review ──► wait for user
       │ (MANDATORY — user approves direction)
       │
       ├─ Amend ──► return to Step 3
       └─ Approve
       │
       ▼

Phase 1B: Design Details
  Step 6: Author (Details) ────────────────────────┐
  Step 6a: Critic + Boundary Analyst (details) ────┤
  Step 6b: Auto-resolve on details (up to 2x) ◄───┘
       │
  Step 7: Design review (MANDATORY, 7 reviewers)
       │ + Steward S3 (intent check after review)
       │
       ├─ Critical findings (details) ──► Step 6b
       ├─ Critical findings (decisions) ──► Step 5b
       └─ Clean
       │
  Step 7a: Detail review ──► wait for user
       │ (MANDATORY — user approves details)
       │
       ├─ Amend ──► return to Step 6b
       ├─ Decision concern ──► return to Step 5b
       └─ Approve
       │
  Step 8: Finalize design (delete -design/, keep .org)
       │
       ▼ (automatic — no pause, recreates -design/ for impl)

Phase 2: Implementation
  Step 9: Plan (module relationships + build order)
  Step 10: Implement + tests + mix paranoid
  Step 11: Self-review (MANDATORY, 9 reviewers)
  Step 11b: Review Collator (dedup + severity + conflicts)
  Step 12: Fix + mix paranoid
       │ + Steward S4 (intent check before user review)
  Step 13: Review plan + notes ──► wait for user
  Step 14: User review loop ◄──── loop until clean
  Step 15: Finalize implementation (delete -design/)
  Step 16: Commit
```

---

## Standalone Commands (unchanged)

These commands from the original flows remain available independently:

- **`boundary-audit [scope]`** — scan existing code for implicit
  subsystem boundaries. Produces a ranked report.
- **`abstraction-check <target>`** — run Abstraction Minimalist on
  any module or function. Informational, no edits.

---

## Concurrent Features

Multiple features can be in-flight simultaneously — each has its
own file set. To avoid confusion:
- Always include `<feature-name>` when switching between features
  in the same session
- Never modify one feature's files while processing another
  feature's loop

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
