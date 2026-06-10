# SPEC: csharp-todo-pipeline

## Overview

This document specifies an orchestrated skill pipeline for complex, multi-step C# development tasks.

The core problem with single-agent task pipelines is that a single agent context becomes overloaded when handling complicated requests, leading to poor unit test generation and inconsistent code review. The solution is to decompose the work into a strict, item-by-item loop managed by a stateful orchestrator that delegates each stage to a dedicated sub-agent.

---

## Inputs

The orchestrator (`csharp-todo-pipeline`) is provided with:

1. **Problem summary** — a human-readable description of what is being implemented and why.
2. **TODO list** — a numbered, ordered list of discrete work items to apply sequentially.

The orchestrator has **no authority to modify the TODO list**. It may only read it and track progress through it.

---

## Skills to Create

| Skill | Role |
|---|---|
| `csharp-todo-pipeline` | Orchestrator — manages the item loop and delegates all work |
| `csharp-todo-work-item` | Applies the changes described by a single TODO item |
| `csharp-todo-work-tests` | Generates or updates xUnit tests for files modified in the current item |
| `csharp-todo-work-review-code` | Reviews modified files for standards compliance and fixes violations |
| `csharp-todo-work-review-spec` | Verifies that the work done for the current item matches the specification |
| `csharp-todo-rebuild` | Performs a full solution rebuild from scratch and confirms all tests pass |
| `csharp-commit-todo-changes` | Commits all accumulated changes once the full pipeline succeeds |

---

## State: Work-in-Progress File

The orchestrator maintains state via an **uncommitted** work-in-progress (WIP) file at the root of the workspace:

```
wip-todo-pipeline.md
```

- **Absence** of this file signals the start of a fresh run → begin at item 1.
- **Presence** of this file indicates a run in progress. The orchestrator reads it to determine the current item index and accumulated context (e.g. list of affected projects across all completed items).
- The WIP file is never committed. It is deleted by the orchestrator only after the final rebuild and commit succeed.

### WIP File Contents

```markdown
# TODO Pipeline — Work in Progress

## Current Item
3

## Problem Summary
<copy of the problem summary passed to the orchestrator>

## TODO List
<copy of the full TODO list>

## Completed Items
1, 2

## Affected Projects (all cycles)
Source/MyProject.Api, Source/MyProject.Domain
```

---

## Orchestrator Loop

For each item `N` in the TODO list (starting from the current item recorded in the WIP file, or item 1 if no WIP file exists):

### Step 1 — Apply the Work Item

Delegate to a sub-agent using the **`csharp-todo-work-item`** skill.

Provide:
- The problem summary.
- The full TODO list (for context only — the sub-agent applies **only item N**).
- The index `N` identifying which item to apply.

The sub-agent returns:
- A list of production files created or modified.
- A list of .NET projects affected (used to scope subsequent build/test steps).

On **unrecoverable error**: stop the pipeline and report. Do not advance.

---

### Step 2 — Generate Unit Tests

Delegate to a sub-agent using the **`csharp-todo-work-tests`** skill.

Provide:
- The list of production files modified in Step 1.
- The list of affected projects for this cycle.

The sub-agent builds and tests only the affected projects (not the full solution) to keep feedback fast.

On **unrecoverable error**: stop the pipeline and report. Do not advance.

---

### Step 3 — Review Code Standards

Delegate to a sub-agent using the **`csharp-todo-work-review-code`** skill.

Provide:
- The list of all files created or modified so far in this cycle (production and test).
- The list of affected projects for this cycle.

The sub-agent applies in-place fixes for standards violations and re-runs the affected-project build and tests.

On **unrecoverable error**: stop the pipeline and report. Do not advance.

---

### Step 4 — Verify Specification Compliance

Delegate to a sub-agent using the **`csharp-todo-work-review-spec`** skill.

Provide:
- The problem summary.
- The full TODO list.
- The index `N` of the current item.
- The list of files modified in this cycle.

The sub-agent verifies that the changes made during steps 1–3 satisfy the intent of TODO item N.

**If compliance check fails:**
- Write (but do not commit) `spec-non-compliance.md` at the workspace root, detailing the non-compliance and the affected item.
- Restart the loop at item N (do not advance to item N+1).
- The restarted sub-agents should treat `spec-non-compliance.md` as additional context.

**If compliance check passes:**
- Delete `spec-non-compliance.md` if it exists.
- Record the accumulated affected projects and advance to item N+1 in the WIP file.

---

### After the Last Item

Once all TODO items have passed steps 1–4:

### Step 5 — Full Rebuild

Delegate to a sub-agent using the **`csharp-todo-rebuild`** skill.

- Perform a **full rebuild** of the entire solution (`dotnet build --no-incremental Fluidite.slnx` or equivalent rebuild-all).
- Run all unit tests (`dotnet test Fluidite.slnx`).

On failure: stop the pipeline and report. Do not commit.

---

### Step 6 — Commit

Delegate to a sub-agent using the **`csharp-commit-todo-changes`** skill.

Provide:
- The problem summary (used as the basis for the commit message).
- The full list of files modified across all items.

On success:
- Delete the WIP file (`wip-todo-pipeline.md`).
- Report the commit hash to the user.

---

## Affected-Project Tracking

To avoid rebuilding the entire solution after each sub-agent step, each sub-agent in a cycle maintains a **list of affected .NET projects** (i.e. the `.csproj` / `.fsproj` files that contain or reference the modified files). This list is:

- Accumulated within a loop cycle across steps 1–4.
- Passed to each sub-agent so it can scope `dotnet build` and `dotnet test` to only the affected projects.
- Aggregated across all cycles in the WIP file for informational purposes.
- **Ignored** during the final Step 5 rebuild, which always builds the full solution.

---

## Error Handling Summary

| Situation | Action |
|---|---|
| Unrecoverable error in steps 1, 2, 3, or 5 | Stop pipeline. Report error. Leave WIP file in place. |
| Spec non-compliance (step 4) | Write `spec-non-compliance.md`. Restart loop at current item N. |
| Spec compliance achieved | Delete `spec-non-compliance.md` (if present). Advance to N+1. |
| Final rebuild or commit fails | Stop pipeline. Report error. Do not delete WIP file. |
| All items complete and committed | Delete WIP file. Report success and commit hash. |

---

## Skill Descriptions Summary

### `csharp-todo-pipeline`
Orchestrator. Receives a problem summary and numbered TODO list. Manages the item-by-item loop, delegates to sub-agents, tracks WIP state, and handles error/compliance branching.

### `csharp-todo-work-item`
Applies the changes described by a single TODO item. Returns modified files and affected projects. Does not run tests — that is delegated to subsequent skills.

### `csharp-todo-work-tests`
Accepts a list of modified production files and affected projects. Generates or updates xUnit 3.x tests for all testable types and verifies the affected-project build and tests pass.

### `csharp-todo-work-review-code`
Accepts a list of files and affected projects. Inspects each file against the project's coding standards, applies behaviour-preserving fixes in-place, and verifies the affected-project build and tests still pass.

### `csharp-todo-work-review-spec`
New skill. Reads the problem summary, the full TODO list, the current item index, and the files modified in the current cycle. Verifies that the implementation satisfies the intent of the current item. Returns pass or fail with a detailed rationale.

### `csharp-todo-rebuild`
Performs a `--no-incremental` (rebuild-all) build of the full solution and runs all unit tests. Returns pass or fail with build/test output.

### `csharp-commit-todo-changes`
Accepts the problem summary, change log, and full file list. Stages all changes, constructs a concise commit message from the problem summary and change log, commits, and returns the commit hash.

---

## File Layout in This Repository

```
skills/
├── csharp-todo-pipeline/
│   └── SKILL.md
├── csharp-todo-work-item/
│   └── SKILL.md
├── csharp-todo-work-tests/
│   └── SKILL.md
├── csharp-todo-work-review-code/
│   └── SKILL.md
├── csharp-todo-work-review-spec/
│   └── SKILL.md
├── csharp-todo-rebuild/
│   └── SKILL.md
└── csharp-commit-todo-changes/
    └── SKILL.md
```

---

## Resolved Design Decisions

### 1. Spec-compliance retry cap

The orchestrator caps non-compliance retries at **3 attempts per item**. On the fourth failure the item is treated as an unrecoverable error: the pipeline stops, the error is reported, and the WIP file is left in place so the run can be resumed once the issue is resolved manually.

The WIP file must track the retry count for the current item:

```markdown
## Retry Count (current item)
2
```

The count resets to 0 whenever the orchestrator advances to a new item.

### 2. Commit message content

The WIP file records a **change log summary** alongside the problem summary. The commit skill uses both to construct the commit message: the problem summary forms the subject line, and the change log provides the optional second line of context.

The change log is a free-form summary of up to two lines that captures the overall effect of all completed items — it does not need to enumerate items individually.

The WIP file gains a new section:

```markdown
## Change Log
Added IFoo interface, FooService implementation, and registered the service in the DI container.
```

### 3. Specification sources for `csharp-todo-work-review-spec`

The skill bases its compliance check on:

1. The **TODO item** being verified (primary intent).
2. Any **specification documents already present in the repository** (e.g. architecture docs, ADRs, PRDs — typically found under `docs/`).

When the TODO item conflicts with an established spec document, the conflict is **not** treated as a non-compliance failure. Instead, the assumption is that the TODO intentionally supersedes the existing spec, and the skill must:

- Flag the conflict explicitly in its report.
- Verify that the implementation matches the TODO item (not the older spec).
- **Update the conflicting spec document** to reflect the new direction, so the repository's specification remains consistent with the implemented behaviour.
