---
id: "015"
title: "Add Kiro CLI v3 hook integration module (full lifecycle authority)"
status: done
priority: high
blocked_by: ["014"]
---

# Add Kiro CLI v3 hook integration module (full lifecycle authority)

## Goal

Give the Kiro CLI the same hook-based lifecycle integration that claude, codex,
and kimi already have, so Kiro reports authoritative `working` / `blocked` /
`idle` state, session identity, and resume data to Herdr through its socket —
instead of relying only on screen scraping. After this change Kiro should appear
in `full_lifecycle_hook_authority`, and screen detection should be paused for a
Kiro pane while the hook holds authority.

This is a product-behavior change (new integration surface). Per
`CONTRIBUTING.md`, align with maintainers before implementing; this ticket
captures the implementation plan so the work is grounded, not a license to open
an unsolicited PR.

## Why (root cause from investigation)

Kiro is screen/OSC-only today. Confirmed facts:

- `src/detect/mod.rs` — `Agent::Kiro` exists (label `"kiro"`, executable
  `kiro-cli`, aliases `kiro`/`kiro-cli`). `full_lifecycle_hook_authority()`
  lists `pi, omp, mastracode, opencode, kilo, kimi`. **Kiro is absent**, so it
  can never report authoritative state. `session_identity_only_integration()`
  lists `hermes, qwen, letta, antigravity_cli`. Kiro is absent there too.
- `src/integration/mod.rs` — no `KIRO_*` constants, no asset. Every integrated
  agent has `*_HOOK_INSTALL_NAME`, `*_HOOK_ASSET`, `*_INTEGRATION_VERSION`.
- `src/integration/registry.rs` — `integration_specs()` returns a fixed-size
  array (currently `; 17]`) of `(IntegrationTarget, path, version)`. Kiro has no
  entry. `IntegrationTarget` has no `Kiro` variant.
- Kiro v3 ships a standalone hook system (`.kiro/hooks/<id>.json`, triggers
  `SessionStart`/`UserPromptSubmit`/`PreToolUse`/`PostToolUse`/`Stop`), which is
  exactly the mechanism Herdr already consumes for kimi/droid/etc. Prior-art
  research: `.scratch/research/kiro-v3-windows-hook-execution.md`.

## Reference / prior art (read these first)

> UPDATE from 014 (live-validated against kiro-cli 2.29.0, herdr 0.9.3 —
> `.scratch/research/kiro-v3-hook-validation.md`):
> - An installed `herdr:kiro` integration ALREADY exists
>   (`~/.kiro/hooks/herdr-agent-state.{json,ps1}`, `HERDR_INTEGRATION_VERSION=1`),
>   but it is **SessionStart-only / identity-only** (reports
>   `pane report-agent-session` from `session_id`). So Kiro currently belongs in
>   `session_identity_only_integration`, NOT `full_lifecycle_hook_authority`.
>   This ticket's real job is to **extend** it to full lifecycle, or decide to
>   keep identity-only and rely on manifest 016 for state. Reconcile with the
>   installed asset; the committed asset must supersede it via version bump.
> - `SessionStart` STDIN keys = `session_id`, `hook_event_name` (value
>   `"SessionStart"`, PascalCase), `cwd`. `session_id` is a **`sess_`-prefixed
>   UUID** — treat the whole string as the id; do not assume bare UUID.
> - **`SessionStart` does NOT fire on `--resume-id`.** A SessionStart-only hook
>   will miss resumed panes. For full lifecycle, add a post-resume signal
>   (first `UserPromptSubmit`/`PreToolUse`) if resume binding matters.
> - `timeout` is integer **seconds**; a slow hook child is killed at timeout and
>   startup is not blocked. Keep the hook fast and guarded.
> - Kiro spawns the opaque Windows command string; the installed hook's explicit
>   `powershell -NoProfile -ExecutionPolicy Bypass -File ...` is the correct
>   portable pattern.


- **Spike results**: ticket `014` and `.scratch/research/kiro-v3-hook-validation.md`
  (produced by 014). Do NOT start 015 until 014 has captured the real
  `SessionStart` payload, the Windows interpreter behavior, resume behavior, and
  timeout behavior. The hook event→state mapping below is provisional until 014
  confirms which Kiro v3 events fire and their exact names/casing.
- **Closest analog = kimi** (full lifecycle authority, maps events to
  working/blocked/idle):
  - Asset: `src/integration/assets/kimi/herdr-agent-state.sh` and
    `herdr-agent-state.ps1`. Copy this structure. Note: posts
    `pane.report_agent` with `--state`, and `pane.report_agent_session` for
    session identity, over the socket at `$HERDR_SOCKET_PATH`. Guards on
    `HERDR_ENV=1`, `HERDR_PANE_ID`, `HERDR_SOCKET_PATH` and exits 0 silently when
    unset (critical: a hook must never block the agent or print into agent
    context).
  - Event→state map: `KIMI_HOOK_EVENTS` in `src/integration/mod.rs`
    (`UserPromptSubmit`→working, `PreToolUse`→working,
    `PermissionRequest`→blocked, `Stop`→idle, etc.).
- **droid asset** `src/integration/assets/droid/herdr-agent-state.sh` shows the
  mktemp-stdin + python3 heredoc variant and `pane.report_agent_session` JSON
  shape (id/method/params, `seq = time.time_ns()`).
- **Registry wiring**: `src/integration/registry.rs` —
  `integration_target_label`, `integration_target_command_names`
  (`&["kiro-cli", "kiro"]` likely), `integration_target_supported` (add Kiro to
  the Windows allow-list match since Kiro runs on Windows), the
  `integration_specs()` array (bump `; 17]` → `; 18]` and add the Kiro tuple),
  and `parse_integration_version` (reads `HERDR_INTEGRATION_VERSION=` marker).
- **Authority wiring**: `src/detect/mod.rs` `full_lifecycle_hook_authority()` —
  add `("herdr:kiro", "kiro")`. Check
  `src/app/api.rs::sync_full_lifecycle_authority_detection_pauses` and
  `runtime.set_full_lifecycle_authority_active` so Kiro panes pause screen
  detection while the hook holds authority.
- **Events consumed**: `src/events.rs` — `HookStateReported`,
  `AgentSessionReported`, `HookAgentReleased`. The asset's socket methods map to
  these.

## Contract / guardrail notes (do not skip)

- `IntegrationTarget` is part of the **frozen client endpoint enum**. Per
  `AGENTS.md` ("Stable client endpoint contract"): named core codecs are
  immutable; enums reachable from a published codec are append-closed. Adding a
  `Kiro` variant must be **appended** (never reorder/reinsert), and older clients
  must treat an unknown target safely. Check whether `IntegrationTarget` is
  reachable from a published codec; if so, confirm the append is handled like
  prior additions (compare how `Grok`/`Mastracode`/`AntigravityCli` were added)
  and whether an `Unknown` fallback already exists. Note: `registry.rs` documents
  that `Letta` was deliberately kept OUT of the frozen enum and surfaced as an
  experimental CLI-only target. Decide with maintainers whether Kiro follows the
  enum path (like grok) or the experimental path (like letta).
- `HERDR_INTEGRATION_VERSION` is a migration version relative to the latest
  released tag, not a per-commit counter (`AGENTS.md` Code Conventions). A new
  integration starts at `1`.
- No `unwrap()` in production Rust. Platform-specific code compile-gated
  (`#[cfg(windows)]` / `#[cfg(unix)]`); the `.ps1`/`.sh` asset selection already
  follows the `if cfg!(windows)` include_str! pattern used by every peer.
- Do not add dependencies.

## What to do

1. Create `src/integration/assets/kiro/herdr-agent-state.sh` and
   `herdr-agent-state.ps1`, modeled on the kimi assets, with headers
   `# HERDR_INTEGRATION_ID=kiro` and `# HERDR_INTEGRATION_VERSION=1`. Source
   label `herdr:kiro`, agent `kiro`. Map the Kiro v3 events confirmed by 014 to
   `session` / `working` / `blocked` / `idle` actions.
2. Add constants in `src/integration/mod.rs`: `KIRO_HOOK_INSTALL_NAME`,
   `KIRO_HOOK_ASSET` (cfg-gated include_str!), `KIRO_INTEGRATION_VERSION = 1`,
   and a `KIRO_HOOK_EVENTS` map mirroring `KIMI_HOOK_EVENTS` with the
   014-confirmed event names. Kiro v3 hooks are standalone JSON files under
   `~/.kiro/hooks/` and `.kiro/hooks/`; follow how the install writer emits the
   per-agent hook config (compare kimi/droid install paths and any JSON hook
   config writer in `src/integration/targets.rs`/`config_file.rs`).
3. Wire `src/integration/registry.rs`: add `IntegrationTarget::Kiro` (appended),
   `integration_target_label` → `"kiro"`, `integration_target_command_names` →
   `&["kiro-cli", "kiro"]`, add to `integration_target_supported` Windows match,
   add the Kiro tuple to `integration_specs()` and bump the array length.
4. Add `("herdr:kiro", "kiro")` to `full_lifecycle_hook_authority()` in
   `src/detect/mod.rs`. Verify detection pausing wiring in `src/app/api.rs`.
5. Decide the Kiro hooks install directory (global `~/.kiro/hooks/` vs project
   `.kiro/hooks/`) based on 014's precedence finding. Never overwrite a
   pre-existing user hook of a different name; the install must be idempotent and
   versioned like peers.

## Testing (per AGENTS.md "Agent Detection Updates" + "Testing")

- Unit-test the **engine/integration plumbing**, not Kiro's screen conventions:
  version-marker parsing, event→state mapping, install/upgrade/outdated status,
  authority classification. Mirror existing tests in `src/integration/tests.rs`
  and detection authority tests.
- Live-validate through the manifest/integration hot-reload loop and the
  disposable-session harness in
  `.scratch/research/herdr-windows-throwaway-commands.md` and the
  `herdr-throwaway-repro` skill. Record Kiro CLI version, observed signals,
  state transitions (idle/working/blocked), and resume behavior.
- Run `just ci` (contributor gate) before any PR; maintainers run `just check`
  (includes Windows cross-compile). Do not bypass failing checks.
- When testing a debug build from inside Herdr, clear inherited socket overrides:
  `env -u HERDR_SOCKET_PATH -u HERDR_CLIENT_SOCKET_PATH cargo run -- <command>`.

## Acceptance criteria

- [x] 014 completed; `SessionStart`/event payloads and Windows behavior confirmed
- [x] `src/integration/assets/kiro/herdr-agent-state.{sh,ps1}` added, version 1, guarded, silent-exit when Herdr vars unset
- [x] `KIRO_*` constants + `KIRO_HOOK_EVENTS` added in `src/integration/mod.rs`
- [x] `IntegrationTarget::Kiro` appended (or experimental-letta-style path chosen) without reordering the frozen enum; contract reviewed
- [x] `registry.rs` label/command/supported/specs wired; array length bumped
- [x] `("herdr:kiro","kiro")` added to `full_lifecycle_hook_authority`; detection pausing verified
- [x] Integration install is idempotent, versioned, and never overwrites unrelated user hooks
- [x] Engine/plumbing unit tests added; `just ci` passes
- [x] Live validation recorded (CLI version, signals, transitions, resume)

## Resolution

Pending implementation. Blocked by 014.
