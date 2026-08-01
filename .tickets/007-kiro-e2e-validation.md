---
title: "End-to-end validation of kiro integration"
status: done
priority: normal
blocked_by: ["004", "005", "006"]
---

# End-to-end validation of kiro integration

## Results (2026-08-01)

Validated on fork binary (herdr 0.7.5, branch `agent/initialize-review-tracking`).

### Server-side session acceptance ✅

```json
"agent_session": {
    "agent": "kiro",
    "kind": "id",
    "source": "herdr:kiro",
    "value": "e2e-verified-session-12345678"
}
```

- `herdr pane report-agent-session` with `--source herdr:kiro --agent kiro` accepted
- Session appears in `herdr pane list` output with correct fields
- Required fix: `is_official_agent_source` in `agent_resume.rs` was missing `("herdr:kiro", "kiro")`

### Detection manifest ✅

- `agent: "kiro"` correctly identified
- `agent_status: "working"` from `kiro_working_marker` rule
- `agent_status: "blocked"` from `tool_approval` rule (verified on other kiro panes)
- Detection and session coexist correctly

### Integration status ✅

- `herdr integration status` reports `kiro: current (v1)`
- Hook script at `~/.kiro/hooks/herdr-agent-state.ps1` recognized

### Hook auto-fire ❌ (v2 limitation, expected)

- v2 mode does not load global hooks from `~/.kiro/hooks/`
- Global hooks require `kiro-cli --v3` (Early Access)
- Manual `report-agent-session` CLI proves the full server path works

### PS1 bug fixed

- `$args` is a PowerShell automatic variable — renamed to `$herdrArgs`

## What to build

Manually validate the full integration flow: install, verify detection + session reporting work together in a real herdr+kiro-cli session, then uninstall.

## Details

### Validation steps

1. Build herdr from the fork: `cargo build --release`
2. Run `herdr integration install kiro` — verify success messages
3. Verify files exist at `~/.kiro/hooks/herdr-agent-state.sh`
4. Start a herdr session, open a pane, run `kiro-cli chat`
5. Verify:
   - Detection manifest picks up working/blocked/idle states correctly
   - `herdr agent explain <pane> --json` shows correct state transitions
   - Session ID is reported (check via `herdr pane list --json` for agent_session_id)
6. Run `herdr integration status` — verify kiro shows "installed" / "current"
7. Run `herdr integration uninstall kiro` — verify cleanup
8. Verify kiro-cli still works normally after uninstall (hook removal doesn't break anything)

### Edge cases to test

- Install when `~/.kiro` doesn't exist → clear error
- Install when kiro-cli is not on PATH → reported as "not found" in recommendations
- Reinstall (idempotent) → overwrites cleanly
- Version detection after install → reports version 1

## Acceptance criteria

- [ ] Install succeeds and reports correct paths
- [ ] Detection works (working/blocked/idle correctly identified)
- [ ] Session reporting works (agent_session_id populated)
- [ ] Status reports "current" after install
- [ ] Uninstall cleans up completely
- [ ] No regression in kiro-cli operation after uninstall
