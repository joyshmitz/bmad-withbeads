# BMAD ↔ Beads: Reverse Engineering

## 1. Файлова структура інтеграції

```
qnb-epic5-worktree/
├── .beads/                           # Beads runtime
│   ├── config.yaml                   # sync-branch: beads-metadata
│   ├── issues.jsonl                  # SSOT для issues (127+ записів)
│   ├── beads.db                      # SQLite cache
│   ├── metadata.json                 # database: beads.db
│   ├── templates/                    # Templates for bd create
│   │   ├── story.yaml                # Story template
│   │   ├── story-orchestrator.yaml   # Domain-specific
│   │   └── story-*.yaml              # Per-epic templates
│   └── README.md                     # Beads intro
│
├── _bmad/
│   ├── bin/
│   │   ├── bd                        # ← WRAPPER SCRIPT (Unix)
│   │   └── bd.cmd                    # ← WRAPPER SCRIPT (Windows)
│   │
│   ├── _tools/beads/                 # ← Beads CLI provisioning
│   │   ├── package.json              # @beads/bd: "latest"
│   │   └── node_modules/.bin/bd      # Actual binary
│   │
│   ├── bmm/data/
│   │   └── beads-conventions.md      # ← CANONICAL CONVENTIONS DOC
│   │
│   ├── bmm/agents/
│   │   ├── dev.md                    # Uses bd in activation steps
│   │   ├── sm.md                     # Uses bd for issue creation
│   │   └── quick-flow-solo-dev.md    # Uses bd for tracking
│   │
│   └── bmm/workflows/
│       ├── 3-solutioning/create-epics-and-stories/
│       │   └── steps/step-03-create-stories.md  # Creates epic/story issues
│       │
│       └── 4-implementation/
│           ├── sprint-planning/instructions.md   # Syncs epics to Beads
│           ├── create-story/instructions.xml     # Creates task hierarchy
│           ├── dev-story/instructions.xml        # Uses bd ready/close
│           └── code-review/instructions.xml      # Creates blocking findings
│
└── ~/.claude/skills/
    └── beads-execute/SKILL.md        # Global skill for "виконай qnb-xxx"
```

---

## 2. CLI Wrapper (_bmad/bin/bd)

```sh
#!/bin/sh
# BMAD Beads CLI wrapper - auto-generated

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
BD_PATH="/Users/sd/github/qnb-epic5-worktree/_bmad/_tools/beads/node_modules/.bin/bd"

# Fallback to relative path
if [ ! -x "$BD_PATH" ]; then
  BD_PATH="$SCRIPT_DIR/../_tools/beads/node_modules/.bin/bd"
fi

if [ ! -x "$BD_PATH" ]; then
  echo "Error: Beads CLI (bd) not found. Run BMAD installer to provision." >&2
  exit 1
fi

exec "$BD_PATH" "$@"
```

### Provisioning (_bmad/_tools/beads/package.json)

```json
{
  "name": "bmad-beads-tools",
  "dependencies": {
    "@beads/bd": "latest"
  }
}
```

---

## 3. Точки інтеграції в Agents

### Dev Agent (_bmad/bmm/agents/dev.md)

```xml
<step n="4">**BEADS PREFLIGHT**: _bmad/bin/bd version</step>
<step n="5">**WORK DISCOVERY**: _bmad/bin/bd ready --parent <story_id></step>
<step n="6">**CLAIM WORK**: _bmad/bin/bd update <story_id> --status in_progress</step>
<step n="9">Execute tasks/subtasks IN ORDER - Beads blockers enforce sequencing</step>
<step n="11">**COMPLETE TASK**: _bmad/bin/bd close <task_id> AND mark [x]</step>
<step n="14">**DISCOVERED WORK**: _bmad/bin/bd create '...' --deps discovered-from:<id></step>
<step n="15">**NEVER CLOSE STORY** with open blockers</step>
```

### SM Agent (_bmad/bmm/agents/sm.md)

```xml
<step n="6">**BEADS REQUIRED**: _bmad/bin/bd version</step>
<step n="7">**CREATE BEADS ISSUES**: epic → story → tasks with sequential blockers</step>
<step n="8">**USE BEADS LABELS**: bmad:stage:* for workflow stages</step>
```

---

## 4. Workflow Integration Points

### A. Sprint Planning (sprint-planning/instructions.md)

**Мета:** Sync epic files → Beads graph

```sh
## Step 0: Beads Preflight
_bmad/bin/bd version

## Step 1.5: Sync Beads Graph
# For each epic without Beads ID:
_bmad/bin/bd create "Epic: {title}" --type epic --label "bmad:stage:backlog"

# For each story:
_bmad/bin/bd create "{title}" --parent {epic_id} --type task \
  --label "bmad:story" --label "bmad:stage:backlog"

# Sequential blockers:
_bmad/bin/bd dep add {story_id} {prev_story_id} --type blocks

## Step 4: Generate sprint-status.yaml (DERIVED VIEW)
# tracking_system: beads
# beads_prefix: qnb
```

### B. Create Story (create-story/instructions.xml)

**Мета:** Story file → Beads task hierarchy

```xml
<step n="0" goal="Beads preflight check">
  <code>_bmad/bin/bd version</code>
</step>

<step n="1" goal="Story discovery (Beads-first)">
  <!-- Primary: Query Beads -->
  <code>_bmad/bin/bd list --json --label "bmad:story" --label "bmad:stage:backlog"</code>

  <!-- Fallback: sprint-status.yaml -->
</step>

<step n="1b" goal="Create Beads task hierarchy">
  <code>_bmad/bin/bd create "{{story_title}}" --parent {{epic_id}} --type task
    --label "bmad:story" --label "bmad:stage:backlog"</code>
</step>

<step n="5b" goal="Create task/subtask children">
  <!-- For each task in story file: -->
  <code>_bmad/bin/bd create "{{task_title}}" --parent {{story_id}}
    --type task --label "bmad:task" --label "bmad:stage:backlog"</code>

  <!-- Sequential blockers: -->
  <code>_bmad/bin/bd dep add {{task_N}} {{task_N_minus_1}} --type blocks</code>
</step>

<step n="6" goal="Update status">
  <code>_bmad/bin/bd label remove {{story_id}} "bmad:stage:backlog"</code>
  <code>_bmad/bin/bd label add {{story_id}} "bmad:stage:ready-for-dev"</code>
</step>
```

### C. Dev Story (dev-story/instructions.xml)

**Мета:** Execute tasks from Beads queue

```xml
<step n="0">Beads preflight</step>

<step n="1" goal="Find next ready story (Beads-first)">
  <code>_bmad/bin/bd ready --json --label "bmad:story" --label "bmad:stage:ready-for-dev"</code>

  <!-- Also check in-progress (resume) -->
  <code>_bmad/bin/bd list --json --label "bmad:story" --label "bmad:stage:in-progress"</code>
</step>

<step n="1" anchor="task_check">
  <!-- Get ready tasks under story -->
  <code>_bmad/bin/bd ready --json --parent {{story_id}}</code>
</step>

<step n="4" goal="Mark in-progress">
  <code>_bmad/bin/bd label remove {{story_id}} "bmad:stage:ready-for-dev"</code>
  <code>_bmad/bin/bd label add {{story_id}} "bmad:stage:in-progress"</code>
  <code>_bmad/bin/bd update {{story_id}} --status in_progress</code>
</step>

<step n="8" goal="Complete task">
  <code>_bmad/bin/bd close {{task_id}}</code>
  <!-- Then check for next ready task -->
  <code>_bmad/bin/bd ready --json --parent {{story_id}}</code>
</step>

<step n="9" goal="Story completion">
  <!-- Verify all tasks closed -->
  <code>_bmad/bin/bd list --json --parent {{story_id}} --status open</code>

  <!-- Update to review -->
  <code>_bmad/bin/bd label add {{story_id}} "bmad:stage:review"</code>
</step>
```

### D. Code Review (code-review/instructions.xml)

**Мета:** Review findings → Blocking issues

```xml
<step n="4" goal="Create action items">
  <!-- HIGH/MEDIUM findings block story completion -->
  <code>
  _bmad/bin/bd create "Fix: {{finding}}" \
    --parent {{story_id}} \
    --type bug \
    --label "bmad:review-finding" \
    --label "bmad:severity:{{severity}}"
  </code>

  <!-- Add blocker so story cannot close -->
  <code>_bmad/bin/bd dep add {{story_id}} {{finding_id}} --type blocks</code>
</step>

<step n="5" goal="Check blockers before done">
  <code>_bmad/bin/bd dep list {{story_id}} --type blocks --json</code>

  <!-- If blockers exist, story stays in review -->
  <!-- Only mark done when all blockers closed -->
</step>
```

---

## 5. Label System (BMAD → Beads)

| BMAD Stage    | Beads Label              | Beads Status |
|---------------|--------------------------|--------------|
| backlog       | bmad:stage:backlog       | open         |
| ready-for-dev | bmad:stage:ready-for-dev | open         |
| in-progress   | bmad:stage:in-progress   | in_progress  |
| review        | bmad:stage:review        | in_progress  |
| done          | bmad:stage:done          | closed       |

### Додаткові labels

- `bmad:story`, `bmad:task`, `bmad:subtask` — type markers
- `bmad:review-finding` — code review issues
- `bmad:severity:high/medium/low` — priority

---

## 6. Hierarchy Mapping

| BMAD Concept   | Beads Type        | Parent | Example ID    |
|----------------|-------------------|--------|---------------|
| Epic           | epic              | None   | qnb-609       |
| Story          | task+bmad:story   | Epic   | qnb-609.1     |
| Task           | task+bmad:task    | Story  | qnb-609.1.1   |
| Subtask        | task+bmad:subtask | Task   | qnb-609.1.1.1 |
| Review Finding | bug               | Story  | qnb-609.1.2   |

---

## 7. Beads-Execute Skill (~/.claude/skills/beads-execute/)

**Trigger:** "виконай qnb-7ic", "execute issue", "run task"

### Execution Flow

1. `bd show <issue-id>` — Read issue
2. Check blockers — Verify dependencies closed
3. `bd update <id> --status=in_progress` — Claim
4. Find `**Workflow:**` in notes — Extract slash command
5. Load agent sidecar — `.bmad-user-memory/<assignee>/`
6. Read story file — From `File:` path in notes
7. Execute workflow — Run slash command
8. Verify acceptance_criteria — All checkboxes done
9. `bd close <issue-id>` — Complete

### Field Policy

| Field               | Purpose                      |
|---------------------|------------------------------|
| assignee            | WHO executes (dev, sm, qa)   |
| description         | WHY/WHAT/OUTPUT              |
| design              | HOW TO (blockers, patterns)  |
| notes               | Workflow command, file links |
| acceptance_criteria | DONE checkboxes              |

---

## 8. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BMAD WORKFLOWS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  sprint-planning ──► create epics/stories in Beads                  │
│         │                                                           │
│         ▼                                                           │
│  create-story ────► create task hierarchy + sequential blockers     │
│         │                                                           │
│         ▼                                                           │
│  dev-story ───────► bd ready (find work) → bd close (complete)      │
│         │                                                           │
│         ▼                                                           │
│  code-review ─────► create blocking findings if issues found        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        BEADS LAYER                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  .beads/issues.jsonl  ◄────── SSOT (git-synced)                     │
│         │                                                           │
│         ├── bd create/update/close ──► Mutations                    │
│         ├── bd ready ────────────────► Unblocked work query         │
│         ├── bd list --parent ────────► Hierarchy query              │
│         ├── bd dep add/list ─────────► Dependency management        │
│         └── bd label add/remove ─────► Stage transitions            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DERIVED VIEWS (Secondary)                        │
├─────────────────────────────────────────────────────────────────────┤
│  sprint-status.yaml ◄──── Refreshed by sprint-planning workflow     │
│  Story files (.md)  ◄──── Rich context, checkboxes mirrored         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Ключові патерни для розширення

### A. Preflight Check (обов'язковий)

```xml
<step n="0" goal="Beads preflight">
  <code>_bmad/bin/bd version</code>
  <check if="fails">HALT - Beads required</check>
  <check if=".beads missing">
    <code>_bmad/bin/bd init --quiet</code>
  </check>
</step>
```

### B. Work Discovery (Beads-first)

```sh
# Find ready work
_bmad/bin/bd ready --json --label "bmad:story" --label "bmad:stage:ready-for-dev"

# Fallback to sprint-status.yaml only if Beads empty
```

### C. Sequential Blockers

```sh
# Story N blocked by Story N-1
_bmad/bin/bd dep add {story_N} {story_N-1} --type blocks

# Task N blocked by Task N-1
_bmad/bin/bd dep add {task_N} {task_N-1} --type blocks

# Review finding blocks story
_bmad/bin/bd dep add {story_id} {finding_id} --type blocks
```

### D. Status Transitions

```sh
# backlog → ready-for-dev
bd label remove {id} "bmad:stage:backlog"
bd label add {id} "bmad:stage:ready-for-dev"

# ready-for-dev → in-progress
bd label remove {id} "bmad:stage:ready-for-dev"
bd label add {id} "bmad:stage:in-progress"
bd update {id} --status in_progress

# in-progress → review
bd label remove {id} "bmad:stage:in-progress"
bd label add {id} "bmad:stage:review"

# review → done (only if no blockers)
bd label remove {id} "bmad:stage:review"
bd label add {id} "bmad:stage:done"
bd close {id}
```

---

## 10. Що можна вдосконалити

| Область        | Поточний стан              | Можливе покращення          |
|----------------|----------------------------|-----------------------------|
| Sync           | Manual via sprint-planning | Auto-sync hook on file save |
| Bidirectional  | Beads → YAML (one-way)     | YAML → Beads sync too       |
| Templates      | Basic story.yaml           | Domain-specific templates   |
| Reports        | bd list --json manual      | Sprint burndown/velocity    |
| Agent memory   | .bmad-user-memory/         | Link to Beads notes field   |
| CI integration | None                       | GitHub Actions for bd sync  |

---

**Це повний reverse engineering. Тепер ти бачиш:**

1. **Де живе код** (wrapper, tools, agents, workflows)
2. **Як працює flow** (preflight → discovery → execution → completion)
3. **Які команди використовуються** (create, update, close, ready, dep add)
4. **Як розширювати** (patterns for new workflows)
