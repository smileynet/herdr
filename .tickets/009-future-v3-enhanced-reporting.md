---
title: "[Future] Enhanced state reporting via v3 lifecycle hooks"
status: open
priority: low
blocked_by: ["008"]
---

# [Future] Enhanced state reporting via v3 lifecycle hooks

## What to build

When v3 is stable, evaluate whether hook-based state reporting (via multiple lifecycle events) provides better accuracy than screen detection alone, and if so, implement richer reporting.

## Details

### v3 lifecycle events available

| Event | Potential herdr state |
|-------|----------------------|
| `SessionStart` | session (identity report) |
| `UserPromptSubmit` | working |
| `PreToolUse` | working |
| `PostToolUse` | working |
| `Stop` | idle |
| `PreTaskExec` | working |
| `PostTaskExec` | idle or working |

### Considerations

- Screen detection is already reliable for kiro-cli (proven manifest exists)
- Hook-based reporting adds a parallel signal source — herdr can reconcile both
- Hook-based is more authoritative (comes from the agent itself) but has latency
- Screen detection is instant but can have false positives on similar-looking content
- The `stop` hook's `assistant_response` field could be used for transcript-like functionality

### Decision criteria for implementation

1. Does screen detection have false positives/negatives with kiro-cli? (Test first)
2. Does hook-based reporting reduce flicker in state transitions?
3. Is the v3 API stable enough to rely on?

## Acceptance criteria

- [ ] Evaluated whether hook reporting improves over screen detection alone
- [ ] If yes: implemented multi-event hooks with appropriate state mappings
- [ ] If no: documented why screen-only is sufficient, closed ticket
