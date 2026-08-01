---
title: "E2E validation: kiro-cli --v3 session reporting in herdr"
status: open
priority: high
blocked_by: []
---

# E2E validation: kiro-cli --v3 session reporting in herdr

## Status

**Not yet tested.** The server-side path is fully proven (manual `report-agent-session` works). What remains is confirming that kiro-cli v3 actually discovers and fires the hook from `~/.kiro/hooks/herdr-agent-state.json`.

## What's proven

- Server accepts `herdr:kiro` session reports ✅
- Hook script (`herdr-agent-state.ps1`) invokes CLI correctly ✅
- Hook JSON config (`herdr-agent-state.json`) uses correct v3 format ✅
- PS1 `$args` bug fixed (`$herdrArgs`) ✅

## What needs testing

Start `kiro-cli --v3` in a herdr pane and verify the hook fires automatically on SessionStart without manual invocation.

## What to validate

Confirm the full integration path: herdr installs the hook, kiro-cli v3 discovers and fires it on SessionStart, herdr receives and displays the agent session ID.

## Test procedure

### Setup

1. Ensure `~/.kiro/hooks/herdr-agent-state.json` and `herdr-agent-state.ps1` (or `.sh`) are installed
2. Verify: `cat ~/.kiro/hooks/herdr-agent-state.json`

### Test 1: Hook fires on v3 session start

1. In a herdr pane, run: `kiro-cli --v3`
2. Send any prompt (e.g., "hello")
3. Check: `herdr pane list` — the kiro pane should have `agent_session` with:
   - `agent`: `"kiro"`
   - `source`: `"herdr:kiro"`
   - `kind`: `"id"`
   - `value`: a UUID matching the kiro-cli session ID

### Test 2: Detection + session coexist

1. While kiro is working: `agent_status` = `"working"` + `agent_session` present
2. While kiro is blocked (tool approval): `agent_status` = `"blocked"` + `agent_session` persists

### Test 3: Session ID stable across state transitions

1. Note the session ID from test 1
2. Send multiple prompts, trigger working/idle cycles
3. Verify `agent_session.value` remains the same UUID

### Test 4: New session gets new ID

1. Exit and start fresh `kiro-cli --v3` in same pane
2. Verify `agent_session.value` is a different UUID

### Test 5: Uninstall is clean

1. Remove `~/.kiro/hooks/herdr-agent-state.*`
2. Start `kiro-cli --v3` — no errors, no session reported
3. Detection still works (working/blocked/idle from screen)

## Acceptance criteria

- [ ] Hook fires on SessionStart in v3 mode
- [ ] `agent_session` appears in herdr pane list
- [ ] Session ID is valid UUID matching kiro-cli's `/session-id`
- [ ] Detection manifest unaffected by hook presence
- [ ] Uninstall clean — no kiro-cli errors after hook removal
