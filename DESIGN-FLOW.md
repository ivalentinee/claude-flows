# Design Loop — Instructions for Claude

An iterative design process for features and sub-features. Every feature goes through the same loop before its design is consolidated into the main design document.

---

## The Eight Files

For a feature named `<feature-name>`, create:

| File | Purpose |
|------|---------|
| `<feature-name>.org` | Design doc — describes what the feature does and how it should be implemented. Grows richer each loop iteration. |
| `<feature-name>-questions.org` | Open questions — things that cannot be decided without user input. Items are moved to `-resolved.org` as answers arrive. When empty, delete the file. |
| `<feature-name>-criticism.org` | Criticisms and concerns about the current design. Items are moved to `-resolved.org` as they are resolved. When empty, delete the file. |
| `<feature-name>-answers.org` | Written by the user. Three sections (see below). Read by Claude at the start of each loop iteration. Never modified by Claude. |
| `<feature-name>-resolved.org` | Archive of resolved questions and criticisms. Two sections: `Questions` and `Criticism`. Append-only — never remove entries. Created on first resolution. |
| `<feature-name>-critic-context.org` | Persistent working memory for the Critic subagent. Tracks active concerns, resolved concerns, and patterns noticed across iterations. Read and updated by the Critic on every invocation. |
| `<feature-name>-boundary-context.org` | Persistent working memory for the Boundary Analyst subagent. Tracks identified subsystem boundaries, their interfaces, and how they evolve across iterations. |
| `<feature-name>-journal.org` | Shared append-only design journal. Every subagent appends a timestamped, role-tagged entry summarizing its key observations. Gives all roles cross-iteration context in one place. |

All files start with **headings only** — no content. Content is added by Claude (design/questions/criticism/resolved) or the user (answers) as the loop progresses.

**File format:** All working files use **Emacs Org mode** (`.org`), not Markdown. Use Org syntax: `*` / `**` / `***` for headings, `*bold*`, `/italic/`, `=verbatim=`, `~code~`, `- ` for lists, `- [ ]` for checkboxes. For code blocks use `#+begin_src lang` / `#+end_src`. Do NOT slip into Markdown syntax (`##` headings, `**bold**`, triple backticks). The flow instruction files themselves (this file) remain Markdown — only the per-feature working files are Org.

---

## `-answers.org` Structure

```org
* Answers

** Considerations

(User writes any context, constraints, or design preferences that
should shape the implementation — things that are neither direct
answers to questions nor responses to criticism.)

** Questions

(After the first loop iteration: open questions copied here from
=-questions.org= as a reference. User writes answers beneath each one.)

** Criticism

(After the first loop iteration: open criticisms copied here from
=-criticism.org= as a reference. User writes responses beneath each one.)
```

On the very first pass (step 1), `-answers.org` has headings only. After every subsequent loop iteration (step 3), Claude recreates it with the still-open questions and criticisms pre-populated under their headings, ready for the user to answer in place.

---

## `-resolved.org` Structure

```org
* Resolved

** Questions

(Resolved questions moved here from =-questions.org=, with their
answers inline. Each entry includes the original question, the
answer, and a *Rationale:* line explaining /why/ this decision
was made — so the reasoning survives even if the design doc only
records the outcome.)

** Criticism

(Resolved criticisms moved here from =-criticism.org=, with their
resolutions inline. Each entry includes the original criticism,
the resolution, and a *Rationale:* line explaining /why/ this
resolution was chosen.)
```

---

## `-critic-context.org` Structure

```org
* Critic Context

** Active Concerns
(Concerns the Critic currently holds about the design. Each entry
includes when it was first raised and a severity: blocking / high /
medium / low. Updated every iteration — resolved items move down,
new items are added.)

** Resolved Concerns
(Concerns that the Critic is satisfied have been genuinely addressed.
Each entry includes the original concern, which iteration resolved
it, and a brief assessment of the resolution quality.)

** Patterns Noticed
(Cross-iteration observations — e.g. "specs tend to lag behind prose
changes", "answers address surface issues without updating formal
contracts". These help the Critic calibrate its scrutiny.)
```

Created by the Critic at step 1b. Read and updated by the Critic on
every subsequent invocation (step 3c and restart).

---

## `-boundary-context.org` Structure

```org
* Boundary Analysis

** Identified Subsystems
(Each subsystem as a named cluster of modules/processes with:
- *Responsibility* — what this subsystem does as a black box
- *Internal components* — modules/processes inside the boundary
- *Public interface* — the narrow set of functions, messages,
  or specs through which the rest of the system interacts with it,
  with recommended formalization (see Interface Formalization below)
- *Internal details hidden* — what callers should NOT know about
- *Stability* — how settled this boundary is: firm / provisional /
  speculative
- *Independence assessment* — could another team develop this
  subsystem given only the public interface? Could it be implemented
  in a different language? If not, what coupling prevents it?
- *Interface formalization* — how should this subsystem's public
  interface be formally described? Pick the appropriate level:
  - /Within a single language:/ typespecs / @type / @callback
    (Elixir Dialyzer), behaviours, protocols, TypeScript types,
    Go interfaces — whatever the language provides
  - /Across language/team boundaries:/ JSON Schema for data
    shapes, OpenAPI for HTTP interfaces, AsyncAPI for
    message-based interfaces, Protocol Buffers / gRPC for
    RPC, GraphQL schema for query APIs
  - /Both:/ when a subsystem has internal callers (same
    language) AND external callers (other teams/languages),
    define both — language-native types for internal use,
    language-agnostic spec for the boundary contract
  List the specific specs to produce and what each covers.)

** Boundary Changes
(How boundaries shifted across iterations. E.g. "Iteration 2:
merged heartbeat and health-check into a single 'worker liveness'
subsystem after user clarified they share timeout config.")

** Distillation Opportunities
(Existing code clusters that already behave as subsystems but lack
a formal boundary. Listed only when formalizing the boundary would
make the new feature's design cleaner. Each entry includes: which
modules form the cluster, what interface the new feature would use,
and what changes would be needed to formalize it.)

** Boundary Tensions
(Places where the design currently reaches across a boundary or
where two subsystems are coupled in a way that undermines the
black-box property. These are not necessarily problems — just
tensions to be aware of.)
```

Created by the Boundary Analyst at step 1b. Read and updated on
every subsequent invocation (step 3c and restart).

---

## `-journal.org` Structure

```org
* Design Journal

** Iteration 1 — Author
(Summary of key design decisions and rationale...)

** Iteration 1 — Critic
(Summary of key concerns raised and why...)

** Iteration 2 — Validator
(Summary of answer quality assessment...)

** Iteration 2 — Critic (re-evaluation)
(What changed, what improved, what's still concerning...)
```

Append-only — subagents never edit prior entries, only add new ones.
Every subagent that runs (Author, Critic, Validator, Reviewers,
Verifier) appends a section tagged with the iteration number and role.
Entries should be concise (3–5 bullet points) — the journal is for
cross-role awareness, not a transcript.

---

## The Loop

### 1. Initial pass — `start <feature-name>`

This step uses two subagents with distinct roles to separate authoring from criticism.

**Step 1a — Author subagent.** Read the relevant source files. Write:
- `<feature-name>.org` — what the feature does and a proposed design. **Documentation-first:** actively look for opportunities to define behavior through formal specifications (JSON Schema for data validation, OpenAPI for HTTP interfaces, AsyncAPI for message-based interfaces, GraphQL schema for query APIs, DB migration schemas for data models). Prefer declarative, machine-readable definitions over prose descriptions of formats and contracts — the spec *is* the documentation.
- `<feature-name>-questions.org` — genuine unknowns that need user input (not self-criticism disguised as questions)

The Author subagent does NOT write criticisms — it advocates for its own design. After writing, it initializes `<feature-name>-journal.org` with an "Iteration 1 — Author" entry summarizing key design decisions and rationale.

**Step 1b — Critic + Boundary Analyst subagents (in parallel).** Both read the design doc and questions produced by the Author, then independently read the same source files.

*Critic subagent:*
- Write `<feature-name>-criticism.org` — concerns, gaps, and risks in the proposed design
- Write `<feature-name>-critic-context.org` — initialize with active concerns, empty resolved section, and any initial patterns noticed
- The Critic has not written the design and is explicitly prompted to distrust it: check every claim against actual code, look for what the Author overlooked, and challenge assumptions. May also add items to `-questions.org` if it identifies unknowns the Author missed.
- Append an "Iteration 1 — Critic" entry to the journal.

*Boundary Analyst subagent:*
- Write `<feature-name>-boundary-context.org` — identify natural subsystem boundaries in the proposed design. Look for clusters of modules/processes that form cohesive units with narrow interfaces to the outside. For each boundary, define what's inside, what's the public interface, and what internal details should be hidden.
- The Boundary Analyst reads the existing codebase to understand where current subsystem boundaries already exist and how the new feature's boundaries relate to them. **Distillation opportunity:** if the new feature would benefit from a formal boundary around existing code that currently lacks one (e.g. a cluster of modules that already behaves as a subsystem but has no defined interface), the Boundary Analyst should propose extracting/formalizing that boundary. This goes into "Identified Subsystems" tagged as `existing — proposed formalization` with a note on what the new feature gains from it. The Analyst does NOT propose refactoring for its own sake — only when the new feature's design would be cleaner or more decoupled as a result.
- For each identified subsystem, also consider: **"Could this subsystem be developed independently by another team?"** and **"Could this subsystem be implemented in a different programming language?"** These are litmus tests for boundary quality — if the answer is no, the interface is probably too leaky or coupled to implementation details. This doesn't mean subsystems *should* be separate services or languages, just that a well-drawn boundary *could* support it.
- **Interface formalization:** for each boundary, recommend how to formally describe the public interface. Use language-native types (typespecs, behaviours, protocols, Dialyzer annotations) for boundaries within a single codebase. Use language-agnostic specs (JSON Schema, OpenAPI, AsyncAPI, Protocol Buffers, GraphQL schema) where the boundary crosses — or could cross — a language or team boundary. When both apply, recommend both: language-native for internal callers, language-agnostic for the contract. The Analyst should be specific: not just "add a JSON Schema" but "define a JSON Schema for the WorkerState payload shape that both the Elixir consumer and the external producer validate against."
- May add items to `-questions.org` if a boundary is ambiguous (e.g. "should heartbeat monitoring be part of the worker lifecycle subsystem or a separate observability subsystem?").
- Append an "Iteration 1 — Boundary Analyst" entry to the journal.

### 2. User fills in `-answers.org`

The user populates Considerations, Questions, and Criticism sections.

### 3. Process answers — `loop <feature-name>`

**Step 3a — Apply answers.** Read `-answers.org`. For each answer:
- Update `<feature-name>.org` to reflect the resolved decision
- Move the answered question from `-questions.org` to the `Questions` section of `-resolved.org`, appending the answer inline
- Move the resolved criticism from `-criticism.org` to the `Criticism` section of `-resolved.org`, appending the resolution inline
- If an answer introduces a new question or concern, add it to the respective file
- If the user marks a question or criticism as **`defer`** (or words to that effect), move it to `-resolved.org` with status "Deferred" and a rationale, and add it to the **Known Deferred Work** section of the main design document

**Step 3b — Validator subagent.** After applying answers, launch a subagent that reads `-answers.org`, `-resolved.org`, and `<feature-name>.org` to check:
- Are any answers too vague or ambiguous to act on? (e.g. "TBD", "maybe", "probably")
- Do any new answers contradict previously resolved decisions?
- Do any answers create implicit new questions the user didn't notice?

Issues found are added to `-questions.org` or `-criticism.org` before the convergence check. If no issues are found, the validator reports clean and processing continues. The Validator appends a journal entry summarizing its assessment.

**Step 3c — Critic + Boundary Analyst re-evaluation (in parallel).** After the Validator, launch both subagents again.

*Critic re-evaluation subagent.* Reads:
- `<feature-name>-critic-context.org` — its own persistent context from prior iterations
- The updated `<feature-name>.org` — to see how the design changed
- `-resolved.org` — to assess whether resolutions genuinely addressed its prior concerns
- `-journal.org` — for cross-role context

The Critic then:
1. Moves concerns it considers genuinely resolved from "Active Concerns" to "Resolved Concerns" in its context file, with a quality assessment
2. Adds new concerns raised by the design changes to "Active Concerns"
3. Updates "Patterns Noticed" if it observes recurring issues (e.g. "specs not updated alongside prose for the third time")
4. Updates `<feature-name>-criticism.org` with any new or escalated concerns
5. Appends a journal entry summarizing what changed in its assessment

If the Critic has active concerns tagged as "blocking" that haven't been addressed for 2+ iterations, it flags them explicitly to the user with an escalation note.

*Boundary Analyst re-evaluation subagent.* Reads:
- `<feature-name>-boundary-context.org` — its own persistent context
- The updated `<feature-name>.org` — to see how resolved decisions affect boundaries
- `-resolved.org` — decisions may have merged, split, or shifted subsystems
- `-journal.org` — for cross-role context

The Boundary Analyst then:
1. Updates "Identified Subsystems" — boundaries may have shifted, merged, or split based on resolved decisions
2. Logs changes in "Boundary Changes" with iteration number and reason
3. Updates "Boundary Tensions" — new decisions may have introduced cross-boundary coupling or resolved prior tensions
4. May add items to `-questions.org` if a decision made a boundary ambiguous
5. Appends a journal entry summarizing boundary evolution

**Convergence check:** After processing, report the net change in open items (e.g. "Resolved 4, added 1 — 3 remain"). If a loop iteration produces more new items than it resolves, flag this to the user explicitly — the design may need to be split into smaller sub-features.

Once all answers are processed, recreate `-answers.org` with the three section headings only, pre-populated with the remaining open questions and criticisms copied under their respective sections as a reference — so the user can fill in answers without switching between files.

### 4. Check for completion

After processing answers, check all three conditions:
1. **No open questions** — `-questions.org` is empty or deleted
2. **No open criticisms** — `-criticism.org` is empty or deleted
3. **No design holes** — `<feature-name>.org` does not contain TBD, TODO, placeholder, or "to be decided" markers, and every section that the design references is fleshed out

- If any condition fails: list specifically what remains and suggest **`loop <feature-name>`**.
- If all three pass: suggest **`review [N]`** or **`finalize <feature-name>`**.

### 4a. Parallel subagent review — `review [N]`

Optional step before finalizing. Launches up to N specialized subagents **in parallel** (default N=7), each with a dedicated review role. Each subagent reads the design doc, `-journal.org` (for cross-iteration context), and all referenced source files but makes **no edits**. Before reporting on any file, each subagent must verify the file exists at the referenced path — if a referenced file is missing or has been renamed, report that as a Critical finding rather than assuming the file's contents.

| # | Role | Focus |
|---|------|-------|
| 1 | **Correctness Reviewer** | Verify claims against actual code, find missing operations or mismatched signatures. Check whether interfaces and data formats are defined via formal specs (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema) rather than only described in prose — flag any contract that could be a schema but isn't |
| 2 | **Edge Case Reviewer** | Concurrent access, race conditions, failure modes, missing error handling |
| 3 | **Integration Reviewer** | Caller/callee contracts, ETS access patterns, cross-module consistency |
| 4 | **API Reviewer** | Naming consistency, function signatures, module boundaries |
| 5 | **Test Reviewer** | Are the proposed tests sufficient to catch regressions? Are there untested code paths? Prefer e2e tests that use real infrastructure (DB, message queues, blob storage) and only stub non-replicable external services. Flag any proposed unit tests that mock infrastructure that could be tested for real. Suggest specific test cases and red-green TDD sequences where applicable |
| 6 | **Spec Reviewer** | Are all formal specifications (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migrations) complete and valid? Do they match the prose design? Are there contracts described only in prose that should be formal specs? |
| 7 | **Boundary Reviewer** | Read `-boundary-context.org`. Are subsystem boundaries clearly defined in the design? Are public interfaces narrow and sufficient? Does the design leak internal details across boundaries? Are there modules that straddle two subsystems and should be split? Does the design respect existing codebase boundaries or intentionally and explicitly change them? Is each boundary's interface formalized at the right level — language-native types for internal boundaries, language-agnostic specs (JSON Schema, OpenAPI, AsyncAPI, Protobuf) for cross-team/cross-language boundaries? Flag any boundary that lacks formal interface description. |

When N < 7, pick the N most relevant roles for the feature. When N > 7, additional subagents repeat with increasing skepticism, cross-referencing findings from the first 7.

Each subagent produces a structured report with severity ratings (Critical / High / Medium / Low / Info). Claude collates the reports, deduplicates overlapping findings, appends a combined "Review" journal entry summarizing findings by role, then updates the design to address all High+ findings and documents Low/Info findings as known tradeoffs.

After the review, re-assess for completion:
- If the review introduced new questions or design changes: suggest **`loop`**.
- If all findings are addressed: suggest **`finalize`**.

### 5. Finalize — `finalize <feature-name>`

**Step 5a — Consolidate.** Merge the completed `<feature-name>.org` design into the main design document under the appropriate section.

**Step 5b — Finalization Verifier subagent.** Before deleting files, launch a subagent that reads the consolidated section in the main design document and diffs it against `<feature-name>.org` + `-resolved.org` + `-journal.org`. It checks:
- Are all resolved decisions and their rationales represented in the final doc?
- Are all formal specs referenced in the design present and accounted for?
- Did any nuances from resolved criticisms get lost during consolidation?
- Does the Critic's context show any unresolved concerns that weren't explicitly deferred?
- Are subsystem boundaries from the Boundary Analyst's context clearly represented in the final doc — including public interfaces and hidden internals?
- Are key insights from the journal reflected in the final doc?

If the verifier finds gaps, Claude patches the consolidated section before proceeding.

**Step 5c — Clean up.** Delete all sub-feature files (`-questions.org`, `-criticism.org`, `-answers.org`, `-resolved.org`, `-critic-context.org`, `-boundary-context.org`, `-journal.org`, and `<feature-name>.org` itself).

After finalizing, read the **Known Deferred Work** section of the main design document and suggest **`next`** if there are remaining deferred items, naming the one you would pick next and why.

### 5a. Next deferred sub-feature — `next`

Read the **Known Deferred Work** section of the main design document. Pick the highest-priority remaining item — prefer items that unblock other items, or that are depended on by the current implementation phase. Run `start <feature-name>` for that item immediately (do not wait for confirmation unless the choice is genuinely ambiguous).

### 6. Continue / restart — `restart <feature-name>`

If the user wants to revisit a finalized feature — or creates a `<feature-name>.org` file manually to kick off a new design — recreate the full file set from the current state of the design:
- `<feature-name>-questions.org` — questions derived from the existing design
- `<feature-name>-criticism.org` — fresh criticism pass against the existing design (via Critic subagent)
- `<feature-name>-answers.org` — empty, ready for the user
- `<feature-name>-resolved.org` — empty (previous resolutions are already baked into the design doc)
- `<feature-name>-critic-context.org` — fresh Critic context initialized from the existing design
- `<feature-name>-boundary-context.org` — fresh Boundary Analyst context initialized from the existing design
- `<feature-name>-journal.org` — initialized with a "Restart" entry noting what triggered the revisit

Then suggest **`loop <feature-name>`** to continue.

---

## Standalone: Boundary Audit — `boundary-audit [scope]`

A standalone command that scans existing code for implicit subsystems
worth formalizing. Runs only when explicitly requested — never
automatic. Does not require a feature to be in progress.

**Input:** An optional scope to narrow the search (e.g. a directory,
a supervision tree, a module cluster). If omitted, scans the full
codebase.

**Process:** Launch a Boundary Analyst subagent that reads the
codebase within the given scope. It applies the same analysis as
during the Design Flow — independence litmus tests, interface
formalization levels, distillation opportunities — but without a
driving feature. To compensate for the missing filter, the output
is capped and ranked.

**Output:** A short report (max 5 candidates) written to
`boundary-audit-report.org` in the current working directory. Each
candidate includes:

```org
** <Subsystem Name>

*Modules:* (which existing modules form this cluster)

*Evidence:* (why these modules behave as a subsystem — e.g.
"called together by 6 different callers", "share a supervision
subtree", "all access the same ETS table")

*Current interface:* (how callers currently interact — scattered
direct calls, shared GenServer, etc.)

*Proposed boundary:* (what the formal public interface would
look like — which functions, which specs)

*Formalization:* (what specs to produce — typespecs/behaviours
for internal, JSON Schema/OpenAPI/AsyncAPI for cross-boundary)

*Benefit:* (what concretely improves — "new features targeting
worker state would depend on 1 interface instead of 3 modules",
"enables independent testing of the AMQP layer")

*Effort:* low / medium / high
```

Candidates are ranked by benefit-to-effort ratio. The report is
informational — the user decides which candidates (if any) to turn
into Design Flow features via **`start <subsystem-name>`**.

The report file is standalone and not part of any feature's file set.
Delete it manually when no longer needed, or keep it as a backlog.

---

## How to Use This Process

### Starting a new feature

1. Create a directory for the feature's documentation
2. Prompt: **`start <feature-name>`**

### Running a loop iteration

1. Open `<feature-name>-answers.org` and fill in the three sections
2. Prompt: **`loop`** (feature name inferred from context)

### Reviewing before finalize (optional)

When the design has no gaps but you want a thorough review: **`review`** (defaults to 7 parallel reviewers) or **`review 3`** for fewer.

### Finalizing

Once Claude suggests it (or after review), prompt: **`finalize`**

### Starting the next deferred sub-feature

After finalizing, prompt: **`next`** to start the next highest-priority item from Known Deferred Work.

### Continuing / restarting a finalized feature

Prompt: **`restart`** (or **`restart <feature-name>`** to specify a different feature)

### Auditing existing code for subsystem boundaries

Prompt: **`boundary-audit`** (full codebase) or **`boundary-audit lib/idunn_monitor/environment`** (scoped). Produces a ranked report of candidates. Turn promising ones into features with **`start`**.

---

## When NOT to Use This Process

Skip the design loop for changes that are:
- **Trivial** — renaming, typo fixes, config changes with obvious values
- **Fully specified** — the user gives exact requirements with no ambiguity
- **Bug fixes** — where the correct behavior is already defined

Use it when there are genuine design decisions to make, trade-offs to weigh, or when the feature touches multiple subsystems.

---

## Session Resumption

All design state lives in the eight files — a fresh Claude session can pick up where the previous one left off. When resuming:
1. Read all existing files for the feature (`<feature-name>.org`, `-questions.org`, `-criticism.org`, `-answers.org`, `-resolved.org`, `-critic-context.org`, `-boundary-context.org`, `-journal.org`)
2. If `-answers.org` has content the user filled in, run **`loop`** to process it
3. If `-answers.org` is empty/headings-only, report the current state (open questions count, open criticisms count) and wait for the user

---

## Concurrent Features

Multiple features can be in-flight simultaneously — each has its own file set. To avoid confusion:
- Always include `<feature-name>` when switching between features in the same session
- Never modify one feature's files while processing another feature's loop

---

## Notes

- After every iteration, suggest the applicable next command(s) to the user — e.g. **`loop <feature-name>`** if gaps remain, **`finalize <feature-name>`** if the design is complete, **`next`** after finalizing.
- All seven commands (`start`, `loop`, `review`, `finalize`, `restart`, `next`, `boundary-audit`) accept an optional `<feature-name>` (except `boundary-audit`, which accepts an optional scope path instead). `review` also accepts an optional reviewer count (default 7). For `next`, a name is not needed — it is derived from Known Deferred Work. When the name is omitted, use the last feature discussed in the current conversation. If no feature has been discussed yet and none can be inferred from context, ask the user to specify one.
- When suggesting next commands after an iteration, omit the feature name from the suggestion if it matches the current feature — e.g. suggest **`loop`** rather than **`loop <feature-name>`**.
- Process all answers in a single pass — do not ask follow-up questions mid-loop.
- **Batch size guidance:** If a loop iteration produces more than 10 open questions, group them by theme and mark the groups with priorities (blocking / important / nice-to-have). This helps the user triage rather than face a wall of undifferentiated questions.
- When a criticism is "accepted as a known gap", move it to `-resolved.org` and also add it to the **Known Deferred Work** section of the main design document.
- Sub-features of a feature follow the same loop — one set of files per sub-feature, or a single set covering all sub-features together if they are tightly related.
- Never delete `-resolved.org` mid-loop. It is only deleted as part of the finalize step.
