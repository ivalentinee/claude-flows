# Implementation Flow — Instructions for Claude

A structured process for implementing features after the Design flow
is finalized. Every feature goes through the same steps: implement,
self-review, fix, then iterate with the user via review-plan and
review-notes.

---

## Prerequisites

- A finalized design exists in the project's main design document
  (produced by the Design flow).
- If no finalized design is found, or the target feature is ambiguous,
  suggest starting the Design flow first or ask the user to specify
  which feature to implement.
- The last feature designed is assumed to be the target. If unsure,
  ask the user.

---

## Files

All implementation files live in the feature's documentation directory
(same directory where the Design flow files lived).

| File                       | Purpose                                                          |
|----------------------------|------------------------------------------------------------------|
| `review-plan.org`          | Emacs org-mode file listing changed files grouped by concern     |
| `review-notes.md`          | User's review notes — first-level headings per file              |
| `review-notes-resolved.md` | Archive of resolved review notes with resolution explanations    |
| `<feature>-review.md`      | Self-review output (persists for user reference)                 |

---

## The Steps

### 1. Implement — `implement <feature-name>`

Read the finalized design section in the main design document.

Before writing code, consider:
- What code properties (correctness, cohesiveness, consistency with
  existing patterns) would benefit the result
- Whether documentation-based approaches apply (JSON Schema
  validations, generated types, OpenAPI/AsyncAPI)
- Whether infrastructure/exploitation info should be read from or
  updated in the project's exploitation doc

Then write the implementation, including tests for the new feature's key behaviors and edge cases — do not rely solely on existing tests to validate new code. **Test strategy:** prefer e2e tests that exercise real infrastructure (DB, message queues, blob storage) over unit tests with mocks. Only stub services that cannot be replicated locally (third-party APIs, payment providers, external SaaS). If a dependency can run in docker-compose or an emulator (e.g. Azurite for Azure Blob), test against the real thing. Run the project's test/lint suite.

### 2. Parallel self-review — automatic after step 1

Launch specialized reviewer subagents **in parallel**, each with an
**adversarial stance** — assume the code is wrong and try to prove it.
Each reviewer is a different "person" than the author; same model, so
actively compensate for shared blind spots by checking every assumption
against the actual code, not memory.

| # | Role | Focus |
|---|------|-------|
| 1 | **Design Fidelity Reviewer** | Re-read the finalized design and diff intent vs. implementation. Flag any drift, missing pieces, undocumented deviations, or design decisions that were silently ignored |
| 2 | **Correctness Reviewer** | Logic errors, off-by-one mistakes, wrong assumptions, null/undefined handling, incorrect return types |
| 3 | **Edge Case Reviewer** | Race conditions, failure modes, error handling paths, boundary conditions, concurrent access |
| 4 | **Test Reviewer** | Are there behaviors or branches not covered by the new tests? Are tests hitting real infrastructure where possible, or unnecessarily mocking things that can run locally? Suggest missing test cases |
| 5 | **Spec Reviewer** | Are formal specifications (JSON Schema, OpenAPI, AsyncAPI, GraphQL schema, DB migrations) implemented correctly? Do runtime validations match the declared schemas? |
| 6 | **Style Reviewer** | Naming consistency, pattern consistency with the existing codebase, code organization, module boundaries |

Each subagent reads the diff from step 1 and all relevant source files
but makes **no edits**. Each produces a structured report with severity
ratings (Critical / High / Medium / Low / Info).

Claude collates and deduplicates the reports into
`<feature>-review.md` in the feature documentation directory. This file
persists — the user may consult it to understand what was auto-fixed.

### 3. Fix — automatic after step 2

Read `<feature>-review.md`. For each issue:
- Fixable → apply the fix
- Blocking (design ambiguity, out of scope) → raise for discussion

Re-run the test/lint suite.

### 4. Review plan + review notes — automatic after step 3

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

Then create `review-notes.md` with first-level headings for each file
from `review-plan.org`, listed in the same order. Each heading uses
the full path from the project root, wrapped in backticks — e.g.
`## \`path/to/file.ex\``. The user fills in notes under each heading.

### 5. Iterate — `loop`

**Step 5a — Apply fixes.** Read `review-notes.md`. For each note:
- Apply the fix or change requested
- Move the note to `review-notes-resolved.md`, appending a
  `**Resolved:**` subsection explaining what was done

**Step 5b — Fix Validator subagent.** After applying all fixes, launch
a subagent that reads the diff of changes made in this iteration,
`review-notes.md`, and the finalized design. It checks:
- Did each fix actually address the review note it claims to resolve?
- Did any fix introduce new issues or regress previously working code?
- Do the changes still match the finalized design?

Issues found are reported to the user alongside the review-plan update.

**Step 5c — Update review plan.** Update `review-plan.org`:
- If a file was modified by this iteration: uncheck its `- [ ]` item
  and update the review description to reflect only the latest change
  (so the user reviews only what changed, not the whole file again).
- If a file was not modified: keep its existing check state.
- If new files were created: add them to the appropriate section.
- If files were removed: move them to a "deleted" list (no checkbox).

Re-run the test/lint suite.

If `review-notes.md` still has unresolved notes, suggest **`loop`**.
If all notes are resolved, suggest **`finalize`**.

### 6. Finalize — `finalize`

**Step 6a — Completeness Verifier subagent.** Before deleting files,
launch a subagent that reads the finalized design and the current
implementation. It checks:
- Does the implementation cover every aspect of the finalized design?
- Are there design requirements that were not implemented or were only
  partially implemented?
- Are all formal specs (JSON Schema, OpenAPI, AsyncAPI) from the design
  present in the codebase?
- Do all tests pass and cover the key behaviors described in the design?

If the verifier finds gaps, report them to the user before proceeding.
The user can choose to address them now (back to step 5) or defer them.

**Step 6b — Clean up.** Delete all review-related files from the
feature documentation directory:
- `review-plan.org`
- `review-notes.md`
- `review-notes-resolved.md`
- `<feature>-review.md`

Suggest **`commit`**.

### 7. Commit — `commit`

Create a commit following the project's existing commit convention.

### 8. Retrospective (optional) — `retro`

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

## Resuming Interrupted Work — `resume`

If the user returns to a partially completed implementation (e.g. new
session, interrupted work):

1. Read all implementation files in the feature documentation directory
2. Check git status/diff to see what code changes exist
3. Report current state: which step was last completed, what remains
4. If `review-notes.md` has unprocessed notes → suggest **`loop`**
5. If implementation is done but no review files exist → run step 2
   (self-review) onward
6. If no code changes exist → start from step 1

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

## Trigger Phrases

- "Let's start the implementation flow"
- "Let's start the implementation"
- Typically follows finalization of the Design flow

---

## How to Use This Process

### Starting implementation

After finalizing a design: **`implement <feature-name>`**

Steps 1–4 run automatically: implement → self-review → fix →
create review-plan.org + review-notes.md.

### User fills in review-notes.md, then

Prompt: **`loop`** — Claude applies fixes, updates review-plan.org,
archives resolved notes

### When all notes resolved

Prompt: **`finalize`** — Claude deletes review files

### After finalize

Prompt: **`commit`**
