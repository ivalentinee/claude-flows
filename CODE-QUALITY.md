# Code Quality — Instructions for Claude

Persistent code quality principles that apply across all projects.
Claude tends to drift on code quality across sessions — these rules
counteract that. Read this file before writing implementation code.

---

## Style Calibration

Before writing code, sample the user's recent commits to calibrate
naming, structure, and idiom choices:

1. Run `git log --author` for the user's committer name, read 1-2
   recently touched files
2. Match the patterns you see — naming length, module granularity,
   error handling style, test structure
3. Do this silently — do not announce it

The user's own code is the authoritative style reference. Written
rules below are the floor; the user's code is the ceiling.

---

## Core Principles

**Use full, unambiguous variable names — no abbreviations, ever.**
- `rp` → `result_publisher`, `cfg` → `config`, `opts` → `options`
- This is the single most common Claude quality drift. Treat as a
  hard rule, not a preference.

**Split modules eagerly — do not let modules grow.**
- When a module starts handling two concerns, split immediately.
- Claude tends to add "just one more function" to existing modules.
  This compounds into grab-bag modules with mixed responsibilities.

**Prefer solutions with minimal impact — reuse existing systems.**
- Before building something new, grep/read the codebase for similar
  patterns. Extend existing modules/functions when possible.
- Claude tends to create new abstractions when extending existing
  ones would be simpler and more consistent.

**Never disable or weaken linter rules to fix a warning.**
- When a linter flags something (especially file length, function
  length, complexity), fix the code — split the module, extract
  a function, simplify the logic.
- Claude's default instinct is to add a disable comment or raise
  a threshold. This hides the problem. The linter is right more
  often than not.
- If a disable is genuinely warranted (rare), document *why* in
  a comment next to the disable, and second-guess the decision:
  "Is there really no way to fix this structurally?"

---

## Naming

**Prefer meaningful verbs for function names, not noun descriptions.**
- `error_state` → `fail` (pairs with `advance`). Nouns describe
  what something is; verbs describe what it does.
- Function names that trigger transitions or actions should be verbs.

**Rename fields to domain terms, not implementation terms.**
- `source_graph` → `template` (it holds a Template object).
- Field names should reflect the domain concept, not where the data
  came from or how it was derived.

---

## Code Structure

**Domain responsibility: put logic in the module that has context.**
- Ask "does this module know *why* the error happened?" — if yes,
  it owns the labelling. Generic modules should not know about
  specific domain contexts.

**No silent fallbacks for required fields. Validate explicitly.**
- Required fields get explicit guards that return error tuples.
- Optional fields get documented defaults.
- Never fall through to a meaningless default for required data.

**Validate/transform data before mutating state.**
- Build the replacement data structure completely *before* clearing
  state. If building fails, the original state is preserved.

**Write only what the recipient expects — not the full struct.**
- Before serializing for an external consumer, read the consumer's
  expected format. Send exactly that, not the internal representation.

---

## Test Structure

**Test directory structure mirrors source structure.**
- `lib/foo/bar/*.ex` → `test/foo/bar/*_test.exs`

**Test support helpers organized into subfolders.**
- Helpers go in subdirectories matching what they support, not flat
  in `test/support/`.

**Positive test cases float to the top of a describe block.**
- Order: success → edge cases → failures. Readers understand the
  happy path first.

**Combine tests that exercise the same code path into one test.**
- If two tests share identical setup and differ only in what they
  assert, merge them. They test a single scenario.

**Extract repeated setup patterns into named helpers.**
- Any `on_exit` restoration pattern or config swap used in more
  than one test → extract to a named helper function.

---

## Grounded Reasoning

Code quality depends on reality-testing, not just reasoning about
code. Five principles:

**Execution over reasoning.**
- When you can test a claim by running code, run it. Don't reason
  about what "would" happen — verify it.
- Experimentation is discovery, not just verification. When you
  don't understand what code does, the first move is to run it —
  not to read it harder.

**Mutate to understand.**
- Code is not an immutable artifact during investigation. To
  understand what a value does, change it and run the code. To
  understand a condition, invert it and observe. To understand a
  function's role, remove its call and see what breaks.
- This is the scientific method applied to code: change one
  variable, observe the effect, form a grounded understanding.
- Always restore changes after investigation (`git checkout` or
  `git stash`), but never hesitate to make them.

**Isolate and run (code is voxels, not a monolith).**
- Any piece of code can be run in isolation. The whole project is
  never the minimum runnable unit.
- To understand `algorithm.js`, create a throwaway 10-line script
  that imports it, feeds it sample data, and prints the result.
  Don't run the entire application.
- To understand an Elixir module, open `iex -S mix` and call its
  functions directly with sample arguments.
- The cost of creating a throwaway caller is almost always less
  than the cost of reasoning about the code's behavior from
  reading alone.

**Separate generator and verifier.**
- Never trust yourself to both generate and validate code. Write
  code, then run tests — separate steps, separate verification.

**Claims require evidence.**
- Assertions about behavior ("this will cause a race condition",
  "this library supports X") must be backed by evidence: file path
  + line number, test output, error reproduction, or benchmark.
  If you have not verified, say "I have not verified this claim."

**Freshness discipline.**
- Re-read files before making claims about their content. If more
  than ~5 reasoning steps have passed since reading a file, re-read
  before asserting facts about it.

**Avoidance requires proof.**
- "Pre-existing issue", "not worth pursuing", "unlikely edge case"
  are claims that require evidence. If you have not tested the claim,
  you cannot make it. Verify: `git stash` + run tests, measure with
  a benchmark, write a property test, or read the actual docs.
- "Behavior unchanged" requires the same evidence as "behavior
  changed." If claiming an issue is pre-existing, prove the code
  is identical in main AND causes the issue there too.

**Ground in reality, not in user's statements.**
- When the user proposes a technical approach, check it against
  actual code before implementing. If the code shows the user's
  model is wrong, inform before complying — state what the code
  actually shows, what the consequences are, let the user decide.
- This is not about disagreeing with the user. It is about Claude
  grounding its output in code reality, which creates productive
  friction that helps BOTH sides discover model drift.
- When Claude disagrees after checking: state the objection in one
  sentence with specific evidence, then comply if the user confirms.
  Record the disagreement. Never silently comply when evidence
  suggests the direction is wrong — but also never refuse.

**Evidence is not only execution.**
- A structural observation ("this module has mixed concerns") is
  grounded when the concerns are enumerated in writing without
  contradiction. Writing them down IS the evidence.
- Evidence spectrum: execution (test output, benchmark) >
  articulable enumeration (written list that survives scrutiny) >
  ungrounded assertion ("I think this might...").

### Property-First Test Design

Before writing test code, enumerate which behavioral properties
the test should validate (Beck's "test list" — the Canon TDD step
most practitioners skip):

1. List every relevant property class (response structure, error
   handling, edge cases, timing, resource cleanup)
2. For each, decide: test now (P0-P1), test later (P2), or not
   tested (with reason)
3. Write the specific predicate for each "test now" property

This prevents under-assertion (checking only that code runs) and
over-assertion (checking implementation details that will change).

---

## Self-Documenting Code

Guiding principle (Mokevnin's Mental Programming): good code encodes
the domain's mental model. When you read the source, you should
reconstruct the author's understanding of the problem. Every rule
below serves this goal.

**Prefer refactoring over commenting.**
- If you write a comment explaining what code does, that is a signal
  to rename, extract a function, or add a type constraint instead.
- Comments are justified only for: why a decision was made, links to
  external specs/tickets, business rules that cannot be modeled in
  types, and non-obvious performance/side-effect warnings.

**Use types to document constraints.**
- TypeScript: prefer branded types for entity IDs (`TemplateId`,
  `RunId`), discriminated unions for multi-state objects, `unknown`
  over `any`.
- Elixir: add @spec to all public functions. Use pattern matching
  in function heads to enumerate valid cases rather than branching
  internally.

**Name functions as meaningful domain verbs.**
- `process`, `handle`, `manage`, `do` are non-names — they
  communicate nothing about what the function actually does.
- Good: `validate_items`, `publish_result`, `advance_stage`
- Bad: `process_data`, `handle_event`, `do_work`

**Avoid vague container names.**
- `data`, `value`, `item`, `result`, `info`, `temp` reveal nothing.
- Name the variable after what it holds: `validated_order`,
  `publisher_config`, `retry_count`.

### Code Clarity Reviewer Checklist

Used by the Code Clarity Reviewer (#9 in self-review). Each item
checks whether the code encodes the domain's mental model:

*Naming clarity:*
1. Do function/variable names reveal purpose, not implementation?
   (`remaining_retries` not `n`, `template` not `source_graph`)
2. Do function names use meaningful verbs, not vague ones?
   (`validate_order` not `process`, `publish_result` not `handle_data`)
3. Do boolean names read as true/false propositions?
   (`is_expired`, `has_permission`, `can_retry`)
4. Do names use domain terminology from the design doc?

*Type expressiveness:*
5. Do types express constraints that would otherwise need comments?
   (branded types for IDs, discriminated unions for state, @spec)
6. Are function signatures self-documenting? (no `any`, no raw
   `string` where a domain type would be clearer)

*Comment discipline:*
7. Does every comment explain *why*, not *what*?
8. Could any comment be eliminated by renaming, extracting a
   function, or adding a type constraint?

*Structural clarity:*
9. Does each function operate at one level of abstraction? Are there
   missing intermediate tiers? (Composed Method — see Abstraction
   Minimalist checklist for the full heuristic)
10. Is control flow self-evident, or does understanding require
    reading non-adjacent code?

---

## Integration with Flows

The implementation steps in all flows (Full Flow Step 9,
Implementation Flow Step 2) include a "Style calibration" step
that references this file. The Style, Principles, and Code Clarity
reviewers in the self-review phase check against these principles.
The Code Clarity Reviewer (#9) specifically uses the "Self-Documenting
Code" checklist above.

This file is generic and portable across projects. Project-specific
coding conventions (language idioms, framework patterns, specific
library usage) belong in project memory files or CLAUDE.md, not here.
