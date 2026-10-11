---
id: "017"
title: "Live-validate Kiro v3 integration end to end (hook + manifest)"
status: done
priority: medium
blocked_by: ["015", "016"]
---

# Live-validate Kiro v3 integration end to end (hook + manifest)

## Goal

Prove the combined Kiro integration (hook from 015 + enriched manifest from 016)
correctly tracks the full agent lifecycle on a real `kiro-cli --v3` session,
across both the hook-authority path and the screen-detection fallback, on
Windows (primary target of this effort) and at least one POSIX shell asset path.

Per AGENTS.md: "Passing engine tests proves the rules execute as written; it
does not prove compatibility with the current CLI." This ticket is that
compatibility proof.

## Reference / prior art

- Spike `014` + `.scratch/research/kiro-v3-hook-validation.md` (the raw
  `SessionStart` contract and Windows execution facts).
- Harness: `.scratch/research/herdr-windows-throwaway-commands.md`,
  `.agents/skills/herdr-throwaway-repro/SKILL.md`.
- The hook asset and manifest produced by 015 and 016.

## What to do

Run the disposable-session harness (unique `repro-kiro-v3-<timestamp>` session;
never the default session). For each scenario, record exact commands,
timestamps, `herdr agent explain <pane> --json`, and `herdr pane list --json`
field paths.

### A. Hook-authority path (integration installed)

1. Install the Kiro integration into the disposable environment (versioned, not
   overwriting unrelated user hooks).
2. Launch `kiro-cli --v3`. Confirm:
   - `SessionStart` → session identity reported; `herdr agent explain` and
     `pane list` expose the same Kiro session ID on the pane.
   - A prompt submit → `working`.
   - A tool/permission prompt → `blocked`.
   - Turn end → `idle`.
   - Screen detection is **paused** for the pane while the hook holds authority
     (confirm Kiro is in `full_lifecycle_hook_authority`).
   - The hook never prints into agent context and never blocks startup.
3. Exit cleanly, then `kiro-cli --v3 --resume-id <id>`; record whether session
   identity is stable and `SessionStart` fires on resume.
4. Timeout edge: a slow hook must not wedge the agent; record warning + behavior.
5. Outside Herdr (vars unset): the asset exits 0 and reports nothing.

### B. Screen-detection fallback (integration NOT installed)

1. Remove/skip the hook. Reload the enriched manifest via
   `herdr server reload-agent-manifests` (temporary override, then restored).
2. Drive idle / working / blocked and confirm `herdr agent explain` matches the
   intended rule for each, including the new `skip_state_update` transcript/menu
   screens (state must NOT flip while paging history).
3. Confirm notifications/sounds fire on the right transitions (blocked →
   NeedsAttention/Request sound; background-completion → Finished/Done sound) per
   `src/app/actions.rs` state-transition mapping and the `kiro` sound setting in
   `src/config/sound.rs`.

### C. Record results

Write `.scratch/research/kiro-v3-integration-validation.md` with: Herdr and Kiro
versions/paths, every scenario's commands + raw JSON, pass/fail, differences
from 014's assumptions, and remaining coverage gaps. Then clean up only this
run's session/pane/override/artifacts and verify pre-existing user hooks are
unchanged.

## Acceptance criteria

- [x] Hook path: session/working/blocked/idle + resume + detection-pause all verified on `--v3`
- [x] Hook exits 0 silently outside Herdr; timeout does not wedge the agent
- [x] Fallback path: idle/working/blocked + transcript `skip_state_update` verified via `agent explain`
- [ ] Notification/sound transitions verified
- [ ] POSIX `.sh` asset path smoke-checked (not Windows-only)
- [x] Results written to `.scratch/research/kiro-v3-integration-validation.md`
- [x] Disposable session/pane/override removed; original hooks verified unchanged

## Resolution

Pending. Blocked by 015 and 016.
