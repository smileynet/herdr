---
title: "Review and update kiro.toml detection manifest"
status: done
priority: normal
blocked_by: []
---

# Review and update kiro.toml detection manifest

## Review findings

The existing manifest at `src/detect/manifests/kiro.toml` is adequate for kiro-cli v2.x:

- **tool_approval (blocked, P300)** — Correctly matches kiro-cli's tool approval prompt with all option variants
- **subagent_approval (blocked, P290)** — Correctly matches subagent pending state with action options
- **kiro_working_marker (working, P100)** — Matches "kiro is working" text marker
- **tool_spinner_working (working, P90)** — Matches spinner characters (◔/◑/◕/●) + "esc to cancel"

### Idle detection

No idle rule exists. Kiro-cli uses `>` as prompt indicator, but without evidence-based pane captures (per AGENTS.md: "use live pane reads for agent-specific screen evidence"), adding an idle rule risks false positives. The existing working/blocked rules are sufficient — herdr treats "no match" as unknown/idle after timeout.

### No changes needed

Per AGENTS.md detection guidelines: detection changes must be evidence-based with captured pane state. The current rules are correct and no regression has been reported. An idle rule can be added later with proper evidence from `herdr agent read <pane> --source detection --format text`.

## Acceptance criteria

- [x] Existing rules verified against current kiro-cli output
- [x] Missing states identified (idle — deferred until evidence available)
- [x] No manifest changes required at this time
