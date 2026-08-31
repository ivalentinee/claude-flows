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

**Function declarations should communicate purpose without reading the body.**
- A reader should make a fair guess at what a function does AND how
  it does it from the declaration alone: name, argument names and
  types, return type.
- Name communicates WHAT: `compile_template`, `validate_segments`
- Argument names communicate WITH WHAT: `(template, compiler_options)`
  not `(data, opts)`
- Argument types communicate CONSTRAINTS: `(ExpandedTemplate, CompilerOptions)`
  not `(map(), keyword())`
- Return type communicates OUTCOME: `{:ok, CompiledTemplate} | {:error, CompilationError}`
  not `any()`
- If the declaration doesn't tell the story, the function is either
  poorly named, has overly generic types, or does too many things.
- Elixir: `@spec` on public functions is mandatory for this reason.
  TypeScript: avoid `any`, use domain types and branded types.

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

## Module Roles

Every module/file should serve one of three roles within its parent
responsibility. This applies to new code AND when refactoring — when
splitting an oversized module, classify the resulting files by role.

**Communication** — the module's inbound interface.
- GenServer handlers (`handle_call`, `handle_cast`, `handle_info`)
- Public API functions, controller actions, route handlers
- Receives requests from other modules and dispatches to action
- Knows the most about the system (routing, protocol, callers)
- Should contain NO business logic and NO data construction

**Action** — business logic dispatched by communication.
- Decisions, transformations, side effects, orchestration
- Knows less than communication (only its domain concern)
- Calls data builders for construction, returns results to
  communication
- Multiple action files per responsibility are valid when the
  module acts in "multiple directions"

**Data Building** — constructing, validating, shaping data structures.
- Node/struct construction, serialization, parsing, normalization
- Knows the least about the system (only data shapes)
- Takes visual space that would clutter action code
- Pure Fabrication (GRASP): exists for design clarity, not domain

### Scope reduction gradient

Dependencies and knowledge flow in one direction:
`communication → action → data building`. Each successive role knows
less about its surroundings. The caller knows more globally; the
callee knows more details. This is the Clean Architecture Dependency
Rule applied within a module.

### "I don't care" principle

Each role is oblivious to the others' internals. A data builder
doesn't know who calls it or why. An action doesn't know how
communication received the request. Communication doesn't know how
data is shaped internally.

### Naming conventions by role

| Role | Filename prefixes | Example |
|------|------------------|---------|
| Communication | module name, handler suffix | `operation_orchestrator.ex` |
| Action | verb + domain noun | `compile_template.ts`, `validate_segments.ex` |
| Data Building | `build_*`, `create_*`, `get_*` | `build_output_node.ts`, `create_run_config.ex` |

### When refactoring existing code

When a module mixes roles (common in legacy code):
1. Identify which lines are communication, action, or data building
2. Extract each role into its own file/module
3. Name files by role convention
4. Verify dependencies flow communication → action → data
5. Check: does each resulting file have a single role?

---

## Grepable Code

Code should be discoverable using `find` (filenames) and
`grep -B10 -A10` (local context) without reading entire modules.

The core principle is **grep continuity**: not grepability by the
same term, but grepability step-by-step without dead ends. Each
grep hop lands at code that reveals the next term to grep for.
The chain never breaks — even when a value is renamed through a
transformation, the transformation function's declaration contains
both the old and new names.

This applies equally to new code AND refactoring existing code.
When touching existing code, improve grepability opportunistically.

**Every concept gets a grep handle.**
- A grep handle is a unique, stable string connecting a concept's
  definition to all usage sites. When you grep for it, you find
  every place that concept appears.
- Generic names (`data`, `extra`, `config`, `info`, `payload`) are
  NOT grep handles — they match everything and identify nothing.
- A generic name is acceptable ONLY when: (a) it's a domain-
  conventional name universal across the codebase (e.g., `config`
  always meaning "operation config" everywhere), or (b) it's fully
  scoped within one file (a grep within the file answers what it is).
- The moment a generic-named untyped bag crosses file boundaries,
  it needs either a domain-qualified name or a typed struct.

**Consistent names across boundaries.**
- When a value crosses a boundary (API → backend → database), the
  root words stay identical. Only case convention may change:
  `scalingFactor` (JS) → `scaling_factor` (Elixir) → `scaling_factor`
  (DB column).
- NEVER rename a concept at a boundary. If `maxHeight` enters the
  system, it stays `maxHeight`/`max_height` everywhere — not
  `scalingFactor` downstream.
- If a computed value is genuinely different, it gets its own name
  AND the transformation function's declaration links the two names:
  `maxHeightToScalingFactor(maxHeight: number): number`. This way
  grepping for `maxHeight` lands at the transformation, and grepping
  for `scalingFactor` continues the trail — no comment needed. Do
  NOT annotate every usage with "derived from X" comments; that
  floods the codebase with noise that a single grep can resolve.

**Function names include subject, not just action.**
- `compile` → `compile_template`. `resolve` → `resolve_graph_node`.
  `setup` → `setup_operation_environment`.
- The grep test: grepping for the function name should find ONLY
  calls related to that specific domain concept.
- When refactoring: rename existing functions to include their
  subject when the current name is ambiguous.

**Filenames reveal purpose.**
- `setup.ex` → `operation_environment_setup.ex`.
  `entries.ts` → `template_run_entries.ts`.
  `utils.ts` → split into domain-specific files.
- The `find` test: scanning filenames should tell you where to look
  for a specific concept without opening files.
- When refactoring: rename files whose current name gives no hint
  of their actual content.

**No opaque map containers crossing file boundaries.**
- `extra: :map` is a grepability black hole — 22 matches in the
  template_runner, all pointing to the same opaque `%{string => any}`.
- Use typed structs (Elixir) or interfaces (TypeScript) to make
  container contents grepable. At minimum, add `@type` specs or
  doc comments listing expected keys.
- String-keyed map access (`get_in(extra, ["scalingFactor"])`) is
  invisible to refactoring tools. Prefer atom keys and struct
  access where possible.
- Dialyzer typespecs bring additional discoverability properties.

**Elixir-specific: use full alias form.**
- `alias A.B.C` (grepable) not `alias A.B.{C, D}` (compact but
  grep-breaking). The full module path must appear as a contiguous
  string in the source.

**Cross-language schemas.**
- When Elixir and TypeScript (or any two languages) share data via
  AMQP, HTTP, or other cross-system interactions, use shared schema
  definitions (JSON Schema, OpenAPI) referenced by both sides.
- Internal Elixir-NodeJS communication within the same system does
  not require formal shared schemas if the contract is small and
  well-typed on both ends.

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

Read and apply `GROUNDED-REASONING.md` (in this directory). The nine
principles (execution over reasoning, experiment to understand,
isolate and test, separate generator/verifier, claims require
evidence, freshness discipline, avoidance requires proof, ground in
reality not user's statements, evidence spectrum) apply to ALL flows.

**Code-specific extensions** (below) supplement those principles:

**Mutate to understand.**
- Code is not an immutable artifact during investigation. To
  understand what a value does, change it and run the code. To
  understand a condition, invert it and observe. To understand a
  function's role, remove its call and see what breaks.
- Always restore changes after investigation (`git checkout` or
  `git stash`), but never hesitate to make them.

**Isolate and run (code is voxels, not a monolith).**
- Any piece of code can be run in isolation. The whole project is
  never the minimum runnable unit.
- To understand `algorithm.js`, create a throwaway 10-line script
  that imports it, feeds it sample data, and prints the result.
- To understand an Elixir module, open `iex -S mix` and call its
  functions directly with sample arguments.
- The cost of creating a throwaway caller is almost always less
  than the cost of reasoning about the code's behavior from
  reading alone.

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

*Grepability:*
11. Can you trace any key value's path through the changed code
    with semantic continuity? (Consistent names across boundaries,
    no opaque containers crossing files, meaningful function/file
    names.) When refactoring, check: did you improve or degrade
    traceability?
12. Do new function names include the domain subject? (Not just the
    action — `setup_operation_environment` not `setup`.) When
    renaming existing functions, add the subject.
13. Are generic container names (`data`, `extra`, `config`) either
    domain-conventional or file-scoped? If they cross file
    boundaries, do they have a typed struct/interface?

---

## Reference Projects

These repos demonstrate the principles above in working code.
Use them as style calibration sources alongside `git log --author`.

**Elixir — [ceiling-ui](https://github.com/ivalentinee/ceiling-ui)**
Phoenix/LiveView app controlling LED ceiling zones. Demonstrates:
- Three-role module split: `Zones` (communication/supervision) →
  `ZoneRunner` (action/GenServer) → `Render`, `State`, `Timer`
  (data building)
- Sub-module decomposition: `ZoneRunner.State` (struct + builders),
  `ZoneRunner.Render` (async frame compilation),
  `ZoneRunner.Render.Frame` (data struct)
- Behaviour-based extensibility: `Generators.Behaviour` defines
  `name/0`, `load/1`, `render/3`; `Generators.Collection` is the
  registry
- Ecto `embedded_schema` for validation without a database table
  (`Scenes.Scene`)
- ETS + File dual storage behind a single facade (`Scenes.Storage`)
- Full alias form (`alias CeilingUI.Zones.ZoneRunner`, not
  `alias CeilingUI.Zones.{ZoneRunner, ...}`)
- Typed structs with `@enforce_keys`

---

## Integration with Flows

The implementation steps in all flows (Full Flow Step 10) include
a "Style calibration" step
that references this file. The Style, Principles, and Code Clarity
reviewers in the self-review phase check against these principles.
The Code Clarity Reviewer (#9) specifically uses the "Self-Documenting
Code" checklist above.

This file is generic and portable across projects. Project-specific
coding conventions (language idioms, framework patterns, specific
library usage) belong in project memory files or CLAUDE.md, not here.
