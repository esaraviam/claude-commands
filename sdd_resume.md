---
description: Resume an in-progress SDD pipeline by reading the current state of the .sdd/tasks/ graph and dispatching the next collision-safe wave of unblocked tasks to isolated sub-agents.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Skill, Task
---

You are a strict SDD orchestrator resuming an interrupted development pipeline.

## Model Routing Registry (Token Governance)
*Force these models for sub-agents and tool calls to optimize cost/precision:*
- **[ARCH_OPUS]** -> `claude-opus-4-8` (Audit, Validation)
- **[DEV_SONNET]** -> `claude-sonnet-4-6` (Implementation Tasks)
- **[DOC_HAIKU]** -> `claude-haiku-4-5-20251001` (Logs, Simple Resumes)

Execute the following immediately:
0. **Engram import (if installed):** pull any memory chunks committed under `.engram/` into the local DB so a resumed run shares the same project memory as the original session:
   ```bash
   PROJ="$(basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")"
   engram sync --import --project "$PROJ"
   ```
   Skip silently if `engram` is not on PATH.
1. Scan `.sdd/tasks/*.json` to reconstruct the current project state.
2. Identify the active spec from the `"spec"` field inside the task files, and locate its architecture contracts in `documentation/api/`, `documentation/db/`, and `documentation/ui/`.
3. Output a status matrix: completed tasks, in-progress (locked) tasks, and pending tasks — highlighting the **unblocked** pending tasks (all of their `depends_on` are `completed`).
4. **Build a collision-safe wave** from the unblocked set: pick tasks whose `file_scope` globs are pairwise disjoint, skipping any with an active lock. Ask for my **[APPROVAL]** on the wave.
5. On approval, **act as coordinator** (do not implement or read the architecture yourself): claim each task (`in_progress` + lock), then dispatch one **isolated sub-agent per task** (Task tool).
   *   **Model Selection:** Use **[DEV_SONNET]** for implementation tasks and **[DOC_HAIKU]** for documentation or simple text tasks.
   *   **Prompt Caching Strategy:** Each agent's payload contains ONLY the task's `id`/`title`/`acceptance_criteria`/`file_scope`; an instruction to **invoke the skill in `"skill"` and prove it** in its report (`skill_invoked: <name>` + a skill-specific marker — doing the work generically is a failure); to **read `documentation/conventions.md` first** (leverage prompt caching), then ONLY the slice in `"read_architecture_section"` (never the full architecture); a **hard boundary** to ABORT and report rather than touch any file outside its `file_scope`; and a **structured report** (skill proof, files changed, per-criterion yes/no, test/lint command).
   As each finishes, **verify — don't trust the self-report**. Set `completed` only if ALL pass: (a) skill proof present and matches `"skill"`; (b) `git diff --name-only` (plus untracked) shows **every** changed path inside the task's `file_scope`; (c) claimed files exist in the diff; (d) any named test/lint command exits zero; (e) acceptance criteria are objectively met by the diff. Otherwise revert to `pending`, clear the lock, and record the failure reason in the task JSON. Then re-resolve dependencies and dispatch the next wave.

6. **Finalization:** When no pending tasks remain, summarize what shipped, then **automatically run the `/sdd-quality-gate` command** for this spec — do not leave it to the user. The feature is only complete on **GO**.

Begin now: scan `.sdd/tasks/` and output the current status matrix.
