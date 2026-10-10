---
topic: Kiro CLI v3 hook STDIN payload contract (SessionStart and others)
date: 2026-10-10
status: reconstructed
confidence: medium
note: >
  Rebuilt for ticket 014. Documented contract from first-party kiro.dev hooks
  docs (and a faithful third-party mirror). The EXACT payload for the installed
  2.29.0 binary — especially hook_event_name casing and session_id format — is a
  live-capture task for 014; this file is the pre-capture reference, not the
  observed result.
---

# Kiro CLI v3 hook STDIN payload contract

## Hook file shape

Standalone `.kiro/hooks/<id>.json`, schema `"version": "v1"`, loaded when a
session starts. One file may define several hooks. A hook has `name`, `trigger`,
optional `matcher`, and `action`. [L4:verified]

```json
{
  "version": "v1",
  "hooks": [
    {
      "name": "guard-writes",
      "trigger": "PreToolUse",
      "matcher": "fs_write",
      "action": { "type": "command", "command": "./guard.sh" }
    }
  ]
}
```

- `action.type`: `command` (shell command, receives the event JSON on STDIN) or
  `agent` (injects a prompt; costs credits). `enabled: false` disables without
  deleting. [L4:verified]
- The installed herdr integration uses `trigger: "SessionStart"`,
  `action.type: "command"`, and `"timeout": 10` (seconds). [L1:verified]

## STDIN payload (documented shape)

Hooks receive a JSON object on STDIN. Documented example: [L4:verified]

```json
{
  "hook_event_name": "preToolUse",
  "cwd": "/current/working/directory",
  "session_id": "abc123-def456-789",
  "tool_name": "read",
  "tool_input": { }
}
```

- `session_id` — the field the herdr integration reads to report agent session
  identity. Documented as a UUID-style string. The installed `.ps1` reads
  `[Console]::In.ReadToEnd()`, parses JSON, and uses `payload.session_id`.
  [L1:verified; L4:verified]
- `cwd` — session working directory.
- `hook_event_name` — present on the payload. **Casing is uncertain**: the docs
  example shows camelCase `preToolUse`, while trigger names in config are
  PascalCase (`PreToolUse`, `SessionStart`). 014 must record the exact value the
  2.29.0 binary emits for a `SessionStart` hook. [L4:verified; casing OPEN]
- `tool_name` / `tool_input` — present only for tool-scoped events
  (`PreToolUse`, `PostToolUse`), not expected for `SessionStart`. [L4:verified]
- `UserPromptSubmit` additionally carries a `prompt` field. [L4:verified]

## Triggers relevant to a state integration

[L4:verified]

- `SessionStart` — new chat session begins (IDE or CLI v3). Canonical v3 trigger
  that replaces v2 `AgentSpawn`. No matcher. STDOUT (exit 0) injected to context.
  Cannot block startup. → maps to herdr `session` identity report.
- `UserPromptSubmit` — user submits a prompt; carries `prompt`. → `working`.
- `PreToolUse` — before a tool runs; gating (exit 0 allow / exit 2 deny / any
  other exit fails closed). Carries `tool_name`, `tool_input`. → `working`
  (and a Kiro permission/approval prompt is the `blocked` signal — confirm which
  event or screen surfaces approval in 2.29.0).
- `PostToolUse` — after a tool; carries `tool_response`. → `working`.
- `Stop` — assistant finished responding (end of turn), no matcher. → `idle`.

Matcher supports canonical names + aliases (`fs_write`/`write`, `fs_read`/`read`,
`execute_bash`/`shell`, `use_aws`/`aws`), MCP forms (`@git`, `@git/status`),
`*` (all), `@builtin`. No matcher = every tool. [L4:verified]

## Exit-code / timeout semantics

- Exit 0 = success; STDOUT captured (injected to context for
  SessionStart-class / UserPromptSubmit; a delivered "allow" for PreToolUse).
- Exit 2 = deny on gating events (PreToolUse, UserPromptSubmit); STDERR to LLM.
- Any other exit (incl. timeout/crash/unexecutable): PreToolUse fails CLOSED
  (blocks the tool); all other events are warn-only and execution continues.
  [L4:verified]
- Timeout: docs say default 30s via `timeout_ms`; the installed 2.29.0 herdr hook
  config uses `"timeout": 10` (seconds). **Schema field name/units drift between
  doc versions — 014 must confirm what 2.29.0 honors.** [L4:verified; L1 drift]

## Integration implications (for ticket 015)

- A `SessionStart` command hook that exits 0 without STDOUT reports session
  identity without injecting anything into the model context — the correct,
  non-intrusive behavior. The installed `.ps1` already does this (silent
  `Out-Null`, guarded on `HERDR_ENV`/`HERDR_PANE_ID`, exits 0 when unset).
- To cover live state (working/blocked/idle) rather than only session identity,
  015 would add `UserPromptSubmit`/`PreToolUse`/`PostToolUse`/`Stop` command
  hooks mapping to `pane report-agent --state ...`, mirroring `KIMI_HOOK_EVENTS`.
  The installed v1 asset only wires `SessionStart` (identity only), which would
  place Kiro in `session_identity_only_integration`, NOT
  `full_lifecycle_hook_authority`. 015 must decide: extend to full lifecycle, or
  keep identity-only + rely on the enriched manifest (016) for state.

## Sources

- [L4:verified] https://kiro.dev/docs/hooks/ and /docs/hooks/types/, /docs/hooks/actions/
  (hook file schema, STDIN payload example, triggers, exit/timeout semantics),
  cross-checked against a faithful third-party mirror
  (github.com/dev-rb-hub/rendu.ai docs/reference/kiro-cli/hooks.md) that quotes
  the same payload and exit contract. Content paraphrased for licensing.
  Content was rephrased for compliance with licensing restrictions.
- [L1:verified] Installed `~/.kiro/hooks/herdr-agent-state.json` (SessionStart,
  timeout 10, powershell command) and `.ps1` (reads STDIN, uses session_id,
  reports `pane report-agent-session`), captured in the spike worktree 2026-10-10.
