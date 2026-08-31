# Flow Guide — Instructions for Claude

Formalized flows produce better results than free-form chat —
they keep effort focused, ensure quality gates run, and create
traceable artifacts. When a task fits a flow, steer the user
toward it rather than handling the task ad-hoc in chat.

**When to consult this file:**
- The user asks which flow to use or says "which flow?"
- The user describes a non-trivial task without specifying a flow
- The user starts working on something in free-form chat that
  would benefit from a structured flow — suggest one proactively
  ("This looks like it would benefit from `research <question>`
  — want me to start that?")

---

## Available Flows

| Flow | Command | When to use |
|------|---------|-------------|
| **Research** | `research <question>` | Explore approaches, find examples, compare options before deciding |
| **Dialog** | `dialog <topic>` | Synchronous discussion to reach a specific decision |
| **Design only** | `init <feature>` → `design` | Design phase only (Steps 1–8 of Full Flow) |
| **Full** | `init <feature>` → `full` | Design + implement end-to-end, Claude-autonomous |
| **Implement only** | `implement <feature>` | Implementation phase only (Steps 9–16, design exists) |
| **Prompt** | `prompt <idea>` | Capture an idea for later — not a flow, just a file |

`design` and `implement` are entry points into the Full Flow's
phases — same steps, same quality gates, same Steward checks.

## Quick Decision

Ask yourself one question: **do I know what to build?**

- **No, I need to explore options** → `research`
- **I have a specific question to answer first** → `dialog`
- **Roughly, but need to flesh out the design** → `design`
- **Yes, and I want Claude to handle design + code** → `full`
- **Yes, and the design doc already exists** → `implement`
- **I just want to capture an idea for later** → `prompt`

## Guided Selection

When the user describes a task without choosing a flow, match
against these patterns:

**→ Dialog** when the user says:
- "let's discuss...", "I want to talk through...", "here's my
  problem...", "I'm not sure how to approach..."
- "what do you think about...", "can we figure out..."
- The user is working toward a *specific decision* (not open-ended
  exploration) and has context that Claude can't get from code alone
- The user starts a back-and-forth that would benefit from tracked
  decisions and a convergence artifact

**→ Research** when the user says:
- "how should we...", "what's the best way to...", "compare..."
- "find examples of...", "research...", "explore..."
- "what approaches exist for...", "assess..."
- The task is a question requiring external investigation, not a
  decision the user can make from their own context

**→ Full Flow** when the user says:
- "build...", "add...", "create..." (a feature, not a question)
- "let's do the full flow", "design and implement..."
- The task is a feature that needs both design and code
- Default for feature requests when the user doesn't specify

**→ Design only** when the user says:
- "let's design...", "start a design process for..."
- "I want to review each decision"
- The user explicitly wants design without implementation
- (This runs Full Flow Phase 1 — same steps, same quality gates)

**→ Implementation only** when the user says:
- "implement...", "start the implementation..."
- A `designs/<feature>.org` already exists with a Design section
- The design is done, only code remains
- (This runs Full Flow Phase 2 — same steps, same quality gates)

**→ No flow** (free-form chat is fine) when the task is:
- A bug fix, typo, config change, or single-file edit
- Fully specified with no ambiguity
- A quick question answerable in one response

If a "quick" chat task starts growing (multiple files, design
decisions, back-and-forth), suggest switching to a flow before
the work becomes hard to track.

**Proactive dialog suggestion.** When a free-form conversation
involves 3+ exchanges on the same topic with decisions being made
implicitly, suggest the Dialog Flow: "We're making decisions here
that might be worth tracking — want to switch to a dialog flow so
we capture them?" The benefit: convergence gate, decision artifact,
contradiction checking, and aggregate drift detection that free-form
chat lacks.

## Presenting Options

When presenting flow options to the user, use this compact format:

```
Available flows:
  prompt <idea>        — capture an idea for later
  research <question>  — explore approaches, compare options
  dialog <topic>       — discuss to reach a specific decision
  full <feature>       — design + implement, Claude-autonomous
  design <feature>     — design only (Full Flow Phase 1)
  implement <feature>  — implement only (Full Flow Phase 2)

Which fits, or describe what you need?
```

Keep it to 5 lines + prompt. Do not explain each flow's internal
steps — the user picks by intent, not by process.

## Flow Files

- Research: `~/.claude/flows/RESEARCH-FLOW.md`
- Dialog: `~/.claude/flows/DIALOG-FLOW.md`
- Full / Design / Implement: `~/.claude/flows/FULL-FLOW.md`
- Architecture: `~/.claude/flows/ARCHITECTURE.md` (read when designing subsystem boundaries)
- Grounded Reasoning: `~/.claude/flows/GROUNDED-REASONING.md` (read by all flows automatically)
- Code Quality: `~/.claude/flows/CODE-QUALITY.md` (read before writing code)
- Denote Metadata: `~/.claude/flows/DENOTE.md` (read by all flows automatically)
- Query Script: `~/.claude/flows/denote-query` (artifact queries and extraction)
- Design Artifacts: `~/.claude/flows/designs/` (denote-named flow design docs)

Read the selected flow file before starting. Do not memorize
flow steps — always read the file for the current instructions.
All flows reference `DENOTE.md` for metadata behavior (active by
default; disabled by `denote: disabled` in project CLAUDE.md).
