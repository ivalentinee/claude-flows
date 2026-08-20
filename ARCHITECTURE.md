# Architecture — Instructions for Claude

System-level architectural guidance. Read this file alongside flow
files when designing features that create new subsystems, modify
subsystem boundaries, or change how subsystems communicate.

CODE-QUALITY.md handles module-level quality (roles, naming, tests).
This file handles system-level structure (boundaries, communication,
contracts).

---

## The "Driver" Heuristic

A subsystem boundary exists where **authorship of action changes.**

Ask: "who decides WHAT happens (driver) and who decides HOW to do
it (responder)?" The boundary is the message/call between them.

- Request handlers **drive** actions (decide what to process)
- Action/service layer **drives** remote calls and storage
- Remote clients **respond** to the action layer's requests
- Storage **responds** to the action layer's persistence requests

```
Request Handler (AMQP) ──► │                       │ ──► Remote System 1 (AMQP)
Request Handler (HTTP) ──► │  Action/Service Layer  │ ──► Remote System 2 (HTTP)
                           │                       │ ──► Storage (DB)
```

All subsystems sit at the **same rank**. No subsystem wraps another.
Dependencies flow from driver to responder, never back.

---

## Single-Rank Architecture

Subsystems are peers connected by message passing, not concentric
layers. The communication layer exists on BOTH sides of the action
layer (inbound handlers AND outbound clients) — it doesn't wrap
around it.

**Why not clean/onion/hexagonal:**
- Communication isn't a single ring — request handlers and gateway
  clients are fundamentally different boundary types
- Wrapping forces artificial equivalence between inbound and outbound
- Feature changes in layered architectures touch every layer;
  single-rank changes touch one or two subsystems

**Single-rank does NOT mean no internal structure.** Each subsystem
has internal organization (see CODE-QUALITY.md "Module Roles":
communication/action/data building). "Rank" refers to how subsystems
relate to each other, not how they're organized internally.

### Supervision ≠ behavioral hierarchy

In Erlang/Elixir, supervision trees manage lifecycle (restart,
cleanup), not behavioral dependency. A supervised actor is a
behavioral peer of its siblings. The supervisor hierarchy is
orthogonal to the single-rank action flow.

---

## Subsystem Naming

Name subsystems by **actor role**, not by technology or convention:

| Instead of | Use |
|------------|-----|
| `services/` | `template_service/`, `run_service/` |
| `gateways/` | `render_farm_client/`, `comfyui_client/` |
| `http/controllers/` | `api_request_handler/` (or keep if idiomatic) |
| `mq/` | `amqp_request_handler/` (or keep if idiomatic) |

The `find` test from grepable code applies: scanning directory names
should tell you which actor handles what responsibility.

Exception: framework-idiomatic names (Phoenix `controllers/`,
Express `routes/`) are acceptable when the framework enforces them.

---

## Message Contracts

Subsystems communicate via **typed contracts** at every boundary.

**Within the same runtime:**
- Elixir: typed structs with `@type` and `@spec`
- TypeScript: interfaces or branded types
- No raw maps or `any` crossing subsystem boundaries
- Reference: `src/platform/types/Services/Operation/index.ts`,
  `src/definitions/types`, `src/compiler/types`

**Across runtime/protocol boundaries (AMQP, HTTP):**
- JSON Schema or OpenAPI for the contract definition
- Both sides reference the same schema
- Runtime validation at the boundary (parse, don't assume)

**The grepability connection:** typed contracts produce grep handles.
`OperationFn` is grepable; `(data: any) => any` is not.

---

## Orchestration for Workflow Visibility

When a multi-step workflow spans subsystems, prefer **orchestration**
(one actor owns the sequence) over choreography (decentralized
event reactions).

**Why orchestration:**
- The workflow is visible in one place (grepable)
- Error handling is centralized (orchestrator decides retry/compensate)
- The "driver" heuristic applies: one actor drives the sequence

**Orchestration retains single-rank** at the "I don't care" level:
as long as the protocol is fulfilled, no module knows about another
module more than its communication protocol. The orchestrator is a
peer that happens to drive the sequence — it doesn't gain special
knowledge of responders' internals.

**When to use choreography:**
- Subsystems genuinely react independently (no single sequence owner)
- Eventual consistency is acceptable
- The workflow is truly emergent, not designed

---

## Anti-Spaghetti Discipline

Single-rank without discipline becomes tangled message spaghetti.
These rules are non-negotiable:

**Unidirectional action flow.**
- For any pair of subsystems, one drives and the other responds.
- If bidirectional communication exists, it must be explicitly
  designed (reactive pipeline) not implicit coupling.
- When refactoring: draw the action flow arrows. If any arrow
  points "backward" (responder driving the driver), that's a
  boundary violation.

**No subsystem bypassing.**
- Request handlers do NOT call remote clients directly.
- The action layer mediates all cross-boundary flows.
- When refactoring: if a controller imports from `gateways/`,
  extract the logic into the service layer.

**Clear state ownership.**
- Each piece of state has exactly one owning subsystem.
- Others read via queries or receive via events.
- When refactoring: if two subsystems write to the same state,
  one of them is overstepping its boundary.

**Typed protocols at boundaries.**
- Every subsystem-to-subsystem call uses typed contracts.
- No raw maps, no `any`, no stringly-typed dispatch.
- When refactoring: introduce types at the boundary first,
  before restructuring internals.

---

## Three-Level Synergy

Architecture, module design, and code quality form a reinforcing
stack:

| Level | Pattern | Determines | Guidance |
|-------|---------|-----------|----------|
| Architecture | Actor-driven single-rank | Where subsystem boundaries go | This file |
| Module | Three-role decomposition | How a subsystem is internally organized | CODE-QUALITY.md |
| Code | Grepable naming | How files/functions are discoverable | CODE-QUALITY.md |

A well-architected system has:
- Subsystems findable by `find` on directory names (actor roles)
- Modules within each subsystem classified by role (communication/
  action/data building)
- Functions and values traceable by `grep` with semantic continuity

---

## Applying to Existing Codebases

When refactoring toward actor-driven architecture:

1. **Map current actors:** which parts "drive" and which "respond"?
   Draw the action flow. Arrows should be unidirectional per pair.

2. **Find boundary violations:** controllers calling gateways,
   business logic in handlers, storage accessed without mediation,
   responders driving drivers.

3. **Fix one boundary at a time:** don't restructure everything.
   Start with the highest-impact violation (usually: move logic out
   of controllers into services).

4. **Introduce typed contracts at boundaries:** even if keeping
   function calls (not message passing), define the interface as a
   typed contract. This makes the boundary explicit and grepable.

5. **Rename subsystems by actor role:** when touching a subsystem,
   rename its directory to reflect its actor role (opportunistic,
   not big-bang).

---

## Integration with Flows

The Boundary Analyst (Design Flow Step 2, Full Flow Step 2) should
reference this file when evaluating subsystem boundaries. The
Abstraction Minimalist should reference the three-level synergy when
proposing module splits.

When designing a new feature that introduces a new subsystem:
1. Apply the "driver" heuristic to determine the boundary
2. Name the subsystem by actor role
3. Define typed contracts for its communication protocol
4. Determine if it drives (orchestration) or responds (service)
5. Organize internally using three-role modules
