# Herdr Fork — Kiro-CLI Integration Plan

## Goal

Add a first-class kiro-cli integration to herdr, enabling session identity reporting via hooks. This enables herdr to track kiro-cli sessions across pane lifecycle (detach/reattach, session restore, etc.).

## Architecture

Two systems work together:
1. **Detection manifest** (already exists) — screen-based pattern matching for working/blocked/idle
2. **Integration hooks** (to build) — reports session ID to herdr server on session start

## Phase 1: v2-compatible integration (target: kiro-cli ≥ 2.13)

Uses kiro-cli's global hooks directory (`~/.kiro/hooks/`) which auto-fires on all agents.

| # | Ticket | Status | Depends on |
|---|--------|--------|------------|
| 001 | Add Kiro to IntegrationTarget enum | done | — |
| 002 | Create hook scripts (sh + ps1) | done | — |
| 003 | Wire directory resolution + registry | done | 001 |
| 004 | Implement install/uninstall | done | 002, 003 |
| 005 | Add tests | done | 004 |
| 006 | Review/update detection manifest | done | — |
| 007 | End-to-end validation | done | 004, 005, 006 |
| 010 | E2E: kiro-cli --v3 hook auto-fire | open | — |
| 011 | v2 functionality review (maximize without hooks) | open | — |

## Current v2 functionality (kiro-cli without --v3)

In v2 mode, herdr provides **screen-based detection only**:

| Feature | Works in v2 | How |
|---------|-------------|-----|
| Agent identification | ✅ | Detection manifest matches "kiro" from process/screen |
| Working state | ✅ | "kiro is working" marker + spinner (◔/◑/◕/●) |
| Blocked state | ✅ | Tool approval prompt, subagent approval prompt |
| Idle state | ⚠️ Partial | Falls back to "unknown" — no positive idle rule yet |
| Session ID reporting | ❌ | Requires v3 global hooks (hook can't fire in v2) |
| Integration status | ✅ | `herdr integration status` shows `kiro: current (v1)` |

**Why session reporting doesn't work in v2:** Global hooks (`~/.kiro/hooks/*.json`) are a v3-only feature. In v2, hooks must be embedded in agent config files, but shadowing `kiro_default` (the built-in agent) breaks tool dispatch entirely. There is no safe v2 hook injection path.

## v3 status (kiro-cli --v3)

**Not yet tested end-to-end.** The infrastructure is in place:
- Hook script + JSON config installed at `~/.kiro/hooks/`
- Server accepts and stores `herdr:kiro` session reports (proven via manual CLI call)
- Remaining: confirm kiro-cli v3 actually discovers and fires the hook on SessionStart

## Phase 2: v3 GA validation (deferred until v3 leaves Early Access)

| # | Ticket | Status | Depends on |
|---|--------|--------|------------|
| 008 | Validate v3 GA and remove caveats | open | 007 |
| 009 | Enhanced state reporting via v3 lifecycle | open | 008 |

## Key decisions

- **Session-only hook**: Only fire on `SessionStart` (v3). Screen detection handles working/blocked/idle.
- **v3 global hooks required**: Global hooks (`~/.kiro/hooks/*.json`) are v3-only. v2 has no safe hook injection path (shadowing `kiro_default` breaks tool dispatch).
- **No transcript_path**: Kiro-cli uses SQLite, not file-based transcripts. We report `session_id` only.
- **Install outputs v3 caveat**: "session reporting requires kiro-cli v3 mode (kiro-cli --v3)"
- **Detection works in v2**: The `kiro.toml` manifest handles working/blocked/idle via screen patterns regardless of hooks.
- **Binary name**: `kiro-cli` (the `kiro` command is an optional router, not guaranteed available).
- **Platform support**: Both Windows and Unix (kiro-cli supports both).

## Lessons learned

- **Do NOT shadow built-in agent names**: Creating `~/.kiro/agents/kiro_default.json` breaks all tool dispatch. See `~/.kiro/steering/agent-configuration-safety.md`.
- **v2 global hooks don't exist**: The 2.13 changelog says "Available in V3 (`kiro-cli --v3`)" — global hooks are v3-only despite being introduced in the 2.13 release.
- **Zig cross-drive path bug**: On Windows, set `ZIG_GLOBAL_CACHE_DIR` to the same drive as the project to avoid zig 0.15.x assertion failure.

## Research findings

- kiro-cli hooks receive JSON on stdin with `hook_event_name`, `cwd`, `session_id`
- `session_id` is a UUID
- `KIRO_HOME` env var overrides `~/.kiro` location
- Hooks introduced in v2.0.0, global hooks dir in v2.13.0
- No `transcript_path` in payload (SQLite-backed sessions)
