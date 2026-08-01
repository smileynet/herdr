---
title: "Create kiro-cli hook scripts (sh + ps1)"
status: done
priority: high
blocked_by: []
---

# Create kiro-cli hook scripts (sh + ps1)

## What to build

Write the `herdr-agent-state.sh` and `herdr-agent-state.ps1` hook scripts for kiro-cli. These are the actual scripts that get installed into kiro's hooks directory and report session state to the herdr server.

## Details

### Integration approach (session-only, v2 compatible)

The existing `kiro.toml` detection manifest already handles working/blocked/idle via screen patterns. The integration only needs to report agent session identity on `agentSpawn`.

The hook script will:
1. Receive JSON payload on stdin (kiro-cli hook contract)
2. Validate presence of `HERDR_ENV=1`, `HERDR_SOCKET_PATH`, `HERDR_PANE_ID`
3. Parse `session_id` from the JSON payload
4. Send `pane.report_agent_session` to herdr's server socket

### Hook payload from kiro-cli (v2)

```json
{
  "hook_event_name": "agentSpawn",
  "cwd": "/path/to/project",
  "session_id": "abc123-def456-789"
}
```

### Key differences from Claude/Codex hooks

- **No `transcript_path`** — kiro-cli stores sessions in SQLite, not files
- **No subagent filtering needed** — kiro-cli doesn't pass subagent hooks to the same shell script (delegates use separate sessions)
- **Binary is `kiro-cli`** — not `kiro` (the shorter command is an optional router)
- **Action arg**: The hook is invoked directly by kiro-cli via its hooks config, not as `./script.sh session` — kiro-cli just runs the command and pipes JSON on stdin. Use the `session` action pattern anyway for forward-compatibility with herdr's hook dispatch model.

### File locations

- `src/integration/assets/kiro/herdr-agent-state.sh` (Unix)
- `src/integration/assets/kiro/herdr-agent-state.ps1` (Windows)

### Pattern to follow

Base on `assets/claude/herdr-agent-state.sh` and `assets/claude/herdr-agent-state.ps1` but simplified:
- No subagent check
- No `transcript_path`
- Source identifier: `"herdr:kiro"`
- Agent name: `"kiro"`
- Version marker: `HERDR_INTEGRATION_VERSION=1`

## Acceptance criteria

- [ ] `src/integration/assets/kiro/herdr-agent-state.sh` exists with correct logic
- [ ] `src/integration/assets/kiro/herdr-agent-state.ps1` exists with correct logic
- [ ] Both include `HERDR_INTEGRATION_ID=kiro` and `HERDR_INTEGRATION_VERSION=1` markers
- [ ] Shell script handles missing env vars gracefully (exit 0)
- [ ] PowerShell script uses `herdr pane report-agent-session` CLI fallback
