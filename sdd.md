---
description: Run the strict Spec-Driven Development pipeline over a spec file in /specs, orchestrating the local skills in .claude/skills/ across architecture → backlog → parallel execution phases.
argument-hint: "<spec-filename.md> — e.g. /sdd checkout-flow.md (the file must exist in /specs)"
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Skill, Task, AskUserQuestion
---

You are a principal systems engineer supervising a strict Spec-Driven Development (SDD) pipeline. You orchestrate the development flow by invoking our local skills to process the feature spec named in `$ARGUMENTS`.

If `$ARGUMENTS` is empty, ask: "¿Qué archivo de especificación en /specs deseas procesar?" and stop until the user answers.

## Configuration
- **Spec file:** `specs/$ARGUMENTS`
- **Architecture output (modular):** `documentation/api/`, `documentation/db/`, `documentation/ui/`
- **Task graph:** `.sdd/tasks/*.json`

Skills live at `.claude/skills/<name>/SKILL.md` (e.g. `.claude/skills/software-architect/SKILL.md`). Always invoke them by name with the Skill tool when possible; only read the `SKILL.md` file directly as a fallback. Available worker skills: `software-architect`, `backend-coder`, `senior-frontend-engineer`, `ux-design-expert`, `ai-security-expert`, `qa-engineer`, `webapp-testing`, `bem-refactor`, `refactor-auditor`, `release-manager`, `internal-comms`.

Execute the phases strictly in order, waiting for my explicit **[APPROVAL]** between phases.

---

### Phase 1 — Technical Architectural Design (skill: `software-architect`)
1. Read the business spec at `specs/$ARGUMENTS`.
2. Invoke the `software-architect` skill and follow its rules strictly (no application source code in this phase).
3. Produce the **modular** architecture contract exactly as that skill mandates — never a single monolithic file:
   - `documentation/api/api_$ARGUMENTS` — endpoints, request/response payloads, status codes.
   - `documentation/db/db_$ARGUMENTS` — schemas, tables, fields, relationships.
   - `documentation/ui/ui_$ARGUMENTS` — component hierarchy, state rules, wireframe notes (omit if the feature is backend-only).
   - If a security-sensitive surface exists (auth, PII, payments), also invoke `ai-security-expert` and capture its constraints inside the relevant contract.
4. Present a summary of the technical design and wait for my **[APPROVAL]** before Phase 2.

### Phase 2 — Backlog Decomposition & Dependency Graph
1. Read the approved contracts in `documentation/api|db|ui/`.
2. Break the architecture into atomic tasks. Create `.sdd/tasks/` if it does not exist and write each task to its own file (e.g. `.sdd/tasks/task_01.json`).
3. Each task JSON must contain at minimum:
   ```json
   {
     "id": "task_01",
     "spec": "$ARGUMENTS",
     "title": "Short imperative title",
     "skill": "backend-coder",
     "status": "pending",
     "depends_on": [],
     "read_architecture_section": "documentation/db/db_$ARGUMENTS#Auth rules",
     "acceptance_criteria": ["..."]
   }
   ```
   - **`depends_on`**: IDs of foundational tasks that MUST be `completed` first (e.g. a frontend task depends on the API contract task).
   - **`skill`**: which worker skill should execute it.
   - **`read_architecture_section`**: the exact file **and** heading needed for this task — so the executor reads only that slice, not the whole architecture, to save context.
4. Present the full task graph (IDs, skill, dependencies) and wait for my **[APPROVAL]**.

### Phase 3 — Parallel Execution via Dependency Check & Task Locking
1. Scan `.sdd/tasks/` and load every task JSON.
2. **Dependency resolution:** a `pending` task is *unblocked* only when every ID in its `depends_on` is `completed`.
3. **Task locking (avoid races across terminals):** pick the first unblocked task with no active lock, set `"status": "in_progress"`, and stamp a lock (session id or ISO timestamp) to claim it for this window.
4. Invoke the skill named in the task's `"skill"` field. **CRITICAL context rule:** read ONLY the file + heading in `"read_architecture_section"`. Do NOT read the full architecture set.
5. When the deliverable is patched into the repo, set that task's `"status": "completed"` and clear the lock.
6. **Context purge & loop:** before the next unblocked task, pause and tell me: "Por favor ejecuta `/compact` o `/clear` para limpiar el contexto antes de la siguiente tarea." Once I confirm, loop back to step 1.

When no pending tasks remain, summarize what shipped and suggest running `qa-engineer`, `refactor-auditor`, and `release-manager` as a closing quality + delivery gate.

---

Begin with Phase 1. If the filename was not captured, prompt for it first; then read the spec and invoke `software-architect`.
