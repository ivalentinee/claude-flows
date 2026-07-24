# Denote Metadata System — Instructions for Claude

Denote-convention metadata for design artifacts. This file is
referenced by all flow files. It defines front matter schema, naming
conventions, status transitions, convergence gate, section heading
standards, and the `denote-query` script interface.

**This file has implicit coupling to flow-defined section names**
(`** Goal`, `** Constraints`, etc.). If a flow changes a heading name,
update the heading standards table and `denote-query` extraction logic.

---

## Activation

Denote metadata behavior is **active by default** for all flows.
A project opts out by adding to its CLAUDE.md:

    denote: disabled

When disabled, all denote behavior (front matter, convergence gate,
status transitions, `denote-query` usage) is skipped. Flows run
exactly as before denote was introduced.

---

## Project Notes

The `notes/` directory holds project knowledge — atomic facts,
discoveries, and domain knowledge that don't fit design artifacts.

### Notes activation

- `notes/` exists, CLAUDE.md silent → Claude may proactively record
  facts using the three-gate framework (default)
- `notes/` exists, CLAUDE.md contains `notes: read-only` (as a bare
  line) → Claude reads and queries but does NOT create notes unless
  explicitly asked
- `notes/` does not exist → no fact recording, no knowledge queries

### Fact recording: three-gate framework

Claude applies these gates before recording a fact:

**Gate 1 — Novelty (~80% elimination):** Don't record if already
documented, grep-able from one file, in git history, standard
technology knowledge, or a behavioral directive (→ memory instead).

**Gate 2 — Stability (~10% elimination):** Don't record if it will
change with the current commit, is a temporary workaround, or
describes actively-refactored code. Exception: volatile facts that
are extremely expensive to rediscover get a `:volatile:` tag.

**Gate 3 — Retrieval Cost (positive test):** Record if rediscovery
requires reading 3+ files, tracing cross-module interactions,
running benchmarks, or trial-and-error with external systems. Also
record if counter-intuitive (a competent developer wouldn't guess).

### What NOT to record

- Code mirrors (values readable from a single source file)
- Git-discoverable history
- Documentation echoes (facts already in CLAUDE.md or design docs)
- Debugging ephemera (session-specific observations)
- In-flight changes (code being actively modified)
- Common technology knowledge
- Behavioral directives (those go in Claude Code memory)

### Fact note format

```org
#+title:      <Fact description>
#+date:       [YYYY-MM-DD Day HH:MM]
#+filetags:   :fact:<domain-keywords>:
#+identifier: <YYYYMMDDTHHmmss>
```

Body: 1-3 paragraphs stating the fact, its context, and how it was
discovered. Cross-reference design docs via `[[file:]]` links.
No prescribed structure beyond front matter.

### Memory vs knowledge boundary

Imperatives ("use X", "prefer Y") → Claude Code memory.
Facts about the system ("pool cap is 40") → `notes/`.

### Convergence gate with notes

When `notes/` exists alongside `designs/`, the convergence gate
scans both directories using `denote-query --dirs designs/,notes/`.
This surfaces relevant facts alongside related design artifacts.

### Lint with notes

When `notes/` exists, lint runs across both `designs/` and `notes/`
at flow init. Fact notes (tagged `:fact:`) are exempt from heading
standards (they use free-form content).

---

## Front Matter Schema

Every artifact created by a flow gets denote-standard front matter
inserted after all `#+` configuration lines (`#+STARTUP:`, `#+TOC:`)
and before the first `*` heading:

```org
#+title:      <Human-readable title>
#+date:       [YYYY-MM-DD Day HH:MM]
#+filetags:   :<type>:<status>:<domain-keywords>:
#+identifier: <YYYYMMDDThhmmss>
#+signature:  <issue-ID or empty>
#+deferred:   <org-style tags of deferred item slugs, or empty>
```

The first four fields (`#+title:`, `#+date:`, `#+filetags:`,
`#+identifier:`) follow the standard denote format exactly as
`denote.el` produces them. The last two (`#+signature:`, `#+deferred:`)
are flow-specific extensions.

### Field rules

- `#+title:` is the human-readable title, matching the first heading
  text (e.g., `Operation Pool`)
- `#+date:` is the creation timestamp in org date format
  (e.g., `[2026-07-14 Mon 15:30]`). Set once at creation, never
  updated. Matches denote standard.
- `#+identifier:` is the denote identifier, generated once at creation.
  Format: `YYYYMMDDTHHmmss` (e.g., `20260714T153000`). This is the
  same value encoded in the filename prefix.
- `#+filetags:` contains type, status (if finalized), and domain
  keywords as colon-delimited org tags. Inferred, open tag system.
- `#+signature:` (extension) is inferred from the title's `[ID]`
  prefix if present, or left empty for exploratory work.
- `#+deferred:` (extension) is empty at creation. Populated at finalize
  from Known Deferred Work section. Uses org-style tags
  (`:memory-recycling:pool-shrinking:`). Slugification: lowercase,
  hyphens for spaces, strip parenthetical notes and punctuation.
- `#+deferred:` is a **cache** of Known Deferred Work at finalize time.
  `denote-query deferred-items` reads the section directly and is always
  fresh.

### Type vocabulary

| Type           | Meaning                            |
|----------------|------------------------------------|
| `research`     | Research document                   |
| `design`       | Design document                     |
| `sub-feature`  | Sub-feature design                  |
| `review`       | Review/audit artifact               |
| `supporting`   | Repro cases, test plans             |
| `working`      | Ephemeral file in `-design/` dir    |
| `fact`         | Atomic project knowledge entry      |

### Status vocabulary (terminal states only)

| Status        | Valid for types         |
|---------------|------------------------|
| `researched`  | research               |
| `designed`    | design, sub-feature    |
| `implemented` | design, sub-feature    |
| `postponed`   | any                    |
| `dismissed`   | any                    |
| `superseded`  | any                    |

Active states (researching, designing, implementing) are already
encoded by working directory existence and must NOT be duplicated in
metadata.

### Working directory files

Files inside `-design/` working directories (questions.org,
criticism.org, journal.org, findings.org, options.org, resolved.org,
impl-plan.org, review.org, review-plan.org, review-notes.org, etc.)
also receive denote front matter:

```org
#+title:      <Descriptive title>
#+date:       [YYYY-MM-DD Day HH:MM]
#+filetags:   :working:<role>:
#+identifier: <YYYYMMDDThhmmss>
```

Rules:
- `#+title:` describes the working file's purpose (e.g., "Critic
  Concerns", "External Survey Findings", "Implementation Plan")
- `#+filetags:` always includes `:working:` plus a role tag. Role
  tags: `criticism`, `questions`, `resolved`, `journal`, `findings`,
  `options`, `review`, `review-plan`, `review-notes`, `impl-plan`,
  `boundary`, `critic-context`, `boundary-context`, `answers`
- `#+identifier:` generated at creation, same format as root artifacts
- No `#+signature:` or `#+deferred:` (not relevant for working files)
- No filename suffix (working files keep their plain names; they are
  deleted at finalize)

Working files use free-form `**` headings — they are NOT subject to
the section heading standards (section below). `denote-query lint`
skips files tagged `:working:`.

Working files ARE indexed by `denote-query` (they have `#+identifier:`).
Filter them out with `denote-query tag working` or exclude them with
AWK post-processing when querying only root artifacts.

---

## Naming Convention

Files follow the standard denote naming scheme:

```
IDENTIFIER--TITLE__KEYWORDS.org
```

Where:
- `IDENTIFIER` is `YYYYMMDDTHHmmss` (e.g., `20260714T153000`)
- `--` separates identifier from title
- `TITLE` is kebab-case (e.g., `operation-pool`)
- `__` separates title from keywords
- `KEYWORDS` are underscore-separated tags (e.g., `design_implemented_pool`)

At creation, the file gets a full denote name immediately:

```
20260714T153000--operation-pool__design.org
```

At finalize, the status keyword is added:

```
20260714T153000--operation-pool__design_implemented.org
```

More examples:
- `20260714T150008--denote-flow-linking__research_researched.org`
- `20260714T160000--code-as-documentation__design.org` (unstarted, no status keyword)
- `20260714T153000--ams-image-final-scaling__sub-feature_implemented.org`

This matches the format that Emacs `denote.el` produces. Files created
by denote.el and files created by Claude are interchangeable — denote
commands (rename, link, backlinks) work on both.

**Title slugs must NOT contain double underscores (`__`) or double
hyphens (`--`).** These are denote's component separators.

### Rename + reference update procedure

1. `mv` old name to new name
2. Grep all `.org` files in `designs/` for old filename string
3. Replace all occurrences (all reference formats: `[[file:]]` links,
   `=code=` markup, and plain text paths)
4. Verify: grep for old filename again; if any remain, report them
5. Scope: `designs/` directory only. References in flow files or
   CLAUDE.md are NOT auto-updated

Recovery: `denote-query fix-names` re-derives filename suffixes from
front matter and fixes stale cross-references.

---

## Status Transitions

| Flow boundary      | Action                                               |
|--------------------|------------------------------------------------------|
| `init`             | Create file as `ID--title__type.org` with             |
|                    | `#+filetags: :<type>:` (no status keyword yet)       |
| `finalize` (design)| Working dir deletion first (existing flow behavior), |
|                    | then: rename to add status keyword                   |
|                    | (`__type.org` → `__type_designed.org`),              |
|                    | update `#+filetags:` to include `:designed:`,        |
|                    | populate `#+deferred:` from Known Deferred Work      |
| `finalize` (impl)  | Rename: `_designed` → `_implemented` in keywords,    |
|                    | update `#+filetags:`: `:designed:` → `:implemented:` |
| `full`             | No direct action — triggers init, then design        |
|                    | finalize, then impl finalize                         |
| `loop`             | No metadata change (iterating within a phase)        |
| `resume`           | No metadata change (re-enters existing phase)        |
| `commit`           | No metadata change (status already set at finalize)  |
| Supersession       | Old file: `:superseded:` tag, remove prior status,   |
|                    | append supersession note, rename with suffix,        |
|                    | update all references. New file: add old to References|
| Postpone/dismiss   | User-triggered: update filetags + add reason to doc  |

Research flow uses `:researched:` instead of `:designed:` at finalize.

### Ordering

Working directory deletion (existing flow behavior) precedes all denote
metadata steps (filetag update, deferred population, rename, reference
update).

### Supersession guard

Supersession is only allowed for finalized artifacts (no working
directory). If a working directory exists, Claude warns the user and
requires explicit confirmation. If confirmed, the working directory is
deleted as part of supersession.

---

## Convergence Gate

Runs at every flow's `init` step, after creating the artifact file
and working directory, before the user fills in intake fields. **Runs
only for root-level inits** — sub-feature inits skip the gate.

### Procedure

1. Run `denote-query overlap "<init-argument>" --format tsv`
   using the full init command argument (not just the derived slug).
2. Run `denote-query signature "<issue-id>" --format tsv`
   (if an issue ID is known) for exact feature matches.
3. Run `denote-query deferred --format tsv`
   to surface all postponed/dismissed work.
4. Run `denote-query deferred-items --structured --format tsv`
   to get cross-referenceable deferred item names (for S2).
5. For each candidate (union of steps 1-3, deduplicated by file path):
   run `denote-query goals --files <candidates> --format tsv`
   and assess semantic relevance. Step 4 produces deferred item names
   (not file paths) — cross-referenced separately in the output.
6. Write a `** Convergence` section in the design/research doc:

```org
** Convergence

*** Related artifacts
- [[file:<path>][<title>]] — <one-line relevance assessment>

*** Deferred work overlap
- <item> (from [[file:<path>]]) — <relevance to current feature>

*** Dismissed/postponed
- [[file:<path>][<title>]] — <why it was dismissed, whether
  conditions have changed>
```

If no matches found:

```org
** Convergence
No related artifacts found.
```

7. Report findings to the user as part of init output.
8. Note in the section: "Re-run with `convergence` after populating
   Goal for refined semantic results."

### Section placement

- Design docs: after `** References`, before `** Acceptance Criteria`
- Research docs: after `** Context`, before `** In Scope`
- Reordered research docs (output-first): after `** Known Leads`

The `** Convergence` section is a **denote-layer section**: it appears
only when denote is active. When denote is disabled (opt-out), this
section does not appear.

### S2/S5 extensions

S2 (Deferred Work Synthesis) and S5 (Dismissed Idea Reconsideration)
are **gate-triggered**: separate `denote-query` calls within the gate
procedure (steps 3-4). They can be tested and invoked independently.

---

## Deferred Item Population

At finalize, Claude reads the `** Known Deferred Work` section and
extracts item names. Each item name is slugified and added as an
org-style tag:

```org
** Known Deferred Work

- *Memory recycling* — Track RSS, recycle after N ops.
- *Pool shrinking* — Stop idle processes after inactivity.

→ #+deferred: :memory-recycling:pool-shrinking:
```

---

## Section Heading Standards

All flows MUST use these exact `**` headings in design artifacts:

### Design document headings

| Heading                    | Required? |
|----------------------------|-----------|
| `** Original Prompt`       | Yes       |
| `** Steering`              | Yes       |
| `** Goal`                  | Yes       |
| `** Constraints`           | Yes       |
| `** Preserved Invariants`  | Optional  |
| `** References`            | Yes       |
| `** Convergence`           | Denote    |
| `** Acceptance Criteria`   | Yes       |
| `** Design`                | Yes       |
| `** Sub-features`          | Yes       |
| `** Known Deferred Work`   | Yes       |

### Research document headings

| Heading                    | Required? |
|----------------------------|-----------|
| `** Original Prompt`       | Yes       |
| `** Question`              | Yes       |
| `** Context`               | Yes       |
| `** In Scope`              | Yes       |
| `** Out of Scope`          | Yes       |
| `** Evaluation Criteria`   | Yes       |
| `** Known Leads`           | Yes       |
| `** Convergence`           | Denote    |
| `** Findings`              | Yes       |
| `** Comparison`            | Yes       |
| `** Recommendation`        | Yes       |
| `** Decision`              | Yes       |
| `** Open Questions`        | Yes       |

Note: finalized research docs place intake headings (Question through
Known Leads) under `** Appendix: Intake` as `***` subheadings. The
`denote-query goals` extractor handles both `** Question` and
`*** Question` under the appendix.

The `denote-query` section extraction commands depend on these exact
headings. Variations (`** Goals`, `** Problem`) are NOT supported.
The AWK matcher is case-insensitive for `#+` keywords but
case-sensitive for heading names.

Use `denote-query lint` to check for non-standard headings.

---

## Synthesis Commands

Synthesis goals are standalone commands, NOT part of regular flows.
Each is invoked explicitly and produces file artifacts within a
research or design flow context.

| Command                 | Goal | Behavior                                    |
|-------------------------|------|---------------------------------------------|
| `roadmap`               | S1   | List designed-not-implemented + researched-  |
|                         |      | not-designed. Produces `designs/roadmap.org` |
| `explore <area>`        | S4   | Curated reading list for an area.            |
|                         |      | Produces `designs/<area>-exploration.org`    |
| `stale-constraints`     | S3   | Cross-reference constraints vs recent impls. |
|                         |      | Produces `designs/stale-constraints.org`     |
| `next`                  | S6   | Topological next action.                     |
|                         |      | Produces `designs/next-actions.org`          |

S2 (deferred work synthesis) and S5 (dismissed reconsideration) are
gate-triggered — not standalone commands.

Synthesis outputs are **ephemeral snapshots** that do NOT participate
in the denote system: no front matter, no filename suffixes,
overwritten on subsequent runs, excluded from convergence gate scanning.

---

## The `denote-query` Script

Location: `~/.claude/flows/denote-query` (executable, git-tracked).

### Architecture

A single bash script with an AWK core. Scans `.org` files from line 1
until the first `*` heading to extract front matter. Recursive by
default; use `--root-only` to limit to top-level directory.

### Global options

| Option            | Default    | Purpose                          |
|-------------------|------------|----------------------------------|
| `--dir <path>`    | `designs/` | Root directory to scan           |
| `--dirs <a,b>`    |            | Scan multiple dirs (comma-sep)   |
| `--format <fmt>`  | `tsv`      | Output: tsv, table, paths        |
| `--root-only`     | off        | Disable recursive traversal      |
| `--max-lines <n>` | unlimited  | Truncate section output per file |
| `--files <list>`  | all        | Newline-separated paths (or `-` for stdin) |
| `--no-header`     | off        | Suppress TSV header row          |

`--dir` and `--dirs` are mutually exclusive.

### TSV output format

Header row (always emitted unless `--no-header`):
`FILE\tTITLE\tDATE\tFILETAGS\tIDENTIFIER\tSIGNATURE\tDEFERRED`

### Subcommands

See `denote-query --help` for the full list. Key subcommands:

**Metadata:** `index`, `status`, `signature`, `tag`, `deferred`,
`overlap`, `backlinks`, `tags`, `statuses`, `signatures`

**Domain queries:** `field` (arbitrary `#+key:` field search)

**Validation:** `validate`, `lint`, `fix-names`

**Section extraction:** `goals`, `constraints`, `deferred-items`, `summary`
