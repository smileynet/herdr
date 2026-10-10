---
id: "014"
title: "Spike: Investigate and test kiro-cli v3 hook harness for herdr integration"
status: done
priority: high
blocked_by: []
---

# Spike: Investigate and test kiro-cli v3 hook harness for herdr integration

## Goal

Confirm the installed kiro-cli v3 global hook mechanism works on Windows, capture
the real `SessionStart` contract, and validate the Kiro integration hook reports
session identity to Herdr without affecting the user's default session or
pre-existing Kiro hooks.

## Observed baseline

> UPDATED 2026-10-10 (spike worktree `spike/kiro-v3-hook-harness`). The original
> baseline below was written against kiro-cli 2.16.2; the installed binary has
> since moved to 2.29.0. Reconstructed context:
> `.scratch/research/kiro-v3-state.md` and
> `.scratch/research/kiro-cli-session-payload.md`.

- **`kiro-cli-chat 2.29.0`** is installed (was 2.16.2 in the original note) and
  exposes `--v3`, `--v2`, `--resume-id`, `--resume-picker`, `--sessions`,
  `--list-sessions --format json`, and a new `kiro-cli serve` v3 WebSocket
  engine. Live-capture facts must be taken from 2.29.0, not 2.16.2.
- `~/.kiro/hooks/herdr-agent-state.json` and `.ps1` already exist and are a
  **v1 herdr Kiro integration** (`HERDR_INTEGRATION_VERSION=1`, source
  `herdr:kiro`, trigger `SessionStart`, `timeout: 10`, command
  `powershell -ExecutionPolicy Bypass -File .../herdr-agent-state.ps1`). The
  `.ps1` reads STDIN, parses JSON, and reports `pane report-agent-session` using
  `session_id`. This is **session-identity-only** (no working/blocked/idle). It
  is user-owned state and must NOT be overwritten or removed by the spike.
  Implication: a prior session already shipped a v1 Kiro identity integration;
  ticket 015 must reconcile/extend rather than build greenfield.
- Herdr is running inside Herdr (`HERDR_ENV=1`). Record the exact
  `herdr --version` at spike start.
- Kiro documents JSON on stdin, non-blocking `SessionStart`, and standalone
  `.kiro/hooks/*.json`. It does NOT pin, for 2.29.0: the exact `hook_event_name`
  casing, the timeout field name/units (`timeout` seconds vs `timeout_ms`),
  resume `SessionStart` behavior, or Windows command spawn/env details. Those
  remain the live-test questions.

## Safety and isolation

- Use the `herdr-throwaway-repro` skill and a unique named session such as
  `repro-kiro-v3-<timestamp>`. Never run the tests in the default session.
- Create one outer pane, record every returned pane ID, and address every
  disposable-session command by clearing inherited socket/caller variables and
  setting only `HERDR_SESSION=<name>` for that command.
- Poll `herdr pane list` for readiness; do not treat a fixed sleep as proof.
- Use unique probe hook, script, and artifact names. Do not replace the existing
  `herdr-agent-state.json` or `.ps1` files.
- Explicitly invoke probe scripts with
  `powershell -NoProfile -ExecutionPolicy Bypass -File <absolute-path>`.
  Do not rely on `%TEMP%`, `$env:TEMP`, or an undocumented implicit shell in the
  JSON command string.
- Keep probe artifacts in the unique reproduction directory. Preserve the
  final evidence in `.scratch/research/kiro-v3-hook-validation.md` before
  deleting that directory.
- Cleanup is required after success or failure: stop/delete only the generated
  named session, close only the created outer pane, remove only probe hook files
  and the reproduction directory, and verify the original hooks are unchanged.

## What to do

### 1. Record authority and create the disposable session

Record:

- `Get-Command herdr` path and `herdr --version`.
- `Get-Command kiro-cli` path and `kiro-cli --version`.
- Existing `~/.kiro/hooks/*.json` and matching scripts, including hashes.
- The generated session name, reproduction directory, outer pane ID, and root
  pane ID returned by Herdr.

Follow `.scratch/research/herdr-windows-throwaway-commands.md` for the installed
Windows CLI sequence. Inspect nested help again if observable syntax differs.

### 2. Confirm v3 global hook loading

Create a uniquely named global hook and PowerShell script beside the existing
hooks. The script should write a marker plus its current directory, process
metadata, and selected inherited variables to the reproduction directory.

Launch `kiro-cli chat --v3` in the disposable root pane. Verify the marker is
created and the Kiro session reaches a usable TUI. Record whether both global
and workspace hooks are present or whether only global loading was tested.

### 3. Capture the real `SessionStart` input

Change only the unique probe hook to invoke a script that captures
`[Console]::In.ReadToEnd()` verbatim. Start a fresh v3 session and record:

- Every JSON key and the exact `hook_event_name` value.
- Whether `session_id` is a non-empty string and its observed format.
- Whether `cwd` equals the reproduction directory.
- `HERDR_ENV`, `HERDR_PANE_ID`, `TEMP`, current directory, and process metadata
  observed inside the explicit PowerShell script.

Do not normalize the raw capture before preserving it. Redact only secrets if
unexpected fields contain them.

### 4. Test the Herdr integration hook

Use the existing matching integration script without changing it. Create a
separate uniquely named hook config that invokes that script, then launch a
fresh v3 Kiro session in the disposable pane.

Verify with disposable-session commands:

- `herdr agent explain <pane> --json` exposes the reported agent session.
- `herdr pane list --json` exposes the same Kiro session ID on that pane.
- The hook exits without blocking startup or emitting output into agent context.

Record the exact field paths returned by the installed Herdr build rather than
assuming `agent_session` appears at a particular JSON depth.

### 5. Test edge cases one variable at a time

1. **Outside Herdr:** invoke the integration script with captured stdin while
   `HERDR_ENV`, socket, and pane variables are unset. Verify exit code 0 and no
   report or output.
2. **Timeout:** use a separate probe hook with a short `timeout` and a longer
   `Start-Sleep`. Record elapsed time, visible warning, whether Kiro remains
   usable, and whether the child process is terminated.
3. **Resume:** exit the fresh v3 session cleanly, then run
   `kiro-cli chat --v3 --resume-id <captured-id>`. Record whether `SessionStart`
   fires and whether the session ID is unchanged.
4. **Event naming:** report the observed value; do not assume `SessionStart` or
   legacy `agentSpawn`.

### 6. Document and clean up

Write `.scratch/research/kiro-v3-hook-validation.md` with:

- Herdr and Kiro binary paths and versions.
- Named-session launch environment and returned IDs.
- Exact commands, timestamps, and raw `SessionStart` JSON.
- Fresh, integration, outside-Herdr, timeout, and resume results.
- Differences from current Kiro documentation or this ticket's assumptions.
- Whether session identity is stable across resume.
- Whether the hook fires before or after the TUI becomes usable.
- Cleanup evidence and any remaining coverage gaps.

Then remove only this spike's hook files and reproduction directory, stop and
delete only its named session, close only its outer pane, and verify the
pre-existing global hook hashes still match the baseline.

## Acceptance criteria

- [x] Installed binary paths, versions, pre-existing hook inventory, and hashes recorded
- [x] Unique disposable Herdr session used; default session untouched
- [x] v3 global hook fires on fresh `SessionStart`
- [x] Exact raw stdin captured; `session_id`, event name, and `cwd` validated
- [x] Windows command process and inherited Herdr environment documented
- [ ] Herdr integration hook reports the same session ID through agent and pane JSON
- [x] Integration hook exits 0 without reporting when Herdr variables are unset
- [x] Timeout warning, elapsed behavior, child termination, and session continuity documented
- [x] `--v3 --resume-id` behavior and session-ID stability documented
- [x] Probe files/session/pane removed and original hook hashes verified unchanged
- [x] Findings written to `.scratch/research/kiro-v3-hook-validation.md`

## Preparation research

- `.scratch/research/kiro-v3-windows-hook-execution.md`
- `.scratch/research/herdr-windows-throwaway-commands.md`
- `.scratch/research/kiro-v3-state.md`
- `.scratch/research/kiro-cli-session-payload.md`

## Resolution

Pending execution.
