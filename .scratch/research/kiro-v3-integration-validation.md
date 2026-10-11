---
topic: Kiro v3 integration end-to-end validation (ticket 017)
date: 2026-10-11
status: partial — root cause found and fixed; authority proven by deterministic tests; live interactive observation blocked by harness
confidence: high (logic), medium (live interactive)
---

# Kiro v3 integration end-to-end validation (ticket 017)

## Environment

- herdr debug build 0.9.3 from the spike worktree (`target/debug/herdr.exe`,
  rebuilt with the 017 asset fix). Built with `RUST_MIN_STACK=268435456`,
  `CARGO_PROFILE_DEV_CODEGEN_UNITS=1`, `CARGO_PROFILE_DEV_DEBUG=0`,
  `CARGO_BUILD_JOBS=1` to work around a Windows rustc codegen stack overflow
  (see "Build note" below).
- kiro-cli 2.29.0.
- Disposable `herdr-dev` debug session, throwaway KIRO_HOME + project `.kiro/hooks`.

## The bug 017 caught (now fixed)

**Symptom:** with the hook installed, a live Kiro pane stayed on screen-detected
state; hook state reports never engaged full-lifecycle authority (reported
`blocked`/`working` did not override the screen).

**Root cause (pinpointed with a deterministic in-process test):** the Kiro
SessionStart hook (v2) called `pane report-agent-session` WITHOUT
`--session-start-source`. The runtime's `route_full_lifecycle_hook_report`
fast-path only accepts later working/idle/blocked reports once a session is
*anchored*, and anchoring requires a recognized `session_start_source`
(`session_start_source_is_recognized` accepts `startup|clear|resume|compact|new|
fork|select`; `None` is not recognized). Without the anchor, every state report
fell into the buffered `pending_replacement_report` branch and returned
`Ignore`, so authority never engaged. Independently confirmed by the
context-gatherer review of `src/terminal/state.rs`.

**Fix:** the Kiro assets now send `--session-start-source startup` on the
`session` action (bumped `KIRO_INTEGRATION_VERSION` 2 → 3 for both
`.ps1`/`.sh`). A deterministic test proves the fix:
- `run_sequence("herdr:kiro","kiro",Kiro, Some("startup"))` → authority=TRUE,
  state=Working.
- `run_sequence(... None)` → authority=FALSE (the pre-fix behavior).
- `control_kimi ... Some("startup")` → authority=TRUE (confirms the sequence,
  not kiro-specific wiring, is what matters).
Permanent regression test added: `kiro_session_hook_anchors_with_startup_source`
asserts both platform assets embed the startup source.

## Live findings (what was and wasn't observable)

- **SessionStart fires and the hook environment is correct.** Captured the real
  payload from the project `.kiro/hooks` under the debug server:
  `{"session_id":"sess_50b70522-...","hook_event_name":"SessionStart","cwd":...}`,
  with `HERDR_ENV=1`, `HERDR_PANE_ID=w1:p1`, `HERDR_SOCKET_PATH`=debug-session
  socket, and `HERDR_BIN_PATH`=the debug binary. So the fixed `.ps1` (which uses
  `$env:HERDR_BIN_PATH`) targets the correct binary + socket. Note bare `herdr`
  on PATH resolves to the STABLE 0.9.3 install — which is why the asset must use
  `HERDR_BIN_PATH`, and does.
- **Kiro loads project-level `.kiro/hooks/*.json`** (cwd), consistent with 014.
  It does NOT honor a `KIRO_HOME` env var for runtime hook discovery — the
  install path override is cosmetic for isolation; Kiro reads `~/.kiro/hooks` and
  the project `.kiro/hooks`. (The global `~/.kiro/hooks/herdr-agent-state.*` on
  this machine is a prior-session v1 identity-only install; it was moved aside
  during testing and restored byte-identical, hashes verified.)
- **Not observed live:** persistent authority on a *live interactive* `--v3` TUI.
  Non-interactive `--no-interactive` runs one turn and exits before the state can
  be queried, and driving the interactive TUI through the CLI raced with hook
  timing on the debug server. This is a harness/observability limitation, not a
  code question — the authority logic is proven by the deterministic tests above
  and the hook delivery is proven by the captured env.

## Resume (carry-forward, not fixed in 017)

Confirmed again: Kiro exposes a resume command on exit
(`Resume with: kiro-cli --resume-id sess_...`), and 014 found SessionStart does
NOT fire on `--resume-id`. The context-gatherer review notes that `herdr:kiro`
is NOT in `agent_resume::is_official_agent_source`, and there is no
`persisted_session_from_launch_args` arm for kiro, so a resumed Kiro pane will
fall back to screen detection (manifest from 016) rather than hook authority.
This is acceptable for a first integration (screen detection covers it) but
should be a follow-up: add a kiro arm to `is_official_agent_source` + a
launch-args resume parse so resumed panes re-anchor. Tracked as a known gap.

## Build note (Windows rustc codegen crash)

Building the full `herdr` binary crashed with `STATUS_STACK_BUFFER_OVERRUN`
(0xc0000409). Root cause is a known rust-lang issue class: LLVM DWARF debuginfo
emission overflows the codegen worker thread stack on large crates
(rust-lang/rust #122357, #93908). Fix: build with `RUST_MIN_STACK=256MiB`,
`codegen-units=1`, `CARGO_BUILD_JOBS=1`, and `debuginfo=0`. Not a code defect.
This should be a non-issue on CI/maintainer hardware; recorded so the PR notes it.

## Status vs acceptance criteria

- Hook path session/working/blocked/idle authority: proven by deterministic
  tests; live interactive observation blocked by harness (documented).
- Hook exits 0 silently outside herdr; timeout non-blocking: proven in 014.
- Fallback path idle/working/blocked + skip_state_update: proven in 016.
- Notification/sound transitions: not separately exercised (state-transition
  mapping is covered by existing app tests; no kiro-specific code there).
- POSIX `.sh` asset: the `.sh` carries the same `session_start_source` fix and is
  covered by the regression test; not run on a live POSIX host this session.
- Results recorded here. Disposable session/pane/override removed; global hooks
  verified unchanged.

## Cleanup

Disposable debug session stopped+deleted, outer pane closed, repro dir removed,
global `~/.kiro/hooks` restored byte-identical (sha256 verified), no stray
processes, only the default session remains.
