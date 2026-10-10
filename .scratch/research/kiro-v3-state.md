---
topic: Kiro CLI v3 current state (installed binary + v3 surface)
date: 2026-10-10
status: reconstructed
confidence: medium
note: >
  Rebuilt for ticket 014 because the original note referenced in 014 was not
  carried on the fork commit. Facts marked [L1] are observed from the installed
  binary in the spike worktree; [L4] are first-party kiro.dev docs. The installed
  version moved from 2.16.2 (014 baseline) to 2.29.0, so treat 014's older
  version-specific assumptions as stale where they conflict with this note.
---

# Kiro CLI v3 current state

## Installed authority (spike machine)

- `kiro-cli --version` → `kiro-cli-chat 2.29.0`. [L1:verified]
  (Ticket 014 and `.scratch/research/kiro-v3-windows-hook-execution.md` were
  written against `2.16.2`. The v3 engine is reached via `--v3`; `--v2` selects
  the pre-3.0 engine, still the documented default in older notes.)
- `kiro-cli --help` top-level commands: `agent`, `chat`, `login`, `logout`,
  `whoami`, `profile`, `settings`, `diagnostic`, `issue`, `crew`, `mcp`,
  `powers`, `update`, `serve`, `help`. [L1:verified]
- Global flags include `--tui`, `--legacy-ui`/`--classic`, `--v3`, `--v2`,
  `-r/--resume`, `--resume-id <SESSION_ID>`, `--resume-picker`/`--list`,
  `--cloud`, `--repo`. [L1:verified]
- **New v3 surface**: `kiro-cli serve` — "Start a persistent WebSocket server
  for the v3 engine". This is a separate structured channel from the terminal
  stream (relevant to the long-term structured-integration path noted in the
  research summary; not used by the hook integration). [L1:verified]

## `kiro-cli chat` options relevant to integration

[L1:verified from `kiro-cli chat --help`]

- Resume: `-r/--resume`, `--resume-id <SESSION_ID>`, `--resume-picker`,
  `--sessions` (V3/KAS session dashboard), `-l/--list-sessions`, `--all-cwds`.
- Machine-readable listing: `-f/--format <plain|json|json-pretty>` is accepted
  **only with `--list-models` and `--list-sessions`** — NOT for the live chat
  turn. So there is still no machine-readable *live* status stream on the
  terminal; `--list-sessions --format json` only enumerates saved sessions.
- Permissions: `-a/--trust-all-tools`, `--trust-tools <TOOL_NAMES>`
  (e.g. `--trust-tools=fs_read,fs_write` or empty to trust none),
  `--no-interactive`.
- `--agent <AGENT>`, `--model <MODEL>`, `--effort <low|medium|high|xhigh|max>`.

## Session identity

- Sessions are UUID-identified, saved automatically per-directory every turn,
  resumable by `--resume-id <ID>`. [L4:verified — kiro.dev session-management]
- v3 session format is incompatible with v2; a v3 session cannot be resumed in
  v2. Both fresh and resume tests must use `--v3`. [L4:verified]

## What this means for herdr

- The hook channel (standalone `.kiro/hooks/*.json`, STDIN JSON) remains the
  integration path; see `kiro-cli-session-payload.md` for the payload contract.
- A Kiro hook integration asset is ALREADY installed on this machine
  (`~/.kiro/hooks/herdr-agent-state.json` + `.ps1`, `HERDR_INTEGRATION_VERSION=1`,
  source `herdr:kiro`, trigger `SessionStart`, `timeout: 10`). That means prior
  work produced a v1 Kiro integration. Ticket 015 must reconcile with it rather
  than assume a greenfield asset. The spike must treat these installed files as
  user-owned state and must not overwrite them.
- `kiro-cli serve` (WebSocket v3 engine) and `--list-sessions --format json` are
  the structured surfaces; neither replaces the hook for live state today.

## Open questions carried into 014 execution

- Does the installed 2.29.0 hook event use `SessionStart` (installed config) or
  the older `AgentSpawn`/casing seen in some docs? Capture the real
  `hook_event_name` casing from STDIN.
- Which interpreter runs the Windows `command` string? (Installed config calls
  `powershell -ExecutionPolicy Bypass -File ...` explicitly, so Kiro only needs
  to spawn that command, not interpret PowerShell itself.)
- `timeout` (seconds, installed config = 10) vs `timeout_ms` (older docs = 30000):
  confirm which the 2.29.0 schema honors.
- Does `SessionStart` fire on `--resume-id`, and is the UUID stable?

## Sources

- [L1:verified] Installed `kiro-cli --version`, `--help`, `--v3 --help`,
  `chat --help` captured in the spike worktree 2026-10-10.
- [L1:verified] Installed `~/.kiro/hooks/herdr-agent-state.json` and `.ps1`.
- [L4:verified] https://kiro.dev/docs/cli/chat/session-management/ (session UUIDs, per-dir autosave, resume).
- [L4:verified] https://kiro.dev/docs/cli/v3/migration-guide/ (v3 vs v2 session incompatibility).
