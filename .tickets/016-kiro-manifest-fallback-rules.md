---
id: "016"
title: "Enrich kiro.toml screen-detection fallback rules"
status: open
priority: medium
blocked_by: []
---

# Enrich kiro.toml screen-detection fallback rules

## Goal

Improve the screen/OSC detection manifest for Kiro so that users who have NOT
installed the hook integration (ticket 015) still get accurate state, and so
transcript/menu screens stop misclassifying state. This is the no-dependency
fallback that complements 015; it is independently useful and can land on its
own.

## Why (gaps vs claude/codex, from investigation)

Current `distribution/agent-detection/kiro.toml` (identical to the bundled
`src/detect/manifests/kiro.toml`, `version = "2026.09.19.1"`,
`min_engine_version = 2`) has 8 rules: `live_prompt_idle`, `tool_approval`,
`tool_approval_edit`, `crew_approval`, `question_panel`, `live_working_footer`,
`osc_title_working`, `osc_progress_working`.

Gaps found when compared with `src/detect/manifests/claude.toml` and
`codex.toml`:

1. **No `skip_state_update` / transcript-viewer rule.** Both claude
   (`transcript_viewer`, `model_picker_menu`) and codex (`transcript_viewer`)
   have `state = "unknown"` + `skip_state_update = true` rules so scrollback,
   history, and picker screens do not flip live state. Kiro has none, so paging
   history or opening a menu can misclassify. This is the highest-value add.
2. **No idle OSC signal.** claude has `osc_title_idle` (`^\x{2733} `) and
   `osc_progress_idle` (`^4;0`); codex has `osc_title_idle`. Kiro only emits OSC
   for *working*, so idle relies on the on-screen prompt line
   (`> ask a question or describe a task`), which scrolls off. AGENTS.md
   "Screen detection is evidence-based" forbids using the user-scrollable
   viewport for status. **Verify via live capture whether Kiro v3 emits any idle
   OSC title/progress at all** — if it does not, this item is not achievable in
   the manifest and should instead be solved by 015's hook. Do not invent a
   signal.
3. **Thinner blocked coverage.** claude has layered permission tiers
   (`bash_permission_prompt`, `generic_permission_prompt`,
   `legacy_no_prompt_blocker`, `mcp_elicitation_prompt`) and codex has
   weak/strong tiers (`live_strong_blocker`, `weak_blocker`, `trust_directory`,
   `startup_update`). Capture additional Kiro approval/question shapes that the
   current 4 blocked rules miss.
4. **No background-work modeling.** claude detects background agents/MCP tasks as
   `working` (`background_agents_working`, `background_mcp_task_working`). Only
   add an equivalent if Kiro v3 actually surfaces a stable background-work line
   — verify by capture first.

## Reference / prior art

- Engine semantics and region vocabulary: `src/detect/manifest.rs` — regions
  (`bottom_non_empty_lines(N)`, `whole_recent`, `after_last_prompt_marker`,
  `top_non_empty_lines(N)`, etc.), AND/OR/NOT gates, `priority` (highest wins),
  `visible_idle/blocker/working`, `skip_state_update`, matcher limits
  (`MAX_RULES_PER_MANIFEST`, `MAX_GATE_DEPTH`, ...).
- Patterns to copy: `src/detect/manifests/claude.toml` (`transcript_viewer`,
  `model_picker_menu`, `osc_title_idle`, `osc_progress_idle`, layered blockers)
  and `codex.toml` (`transcript_viewer`, weak/strong blocker tiers, `osc_title_idle`
  with `not` guards against the working spinner).
- Bundled vs published: edit BOTH `src/detect/manifests/kiro.toml` (bundled,
  compiled in via `BUNDLED_MANIFESTS` in `src/detect/manifest.rs`) and
  `distribution/agent-detection/kiro.toml` (remote catalog listed in
  `distribution/agent-detection/index.toml`). AGENTS.md requires the published
  catalog stay aligned with the bundled manifest for a released agent; keep them
  byte-aligned unless an exact compatibility exception is recorded.
- Bump `version` (date-stamped, e.g. `2026.10.10.1`) and `updated_at`. If any new
  rule uses a region/feature introduced after `min_engine_version = 2`, raise
  `min_engine_version` accordingly (codex uses `3`).

## How to capture evidence (required before writing rules — AGENTS.md)

Use the `herdr-throwaway-repro` skill + the disposable-session harness in
`.scratch/research/herdr-windows-throwaway-commands.md`. Drive the real Kiro v3
UI into each target state, then:

- `herdr agent read <pane> --source detection --format text` for the
  bottom-buffer text the engine actually sees.
- `herdr agent read <pane> --source detection --format ansi` when styling /
  alternate-screen behavior matters (OSC title/progress).
- `herdr agent explain <pane> --json` to see which rule matched and why.

Encode invariant visible controls as explicit AND/OR gates. Do not match
whole-pane incidental text. Do not use the user-visible scrollable viewport for
status.

## Local hot-reload loop (AGENTS.md "Agent Detection Updates")

1. Edit `src/detect/manifests/kiro.toml`.
2. Copy it to the override path `~/.config/herdr/agent-detection/kiro.toml`
   (on Windows the equivalent config dir). **Check first whether an override
   already exists; never overwrite a pre-existing override without alignment.**
3. `herdr server reload-agent-manifests` against the session under test.
4. Re-drive the states; confirm with `herdr agent explain`.
5. When correct, remove the temporary override (or restore the prior one
   exactly) so the committed bundled manifest is the source of truth. Mirror the
   final rules into `distribution/agent-detection/kiro.toml`.

## Testing (AGENTS.md)

- Do NOT add tests that classify captured/invented Kiro screens against the
  bundled rules, and do not freeze Kiro's specific rule IDs/priorities. Test the
  **engine** with synthetic manifests + minimal input (parsing, regions, gates,
  priority, skip-state, source precedence) — those live in
  `src/detect/manifest.rs` tests.
- Validate Kiro-specific behavior with the live smoke loop above; record CLI
  version, observed signals, and outcomes for idle/working/blocked/background.
- `just ci` must pass; keep bundled schema validation and `just release-docs-check`
  green (no unpublished-exception drift for an already-released agent).

## Acceptance criteria

- [ ] Evidence captured for each target state via `agent read --source detection`
- [ ] `transcript_viewer`-style `skip_state_update` rule added for Kiro history/menu screens
- [ ] Idle OSC rule added IF Kiro v3 emits one (verified); otherwise documented as hook-only
- [ ] Additional blocked/approval shapes covered where captured
- [ ] Background-work rule added only if a stable signal exists
- [ ] `version` + `updated_at` bumped; `min_engine_version` raised if required
- [ ] Bundled `src/detect/manifests/kiro.toml` and `distribution/agent-detection/kiro.toml` kept aligned
- [ ] Local override removed/restored; `just ci` + release-docs-check pass

## Resolution

Pending implementation.
