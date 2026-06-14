# Manual de uso — SDD Flow

Guía de operación del pipeline de **Spec-Driven Development (SDD)** implementado con comandos slash de Claude Code (`~/.claude/commands/`) y skills locales (`~/.claude/skills/`).

> El flujo convierte una idea en software siguiendo 5 etapas: **especificar → diseñar → descomponer → ejecutar → validar**, con una compuerta de aprobación humana (`[APPROVAL]`) entre fase y fase y un **quality gate GO/NO-GO** obligatorio al cierre.

---

## 1. Mapa rápido

| Paso | Comando | Qué hace | Produce |
|------|---------|----------|---------|
| 1 | `/create-spec <archivo.md>` | Te entrevista y escribe la especificación de negocio | `specs/<archivo>.md` |
| 2 | `/sdd <archivo.md>` | Orquesta arquitectura → backlog → ejecución → quality gate | `documentation/`, `.sdd/tasks/`, código |
| 3 | `/sdd_resume` | Retoma un pipeline interrumpido desde el grafo de tareas | continúa la siguiente tarea desbloqueada |
| 4 | `/sdd-quality-gate [archivo.md]` | Verifica completitud + calidad y emite veredicto **GO / NO-GO** | `.sdd/quality-gate-report.md` |

Skills trabajadoras disponibles (invocadas por `/sdd`):
`software-architect`, `backend-coder`, `senior-frontend-engineer`, `ux-design-expert`, `ai-security-expert`, `qa-engineer`, `webapp-testing`, `bem-refactor`, `refactor-auditor`, `release-manager`, `internal-comms`.

---

## 2. Requisitos previos

- Ejecutar los comandos **desde la raíz del proyecto** sobre el que vas a trabajar (no desde `~/.claude`).
- El proyecto debe tener (o se crearán) estas carpetas:
  - `specs/` — especificaciones de negocio.
  - `documentation/api|db|ui/` — contratos modulares de arquitectura.
  - `.sdd/tasks/` — grafo de tareas en JSON.
- Las skills viven en `~/.claude/skills/<nombre>/SKILL.md` y se cargan automáticamente.

---

## 3. Paso 1 — Crear la especificación

```
/create-spec checkout-flow.md
```

El comando actúa como analista de negocio y te entrevista **una pregunta a la vez**:

1. **Nombre y objetivo** — propuesta de valor, qué problema resuelve.
2. **Actores / usuarios** — quién interactúa con la funcionalidad.
3. **Requisitos funcionales** — comportamientos no negociables / user stories.
4. **Entidades de datos** (opcional) — modelos, atributos, reglas de negocio.
5. **Criterios de aceptación** — cómo sabemos que está terminado.

Al terminar, escribe `specs/checkout-flow.md` con esta estructura fija:

```
# Spec: [Feature]
## 1. Objective & Value Proposition
## 2. User Personas & Actors
## 3. Functional Requirements & User Stories
## 4. Business Logic & Constraints
## 5. Explicit Acceptance Criteria
- [ ] ...
```

> 💡 Revisa el archivo generado antes de pasar a `/sdd`. Esta es la única etapa pensada para iterar a mano.

---

## 4. Paso 2 — Ejecutar el pipeline

```
/sdd checkout-flow.md
```

> El argumento es **el nombre del archivo dentro de `specs/`**, no la ruta completa. Si lo omites, el comando te preguntará cuál procesar y se detendrá hasta que respondas.

El pipeline corre en **3 fases estrictas** con `[APPROVAL]` obligatorio entre cada una, más una **Fase 4** de quality gate que se dispara sola al final (sin `[APPROVAL]` previo).

### Fase 1 — Diseño arquitectónico (`software-architect`)

- Lee `specs/checkout-flow.md`.
- Invoca `software-architect` (prohibido escribir código de aplicación aquí).
- Produce contratos **modulares** (nunca un archivo monolítico):
  - `documentation/api/api_checkout-flow.md` — endpoints, payloads, status codes.
  - `documentation/db/db_checkout-flow.md` — esquemas, tablas, campos, relaciones.
  - `documentation/ui/ui_checkout-flow.md` — jerarquía de componentes, estado, wireframes *(se omite si es backend-only)*.
- Si hay superficie sensible (auth, PII, pagos) invoca además `ai-security-expert` y registra sus restricciones en el contrato correspondiente.
- Presenta un resumen y **espera tu `[APPROVAL]`**.

### Fase 2 — Backlog y grafo de dependencias

- Lee los contratos aprobados.
- Descompone la arquitectura en **tareas atómicas**, una por archivo en `.sdd/tasks/task_01.json`, `task_02.json`, …

Esquema mínimo de cada tarea:

```json
{
  "id": "task_01",
  "spec": "checkout-flow.md",
  "title": "Título imperativo corto",
  "skill": "backend-coder",
  "status": "pending",
  "depends_on": [],
  "read_architecture_section": "documentation/db/db_checkout-flow.md#Auth rules",
  "acceptance_criteria": ["..."]
}
```

- `depends_on` — IDs que deben estar `completed` antes (p.ej. una tarea de frontend depende del contrato de API).
- `skill` — qué skill ejecuta la tarea.
- `read_architecture_section` — **archivo + heading exactos**; el ejecutor lee solo esa porción, no toda la arquitectura (ahorra contexto).

Presenta el grafo completo (IDs, skill, dependencias) y **espera tu `[APPROVAL]`**.

### Fase 3 — Ejecución paralela con locking

1. Escanea `.sdd/tasks/` y carga todas las tareas.
2. **Resolución de dependencias:** una tarea `pending` se *desbloquea* solo cuando todos sus `depends_on` están `completed`.
3. **Task locking:** toma la primera tarea desbloqueada sin lock activo, la marca `in_progress` y estampa un lock (session id / timestamp) para reclamarla en esta ventana — evita carreras entre terminales.
4. Invoca la skill del campo `"skill"`. **Regla crítica de contexto:** lee solo el `read_architecture_section`, nunca toda la arquitectura.
5. Cuando el entregable queda en el repo, marca la tarea `completed` y libera el lock.
6. **Purga de contexto:** antes de la siguiente tarea te pedirá ejecutar `/compact` o `/clear`. Confirmas y vuelve al paso 1.

### Fase 4 — Quality Gate obligatorio (auto-invocado)

Cuando no quedan tareas `pending`, el pipeline **todavía no está terminado**. `/sdd` invoca automáticamente `/sdd-quality-gate` (ver Paso 4). El feature solo se considera completo cuando el gate devuelve **GO**.

- Si devuelve **NO-GO**, te indica la tarea/skill exacta a corregir (normalmente vía `/sdd_resume`) y vuelves a correr el gate.
- El gate **nunca muta git**; tras un GO, tú ejecutas el release real.

---

## 5. Paso 3 — Retomar tras una interrupción

Si cerraste la sesión, hiciste `/clear`, o cambiaste de terminal a mitad de la Fase 3:

```
/sdd_resume
```

- Reconstruye el estado escaneando `.sdd/tasks/*.json`.
- Detecta el spec activo y localiza sus contratos en `documentation/api|db|ui/`.
- Imprime una **matriz de estado**: completadas / en progreso (locked) / pendientes, resaltando la **primera tarea desbloqueada**.
- Pide `[APPROVAL]` y ejecuta esa tarea con contexto aislado (solo su definición + criterios + la porción de arquitectura referenciada).

> No necesita argumentos: el estado vive completo en `.sdd/tasks/`.

---

## 6. Ciclo de trabajo recomendado (paralelismo real)

El locking permite avanzar varias tareas a la vez en **terminales distintas**:

```
Terminal A: /sdd checkout-flow.md     # corre fases 1–2, arranca fase 3
Terminal B: /sdd_resume               # toma la siguiente tarea desbloqueada
Terminal C: /sdd_resume               # toma otra tarea sin dependencias
```

Cada terminal estampa su lock, así dos sesiones no pelean por la misma tarea. Entre tareas, ejecuta `/compact` o `/clear` para mantener el contexto limpio.

---

## 7. Estructura de carpetas resultante

```
proyecto/
├── specs/
│   └── checkout-flow.md              # Paso 1
├── documentation/
│   ├── api/api_checkout-flow.md      # Fase 1
│   ├── db/db_checkout-flow.md
│   └── ui/ui_checkout-flow.md
├── .sdd/
│   ├── tasks/
│   │   ├── task_01.json              # Fase 2
│   │   ├── task_02.json
│   │   └── ...
│   └── quality-gate-report.md        # Fase 4 (veredicto GO/NO-GO)
└── src/ ...                          # Fase 3 (código entregado)
```

---

## 8. Paso 4 — Quality Gate (`/sdd-quality-gate`)

El gate es la **compuerta de cierre obligatoria** del pipeline. Se ejecuta solo (lo invoca `/sdd` al terminar la Fase 3) o lo corres a mano:

```
/sdd-quality-gate                 # infiere el spec desde .sdd/tasks/
/sdd-quality-gate checkout-flow.md
```

Emite un único veredicto vinculante: **GO** o **NO-GO**. El pipeline solo está "terminado" con un GO.

**Etapas internas:**

| Etapa | Qué valida | Bloquea si… |
|-------|------------|-------------|
| **0. Completitud** *(programática)* | Todas las tareas `completed`, contratos presentes, `acceptance_criteria` y `read_architecture_section` válidos | Hay tareas sin terminar o locks colgados → **NO-GO** sin correr las skills |
| **1. QA** (`qa-engineer` + `webapp-testing`) | Cumplimiento del spec, regresiones, flujos UI | Veredicto **REJECTED** → NO-GO |
| **2. Arquitectura** (`refactor-auditor`) | Salud arquitectónica, deuda técnica | Cualquier issue **BLOCKING** → NO-GO |
| **3. Release** (`release-manager`, *solo análisis*) | Bump SemVer, changelog, estrategia de merge | Reporta "no safe" si QA rechazó |

**Garantías clave:**

- La Etapa 0 es prerrequisito duro: **no se corren las skills de calidad sobre un pipeline incompleto**.
- El gate **nunca crea commits ni tags** — produce un *plan* de release. Tú ejecutas el release real tras el GO.
- Cada item bloqueante viene con su **route-back** explícito (qué tarea/skill corregir).
- El reporte completo queda en `.sdd/quality-gate-report.md`.

---

## 9. Errores y trampas comunes

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| `/sdd` pregunta qué archivo procesar | Invocaste sin argumento | `/sdd <archivo.md>` con el nombre dentro de `specs/` |
| No encuentra el spec | Lo corriste fuera de la raíz del proyecto | `cd` a la raíz y vuelve a ejecutar |
| Una tarea nunca se desbloquea | Dependencia mal puesta en `depends_on` o una tarea quedó `in_progress` colgada | Revisa/edita el JSON en `.sdd/tasks/`; limpia el lock manualmente |
| El arquitecto intenta escribir código | — | Es comportamiento prohibido por diseño; reitera que la Fase 1 es solo contratos |
| Se pierde contexto entre tareas | No purgaste | Ejecuta `/compact` o `/clear` cuando el pipeline lo pida, luego `/sdd_resume` |

---

## 10. Reglas de oro

1. **Un spec, un pipeline.** No mezcles features en el mismo `.sdd/tasks/`.
2. **Respeta los `[APPROVAL]`.** Son la oportunidad de corregir antes de que el costo suba.
3. **Lee solo la porción que necesitas.** El `read_architecture_section` existe para ahorrar contexto; no leas toda la arquitectura.
4. **Purga entre tareas.** Contexto limpio = ejecución más precisa.
5. **No declares el feature terminado sin un GO** de `/sdd-quality-gate`. El pipeline solo cierra con veredicto GO.
