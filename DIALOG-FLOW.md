# Dialog Flow — Instructions for Claude

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

A synchronous conversation mode for architectural decisions and design clarification. The human is the primary participant — Claude steers, probes, and captures decisions. Use when the human has domain context that subagents cannot replicate from code alone.

---

## When to Use

- **Architectural decisions** where trade-offs depend on project philosophy, past experience, or intuition that isn't in the codebase
- **Design clarification** when a Full Flow design phase is stuck in a local maximum (alternatives debated at the wrong level of abstraction)
- **Concern exploration** when a specific pain point needs diagnosis before committing to a solution

### When NOT to Use

- Exhaustive code verification (use the Full Flow's reviewer subagents)
- Implementation planning (use the Full Flow's implementation phase)
- When the topic is vague and has no concrete decision target (push back — see Input Sufficiency below)

---

## Input Sufficiency Gate

A dialog requires two things from the user:

1. **A concrete decision or concern** — phrased as a question with a finite set of possible answers. Not "let's discuss state management" but "should UI state be centralized, and if so, how?"

2. **Grounding context** — at least one of:
   - A specific pain point experienced
   - A specific upcoming need
   - A reference to existing behavior or external experience

### When to Push Back

If the input is only a topic area (no decision target):

> "What specific decision are you trying to make, or what specific problem are you trying to solve? A dialog works best when we have a concrete question to answer."

If the input has a topic but no grounding (no pain point, need, or reference):

> "What's driving this question? Is there a specific pain point you're hitting, an upcoming feature that needs it, or a pattern from another project that seems relevant?"

The bar is deliberately low — one concrete question + one grounding anchor. The dialog itself discovers the rest.

---

## The Artifact

One file per dialog: **`<topic>-dialog.org`**

Location:
- If a Design Process is in progress: in the feature's design directory
- If standalone: in `design/` at the repository root

Structure:

```org
* Dialog: <topic>

** Original Prompt
(Verbatim copy of the user's chat message that triggered this dialog.
Preserves raw intent for drift detection. Set once, never modified.)

** Goal
<The concrete decision/concern>

** Grounding
<The pain point, upcoming need, or reference that anchors it>

** Decisions
(Appended as each decision crystallizes during the conversation.
Each decision is checked against Goal, Grounding, and all previous
Decisions before confirmation.)

- <decision>: <rationale, one line>

** Open
(Anything unresolved when the dialog paused/ended)

- <open item>
```

**Create the file at the start of the dialog** with Goal and Grounding filled in. Append to Decisions as checkpoints are reached. Update Open when pausing or ending.

No journal, no criticism file, no answers file. The conversation itself is the journal. The Decisions section is the only durable artifact.

---

## Conversation Style

### Start from the root

Begin at the root of the architectural decision tree, not at a specific alternative. The first question should probe **what problem we're actually solving**, not which solution to pick.

### Flow downward

Move from general to specific. Don't jump between unrelated topics. Each exchange should narrow the decision space. If the conversation needs to shift direction, acknowledge the shift explicitly.

### Ground in code, not in conversation

When the user makes a technical claim about the codebase during the
dialog ("module X works like Y", "we always use pattern Z here"),
**verify it against actual code before building a decision on it.**
Read the relevant file. If the code shows something different, state
what the code actually shows before continuing.

This is the dialog's highest sycophancy risk: the user speaks with
domain authority, Claude defers, and a wrong claim becomes the
foundation for subsequent decisions. The compounding starts here.

### Steer toward decisions

Claude's role is to:
- **Probe** assumptions and framings ("is this actually a problem, or just aesthetically unsatisfying?")
- **Surface tensions** between stated goals and proposed approaches
- **Challenge claims that contradict code reality** — don't accept
  technical assertions without checking. The user's domain knowledge
  is valuable but may be stale or based on a mental model that has
  drifted from the current codebase.
- **Offer concrete alternatives** when the conversation stalls
- **Deduce answers** from the human's responses — AND challenge
  deductions that contradict the stated Goal, Grounding, or actual
  code. Don't wait for explicit declarations, but don't build on
  unchecked premises either.

### Checkpoint decisions

Whenever a key decision can be firmly deduced — either a shift in
approach, an answer to a question, or a resolution of a concern —
Claude checks BEFORE confirming:

1. Does this decision contradict the Goal or Grounding?
2. Does it contradict any previously confirmed Decision?
3. If the decision rests on a technical claim, has that claim been
   verified against code?

If all checks pass:

> "I think we've arrived at this: **[decision statement]**. Is that correct?"

If a contradiction is found:

> "This would contradict [Goal/previous Decision/code reality]:
> [specific contradiction]. Do you want to proceed anyway, or
> should we revisit?"

Append confirmed decisions to the artifact's Decisions section.

### Handle breaks

If a decision is challenged or contradicted by a later finding, acknowledge the shift:

> "This changes our earlier decision about X. The new direction is Y because Z."

Update the Decisions section to reflect the current state, not the history.

### Keep turns concise

The dialog's value is in rapid exchange. Claude should not write essays per turn. Lead with the key point, ask one focused question. Save thorough analysis for the Full Flow.

---

## Ending Conditions

A dialog is "done" when:

1. The Goal question has a corresponding decision in the Decisions section, OR
2. The user explicitly redirects to the Full Flow (`design` or `full`), OR
3. The user pauses the dialog (fill in Open section with unresolved items)

**End-of-dialog aggregate check.** Before declaring the dialog done,
re-read all Decisions and check them as a set against the original
Goal and Grounding. Individually reasonable decisions can drift in
aggregate — the whole set should still address the original concern.
If aggregate drift is detected, flag it:

> "Looking at our decisions together, we've drifted from the original
> goal [Goal]. Specifically: [what shifted]. Is this the direction you
> want, or should we revisit?"

---

## Integration with the Full Flow

### Dialog during a Full Flow

When a dialog happens mid-flow (e.g., during `loop` or before `finalize`), the decisions feed **directly into the design document** — not into the answers file. The dialog replaces a loop iteration; it doesn't produce input for one.

After the dialog ends, resume the Full Flow from wherever it was (typically `loop` to process any remaining open items, or `finalize` if the dialog resolved everything).

### Standalone dialog

The dialog file stands alone in `designs/`. It can later:
- Seed a new Full Flow (`init <feature>` → `full`, using the decisions as input)
- Be referenced by a subsequent `implement` phase
- Be discarded if the decision is superseded

---

## Trigger Phrases

Explicit triggers:
- "Let's discuss...", "I want to talk through..."
- "Can we have a dialog about...", "Let's use the dialog flow for..."
- "dialog <topic>"

Implicit triggers (suggest the dialog flow proactively):
- "Here's my problem...", "I'm not sure how to approach..."
- "What do you think about...", "Can we figure out..."
- "I have a question about the architecture/design of..."
- Any back-and-forth that has gone 3+ exchanges on the same topic
  with decisions being made implicitly in chat

When an implicit trigger is detected, suggest:

> "This looks like it could benefit from a dialog flow — we'd get
> tracked decisions and contradiction-checking. Want me to start one?"

When the user invokes a dialog (explicitly or after accepting a
suggestion), apply the Input Sufficiency Gate before proceeding.
If sufficient, create the artifact file and begin.
