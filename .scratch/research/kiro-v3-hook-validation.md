---
topic: Kiro CLI v3 hook validation — live results (ticket 014)
date: 2026-10-10
status: complete
confidence: high
harness: disposable herdr named session inside spike worktree workspace w62
---

# Kiro CLI v3 hook validation — live results

Executed per ticket 014 using the `herdr-throwaway-repro` discipline inside the
isolated worktree `spike/kiro-v3-hook-harness` (workspace `w62`). All agent work
ran in a disposable named session; the user's default session was never touched.

## Environment

- herdr `0.9.3` (host, `HERDR_ENV=1`).
- `kiro-cli-chat 2.29.0` (observed in the disposable pane).
- Disposable session: `repro-kiro-v3-20261010-095909`; outer pane `w62:p2`
  (split from spike root `w62:p1`); disposable root pane `w1:p1`.
- Repro dir: `%TEMP%\repro-kiro-v3-20261010-095909` with test config
  `[experimental] allow_nested = true` passed via `HERDR_CONFIG_PATH`.
- Project-level probe hook: `<reproDir>\.kiro\hooks\probe-sessionstart.json`
  invoking `powershell -NoProfile -ExecutionPolicy Bypass -File <abs>\probe-capture.ps1`.
  The installed global hooks (`~/.kiro/hooks/herdr-agent-state.{json,ps1}`) were
  NOT modified (baseline SHA256 recorded and re-verified; see Cleanup).

## Baseline hashes (installed global Kiro hooks — must stay unchanged)

- `herdr-agent-state.json` SHA256
  `6F01254EF97CD5DFE9A33412211C2205D89CECFA76432535846B1BAC0C7A50A8`
- `herdr-agent-state.ps1` SHA256
  `8EF67F1E122ED893F7BD1D26CA786DD71CC1F00642396EE3797EEE98C0C27B73`

## Findings

### 1. Fresh `SessionStart` fires; exact payload captured

Launching `kiro-cli chat --v3 --no-interactive "say ready"` in the repro dir
fired the project-level `SessionStart` hook within ~1s. Raw STDIN captured
verbatim:

```json
{"session_id":"sess_a40b8637-d602-4282-b9f5-ce5ff07c9ae6","hook_event_name":"SessionStart","cwd":"C:\\Users\\uosmi\\AppData\\Local\\Temp\\repro-kiro-v3-20261010-095909"}
```

- **`hook_event_name` = `"SessionStart"` (PascalCase)** — resolves the casing
  open question. The docs example's camelCase `preToolUse` does NOT apply here.
- **`session_id` = `sess_`-prefixed UUID** (`sess_<uuid>`), not a bare UUID.
  Integration code must treat the full `sess_...` string as the identifier and
  must not assume a plain UUID.
- SessionStart payload keys = exactly `session_id`, `hook_event_name`, `cwd`.
  No `tool_name`/`tool_input` (expected for a non-tool event).
- `cwd` = the launch directory (repro dir), confirming **project-level
  `.kiro/hooks/` is loaded** and the hook runs in project root.

### 2. Windows execution + inherited environment

The hook ran via `powershell -NoProfile -ExecutionPolicy Bypass -File ...`
spawned by Kiro (Kiro only spawns the command string; it does not interpret
PowerShell itself — so the installed integration's explicit `powershell ... -File`
invocation is the correct portable pattern). Inside the hook:

- `HERDR_ENV=1`, `HERDR_PANE_ID=w1:p1` were inherited correctly.
- `HERDR_SOCKET_PATH` pointed at the **disposable** session socket
  (`...\sessions\repro-kiro-v3-20261010-095909\herdr.sock`), i.e. the hook would
  report to the correct session, not the parent.
- `TEMP` present; hook `pid`/`parent_pid` captured.

### 3. `SessionStart` does NOT fire on `--resume-id` (2.29.0)

`kiro-cli chat --v3 --resume-id sess_a40b8637-... --no-interactive "say resumed"`
produced a correct model response ("resumed") but **no second hook capture** was
written (verified twice, incl. a trailing wait). Conclusion: **resuming a session
does not re-trigger `SessionStart`** in 2.29.0.

Implication for 015: a SessionStart-only integration will NOT re-report session
identity on resume. If herdr relies on SessionStart to (re)bind a resumed Kiro
pane to its session, resumed panes will be missed. Options: also wire a hook on
an event that fires post-resume (e.g. first `UserPromptSubmit`/`PreToolUse`),
or accept identity is only reported on fresh starts.

### 4. Timeout: child is killed, startup is NOT blocked

Probe hook with `timeout: 2` invoking a script that writes a start marker,
sleeps 8s, then writes an end marker:

- Start marker written; **end marker never written** → Kiro terminated the hook
  child at the timeout.
- The session still reached a usable state ("Hi! What can I help you with
  today?") and returned to the prompt. **A slow/timed-out SessionStart hook does
  not wedge or block the Kiro session** (consistent with SessionStart being
  non-blocking).
- No explicit timeout warning was visible in the `--no-interactive` transcript;
  interactive-TUI warning text was not separately captured (minor gap; not
  blocking for 015 since the herdr hook is fast and guarded).
- `timeout` is honored as an **integer number of seconds** (config used `2`,
  killed at ~2s), matching the installed herdr hook's `"timeout": 10`. The older
  `timeout_ms`/30000 doc form does not apply to the 2.29.0 field used here.

### 5. Installed integration guard works outside herdr

Piping a sample `SessionStart` payload to the installed
`~/.kiro/hooks/herdr-agent-state.ps1` with `HERDR_ENV`/`HERDR_PANE_ID`/
`HERDR_SOCKET_PATH`/`HERDR_SESSION` unset → **exit 0, no output**. Correct
non-intrusive behavior when not running under herdr.

## Answers to 014's open questions

- Raw `SessionStart` STDIN: captured (above). Keys: `session_id`,
  `hook_event_name`, `cwd`. Event name casing: `SessionStart`.
- `session_id` format: `sess_`-prefixed UUID.
- Windows interpreter: Kiro spawns the opaque command string; explicit
  `powershell -NoProfile -ExecutionPolicy Bypass -File` works.
- Inherited env: `HERDR_ENV`, `HERDR_PANE_ID`, `HERDR_SOCKET_PATH` all inherited
  and point at the owning (disposable) session.
- Resume: `SessionStart` does NOT fire on `--resume-id`; session responds
  normally.
- Timeout: integer seconds; child killed at timeout; startup not blocked.
- Hook loading: project-level `.kiro/hooks/*.json` loaded (global not re-tested
  this run; the installed global hook is known to load from prior installs).

## Carry-forward for ticket 015

1. The installed `herdr:kiro` integration is **SessionStart-only / identity-only**
   → currently belongs in `session_identity_only_integration`, not
   `full_lifecycle_hook_authority`. To get authoritative working/blocked/idle,
   015 must add `UserPromptSubmit`/`PreToolUse`/`PostToolUse`/`Stop` command
   hooks (mirroring `KIMI_HOOK_EVENTS`) and move Kiro to full lifecycle authority.
2. Use the full `sess_...` string as the session id; do not assume bare UUID.
3. Do not rely on `SessionStart` for resumed sessions — add a post-resume signal
   if resume binding matters.
4. `timeout` is seconds; keep the herdr hook fast and guarded (it already is).

## Cleanup

Performed after capture (see session report). Only this run's disposable
session, outer pane, probe hooks/scripts, and repro dir were removed. Installed
global hook hashes re-verified against the baseline above.
