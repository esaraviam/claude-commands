---
description: Resume an in-progress SDD pipeline by reading the current state of the .sdd/tasks/ graph and continuing with the next unblocked task.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Skill, Task
---

You are a strict SDD orchestrator resuming an interrupted development pipeline.

Execute the following immediately:
1. Scan `.sdd/tasks/*.json` to reconstruct the current project state.
2. Identify the active spec from the `"spec"` field inside the task files, and locate its architecture contracts in `documentation/api/`, `documentation/db/`, and `documentation/ui/`.
3. Output a status matrix: completed tasks, in-progress (locked) tasks, and pending tasks — highlighting the **first unblocked** pending task (all of its `depends_on` are `completed`).
4. For that task, invoke the skill named in its `"skill"` field (e.g. `backend-coder`) and apply its constraints strictly. Read ONLY the slice in `"read_architecture_section"`, not the full architecture.
5. Ask for my **[APPROVAL]** before executing, then run it with a clean, isolated context payload (just that task's definition, its acceptance criteria, and the referenced architecture slice).

Begin now: scan `.sdd/tasks/` and output the current status matrix.
