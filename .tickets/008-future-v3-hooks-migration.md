---
title: "[Future] Migrate to kiro-cli v3 standalone hooks format"
status: open
priority: low
blocked_by: ["007"]
---

# [Future] Migrate to kiro-cli v3 standalone hooks format

## What to build

When kiro-cli v3 is fully released and stable, update the integration to use the standalone `.kiro/hooks/<name>.json` file format instead of relying on the global hooks directory or agent config embedding.

## Why deferred

v3 is currently in "Early Access" (since v2.8.0, June 2026) and requires `--v3` flag. The format may change before GA. We'll implement once it's finalized.

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
