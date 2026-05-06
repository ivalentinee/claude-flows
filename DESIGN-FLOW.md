# Design Loop — Instructions for Claude

An iterative design process for features and sub-features. Every feature goes through the same loop before its design is consolidated into the main design document.

---

## The Five Files

For a feature named `<feature-name>`, create:

| File | Purpose |
|------|---------|
| `<feature-name>.md` | Design doc — describes what the feature does and how it should be implemented. Grows richer each loop iteration. |
| `<feature-name>-questions.md` | Open questions — things that cannot be decided without user input. Items are moved to `-resolved.md` as answers arrive. When empty, delete the file. |
| `<feature-name>-criticism.md` | Criticisms and concerns about the current design. Items are moved to `-resolved.md` as they are resolved. When empty, delete the file. |
| `<feature-name>-answers.md` | Written by the user. Three sections (see below). Read by Claude at the start of each loop iteration. Never modified by Claude. |
| `<feature-name>-resolved.md` | Archive of resolved questions and criticisms. Two sections: `Questions` and `Criticism`. Append-only — never remove entries. Created on first resolution. |

All files start with **headings only** — no content. Content is added by Claude (design/questions/criticism/resolved) or the user (answers) as the loop progresses.

---

## `-answers.md` Structure

```markdown
# Answers

## Considerations

(User writes any context, constraints, or design preferences that
should shape the implementation — things that are neither direct
answers to questions nor responses to criticism.)

## Questions

(After the first loop iteration: open questions copied here from
`-questions.md` as a reference. User writes answers beneath each one.)

## Criticism

(After the first loop iteration: open criticisms copied here from
`-criticism.md` as a reference. User writes responses beneath each one.)
```

On the very first pass (step 1), `-answers.md` has headings only. After every subsequent loop iteration (step 3), Claude recreates it with the still-open questions and criticisms pre-populated under their headings, ready for the user to answer in place.

---

## `-resolved.md` Structure

```markdown
# Resolved

## Questions

(Resolved questions moved here from `-questions.md`, with their
answers inline. Each entry includes the original question, the
answer, and a **Rationale:** line explaining *why* this decision
was made — so the reasoning survives even if the design doc only
records the outcome.)

## Criticism

(Resolved criticisms moved here from `-criticism.md`, with their
resolutions inline. Each entry includes the original criticism,
the resolution, and a **Rationale:** line explaining *why* this
resolution was chosen.)
```

---

## The Loop

### 1. Initial pass — `start <feature-name>`

This step uses two subagents with distinct roles to separate authoring from criticism.

**Step 1a — Author subagent.** Read the relevant source files. Write:
- `<feature-name>.md` — what the feature does and a proposed design. **Documentation-first:** actively look for opportunities to define behavior through formal specifications (JSON Schema for data validation, OpenAPI for HTTP interfaces, AsyncAPI for message-based interfaces, GraphQL schema for query APIs, DB migration schemas for data models). Prefer declarative, machine-readable definitions over prose descriptions of formats and contracts — the spec *is* the documentation.
- `<feature-name>-questions.md` — genuine unknowns that need user input (not self-criticism disguised as questions)

The Author subagent does NOT write criticisms — it advocates for its own design.

**Step 1b — Critic subagent.** Read the design doc and questions produced by the Author, then independently read the same source files. Write:
- `<feature-name>-criticism.md` — concerns, gaps, and risks in the proposed design

The Critic has not written the design and is explicitly prompted to distrust it: check every claim against actual code, look for what the Author overlooked, and challenge assumptions. The Critic may also add items to `-questions.md` if it identifies unknowns the Author missed.

### 2. User fills in `-answers.md`

The user populates Considerations, Questions, and Criticism sections.

### 3. Process answers — `loop <feature-name>`

**Step 3a — Apply answers.** Read `-answers.md`. For each answer:
- Update `<feature-name>.md` to reflect the resolved decision
- Move the answered question from `-questions.md` to the `Questions` section of `-resolved.md`, appending the answer inline
- Move the resolved criticism from `-criticism.md` to the `Criticism` section of `-resolved.md`, appending the resolution inline
- If an answer introduces a new question or concern, add it to the respective file
- If the user marks a question or criticism as **`defer`** (or words to that effect), move it to `-resolved.md` with status "Deferred" and a rationale, and add it to the **Known Deferred Work** section of the main design document

**Step 3b — Validator subagent.** After applying answers, launch a subagent that reads `-answers.md`, `-resolved.md`, and `<feature-name>.md` to check:
- Are any answers too vague or ambiguous to act on? (e.g. "TBD", "maybe", "probably")
- Do any new answers contradict previously resolved decisions?
- Do any answers create implicit new questions the user didn't notice?

Issues found are added to `-questions.md` or `-criticism.md` before the convergence check. If no issues are found, the validator reports clean and processing continues.

**Convergence check:** After processing, report the net change in open items (e.g. "Resolved 4, added 1 — 3 remain"). If a loop iteration produces more new items than it resolves, flag this to the user explicitly — the design may need to be split into smaller sub-features.

Once all answers are processed, recreate `-answers.md` with the three section headings only, pre-populated with the remaining open questions and criticisms copied under their respective sections as a reference — so the user can fill in answers without switching between files.

### 4. Check for completion

After processing answers, check all three conditions:
1. **No open questions** — `-questions.md` is empty or deleted
2. **No open criticisms** — `-criticism.md` is empty or deleted
3. **No design holes** — `<feature-name>.md` does not contain TBD, TODO, placeholder, or "to be decided" markers, and every section that the design references is fleshed out

- If any condition fails: list specifically what remains and suggest **`loop <feature-name>`**.
- If all three pass: suggest **`review [N]`** or **`finalize <feature-name>`**.

### 4a. Parallel subagent review — `review [N]`

Optional step before finalizing. Launches up to N specialized subagents **in parallel** (default N=6), each with a dedicated review role. Each subagent reads the design doc and all referenced source files but makes **no edits**. Before reporting on any file, each subagent must verify the file exists at the referenced path — if a referenced file is missing or has been renamed, report that as a Critical finding rather than assuming the file's contents.

| # | Role | Focus |
|---|------|-------|
| 1 | **Correctness Reviewer** | Verify claims against actual code, find missing operations or mismatched signatures. Check whether interfaces and data formats are defined via formal specs (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema) rather than only described in prose — flag any contract that could be a schema but isn't |
| 2 | **Edge Case Reviewer** | Concurrent access, race conditions, failure modes, missing error handling |
| 3 | **Integration Reviewer** | Caller/callee contracts, ETS access patterns, cross-module consistency |
| 4 | **API Reviewer** | Naming consistency, function signatures, module boundaries |
| 5 | **Test Reviewer** | Are the proposed tests sufficient to catch regressions? Are there untested code paths? Prefer e2e tests that use real infrastructure (DB, message queues, blob storage) and only stub non-replicable external services. Flag any proposed unit tests that mock infrastructure that could be tested for real. Suggest specific test cases and red-green TDD sequences where applicable |
| 6 | **Spec Reviewer** | Are all formal specifications (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migrations) complete and valid? Do they match the prose design? Are there contracts described only in prose that should be formal specs? |

When N < 6, pick the N most relevant roles for the feature. When N > 6, additional subagents repeat with increasing skepticism, cross-referencing findings from the first 6.

Each subagent produces a structured report with severity ratings (Critical / High / Medium / Low / Info). Claude collates the reports, deduplicates overlapping findings, then updates the design to address all High+ findings and documents Low/Info findings as known tradeoffs.

After the review, re-assess for completion:
- If the review introduced new questions or design changes: suggest **`loop`**.
- If all findings are addressed: suggest **`finalize`**.

### 5. Finalize — `finalize <feature-name>`

**Step 5a — Consolidate.** Merge the completed `<feature-name>.md` design into the main design document under the appropriate section.

**Step 5b — Finalization Verifier subagent.** Before deleting files, launch a subagent that reads the consolidated section in the main design document and diffs it against `<feature-name>.md` + `-resolved.md`. It checks:
- Are all resolved decisions and their rationales represented in the final doc?
- Are all formal specs referenced in the design present and accounted for?
- Did any nuances from resolved criticisms get lost during consolidation?

If the verifier finds gaps, Claude patches the consolidated section before proceeding.

**Step 5c — Clean up.** Delete all sub-feature files (`-questions.md`, `-criticism.md`, `-answers.md`, `-resolved.md`, and `<feature-name>.md` itself).

After finalizing, read the **Known Deferred Work** section of the main design document and suggest **`next`** if there are remaining deferred items, naming the one you would pick next and why.

### 5a. Next deferred sub-feature — `next`

Read the **Known Deferred Work** section of the main design document. Pick the highest-priority remaining item — prefer items that unblock other items, or that are depended on by the current implementation phase. Run `start <feature-name>` for that item immediately (do not wait for confirmation unless the choice is genuinely ambiguous).

### 6. Continue / restart — `restart <feature-name>`

If the user wants to revisit a finalized feature — or creates a `<feature-name>.md` file manually to kick off a new design — recreate the full file set from the current state of the design:
- `<feature-name>-questions.md` — questions derived from the existing design
- `<feature-name>-criticism.md` — fresh criticism pass against the existing design
- `<feature-name>-answers.md` — empty, ready for the user
- `<feature-name>-resolved.md` — empty (previous resolutions are already baked into the design doc)

Then suggest **`loop <feature-name>`** to continue.

---

## How to Use This Process

### Starting a new feature

1. Create a directory for the feature's documentation
2. Prompt: **`start <feature-name>`**

### Running a loop iteration

1. Open `<feature-name>-answers.md` and fill in the three sections
2. Prompt: **`loop`** (feature name inferred from context)

### Reviewing before finalize (optional)

When the design has no gaps but you want a thorough review: **`review`** (defaults to 6 parallel reviewers) or **`review 3`** for fewer.

### Finalizing

Once Claude suggests it (or after review), prompt: **`finalize`**

### Starting the next deferred sub-feature

After finalizing, prompt: **`next`** to start the next highest-priority item from Known Deferred Work.

### Continuing / restarting a finalized feature

Prompt: **`restart`** (or **`restart <feature-name>`** to specify a different feature)

---

## When NOT to Use This Process

Skip the design loop for changes that are:
- **Trivial** — renaming, typo fixes, config changes with obvious values
- **Fully specified** — the user gives exact requirements with no ambiguity
- **Bug fixes** — where the correct behavior is already defined

Use it when there are genuine design decisions to make, trade-offs to weigh, or when the feature touches multiple subsystems.

---

## Session Resumption

All design state lives in the five files — a fresh Claude session can pick up where the previous one left off. When resuming:
1. Read all existing files for the feature (`<feature-name>.md`, `-questions.md`, `-criticism.md`, `-answers.md`, `-resolved.md`)
2. If `-answers.md` has content the user filled in, run **`loop`** to process it
3. If `-answers.md` is empty/headings-only, report the current state (open questions count, open criticisms count) and wait for the user

---

## Concurrent Features

Multiple features can be in-flight simultaneously — each has its own file set. To avoid confusion:
- Always include `<feature-name>` when switching between features in the same session
- Never modify one feature's files while processing another feature's loop

---

## Notes

- After every iteration, suggest the applicable next command(s) to the user — e.g. **`loop <feature-name>`** if gaps remain, **`finalize <feature-name>`** if the design is complete, **`next`** after finalizing.
- All six commands (`start`, `loop`, `review`, `finalize`, `restart`, `next`) accept an optional `<feature-name>`. `review` also accepts an optional reviewer count (default 6). For `next`, a name is not needed — it is derived from Known Deferred Work. When the name is omitted, use the last feature discussed in the current conversation. If no feature has been discussed yet and none can be inferred from context, ask the user to specify one.
- When suggesting next commands after an iteration, omit the feature name from the suggestion if it matches the current feature — e.g. suggest **`loop`** rather than **`loop <feature-name>`**.
- Process all answers in a single pass — do not ask follow-up questions mid-loop.
- **Batch size guidance:** If a loop iteration produces more than 10 open questions, group them by theme and mark the groups with priorities (blocking / important / nice-to-have). This helps the user triage rather than face a wall of undifferentiated questions.
- When a criticism is "accepted as a known gap", move it to `-resolved.md` and also add it to the **Known Deferred Work** section of the main design document.
- Sub-features of a feature follow the same loop — one set of files per sub-feature, or a single set covering all sub-features together if they are tightly related.
- Never delete `-resolved.md` mid-loop. It is only deleted as part of the finalize step.
