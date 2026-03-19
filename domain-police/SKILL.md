---
name: domain-police
description: Analyze a backend module and propose domain extraction. Identifies pure domain logic, business rules, and determines whether the domain should be extracted as simple pure functions/types or as a DDD aggregate implemented as a decider (decide/evolve pattern).
---

Analyze the module provided by the user and produce a domain extraction proposal. Do NOT modify any files.

## Step 0 — Validate scope

Before doing anything else, resolve the target path and count the files inside it recursively.

**Reject and stop** if any of the following are true:

- The path resolves to more than **one module** (e.g. an entire `modules/` directory)
- The file count exceeds **20 files**
- The path is a top-level source folder (e.g. `src/`, `packages/`)

If the scope is too big, respond with:

> **Scope too large.** domain-police requires a single, specific module or subfolder. You provided `<path>` which contains `<N>` files across `<M>` directories. Please narrow the scope to a single module (e.g. `modules/payout`) or a specific subfolder within it.

Do not proceed until the user provides a valid scope.

## Step 1 — Read the module

The user will provide a module path or name. Resolve it (using project aliases if needed) and read all files in the module recursively. Also read any utility files referenced by the module that are relevant to understanding patterns (e.g. `createDeciderAggregate`, `decider` utilities).

## Step 2 — Classify every file and function

For each file, classify every export into one of these layers:

- **Domain** — pure business logic: no I/O, no framework deps, no external module deps. Includes: pure functions, calculations, business rules, value types, enums, constants that encode business rules.
- **Application** — orchestrates domain + infrastructure: services, policies, event handlers. Has I/O but delegates decisions to domain.
- **Infrastructure** — repos, DB queries, external API calls, feature flag reads.
- **Presentation** — GraphQL schema, resolvers, HTTP handlers.

## Step 3 — Evaluate aggregate fit

Apply these rules to decide if the domain warrants a **decider aggregate** rather than simple pure functions:

1. **State guards decisions** — business logic says "what should happen depends on what already happened". Current state must be consulted before acting.
2. **Invariants span multiple sub-entities** — a consistency rule must hold across a group of related objects simultaneously. They cannot be updated independently.
3. **Commands can be rejected** — inputs can be modeled as `Command` types and the answer is sometimes "no, because current state says so".
4. **Lifecycle with meaningful transitions** — domain objects have statuses with valid transition rules (e.g. `ESTIMATED → BILLED → VOIDED`).
5. **History/audit matters** — the events that led to the current state are as important as the state itself.
6. **Concurrent writes risk inconsistency** — two concurrent operations on the same entity could produce conflicting state without an explicit consistency boundary.

7. **Asymmetric write/read paths (CQS signal)** — The module has a narrow write path (one or very few mutating operations) that follows the pattern `load aggregate → run(command) → save`, while the read path bypasses the aggregate entirely: it either calls domain functions directly or queries the underlying table treating it as a read model/projection. The two paths are structurally different by design — writes are thick (all business logic flows through the decider), reads are thin (direct projections with no state mutation). When you see a service with very few mutating methods alongside several read-only query methods that go straight to the DB, this is a strong CQS signal.

If **3 or more rules** are triggered → propose a **decider aggregate**.
If fewer → propose **simple pure functions + types**.

## Step 4 — Produce the proposal

Output a structured report with these sections:

### Module overview

Brief description of what the module does.

### Layer classification

Table or list mapping each file/function to its layer (Domain / Application / Infrastructure / Presentation). Call out anything that is domain logic currently living in the wrong layer.

### Aggregate assessment

List which of the 6 rules are triggered and why, with concrete references to the code. State the verdict: **decider aggregate** or **pure functions**.

### Proposed domain structure

**If pure functions:**
List the functions and types that belong in the domain, with suggested filenames (e.g. `domain/types.ts`, `domain/calculations.ts`).

**If decider aggregate:**
Propose the full `domain/` folder shape:

- `domain/types.ts` — `State`, `Command`, `Event` discriminated unions to define
- `domain/decide.ts` — which existing logic maps to `decide(command, state): Event[]`
- `domain/evolve.ts` — which existing logic maps to `evolve(state, event): State`
- `domain/domain.ts` — pure helper functions to extract from service/policies
- `domain/index.ts` — exports `{ decide, evolve }` as the decider

For each file, describe concretely which existing code should move there.

### Caveats

Note any ambiguous cases, missing context, or decisions that require product/domain knowledge to resolve.
