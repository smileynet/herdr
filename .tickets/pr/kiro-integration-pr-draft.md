# DRAFT — Pull Request body — NOT SUBMITTED

> Internal draft for review. Do NOT open against herdrdev/herdr until: the
> maintainer approves the direction (Discussion first — this is a new surface
> and the acting account is not in `.github/MAINTAINERS`/`APPROVED_CONTRIBUTORS`
> at time of writing, though it authored merged #2301), we've dogfooded for a few
> weeks, the branch is rebased on current `origin/master`, and `.tickets/` +
> `.scratch/research/` are stripped from the diff.
>
> Prior art to stay consistent with / not repeat:
> - #4372 (MERGED) current kiro manifest — our changes are ADDITIVE to it.
> - #4499 (CLOSED) kiro v3 identity hooks — CodeRabbit flagged (1) tests touching
>   the real ~/.kiro config and (2) kiro events silently dropped. We address both.
> - #2301 (MERGED, same author) kiro idle detection — the template for small
>   kiro detection PRs.

---

**Title:** `feat: add kiro cli v3 hook integration`

## Summary

- Adds an optional `herdr integration install kiro` that installs a Kiro v3
  standalone hook (`.kiro/hooks/herdr-agent-state.json` + script) reporting
  authoritative working / idle state and session identity.
- Enriches the existing `kiro.toml` screen manifest (additive to #4372) with a
  scroll-proof idle signal and a `/sessions` picker skip-state rule, so no-hook
  users still get accurate state.
- Registers Kiro as an experimental, CLI-only integration target — it does NOT
  touch the frozen generation-1 `IntegrationTarget` enum or the JSON API schema.

## What changed

Hook integration (install/uninstall/status via `herdr integration … kiro`):

- Writes `herdr-agent-state.{sh,ps1}` and a schema-`v1` `herdr-agent-state.json`
  into Kiro's `hooks/` dir. Config dir resolves via `KIRO_HOME` then `~/.kiro`.
- Maps Kiro v3 events to state: `SessionStart`→session identity (with
  `--session-start-source startup` so the runtime anchors the session),
  `UserPromptSubmit`/`PreToolUse`/`PostToolUse`→working, `Stop`→idle. Reported
  under `source: herdr:kiro`; Kiro gains full-lifecycle authority so hook state
  overrides screen detection while active.
- The hook uses `HERDR_BIN_PATH` so its client matches the running server, exits
  0 silently when `HERDR_ENV`/`HERDR_PANE_ID`/`HERDR_SOCKET_PATH` are unset, and
  never prints into agent context. Uninstall removes only the Herdr-owned files
  and leaves unrelated user hooks untouched. `status` reports `outdated` when the
  installed JSON config no longer matches the Herdr-owned shape.
- `blocked` is intentionally left to screen detection: Kiro v3 has no documented
  permission/notification hook event, so inventing a blocked report would risk
  silently-wrong state. (Addresses the "events silently dropped" concern from
  #4499 — we only report events Kiro actually fires, and document the omission.)

Manifest (`src/detect/manifests/kiro.toml` + `distribution/agent-detection/kiro.toml`,
kept byte-identical; version bumped, min engine unchanged at 2):

- `osc_progress_idle` (`^4;0`, low priority): Kiro v3 drives OSC progress `4;0`
  at idle and `4;3` while working, so idle survives the prompt scrolling off the
  viewport. Lower priority than every live working/blocked rule.
- `sessions_picker` (`skip_state_update`): the `/sessions` resume list is a
  navigable menu, not a live prompt — preserve prior state instead of asserting
  idle. Guarded with `not` so a real tool-approval still wins.
- All 8 existing #4372 rules are unchanged.

## Endpoint / backwards-compatibility note

Kiro is wired as an **experimental, CLI-only** integration (same approach as the
closed-but-correct #4499 and the merged experimental Letta path): it is kept out
of the frozen client-endpoint `IntegrationTarget` enum, so published
generation-1 clients never receive an unknown value over `integration.install` /
`integration.list`. No change to `docs/next/api/herdr-api.schema.json`, no frozen
enum variant, no change to any other agent's manifest. The frozen
endpoint-method-shape contract test stays green. Requires kiro-cli with v3
standalone hooks (verified against 2.29.0).

## Validation

- `just ci` — <to be re-run green on a clean CI/maintainer machine before
  submission; see build note below>.
- Targeted: `cargo nextest run --bin herdr -E 'test(kiro) or test(manifest) or
  test(bundled_integration_asset_versions) or
  test(advertised_client_shell_method_shapes_stay_at_the_v1_contract)'` — all
  pass locally (manifest schema validation, kiro install/uninstall/status/config,
  the session-anchor regression test, frozen-contract test).
- Live, kiro-cli 2.29.0 on Windows:
  - [x] SessionStart fires; hook env confirmed (`HERDR_BIN_PATH`,
    `HERDR_SOCKET_PATH`, `HERDR_PANE_ID`), real payload captured
    (`session_id` is a `sess_`-prefixed UUID; `hook_event_name` PascalCase).
  - [x] Full-lifecycle authority engages given a startup-anchored session
    (proven by deterministic in-process tests; the missing `session_start_source`
    was found and fixed here).
  - [x] Screen fallback: idle / working / tool-approval(blocked) / `/sessions`
    picker classified correctly via `herdr agent explain`; OSC `4;0`/`4;3`
    confirmed live; `osc_progress_idle` matches on scroll.
  - [x] `SessionStart` does NOT fire on `--resume-id` (kiro behavior) — see Notes.
  - [x] Hook exits 0 silently outside herdr; a slow hook does not block startup
    (timeout kills the child; session stays usable).
  - [ ] Not run: persistent authority observed on a *live interactive* TUI
    (non-interactive exits before query; interactive driving raced hook timing —
    a harness limitation, logic is covered by tests).
  - [ ] Not run: native POSIX `.sh` path on a live Linux/macOS host (the `.sh`
    carries the same session-anchor fix and is covered by a regression test).
- Test isolation: all install/uninstall tests target a throwaway dir, never the
  developer's real `~/.kiro` (addresses the test-isolation concern from #4499).

## Notes

- Resume: because Kiro's `SessionStart` doesn't fire on `--resume-id`, a resumed
  pane currently falls back to screen detection rather than hook authority. This
  is a deliberate, documented scope boundary for a first integration; a
  follow-up can add a launch-args resume parse + `is_official_agent_source` arm.
- Build note (reviewer context): building the full binary on one Windows machine
  hit a rustc DWARF-codegen stack overflow (`STATUS_STACK_BUFFER_OVERRUN`,
  rust-lang/rust #122357 class) under memory pressure; worked around with
  `RUST_MIN_STACK` + `codegen-units=1` + `debuginfo=0`. Not a code change; should
  be a non-issue on CI.
- No edits to root README/CHANGELOG or `docs/next/CHANGELOG.md`. User-facing docs
  under `docs/next/website/...` to be added if the direction is approved.

refs #<discussion-or-issue-number>
