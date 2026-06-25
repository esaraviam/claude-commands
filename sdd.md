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

## Model Routing Registry (Token Governance)
*Force these models for sub-agents and tool calls to optimize cost/precision:*
- **[ARCH_OPUS]** -> `claude-opus-4-8` (Architecture, Security, Quality Gate)
- **[DEV_SONNET]** -> `claude-sonnet-4-6` (Backlog Decomposition, Implementation Tasks)
- **[DOC_HAIKU]** -> `claude-haiku-4-5-20251001` (Documentation, Logs, Spec Interviews)

Skills live at `.claude/skills/<name>/SKILL.md` (e.g. `.claude/skills/software-architect/SKILL.md`). Always invoke them by name with the Skill tool when possible; only read the `SKILL.md` file directly as a fallback. Available worker skills: `software-architect`, `backend-coder`, `senior-frontend-engineer`, `ux-design-expert`, `ai-security-expert`, `qa-engineer`, `webapp-testing`, `bem-refactor`, `refactor-auditor`, `release-manager`, `internal-comms`.

Execute the phases strictly in order, waiting for my explicit **[APPROVAL]** between phases.

---

### Phase 0 — Project Bootstrap (Engram git-travel guard)
*Trivial programmatic check via Bash; no model routing needed. Run once at the start, idempotent, silent when everything is already fine. Skip entirely if `engram` is not installed.*

For the per-project Engram memory to **travel with the repo via git**, the `.engram/` directory (manifest + compressed chunks) must be committed — never git-ignored. Before Phase 1, verify this:

```bash
if command -v engram >/dev/null 2>&1 && git rev-parse --git-dir >/dev/null 2>&1; then
  # 1. .engram/ must NOT be ignored, or sync output would never get committed.
  #    Probe a file path *inside* .engram/ — a trailing-slash pattern (".engram/")
  #    only matches when checked against a path under the dir, even if it doesn't exist yet.
  if git check-ignore -q .engram/manifest.json 2>/dev/null; then
    echo "⚠️  .engram/ is git-ignored — Engram memory will NOT travel with the repo."
    echo "    Remove the matching pattern from .gitignore so '.engram/' can be committed."
  fi
  # 2. If chunks already exist but are untracked, remind to stage them.
  if [ -d .engram ] && [ -n "$(git status --porcelain .engram 2>/dev/null)" ]; then
    echo "ℹ️  Engram chunks under .engram/ are uncommitted — include them in your release commit:  git add .engram/"
  fi
fi
```

- If `.engram/` is git-ignored, surface it to the user and **offer to remove** the offending `.gitignore` line (do not delete other ignore rules). This is the one-time setup that makes memory shareable across clones.
- This phase **never commits** — it only checks and reminds; staging/committing `.engram/` happens in the user's post-GO release step. Skip the whole phase silently if `engram` is absent.

---

### Phase 1 — Technical Architectural Design (skill: `software-architect`)
*   **Model:** Use **[ARCH_OPUS]** for this phase.
1. Read the business spec at `specs/$ARGUMENTS`.
2. **Context Injection (Long-term Memory):**
   - Read `documentation/SYSTEM_MAP.md` (if exists) to understand current system constraints, existing modules, and avoid redundant designs.
   - **Engram — import teammate memories + semantic recall (per-project).** Engram memory travels with the repo under `.engram/` (committed to git). First derive the project key (the repo basename — this MUST match what `engram sync` writes, because `engram save` does **not** auto-detect the project), import any chunks pulled from git, then recall only the *relevant* prior design decisions:
     ```bash
     PROJ="$(basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")"
     engram sync --import --project "$PROJ"    # load .engram/ chunks committed by teammates/past runs
     engram search "<feature domain + key entities>" --type architecture --project "$PROJ" --limit 5
     ```
     Fold the results into the architect's context as "Existing decisions to respect / not contradict", instead of skimming the whole `SYSTEM_MAP.md`. If the `engram` binary is not installed, skip both steps silently and rely on `SYSTEM_MAP.md` alone.
3. Invoke the `software-architect` skill and follow its rules strictly.
4. Produce the **modular** architecture contract:
   - `documentation/api/api_$ARGUMENTS`
   - `documentation/db/db_$ARGUMENTS`
   - `documentation/ui/ui_$ARGUMENTS`
   - **Modular Conventions:** Write `documentation/conventions.md`. If the spec is complex, you may split conventions into a directory `documentation/conventions/*.md` (e.g., `auth.md`, `testing.md`) and reference them from the main `conventions.md`.
   - Security constraints via `ai-security-expert`.
4. Present a summary and wait for **[APPROVAL]**.

### Phase 2 — Backlog Decomposition & Dependency Graph
*   **Model:** Use **[DEV_SONNET]** for this phase.
1. Read the approved contracts.
2. **Behavioral Injection (Lessons Learned):**
   - Read `.sdd/retrospectives.json` (if exists). Extract previous failures, rejected patterns, or QA edge cases.
   - **Engram semantic recall (per-project):** run `engram search "<feature/task domain>" --type retrospective --project "$PROJ" --limit 5` (same `PROJ` = repo basename as Phase 1) to surface lessons from *any* prior spec in this project, not just this spec's local JSON.
   - Inject both sources as explicit "Constraint" or "Anti-pattern avoidance" rules into the relevant tasks' `acceptance_criteria`. If `engram` is not installed, skip the recall silently and rely on `retrospectives.json` alone.
3. Break architecture into atomic tasks in `.sdd/tasks/task_01.json`, etc.
3. Each task JSON must contain:
   ```json
   {
     "id": "task_01",
     "spec": "$ARGUMENTS",
     "title": "Short title",
     "skill": "backend-coder",
     "model_hint": "sonnet",
     "status": "pending",
     "depends_on": [],
     "read_architecture_section": "documentation/db/db_$ARGUMENTS#Auth rules",
     "file_scope": ["..."],
     "acceptance_criteria": ["..."]
   }
   ```
   - **`model_hint` Selection Rules (Mandatory):**
     - **"sonnet"**: Default for ANY task involving code, logic, refactoring, or architectural analysis.
     - **"haiku"**: ONLY for tasks purely descriptive, boilerplate documentation, or simple file formatting. **CRITICAL: NEVER assign "haiku" to a task that requires reasoning or analysis.**
4. Present the full graph and wait for **[APPROVAL]**.

### Phase 3 — Parallel Execution via Agent Fan-Out
You are now a **coordinator**: you do NOT implement tasks yourself and you do NOT read the architecture contracts. You dispatch each task to an **isolated sub-agent** (Task tool) that runs in its own context and returns only a result summary. This keeps your context lean across the whole run — no manual `/compact` between tasks.

Work in **waves**:

1. **Scan & resolve.** Load every task JSON in `.sdd/tasks/`. A `pending` task is *unblocked* only when every ID in its `depends_on` is `completed`.
2. **Build the wave (collision-safe).** From the unblocked set, select a batch whose `file_scope` globs are **pairwise disjoint**. Two tasks that share any file go in different waves — never dispatch them together. Tasks with no active lock only.
3. **Claim.** For each task in the wave, set `"status": "in_progress"` and stamp a lock (session id / ISO timestamp) before dispatch, so parallel terminals don't double-pick.
4. **Fan out in background.** Launch one sub-agent per task in the wave **in parallel** (`run_in_background: true`). 
   *   **Model Selection:** Respect the `"model_hint"` from the task JSON. If `"model_hint"` is missing, default to **[DEV_SONNET]**.
   *   **Prompt Caching Strategy:** Agent must read `documentation/conventions.md` (and any relevant modular convention file) first.
   *   **Payload:** Each agent's prompt must contain ONLY a tight payload:
       - the task's `id`, `title`, `acceptance_criteria` and `file_scope`;
       - an instruction to **invoke the skill named in `"skill"`** and apply its rules — and to **prove it** in the report (see below); the skill is where the expertise lives, so doing the work generically without invoking it is a failure, not a shortcut;
       - an instruction to **read `documentation/conventions.md` first** (to leverage prompt caching), then **read ONLY** the file + heading in `"read_architecture_section"` — never the full architecture set;
       - a **hard boundary**: it may create/modify only files inside its `file_scope`. If it discovers it needs to touch any file outside that scope, it must **ABORT and report it** — never edit out-of-scope files;
       - a required **structured report** containing exactly: (a) `skill_invoked: <name>` plus a one-line skill-specific marker proving the skill actually ran (e.g. the rule/checklist it applied); (b) the list of files it created/modified; (c) per acceptance criterion, satisfied yes/no; (d) the test/lint command to validate its work, if any.
5. **Await & reconcile (verify, don't trust).** A self-reported "done" is not enough. As each agent finishes, mark `"completed"` **only if all** of these pass; otherwise set it back to `"pending"`, clear the lock, and record the failure reason in the task JSON:
   - **Skill proof:** The report MUST include the marker `[SKILL-CONFIRMATION: <skill_name> | ... ]` as defined in the skill's `SKILL.md`. Missing marker → reject.
   - **Scope check:** run `git diff --name-only` (plus untracked) and confirm **every** changed path falls inside the task's `file_scope`. Any out-of-scope file → reject (also flags a potential collision).
   - **Existence check:** the files the report claims it created/modified actually exist and show in the diff.
   - **Test check:** if the task or report names a test/lint command, run it; non-zero exit → reject.
   - **Criteria check:** the task's `acceptance_criteria` are objectively satisfied by the diff, not merely asserted.
6. **Loop.** Re-resolve dependencies (completed tasks may unblock new ones) and dispatch the next wave. No context purge needed — each worker's context died with its agent.

> **Cross-terminal note:** the lock still lets you also run extra `/sdd_resume` terminals; but in-session background fan-out is now the primary parallelism, not multiple terminals.

### Phase 4 — Mandatory Quality Gate (auto-invoked)
When no pending tasks remain, the pipeline is **not** done yet. Summarize what shipped, then **automatically run the `/sdd-quality-gate` command** for this spec — do not leave it to the user. 
*   **Model:** This command will use **[ARCH_OPUS]** for final validation.
That gate verifies completeness and runs `qa-engineer` → `refactor-auditor` → `release-manager` (analysis only), and returns a single **GO / NO-GO** verdict.

- If the gate returns **NO-GO**, route back to the specific task or skill it names (usually via `/sdd_resume`) and re-run the gate. The feature is only complete on **GO**.
- The gate never mutates git; once it returns GO, the user performs the actual release.

---

Begin with Phase 1. If the filename was not captured, prompt for it first; then read the spec and invoke `software-architect`.
