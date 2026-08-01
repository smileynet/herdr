---
title: "v2 functionality review: maximize herdr+kiro integration without hooks"
status: open
priority: high
blocked_by: []
---

# v2 functionality review: maximize herdr+kiro integration without hooks

## Goal

Ensure herdr provides maximum value for kiro-cli v2 users (who can't use session hooks). Review what currently works, identify gaps, and implement improvements that don't require hooks.

## What works today (v2, no hooks)

The detection manifest (`kiro.toml`) provides:
- **Working detection**: "kiro is working" marker + spinner (◔/◑/◕/●)
- **Blocked detection**: Tool approval prompts, subagent approval prompts
- **Agent identification**: herdr correctly identifies the pane as running kiro

## What's missing in v2

1. **Idle detection** — No rule identifies when kiro is waiting for user input. herdr falls back to "unknown" after working/blocked clear.
2. **Session identity** — No `agent_session` without hooks. Herdr can't track which session is running.
3. **Agent resume awareness** — herdr doesn't know if a pane has a resumable kiro session after detach/reattach.

## Review tasks

### A. Idle detection improvement

Research kiro-cli's idle prompt indicator and add an idle rule to `kiro.toml`:
- Capture pane state with `herdr agent read <pane> --source detection --format text` when kiro is idle
- Identify invariant prompt elements (e.g., `>` prefix, status bar text)
- Add a rule with appropriate priority and region

### B. Additional blocked states

Check if kiro-cli has other blocking prompts beyond tool approval:
- MCP server connection prompts
- Permission escalation dialogs
- Rate limit / quota warnings that need user action
- Subagent delegation confirmations

### C. Working state completeness

Verify we catch all working indicators:
- Are there other spinners or progress text beyond the ones in the manifest?
- Does kiro-cli show different UI during file writes, web searches, or long tool calls?
- Subagent execution — does "pending from subagents" cover the entire subagent lifecycle?

### D. OSC title/progress sequences

Check if kiro-cli emits OSC terminal sequences (like Claude does):
- OSC title with spinner → working
- OSC title with idle marker → idle
- OSC progress sequences
- These are high-priority signals when present (see claude.toml for pattern)

### E. Integration command detection

Verify `herdr agent explain <pane> --json` correctly reports:
- Agent name as "kiro"
- Integration available/not-installed status
- Correct state transitions without flicker

## Acceptance criteria

- [ ] Idle detection rule added (if evidence supports it)
- [ ] Additional blocked states documented or added
- [ ] Working state coverage verified complete
- [ ] OSC sequences checked (add rules if present)
- [ ] No false positives introduced (test with real kiro-cli session)
- [ ] Manifest version bumped if rules changed
- [ ] `agent_detection_manifest_check.py` passes
