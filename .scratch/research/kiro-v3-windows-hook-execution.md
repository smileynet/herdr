---
topic: Kiro CLI 2.16.2 v3 Windows hook execution
date: 2026-08-08
status: complete
confidence: medium
---

# Kiro CLI 2.16.2 v3 Windows hook execution

## Summary

Kiro CLI 2.16.2 exposes the v3 harness through `kiro-cli --v3`. V3 command hooks are standalone `.kiro/hooks/*.json` files, run in the project root, receive JSON session context on STDIN, and use a timeout measured in seconds (default 60; `0` disables it). Official documentation does not specify the Windows command interpreter, environment-expansion syntax, the exact `SessionStart` payload, or whether `SessionStart` fires during `--resume-id`; ticket 014 must measure those details with a disposable session.

## Sources

- [L1:verified] `kiro-cli --version` and `kiro-cli --v3 --help` on this machine — directly observed version `kiro-cli-chat 2.16.2`, v3 flag, and top-level `--resume-id <SESSION_ID>` support.
- [L4:verified] [Kiro Hooks](https://kiro.dev/docs/hooks/) — current first-party v3 schema, hook discovery, project-root execution, JSON-on-STDIN contract, triggers, timeout field, and exit behavior; updated 2026-08-06.
- [L4:verified] [What's new in CLI 3.0](https://kiro.dev/docs/cli/v3/) — current first-party v3 migration and compatibility notes; updated 2026-08-05.
- [L4:verified] [Kiro Hooks migration](https://kiro.dev/docs/cli/v3/hooks-migration/) — current first-party mapping from v2 `agentSpawn` to v3 `SessionStart` and standalone-file examples; updated 2026-08-04.
- [L4:verified] [Kiro Hook actions](https://kiro.dev/docs/hooks/actions/) — current first-party command exit and timeout behavior; updated 2026-08-04.
- [L4:verified] [Kiro CLI commands](https://kiro.dev/docs/reference/cli-commands/) — first-party session-management reference for `--resume-id` and UUID session identifiers.
- [L1:verified] `C:/Users/uosmi/.kiro/hooks/herdr-agent-state.json` and `.ps1` — local installed integration explicitly invokes `powershell -ExecutionPolicy Bypass -File ...` and reads all STDIN before accessing `session_id`; evidence of Herdr's intended portable invocation, not evidence of Kiro's implicit shell.

## Related Topics

- Herdr Kiro integration contract — confirm whether `session_id` is sufficient or whether v3 exposes a more stable identifier.
- Hook isolation — validate whether global and workspace hook files are merged and how duplicate names behave.

## Related Tools & Resources

- `kiro-cli --v3 --help` — read-only confirmation of installed v3 and resume switches.
- A uniquely named Herdr throwaway session — safe live harness for capturing `SessionStart` input and timeout behavior.
- A uniquely named temporary hook file and PowerShell capture script — avoids replacing the installed Herdr integration.

## Findings

### Confirmed contract

- The installed binary reports `kiro-cli-chat 2.16.2`; `kiro-cli --v3 --help` accepts `--v3`, `--resume`, `--resume-id`, and `--resume-picker`. [L1:verified]
- CLI 3.0 uses standalone `.kiro/hooks/*.json` files with schema version `v1`. `SessionStart` replaces v2's `agentSpawn`, fires when a session begins, does not evaluate a matcher, and cannot block startup. [L4:verified]
- A command action has the form `{"type":"command","command":"..."}`. Kiro runs it in the project root and supplies session context as JSON on STDIN. [L4:verified]
- `timeout` is an integer number of seconds. Its default is 60; `0` disables the timeout. Current v3 docs supersede the older v2 `timeout_ms`/30-second contract. [L4:verified]
- Exit code 0 is success; current hook docs say stdout is added to context for `SessionStart` and `UserPromptSubmit`. Other exit codes produce a warning. Because `SessionStart` cannot block, a failure or timeout should not intentionally prevent startup, but the visible v3 TUI behavior still needs observation. [L4:verified]
- V3 sessions use a format incompatible with v2. V3 sessions cannot be resumed in v2, so both fresh and resume tests must use `--v3`. [L4:verified]
- Kiro documents session IDs as UUIDs and supports resuming a specific session with `--resume-id <ID>`. [L4:verified]

### Windows execution guidance

- The v3 schema stores one command string; it does not offer separate `powershell`/`bash` fields. Official Kiro docs show POSIX shell examples but do not state which interpreter executes that string on Windows. [L4:verified]
- Therefore ticket 014 should not assume `%TEMP%`, `$env:TEMP`, `cmd.exe`, or PowerShell interpolation in the hook command. Use an explicit executable invocation such as `powershell -NoProfile -ExecutionPolicy Bypass -File "C:/.../capture.ps1"` and let the script read `$env:TEMP` or a preselected absolute artifact path. This is a recommendation inferred from the documented opaque command string and the installed Herdr hook's explicit PowerShell invocation. [L1:verified; L4:verified]
- Environment variables inherited by the Kiro process should be read inside the explicit PowerShell script (`$env:HERDR_ENV`, `$env:HERDR_PANE_ID`) rather than expanded in JSON command text. Whether Kiro alters inherited environment variables is undocumented and must be observed. [L1:verified; L4:verified]
- `[Console]::In.ReadToEnd()` is appropriate for capturing the complete JSON payload because Kiro documents STDIN delivery. The installed Herdr script already uses this pattern. [L1:verified; L4:verified]

### Required live observations for ticket 014

1. Capture raw `SessionStart` STDIN for a fresh `kiro-cli --v3` launch; record all keys, exact trigger-name casing if present, `cwd`, and `session_id` format.
2. Resume that same v3 UUID with `kiro-cli --v3 --resume-id <ID>`; record whether `SessionStart` fires and whether its `session_id` is identical.
3. Use an explicit PowerShell command and record inherited `HERDR_ENV`, `HERDR_PANE_ID`, `TEMP`, current directory, and process metadata. This answers Windows execution without relying on undocumented interpreter semantics.
4. Run a separate hook with a short `timeout` and a longer `Start-Sleep`; observe warning text, elapsed time, hook-process termination, and whether the TUI reaches a usable session.
5. Run the integration with Herdr variables unset; verify it exits successfully without reporting a session.

## Open Questions

- Which executable/interpreter Kiro 2.16.2 uses internally for a Windows command string?
- What exact fields and casing appear in v3 `SessionStart` STDIN?
- Does `SessionStart` fire on `--resume-id`, and does the UUID remain stable?
- What warning is shown on timeout, and is the child process tree terminated?
- Are global `~/.kiro/hooks` and project `.kiro/hooks` both loaded by CLI v3, and with what precedence?
