---
id: "019"
title: "Make agent resume trust/auto-accept flag configurable"
status: open
blocked_by: []
priority: medium
---

# Make agent resume trust/auto-accept flag configurable

## Problem

When herdr resumes an agent session (on session restore, or any
`AgentResumePlan` path), it relaunches the agent with a bare resume argv and
**never passes the agent's trust / auto-accept flag** (e.g. `-a` /
`--trust-all-tools` for kiro-cli, `--dangerously-skip-permissions` for Claude
Code, `--yolo` / `--full-auto` for Codex). As a result a resumed agent comes
back in its default *prompting* permission mode, which:

- loses the trust posture the user originally launched with, and
- stalls an otherwise unattended resume on the first tool-permission prompt
  (the pane sits `blocked` waiting for a human), defeating the point of
  resuming many panes at once.

The flag is agent-specific and carries real risk (auto-approving tool
execution), so it must be **off by default and explicitly opt-in**, configured
per agent — never forced on globally.

## Current behavior (grounding)

- `src/agent_resume.rs::plan(source, agent, session_ref)` builds the resume
  `argv` per agent, e.g.:
  - claude → `["claude", "--resume", <id>]`
  - codex → `["codex", "resume", <id>]`
  - kimi → `["kimi", "--session", <id>]`
  - kiro (once wired) → `["kiro-cli", "--resume-id", <id>]`
  None of the arms append a trust/auto-accept flag.
- The plan's `argv` is stored as `terminal.pending_agent_resume_plan` and
  executed on restore. The restore path is gated by
  `[session] resume_agents_on_restore` (`src/persist/restore.rs`,
  `persisted_resume_plan_for_session(..., resume_agents_on_restore)` and
  `src/config/io.rs`). `validate_resume_argv` (`src/persist/restore.rs:868`,
  defined in `src/agent_resume.rs`) requires a bare command name and forbids
  control chars / apostrophes — any added flag must pass this validation.
- Per-agent config precedent already exists: `[ui.sound.agents.<agent>]`
  (`src/config/sound.rs` `AgentSoundOverrides`, one field per agent) and
  `[session] resume_agents_on_restore` (`src/config/model.rs` / `io.rs`). Follow
  these patterns rather than inventing a new config shape.

## Flag reference (per agent — verify each against the installed CLI before use)

The trust/auto-accept switch differs per agent and must be looked up, not
guessed. Known examples (confirm with `<agent> --help` / `<agent> chat --help`):

- kiro-cli: `-a` / `--trust-all-tools` (also `--trust-tools=<list>` for a scoped
  set). Observed in `kiro-cli chat --help`, 2.29.0.
- claude: `--dangerously-skip-permissions`.
- codex: `--yolo` / `--full-auto` / `--dangerously-bypass-approvals-and-sandbox`
  (confirm current spelling).
- Others (droid, qwen, kimi, etc.): verify individually; some have no such flag,
  in which case the config is a no-op for that agent.

Do NOT hardcode a single `-a`: the token is agent-specific. The mapping belongs
next to the per-agent resume argv construction in `src/agent_resume.rs`.

## Proposed design

1. Add a per-agent resume-trust config, mirroring `AgentSoundOverrides`:
   `[session.resume_trust_all]` (or `[agents.<agent>] resume_trust_all = true`),
   default `false` for every agent. Decide the exact table name with maintainers;
   keep it under the existing `[session]`/agents config, not a new top-level key.
2. Thread the resolved per-agent setting into `agent_resume::plan(...)` (or into
   the caller that builds the plan) and, when enabled AND the agent actually
   supports a trust flag, append the agent-specific token to `argv`. Keep a
   single source of truth mapping `agent -> Option<trust_flag_tokens>`.
3. Ensure the appended flag passes `validate_resume_argv` and the dedupe-key
   logic (the trust flag changes the argv, so it participates in the dedupe key —
   that is correct; a trusted vs untrusted resume are different plans).
4. Surface it in docs (`docs/next/website/...`) and the config reference checked
   by `scripts/config_reference_check.py` (run under `just release-docs-check`).

## Guardrails / risk

- **Security-sensitive**: this auto-approves tool execution on resume. Default
  OFF, explicit opt-in, documented clearly as such. This is exactly the kind of
  "removing/relaxing permission controls" change that warrants maintainer
  alignment before implementation (AGENTS.md / CONTRIBUTING.md: features and
  behavior changes need discussion + approval first).
- Runtime/client boundary (AGENTS.md): resume behavior is a shared runtime/
  session fact, so the config + flag belong in server/runtime config, not TUI-
  only state. Use neutral names.
- No `unwrap()` in production; per-agent flag mapping compile-safe for all
  agents (a missing flag = no-op, not a panic).

## Testing

- Unit-test `agent_resume::plan` (or the plan builder) for: default off = no flag
  appended; enabled + supported agent = correct agent-specific token appended;
  enabled + unsupported agent = no change; appended argv still passes
  `validate_resume_argv`; dedupe key differs between trusted/untrusted.
- Config round-trip test for the new per-agent setting (default false, parse
  `true`), mirroring `src/config/sound.rs` tests and the
  `resume_agents_on_restore` test in `src/config/io.rs`.
- `just ci` / `just check` green.

## Acceptance criteria

- [ ] Per-agent resume-trust config added (default off), following the
      `AgentSoundOverrides` / `[session]` precedent
- [ ] Agent-specific trust-flag mapping lives beside the resume argv builder in
      `src/agent_resume.rs`; no hardcoded single `-a`
- [ ] Flag appended only when enabled AND the agent supports it; appended argv
      passes `validate_resume_argv`
- [ ] Dedupe key accounts for the added flag
- [ ] Unit + config round-trip tests added
- [ ] Documented as security-sensitive, off by default; config reference updated
- [ ] Maintainer alignment obtained before implementation (new behavior)
- [ ] `just ci` passes

## Resolution

Pending. Not blocked; independent of the Kiro integration (015-018), though the
kiro resume arm (currently a follow-up gap noted in 017) is one consumer.
