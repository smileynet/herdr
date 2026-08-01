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
| 007 | End-to-end validation | open | 004, 005, 006 |

## Phase 2: v3 compat (deferred until v3 GA)

| # | Ticket | Status | Depends on |
|---|--------|--------|------------|
| 008 | Migrate to v3 standalone hooks format | open | 007 |
| 009 | Enhanced state reporting via v3 lifecycle | open | 008 |

## Key decisions

- **Session-only hook**: Only fire on `agentSpawn` (v2) / `SessionStart` (v3). Screen detection handles working/blocked/idle.
- **No transcript_path**: Kiro-cli uses SQLite, not file-based transcripts. We report `session_id` only.
- **Target ≥ 2.13**: Uses global hooks directory for clean installation (no agent config modification needed).
- **v3 deferred**: v3 hooks are "Early Access" and may change. We'll implement when GA.
- **Binary name**: `kiro-cli` (the `kiro` command is an optional router, not guaranteed available).
- **Platform support**: Both Windows and Unix (kiro-cli supports both).

## Research findings

- kiro-cli hooks receive JSON on stdin with `hook_event_name`, `cwd`, `session_id`
- `session_id` is a UUID
- `KIRO_HOME` env var overrides `~/.kiro` location
- Hooks introduced in v2.0.0, global hooks dir in v2.13.0
- No `transcript_path` in payload (SQLite-backed sessions)
