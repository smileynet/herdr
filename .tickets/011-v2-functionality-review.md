---
title: "v2 functionality review: maximize herdr+kiro integration without hooks"
status: done
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

- [x] Idle detection rule added (if evidence supports it)
- [x] Additional blocked states documented or added
- [x] Working state coverage verified complete
- [x] OSC sequences checked (add rules if present)
- [x] No false positives introduced (test with real kiro-cli session)
- [x] Manifest version bumped if rules changed
- [x] `agent_detection_manifest_check.py` passes

## Resolution

Validated on 2026-08-01 with kiro-cli 2.16.0 v2 in a disposable Herdr session.

- Added `prompt_idle`, scoped to the bottom three non-empty lines and requiring both the prompt text and copy affordance. Live hot reload matched it only while Kiro was waiting for input and set `visible_idle = true`.
- Existing working detection matched normal thinking and tool preparation through the `Kiro is working` footer. The idle rule did not match while working.
- Existing tool approval detection matched the v2 `shell requires approval` prompt. The idle rule did not match, and the blocker retained priority.
- Repository and live review found no evidence for additional MCP, quota, or subagent-execution rules. Subagent approval remains covered; ordinary subagent lifecycle behavior needs a future captured reproduction before adding broader rules.
- The terminal title stayed `Windows PowerShell` through idle, working, and blocked states, and text/ANSI captures exposed no Kiro-specific OSC signal. No OSC rules were added without evidence.
- Version `2026.08.01.1` was hot-reloaded as a local override, then verified with `agent explain` against real idle, working, and blocked screens.
- `agent_detection_manifest_check.py` passed against the Kiro bundled/published pair. The repository-wide invocation remains blocked by unrelated pre-existing Grok publication drift (`2026.07.16.1` published vs `2026.07.16.2` bundled).
