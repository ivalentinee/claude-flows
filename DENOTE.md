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

## Front Matter Schema

Every artifact created by a flow gets this front matter block inserted
after all `#+` configuration lines (`#+STARTUP:`, `#+TOC:`) and before
the first `*` heading:

```org
#+title:      <Human-readable title>
#+filetags:   :<type>:<status>:<domain-keywords>:
#+identifier: <YYYYMMDDThhmmss from current time>
#+signature:  <issue-ID or empty>
#+deferred:   <org-style tags of deferred item slugs, or empty>
```

### Field rules

- `#+title:` reproduces the first heading text (e.g., `[15987] Operation Pool`)
- `#+identifier:` is generated once at creation and never changes.
  Format: `YYYYMMDDTHHmmss` (e.g., `20240713T113400`)
- `#+signature:` is inferred from the title's `[ID]` prefix if present,
  or left empty for exploratory work
- `#+filetags:` always contains exactly one type tag and one status tag
  (or no status tag during active work). Domain keywords are added by
  Claude based on Goal section content. Inferred, open tag system — no
  predefined vocabulary for domain keywords.
- `#+deferred:` is an empty string at creation. Populated at finalize
  from Known Deferred Work section. Uses org-style tags
  (`:memory-recycling:pool-shrinking:`). Slugification: lowercase,
  hyphens for spaces, strip parenthetical notes and punctuation. Only
  the primary item name is slugified.
- `#+deferred:` is a **cache** of Known Deferred Work at finalize time.
  `denote-query deferred-items` reads the section directly and is always
  fresh.

### Type vocabulary

| Type           | Meaning                |
|----------------|------------------------|
| `research`     | Research document       |
| `design`       | Design document         |
| `sub-feature`  | Sub-feature design      |
| `review`       | Review/audit artifact   |
| `supporting`   | Repro cases, test plans |

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

---

## Naming Convention

At creation, artifacts are named: `<slug>.org` (no suffix metadata).

At finalize, artifacts are renamed to: `<slug>__<status>_<type>.org`

Examples:
- `operation-pool__implemented_design.org`
- `operation-pool-reuse__researched_research.org`
- `resource-scaling/ams-image-final-scaling__implemented_sub-feature.org`

**Slugs must NOT contain double underscores (`__`).** Use a single
hyphen instead.

During active work (working directory exists), the file keeps its
plain `<slug>.org` name. The suffix is added only at finalize.

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
| `init`             | Create file with `#+filetags: :<type>::`             |
|                    | (no status tag — working dir signals active)         |
| `finalize` (design)| Working dir deletion first (existing flow behavior), |
|                    | then: rename file with `__designed_<type>` suffix,   |
|                    | update `#+filetags:` to include `:designed:`,        |
|                    | populate `#+deferred:` from Known Deferred Work      |
| `finalize` (impl)  | Change `__designed_` to `__implemented_`,            |
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
| `--format <fmt>`  | `tsv`      | Output: tsv, table, paths        |
| `--root-only`     | off        | Disable recursive traversal      |
| `--max-lines <n>` | unlimited  | Truncate section output per file |
| `--files <list>`  | all        | Newline-separated paths (or `-` for stdin) |
| `--no-header`     | off        | Suppress TSV header row          |

### TSV output format

Header row (always emitted unless `--no-header`):
`FILE\tTITLE\tFILETAGS\tIDENTIFIER\tSIGNATURE\tDEFERRED`

### Subcommands

See `denote-query --help` for the full list. Key subcommands:

**Metadata:** `index`, `status`, `signature`, `tag`, `deferred`,
`overlap`, `backlinks`, `tags`, `statuses`, `signatures`

**Validation:** `validate`, `lint`, `fix-names`

**Section extraction:** `goals`, `constraints`, `deferred-items`, `summary`
