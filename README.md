# Claude Flows

Structured workflows for Claude Code that guide feature development through explicit iteration loops with the user.

## Design Flow

An iterative design process that produces a finalized feature design through question/criticism/answer cycles. The user and Claude converge on a design via file-based state: a design doc, open questions, criticisms, user answers, a resolved archive, a persistent Critic context, and a shared design journal. Emphasizes documentation-first thinking (JSON Schema, OpenAPI, AsyncAPI) and formal specifications over prose. Uses specialized subagents to separate authoring from criticism, validate answers, run parallel reviews, and verify finalization completeness. The Critic subagent persists its working memory across iterations and re-evaluates the design every loop.

```
start <feature>     — Author subagent writes design + questions + journal; Critic subagent writes criticisms + initializes its context
loop                — apply answers, Validator checks consistency, Critic re-evaluates with persistent context, check convergence
review [N]          — N parallel reviewer subagents (default 6: correctness, edge cases, integration, API, tests, specs)
finalize            — consolidate into main design doc, Verifier checks nothing was lost (reads journal + critic context)
next                — pick and start the next deferred sub-feature
restart <feature>   — reopen a finalized feature for further iteration
```

## Implementation Flow

A structured implementation process that follows a finalized design through code, self-review, and user review cycles. Prefers e2e tests with real infrastructure over unit tests with mocks. Uses parallel specialized reviewer subagents (design fidelity, correctness, edge cases, tests, specs, style) for adversarial self-review, a fix validator subagent during iteration, and a completeness verifier before finalization.

```
implement <feature> — write code + tests; 6 parallel reviewer subagents produce review; fix; produce review plan
loop                — apply user notes, Fix Validator subagent checks fixes, update review plan
finalize            — Completeness Verifier subagent checks design coverage, then clean up
commit              — create a commit
resume              — pick up interrupted work from file state
retro               — optional retrospective on the cycle
```
