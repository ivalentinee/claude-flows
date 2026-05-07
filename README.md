# Claude Flows

Structured workflows for Claude Code that guide feature development through explicit iteration loops with the user.

## Design Flow

An iterative design process that produces a finalized feature design through question/criticism/answer cycles. The user and Claude converge on a design via file-based state: a design doc, open questions, criticisms, user answers, a resolved archive, persistent context files for the Critic and Boundary Analyst, and a shared design journal. Emphasizes documentation-first thinking (JSON Schema, OpenAPI, AsyncAPI) and formal specifications over prose. The Critic and Boundary Analyst subagents persist working memory across iterations, re-evaluating the design every loop — the Critic tracking concerns, the Boundary Analyst tracking subsystem boundaries and their interfaces.

```
start <feature>       — Author writes design + questions; Critic + Boundary Analyst run in parallel (criticisms, boundaries)
loop                  — apply answers, Validator checks consistency, Critic + Boundary Analyst re-evaluate, check convergence
review [N]            — N parallel reviewer subagents (default 7: correctness, edge cases, integration, API, tests, specs, boundaries)
finalize              — consolidate into main design doc, Verifier checks nothing was lost
next                  — pick and start the next deferred sub-feature
restart <feature>     — reopen a finalized feature for further iteration
boundary-audit [scope] — standalone: scan existing code for implicit subsystems, produce ranked report (max 5)
```

## Implementation Flow

A structured implementation process that follows a finalized design through code, self-review, and user review cycles. Starts with a two-phase planning step: a Module Relationship subagent maps the topology (modules, interfaces, supervision tree, data flow), then a Plan subagent sequences the build order. Prefers e2e tests with real infrastructure over unit tests with mocks. Uses 8 parallel reviewer subagents including a Principles Reviewer (SOLID/GRASP smell detector) and an Abstraction Minimalist (consistent abstraction tiers within functions and modules). The Abstraction Minimalist also runs as a lightweight trailing check after every code-writing step to catch tier violations before they compound.

```
implement <feature>          — plan → write code + tests → trailing abstraction check → 8 parallel reviewers → fix → review plan
loop                         — apply user notes, trailing abstraction check, Fix Validator, update review plan
finalize                     — Completeness Verifier checks design coverage, then clean up
commit                       — create a commit
resume                       — pick up interrupted work from file state
retro                        — optional retrospective on the cycle
abstraction-check <target>   — standalone: analyze abstraction tier consistency of any module/function
```
