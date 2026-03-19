---
name: coupling-police
description: Analyze a single backend module and produce a full coupling overview: outgoing dependencies (external libs + other modules), DB table access and column-level ownership, and incoming dependencies (which modules depend on it).
---

Analyze the module provided by the user and produce a coupling report. Do NOT modify any files.

## Step 0 — Validate scope

Before doing anything else, resolve the target path and count the files inside it recursively.

**Reject and stop** if any of the following are true:

- The path resolves to more than **one module** (e.g. an entire `modules/` directory)
- The file count exceeds **20 files**
- The path is a top-level source folder (e.g. `src/`, `packages/`)

If the scope is too big, respond with:

> **Scope too large.** coupling-police requires a single, specific module or subfolder. You provided `<path>` which contains `<N>` files across `<M>` directories. Please narrow the scope to a single module (e.g. `modules/payout`) or a specific subfolder within it.

Do not proceed until the user provides a valid scope.

## Step 1 — Read the module

Read all files in the module recursively. Pay close attention to:

- All `import` statements in every file
- All DB access patterns in repo files (`selectFrom`, `updateTable`, `insertInto`, `deleteFrom`, `.select([...])`, `.selectAll()`, `.set({...})`, `.values({...})`)

## Step 2 — Map outgoing dependencies and incoming events

**Module dependencies** — imports from sibling modules (`../otherModule`). Ignore external libraries and internal utilities (database, config, featureFlags, utils, auth, etc.) — only list inter-module dependencies. For each dependency:

- Which module is imported
- What is consumed (function names, types)
- Purpose: read data / trigger action

**Incoming events** — look at the module's `policies.ts` (or equivalent). Methods whose name starts with `on` (e.g. `onOnlineTherapistInvoiceGenerated`, `onPathPricingConfigured`) are event handlers, not outgoing dependencies. List them separately as incoming events, noting:

- The event name (derived from the method name or the event type imported)
- Which module emits it (infer from the import path of the event type, if available)
- What the handler does in response

## Step 3 — Map DB table access and column ownership

Scan all repo files and any file that directly touches the database. For each table accessed:

- **Table name** and which DB connection is used (`database.therapySessions`, `database.paymentsReplica`, etc.)
- **Read operations**: list the columns selected. If `.selectAll()` is used, flag it as a broad read.
- **Write operations**: list the columns written (from `.set({...})` or `.values({...})`). Classify the operation type: INSERT, UPDATE, DELETE.
- **Ownership assessment**:
  - **Owns** — the module is the primary writer (inserts rows, updates many columns, or the table name clearly belongs to this domain)
  - **Borrows** — the module only reads a few columns from a table whose name suggests it belongs to another domain
  - **Crosses boundary** ⚠️ — the module writes to a table that clearly belongs to another module's domain. Flag this explicitly.

## Step 4 — Map incoming dependencies (reverse)

Search the entire `modules/` directory for files that import from the subject module. Exclude `modulesHooks.ts` — it is the internal event broker responsible for wiring `on*` calls between modules, not a real semantic dependency.

For each dependent module found:

- Which files import it
- What they import (exported names)
- Whether the dependency is on the public exports (via the module's `index.ts`) or on internal files (direct path imports — flag these as ⚠️ internal coupling)

## Step 5 — Produce the report

Output a structured report with these sections:

### Module overview

One paragraph: what the module does and its role in the system.

### Outgoing dependencies

**Module dependencies**
For each imported module: what is consumed and purpose (read data / trigger action).

### Incoming events

List of events this module handles (from `on*` methods in policies), with the emitting module and what the handler does.

### DB surface

Table per accessed database table:
| Table | DB connection | Reads (columns) | Writes (columns + operation) | Ownership |
|---|---|---|---|---|

Flag any **boundary crossings** (writes to tables owned by other modules) prominently.

### Incoming dependencies

For each module that depends on the subject module:

- List what it consumes
- Flag any direct internal imports (not going through `index.ts`) as ⚠️

### Coupling summary

Close with a short paragraph identifying:

- The most significant coupling risks (high fan-in, boundary crossings, internal imports)
- Tables that are contested between multiple modules
- Any surprising or non-obvious dependencies worth a second look
