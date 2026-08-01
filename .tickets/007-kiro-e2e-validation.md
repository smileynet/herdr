---
title: "End-to-end validation of kiro integration"
status: open
priority: normal
blocked_by: ["004", "005", "006"]
---

# End-to-end validation of kiro integration

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
