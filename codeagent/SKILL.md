---
name: codeagent
description: "Run `codeagent-wrapper` to delegate tasks to an AI backend (codex/claude/gemini), resume sessions, or run parallel subtasks from stdin. Use when the user asks to use codeagent-wrapper, wants multi-backend execution, needs parallelization, or provides a SESSION_ID to continue."
---

# codeagent-wrapper

## Prereqs

- Verify the binary exists: `command -v codeagent-wrapper`
- Check version: `codeagent-wrapper --version`

## Single Task

**Default backend (codex):**
```bash
codeagent-wrapper "task" [workdir]
```

**Explicit backend:**
```bash
codeagent-wrapper --backend codex "task" [workdir]
codeagent-wrapper --backend claude "task" [workdir]
codeagent-wrapper --backend gemini "task" [workdir]
```

**Stdin mode (recommended for long prompts):**
```bash
codeagent-wrapper --backend codex - [workdir] <<'EOF'
task content
EOF
```

## Backends

- `codex` (default): best for code changes, refactors, tests
- `claude`: best for docs/specs/summaries
- `gemini`: best for UI scaffolding/prototypes

If the selected backend command is missing, the wrapper exits `127`.

## Resume Session

The wrapper prints a `SESSION_ID: ...` line in its output. Continue a session with:

```bash
codeagent-wrapper resume <session_id> "follow-up task" [workdir]
codeagent-wrapper resume <session_id> - [workdir] <<'EOF'
follow-up task content
EOF
```

## Parallel Execution

Parallel mode reads task configuration from stdin:

```bash
codeagent-wrapper --parallel --backend codex <<'EOF'
---TASK---
id: task1
workdir: /absolute/path/to/repo
---CONTENT---
analyze the current code structure
---TASK---
id: task2
dependencies: task1
workdir: /absolute/path/to/repo
---CONTENT---
propose a refactor plan and implement it
EOF
```

Optional per-task overrides (when needed):
- `backend: codex|claude|gemini`
- `dependencies: task_id_1, task_id_2`

## Controls

- `CODEX_TIMEOUT` (ms): override timeout (default: 7200000)
- `CODEAGENT_MAX_PARALLEL_WORKERS`: cap parallel workers (optional)
- `CODEAGENT_SKIP_PERMISSIONS`: only relevant to the Claude backend; use sparingly

## Output

Expect normal agent output plus a trailing `SESSION_ID: ...` line.
