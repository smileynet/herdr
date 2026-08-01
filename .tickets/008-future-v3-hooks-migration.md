---
title: "[Future] Validate v3 GA and remove Early Access caveats"
status: open
priority: low
blocked_by: ["007"]
---

# [Future] Validate v3 GA and remove Early Access caveats

## Context

The kiro integration already targets v3 hooks (standalone `~/.kiro/hooks/herdr-agent-state.json`). However, v3 is currently "Early Access" (since 2.8.0, June 2026) and requires `kiro-cli --v3` flag.

## What to do when v3 reaches GA

1. Remove the "session reporting requires kiro-cli v3 mode" install message
2. Verify hook auto-loading works without `--v3` flag
3. Test with the GA version for any format changes
4. Consider adding a minimum kiro-cli version check (`AgentVersionRequirement`)
5. Update docs to remove Early Access caveats

## Known v3 risks to re-check at GA

- GitHub #9490: Custom agents not discovered in v3 (shouldn't affect global hooks)
- Hook schema labeled "v1" — confirm it didn't change
- Session format incompatibility between v2↔v3 (user awareness only)

## Acceptance criteria

- [ ] v3 is GA (no `--v3` flag needed)
- [ ] Hook fires without `--v3` flag
- [ ] Install message updated to remove v3 caveat
- [ ] E2E validation passes on GA version

## Details

### v3 format

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "herdr-agent-state",
      "trigger": "SessionStart",
      "action": { "type": "command", "command": "~/.kiro/hooks/herdr-agent-state.sh" },
      "timeout": 10,
      "enabled": true
    }
  ]
}
```

### Changes needed

1. Install a `.kiro/hooks/herdr-agent-state.json` alongside the script
2. Use `SessionStart` trigger (v3 name) instead of `agentSpawn` (v2 name)
3. Consider using additional v3 triggers for richer state reporting:
   - `UserPromptSubmit` → working
   - `Stop` → idle
   - `PreToolUse` → working
   This would reduce reliance on screen detection
4. Update minimum version requirement
5. Potentially remove some detection manifest rules if hook-based reporting is more reliable

### Migration path

- Detect kiro-cli version at install time
- If v3+: use standalone hooks format
- If v2: use current global hooks approach
- Or: install both (v3 file + legacy script) for cross-version support

## Acceptance criteria

- [ ] v3 hook format used when kiro-cli reports v3+ version
- [ ] Backward compatibility with v2 maintained or clearly versioned
- [ ] Session reporting works with v3 lifecycle events
- [ ] Tests cover both v2 and v3 code paths
