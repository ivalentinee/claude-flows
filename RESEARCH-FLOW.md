# Research Flow — Instructions for Claude

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

A lightweight, autonomous flow for researching questions before
committing to a design. Produces a structured research document that
feeds into the Design or Full flow as a Reference.

---

## Principles

1. **Autonomous by default.** Claude runs the full flow without
   pausing unless it hits a Critical item it genuinely cannot
   resolve. When the user starts the flow with conversational
   context suggesting interactivity (e.g. "let's explore together",
   "walk me through options"), switch to interactive mode: present
   findings incrementally and ask for direction at each phase
   boundary instead of proceeding automatically.

2. **Preserve alternatives.** Unlike the design flow, research does
   NOT converge to a single answer prematurely. The output presents
   multiple options with trade-offs. The recommendation is Claude's
   informed opinion, not a binding decision.

3. **External awareness.** "How do others solve this?" is a
   first-class step. Search for existing solutions, libraries,
   patterns, and prior art before inventing from scratch.

4. **Feed-forward.** The research doc is designed to slot directly
   into a design flow's References field. It is not a design — it
   is input to one.

5. **Convention over configuration.** Sensible defaults for
   everything. The user provides a question; Claude infers the rest.
   Optional fields are discovered via in-chat dialog when missing,
   not demanded upfront. File names, directory structure, and
   output format follow conventions the user never configures.

6. **Communicate effort.** Before entering an autonomous phase,
   emit an effort estimate so the user knows whether to stay or
   multitask. After a moderate or heavy phase, re-establish context
   — assume the user was away.

---

## Directory Structure

```
designs/
  <topic>.research.org              # Research doc (intake + output)
  <topic>-research/                 # Working files
    options.org                     # Options registry — all discovered
                                    #   approaches, including discarded
    findings.org                    # Raw findings from gather phase
    criticism.org                   # Critic's concerns
    resolved.org                    # Resolved concerns with rationale
    journal.org                     # Subagent journal
```

The `.research.org` suffix distinguishes research docs from design
docs in the `designs/` directory. Working files are implementation
details — the user's interaction surface is the `.research.org` file
and chat. The user *can* read working files but is not expected to.

---

## Intake Template

The `init-research` command creates `designs/<topic>.research.org`.
The `research <question>` command creates it automatically with
Question and Context inferred from the prompt.

All artifacts use `#+STARTUP: overview` so only top-level headings
are visible by default in org-mode.

```org
#+STARTUP: overview
* Research: <Topic>

** Original Prompt
(Verbatim copy of the user's chat message that triggered this
research. Preserves raw intent for Steward drift detection.
Set once at init, never modified.)

** Steering
(Append-only log of user messages during the flow that change
direction, add scope, constrain approach, or redirect focus.
Timestamped entries, added as they happen. Only directional
messages — not "continue" or "finalize".)

** Question
(The specific question or problem being researched. Frame as a
question, not a statement — "How should we handle real-time sync
between GM and player views?" not "Real-time sync".)

** Context
(Why this research is needed. What decision does it feed into?
What prompted the investigation? This helps Claude judge which
findings are relevant vs. merely interesting.)

# The following sections are optional. If missing when the flow
# starts, Claude discovers them via in-chat dialog during Frame.

** In Scope
(What to include in the research.)

** Out of Scope
(What to explicitly exclude. Without boundaries, research expands
indefinitely.)

** Evaluation Criteria
(What properties matter when comparing approaches, in priority
order. E.g. simplicity > performance > flexibility. These become
the columns in the comparison table.)

** Known Leads
(Starting points the user already has: files in the codebase,
URLs, library names, patterns, prior conversations. Claude reads
these first before exploring further.)

# Output Expectation is derived by Claude from the question.
# Confirmed in chat only if Claude is unsure.
# Options: comparison, recommendation (default), survey,
# proof-of-concept, assessment.

** Findings
(Empty at init. Populated during the research flow.)

** Comparison
(Empty at init. Populated during synthesis.)

** Recommendation
(Empty at init. Populated during synthesis.)

** Decision
(Empty at init. Populated during convergence — if the user
chooses to converge. Records the final choice, the rationale,
and what was traded away. Absent if the research ends at
comparison/survey without a decision.)

** Open Questions
(Empty at init. Unresolved items that should carry forward
into the design phase.)
```

After synthesis, the document is **reordered** so output comes first:
Recommendation → Decision → Comparison → Findings → then the intake
sections (Question, Context, etc.) as an appendix. During init the
intake sections are at the top since they are the only content.

---

## Options Registry — `options.org`

Every approach, solution, or pattern discovered during research gets
an entry in `options.org`, regardless of whether it is ultimately
recommended, viable, or discarded. This gives the user the full
picture — including why something was ruled out.

All working files use `#+STARTUP: overview` so option headings
(with status and one-line verdict) are visible and details
(Approach/Fit/Gaps) are folded away by default.

**Entry format** — use the structured format for design/architecture
research. For pure code research (finding a function, tracing a
call path, locating a config value), use the lightweight format
instead.

### Structured format (design/architecture research)

```org
#+STARTUP: overview
* Options Registry

** <Option Name>                                       :active:

*Status:* active | discarded | conditional
*Source:* (where this was found — codebase path, URL, library name)

*** Approach
(What this option does and how it works. Free-form, as detailed
as needed.)

*** Fit
(What makes this option good for the question at hand. Tied to
the Evaluation Criteria from the intake — reference specific
criteria by name.)

*** Gaps
(What makes this option bad, risky, or insufficient. Be specific:
"adds ~200ms latency per request" not "might be slow".)

*** Verdict
(Why this option was kept or discarded. One or two sentences.
For =discarded= options, this is the key value — it explains
what was considered and why it was ruled out, so the user or a
future design flow does not re-discover and re-evaluate it.)
```

Use org tags on headings (`:active:`, `:discarded:`, `:conditional:`)
for filtering and at-a-glance status when folded.

`Status` values:
- `active` — viable option, included in the comparison
- `discarded` — evaluated and ruled out (Verdict explains why)
- `conditional` — viable only under specific circumstances
  (Verdict explains which)

### Lightweight format (code research)

When the research is about finding code, tracing behavior, or
locating existing implementations — where Fit/Gaps analysis would
be overhead — use a shorter format:

```org
** <Finding>

*Source:* (file path, URL, or library)
*Relevance:* (one line — why this matters to the question)
*Notes:* (free-form observations, caveats, or pointers)
```

Claude picks the format based on the research type. If the
Question is about code behavior or the research is an assessment
of existing code, default to lightweight. Otherwise, default to
structured. Mixed usage is fine within one registry — some options
may warrant full analysis while others are simple pointers.

### Registry lifecycle

- **Gather phase (Step 2):** subagents add entries as they discover
  options. Initial status is `active` — nothing is discarded yet.
- **Critic phase (Step 3):** the Critic may flag options for
  reconsideration but does not discard them directly.
- **Auto-resolve (Step 4):** Claude may change status to `discarded`
  with a Verdict when an option is clearly unviable. Discarded
  entries are never deleted — only their status changes.
- **Synthesize (Step 5):** the Comparison section in the research
  doc draws from `active` and `conditional` entries. The full
  registry (including discarded) remains in the working directory
  until finalize.
- **Finalize (Step 6):** the Options Registry is summarized into
  the research doc's Findings section before the working directory
  is deleted. Discarded options get a collapsed summary (name +
  one-line verdict) so the full picture survives cleanup.

---

## The Flow

### Step 1 — Frame

If started via `research <question>` (one-command start):
1. Derive a short topic name from the question (strip question
   words, extract noun phrases, max 3-4 words, kebab-case)
2. Create `designs/<topic>.research.org` with Question and Context
   filled from the user's prompt
3. Create `designs/<topic>-research/` working directory
4. Derive Output Expectation from the question (default:
   `recommendation`). If unsure, confirm in chat.

If started via `init-research` then `research`:
1. Read the existing intake doc

**Discover missing optional fields.** For each missing optional
field (In Scope, Out of Scope, Evaluation Criteria, Known Leads):
- In **interactive mode**: use in-chat dialog to elicit them.
  Ask focused questions ("What should I exclude?" / "What
  properties matter most when comparing approaches?"), not
  "please fill in the template."
- In **autonomous mode**: infer from the question and codebase
  context. Emit inferred values as a progress breadcrumb. The
  user can steer via chat if the inference is wrong.

Verify the Question is specific enough to research. If too vague,
ask the user to clarify.

### Step 2 — Gather (3 parallel subagents)

**Effort forecast:** Emit before launching: "Proceeding — *heavy*
pass (3 parallel gather agents + critic + synthesis). Good time to
multitask."

Launch three subagents in parallel. Each writes raw notes into
`findings.org` AND adds structured entries to `options.org` for
each distinct approach or solution discovered. Use the structured
or lightweight option format based on research type (see Options
Registry section above).

**Progress breadcrumbs:** Emit a brief status as each subagent
completes (e.g. "Codebase auditor: found 3 relevant patterns.
External surveyor still running...").

**Codebase Auditor:**
- Search the existing codebase for anything relevant to the
  question: existing implementations, partial solutions, related
  patterns, utility functions, data structures, test fixtures
- Read files listed in Known Leads
- Report what exists, how it works, and how well it addresses
  the question
- Note reuse opportunities ("this existing module already does
  80% of what's needed")
- Add each distinct approach/pattern found as an entry in
  `options.org` with status `active`
- Append journal entry

**External Surveyor:**
- Search for how others solve the same or similar problems:
  common patterns, libraries, framework features, blog posts,
  documentation, reference implementations
- Focus on solutions in the same ecosystem first (Elixir/OTP,
  Phoenix LiveView), then broaden to general approaches
- For each finding: what it is, how it works, where to find it,
  and what trade-offs it makes
- Add each distinct approach as an entry in `options.org` with
  status `active` — fill in Fit/Gaps relative to the Evaluation
  Criteria
- Prioritize battle-tested approaches over novel ones
- Append journal entry

**Constraint Analyst:**
- Read the existing architecture (supervision trees, module
  boundaries, data flow, deployment setup) to identify what
  constrains the solution space
- Check for: performance requirements, compatibility needs,
  existing patterns that new code should follow, deployment
  constraints, dependencies that are already in the project
- Report hard constraints (must satisfy) vs. soft constraints
  (should satisfy, but could be relaxed)
- Does NOT add options, but may annotate existing entries in
  `options.org` with constraint-related Gaps
- Append journal entry

**Step-boundary steering:** After all gather subagents complete,
check if the user typed anything in chat during execution. If so,
treat the message as an inline amendment:
- Additional Known Leads → note in journal, feed into critic
- Scope adjustments → update research doc's In Scope / Out of
  Scope sections, note in journal
- "Also consider X" → run a targeted follow-up search
- "Drop option Y" → mark as discarded in options registry

Acknowledge briefly ("Noted — adding geometry module to search
scope") and incorporate into the next step.

### Step 3 — Critic

**Progress breadcrumb:** "Critic: reviewing N options for bias and
gaps..."

Launch a Critic subagent that reads `options.org`, `findings.org`,
and the intake doc. The Critic:

- Checks for **confirmation bias** — did the gather phase only find
  evidence supporting an obvious answer while ignoring alternatives?
- Checks for **missing options** — are there approaches the
  surveyors didn't consider? Common solutions they overlooked?
- Checks for **stale information** — are any external references
  outdated or deprecated?
- Checks for **false constraints** — did the Constraint Analyst
  flag something as a hard constraint that is actually changeable?
- Checks for **scope drift** — did any subagent research beyond
  the stated scope?
- Writes concerns to `criticism.org`, tagged `[Critical]` or
  `[Auto]` per the same classification as the Full Flow
- Appends journal entry

**Step-boundary steering:** Check for user chat messages after
critic completes. Apply amendments as above.

### Step 4 — Auto-resolve

**Progress breadcrumb:** "Auto-resolving N concerns..."

Same pattern as the Full Flow:

1. Auto-resolve `[Auto]` items from `criticism.org`
2. If the Critic flagged missing options or confirmation bias,
   Claude runs a **targeted follow-up search** for the specific
   gaps identified (not a full re-gather)
3. Move resolved items to `resolved.org` with rationale
4. If `[Critical]` items remain, escalate to user

Circuit breaker: max 2 auto-resolve iterations for research
(lighter than the design flow's 3 — research should not loop
extensively).

**Progress breadcrumb:** "Auto-resolved N concerns. M Critical
items remain." (or "No critical items — proceeding to synthesis.")

### Step 5 — Synthesize

**Progress breadcrumb:** "Synthesizing comparison of N active
options..."

Read `options.org`, `findings.org`, and resolved criticisms. Derive
the Output Expectation if not already set. Write into the research
doc:

**Findings section:** Consolidated narrative of what was discovered,
organized by theme (not by which subagent found it). Include:
- What exists in the codebase
- What external solutions are available
- What constraints apply
- **Discarded options summary** — a collapsed list of discarded
  options (name + one-line Verdict each), so the full picture
  survives even after the working directory is deleted

**Comparison section:** Structured comparison of `active` and
`conditional` options from the registry. Format depends on Output
Expectation:

- `comparison` / `recommendation`: A table or structured list with
  one row per option, one column per Evaluation Criterion, plus
  a "Trade-offs" column. Each cell draws from the option's
  Fit/Gaps fields — a brief assessment, not just a checkmark.
- `survey`: Grouped list of approaches by category, with brief
  descriptions and pointers.
- `assessment`: Strengths/weaknesses analysis of the specific
  approach being evaluated.
- `proof-of-concept`: Comparison table plus a "Feasibility" column
  noting what a PoC would need to demonstrate.

**Recommendation section** (if Output Expectation includes it):
- Claude's pick with explicit rationale tied to the Evaluation
  Criteria
- What the recommendation does NOT address (known gaps)
- Confidence level: high / medium / low — and what would change
  it

**Open Questions section:** Items that surfaced during research but
couldn't be resolved — these carry forward into the design phase.

**Reorder the document:** Move output sections (Recommendation,
Decision, Comparison, Findings) above the intake sections (Question,
Context, In Scope, etc.). The intake sections become an appendix.
Add `#+TOC: headlines 2` at the top and internal navigation links
(`[[*Recommendation]]`, `[[*Comparison]]`).

**Inline convergence:** If the Recommendation's confidence is low
or 2+ options score similarly, present the convergence decision
frame directly in the synthesis output — do not require a separate
`converge` command. Show the active options, trade-offs, and ask
the user to choose. If confidence is high and one option clearly
dominates, proceed to finalize.

**Context re-establishment:** Since this was a heavy phase, open
the synthesis output with a 2-3 line recap: what was researched,
how many options were found, what the recommendation is. Assume
the user was away.

### Step 5a — Converge (optional, interactive) — `converge`

An interactive process for choosing between options when the
research surfaces multiple viable approaches. Triggered inline
after synthesis (when options are close) or by the user running
`converge` to re-enter the decision process later.

**When to suggest convergence:**
- 2+ options are close in the comparison (no clear winner)
- The choice depends on priorities only the user can rank
- The research was about balancing competing concerns (performance
  vs. simplicity, flexibility vs. safety, etc.)
- The Output Expectation is `recommendation` but Claude's
  confidence is medium or low

**When to skip:**
- One option clearly dominates across all criteria
- Output Expectation is `survey` or `comparison` (the user
  explicitly didn't ask for a decision)
- Only one `active` option remains after the critic phase

**The convergence process:**

1. **Present the decision frame.** Show the user:
   - The `active` / `conditional` options with a brief summary
     of each (drawn from the Comparison section)
   - The Evaluation Criteria in their current priority order
   - Claude's recommendation and its rationale (if one exists)
   - Specific trade-offs the user needs to weigh — framed as
     choices, not questions: "Option A optimizes for X at the
     cost of Y. Option B does the reverse."

2. **User input.** The user may:
   - Pick an option directly ("go with A")
   - Re-rank the Evaluation Criteria ("actually, simplicity
     matters more than performance here")
   - Combine options ("use A's approach for X but B's approach
     for Y")
   - Rank all options ("A > C > B for our case")
   - Request more detail on a specific option before deciding
   - Reject all options and redirect the research

3. **Record the decision.** Write into the research doc's
   **Decision** section:

   ```org
   ** Decision

   *** Chosen
   (The selected option or combined approach. If ranked: the
   full ranking with rationale for the ordering.)

   *** Rationale
   (Why this was chosen — which criteria dominated, what the
   user's reasoning was, how trade-offs were resolved. Be
   specific: "Chose A because the team prioritizes deployment
   simplicity over raw throughput, and the 15% performance gap
   is acceptable given current load.")

   *** Traded Away
   (What the chosen option sacrifices compared to alternatives.
   This is the most valuable part — it documents what was
   knowingly given up, so future revisits don't re-litigate
   without new information.)

   *** Decided By
   (Who made the call: "user", "Claude (autonomous)", or
   "user + Claude (convergence)". Timestamped.)
   ```

4. **Update options.** In `options.org`, update the chosen
   option's status to `active — chosen` and non-chosen options
   to `discarded` (with Verdict: "Not chosen during convergence
   — see Decision section for rationale"). If the decision was
   a ranking rather than a single pick, annotate each option
   with its rank.

If the user changes the Evaluation Criteria priority during
convergence, update them in the intake section of the research
doc and note the change in the journal.

Convergence may take multiple rounds if the user needs more
information or wants to explore a hybrid approach. Each round
is a `converge` command.

### Step 6 — Finalize

1. Verify the research doc is complete and self-consistent
2. Verify the Findings section includes a discarded options summary
   (so the full picture is preserved after cleanup)
3. If convergence ran, verify the Decision section captures the
   rationale and traded-away items
4. Verify the document has been reordered (output first, intake
   as appendix)
5. Delete the working directory (`designs/<topic>-research/`)
6. The research doc `designs/<topic>.research.org` remains as a
   permanent artifact

Report completion to the user. If the research feeds into a known
feature, suggest: "To start the design, run `init <feature-name>`
and add this research doc to the References field."

---

## Commands

| Command | Action |
|---------|--------|
| `research <question>` | **One-command start.** Derive topic name, create research doc with Question/Context inferred from the prompt, discover missing fields, and run the full flow (Steps 1–6). Pauses only for Critical escalations, inline convergence (when options are close), and user review. If targeting a prompt file (`:prompt:` tag in `designs/`), hydrate it into a research doc first (see FULL-FLOW.md Prompt Files). |
| `init-research <topic>` | **Escape hatch.** Create `designs/<topic>.research.org` with the intake template and working directory. The user fills in fields manually, then runs `research` to start. Derives kebab-case filename (strip question words, noun phrases, max 3-4 words). |
| `research` | **(no argument)** Resume or start the most recently initialized research. Infer topic from the most recent `.research.org` by modification time. Also checks for `:prompt:` files. If ambiguous, ask. |
| `converge` | Interactive convergence process (Step 5a). Choose between options, rank results, or resolve trade-offs. Can be run multiple rounds. |
| `finalize` | Run Step 6 — verify completeness, clean up working directory. |
| `resume` | Detect current state from existing files, report in human-readable terms (see Session Resumption), and continue. |

---

## Mid-Flow Communication

These conventions apply throughout the flow and in all modes
(autonomous and interactive).

### Effort Forecast

Before entering an autonomous phase, emit a one-line effort
estimate:

- **Light** — a single step, one or two subagents. "Stay here."
- **Moderate** — multiple steps or parallel subagents, a few
  minutes. "Check back shortly."
- **Heavy** — full multi-phase autonomous run (gather + critic +
  resolve + synthesize). "Good time to multitask."

Example: "Proceeding — *heavy* pass (gather + critic + synthesis).
Good time to multitask."

### Progress Breadcrumbs

Between subagent launches and completions, emit brief status lines.
Not pauses, not questions — just status the user sees scroll by:

- "Gathering: searching codebase for relevant patterns..."
- "Codebase auditor: found 3 relevant modules. External surveyor
  still running..."
- "Critic: reviewing 5 options for bias and gaps..."
- "Auto-resolved 2 concerns. No critical items."

### Context Re-establishment

After a **moderate** or **heavy** autonomous phase, Claude's next
user-facing message opens with a 2-3 line context recap:

- What was being done
- What happened (key numbers: options found, concerns resolved)
- What comes next

Example: "Ran gather + critic + synthesis on your research about
real-time sync. Found 5 approaches, critic flagged 1 concern
(auto-resolved), 3 options are viable. Here's the comparison:"

After a **light** pass, skip the recap — the user is still
watching.

### Step-Boundary Steering

Between flow phases, check if the user typed anything in chat
during execution. If so, treat the message as an inline amendment:
- Additional Known Leads → feed into subsequent steps
- Scope adjustments → update the research doc, note in journal
- "Also consider X" → targeted follow-up search
- "Drop option Y" → mark as discarded in options registry

Acknowledge briefly and incorporate into the next step.

If both a chat message and a `steering.org` file exist in the
working directory, chat takes precedence. Process both; log
conflicts in the journal.

---

## Filesystem Steering (alternative to chat)

The user can create a `steering.org` file in the working directory
at any point. At each step boundary, Claude checks for this file
and incorporates its contents as amendments (same effect as chat
steering). Format:

```org
* Steering
- Also consider: Phoenix PubSub patterns
- Narrow scope: exclude deployment concerns
- Drop option: manual polling
```

After processing, append entries to the journal and delete
`steering.org`. This is an alternative to chat steering for users
who prefer file editing, not the primary mechanism.

---

## Interactive Mode

When the user's prompt suggests interactivity, the flow pauses at
phase boundaries instead of running straight through:

- After **Frame** (Step 1): "Here's how I'm interpreting the
  question and scope. Adjust before I search?"
- After **Gather** (Step 2): "Here's what I found. Any leads I
  missed before I run the critic?"
- After **Critic** (Step 4): "The critic raised these points.
  Want to weigh in before I synthesize?"
- After **Synthesize** (Step 5): "Here's the comparison and my
  recommendation. Want to `converge` on a choice, or `finalize`?"

In interactive mode, convergence is the natural next step after
synthesis — the user has seen the options and can decide. In
autonomous mode, convergence is presented inline only when options
are close; otherwise Claude proceeds to finalize.

In autonomous mode, none of these pauses happen — Claude runs
Steps 1–6 continuously, emitting effort forecasts, breadcrumbs,
and context recaps as it goes.

---

## Proof-of-Concept Output

When Output Expectation is `proof-of-concept`, Step 5 adds an
extra sub-step:

**Step 5b — Minimal PoC.** After synthesis, implement the smallest
possible working example that demonstrates the recommended approach.
This is:
- A single file or module, not a full feature
- Focused on proving feasibility, not production quality
- Accompanied by a note on what the PoC does and does NOT prove

The PoC code lives in the codebase (not in the research doc). The
research doc references it with a file link and a summary of what
it demonstrated.

---

## When NOT to Use This Flow

Skip the research flow when:
- The answer is already known and just needs documenting
- A quick web search or codebase grep suffices (just do it inline)
- The question is purely about preference, not trade-offs

Use it when the question has multiple viable answers, when
external prior art matters, or when you need to audit existing
code before deciding on an approach.

---

## Session Resumption

When resuming, report state in **human-readable terms** — what was
found and what happens next, never step numbers. The user should
not need to know the flow's internal structure.

Example: "Your research on real-time sync found 5 approaches. The
critic flagged 2 concerns (both auto-resolved). 3 viable options
remain. Next: I'll synthesize a comparison. Say `research` to
continue."

Detection logic:
1. Scan `designs/` for `.research.org` files, `-research/`
   working directories, and `:prompt:` files
2. Read all existing files
3. Determine phase:
   - Research doc exists, no working directory → just initialized
   - `findings.org` is empty → gather phase
   - `findings.org` has content, no `criticism.org` → critic phase
   - `criticism.org` has content → auto-resolve phase
   - Comparison section populated, no Decision section → suggest
     `converge` or `finalize`
   - Decision section populated → finalize phase
   - Output sections at top of research doc, working directory
     gone → already finalized
4. Report current state in human-readable terms and continue
