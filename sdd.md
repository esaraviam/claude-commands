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
   - `documentation/conventions.md` — **cross-cutting conventions every task must obey**: language/stack, naming rules, error/response format, auth & logging patterns, shared utilities and where they live, test conventions. Keep it short and concrete; this is the only shared truth cold agents get besides their own architecture slice. If it already exists from a prior spec, update it instead of overwriting.
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
     "file_scope": ["src/api/urls/**", "src/db/migrations/003_urls.sql"],
     "acceptance_criteria": ["..."]
   }
   ```
   - **`depends_on`**: IDs of foundational tasks that MUST be `completed` first (e.g. a frontend task depends on the API contract task).
   - **`skill`**: which worker skill should execute it.
   - **`read_architecture_section`**: the exact file **and** heading needed for this task — so the executor reads only that slice, not the whole architecture, to save context.
   - **`file_scope`**: the files/globs this task is allowed to create or modify. Two tasks may run **in parallel only if their `file_scope` sets are disjoint** — this is what lets Phase 3 fan out safely across background agents. Keep scopes tight and non-overlapping; if two tasks must touch the same file, give one a `depends_on` the other instead.
4. Present the full task graph (IDs, skill, dependencies) and wait for my **[APPROVAL]**.

### Phase 3 — Parallel Execution via Agent Fan-Out
You are now a **coordinator**: you do NOT implement tasks yourself and you do NOT read the architecture contracts. You dispatch each task to an **isolated sub-agent** (Task tool, `subagent_type: general-purpose`) that runs in its own context and returns only a result summary. This keeps your context lean across the whole run — no manual `/compact` between tasks.

Work in **waves**:

1. **Scan & resolve.** Load every task JSON in `.sdd/tasks/`. A `pending` task is *unblocked* only when every ID in its `depends_on` is `completed`.
2. **Build the wave (collision-safe).** From the unblocked set, select a batch whose `file_scope` globs are **pairwise disjoint**. Two tasks that share any file go in different waves — never dispatch them together. Tasks with no active lock only.
3. **Claim.** For each task in the wave, set `"status": "in_progress"` and stamp a lock (session id / ISO timestamp) before dispatch, so parallel terminals don't double-pick.
4. **Fan out in background.** Launch one sub-agent per task in the wave **in parallel** (`run_in_background: true`). Each agent's prompt must contain ONLY a tight payload:
   - the task's `id`, `title`, `acceptance_criteria` and `file_scope`;
   - an instruction to **invoke the skill named in `"skill"`** and apply its rules — and to **prove it** in the report (see below); the skill is where the expertise lives, so doing the work generically without invoking it is a failure, not a shortcut;
   - an instruction to **read `documentation/conventions.md` first**, then **read ONLY** the file + heading in `"read_architecture_section"` — never the full architecture set;
   - a **hard boundary**: it may create/modify only files inside its `file_scope`. If it discovers it needs to touch any file outside that scope, it must **ABORT and report it** — never edit out-of-scope files;
   - a required **structured report** containing exactly: (a) `skill_invoked: <name>` plus a one-line skill-specific marker proving the skill actually ran (e.g. the rule/checklist it applied); (b) the list of files it created/modified; (c) per acceptance criterion, satisfied yes/no; (d) the test/lint command to validate its work, if any.
5. **Await & reconcile (verify, don't trust).** A self-reported "done" is not enough. As each agent finishes, mark `"completed"` **only if all** of these pass; otherwise set it back to `"pending"`, clear the lock, and record the failure reason in the task JSON:
   - **Skill proof:** the report includes `skill_invoked` matching the task's `"skill"` and a plausible skill-specific marker. Missing/generic → reject (the skill likely never ran).
   - **Scope check:** run `git diff --name-only` (plus untracked) and confirm **every** changed path falls inside the task's `file_scope`. Any out-of-scope file → reject (also flags a potential collision).
   - **Existence check:** the files the report claims it created/modified actually exist and show in the diff.
   - **Test check:** if the task or report names a test/lint command, run it; non-zero exit → reject.
   - **Criteria check:** the task's `acceptance_criteria` are objectively satisfied by the diff, not merely asserted.
6. **Loop.** Re-resolve dependencies (completed tasks may unblock new ones) and dispatch the next wave. No context purge needed — each worker's context died with its agent.

> **Cross-terminal note:** the lock still lets you also run extra `/sdd_resume` terminals; but in-session background fan-out is now the primary parallelism, not multiple terminals.

### Phase 4 — Mandatory Quality Gate (auto-invoked)
When no pending tasks remain, the pipeline is **not** done yet. Summarize what shipped, then **automatically run the `/sdd-quality-gate` command** for this spec — do not leave it to the user. That gate verifies completeness and runs `qa-engineer` → `refactor-auditor` → `release-manager` (analysis only), and returns a single **GO / NO-GO** verdict.

- If the gate returns **NO-GO**, route back to the specific task or skill it names (usually via `/sdd_resume`) and re-run the gate. The feature is only complete on **GO**.
- The gate never mutates git; once it returns GO, the user performs the actual release.

---

Begin with Phase 1. If the filename was not captured, prompt for it first; then read the spec and invoke `software-architect`.
