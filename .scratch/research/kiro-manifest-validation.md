---
topic: Kiro kiro.toml manifest enrichment — live validation (ticket 016)
date: 2026-10-10
status: complete
confidence: high
harness: disposable herdr named session (hookless KIRO_HOME) in spike worktree w62
---

# Kiro manifest enrichment — live validation (ticket 016)

Captured real `kiro-cli --v3` screens/OSC in a disposable session with a
throwaway `KIRO_HOME` containing **no** herdr hook, so 015's full-lifecycle
authority never paused screen detection. This exercises the pure screen/OSC
fallback path.

## Environment

- herdr `0.9.3`; kiro-cli `2.29.0`.
- Disposable session `repro-kiro-manifest-20261010-203626`, root pane `w1:p1`,
  outer pane `w62:p3`.
- `KIRO_HOME` = throwaway dir with empty `hooks/` (no herdr integration).

## Captured signals per state (verified)

| State | Screen (bottom buffer) | osc_progress | osc_title |
|-------|------------------------|--------------|-----------|
| idle | `› ask a question or describe a task ↵` + `/sessions to resume` | **`4;0`** | `Windows PowerShell` (unset) |
| working | `Kiro is working · Ns · Type to steer · Ctrl+S to queue` | **`4;3`** | `Windows PowerShell` (unset) |
| blocked (tool approval) | `… requires approval` + `❯ Allow / Always allow / Deny / Always deny` + `esc to close · ↑↓ to navigate · ↵ to select · Tab to edit` | `4;4;100` | (unset) |
| sessions picker (`/sessions`) | `↑↓ to navigate · ↵ to select` + `ctrl+p to preview · + to view all controls` | `4;0` | (unset) |

### Key conclusions

1. **Idle OSC is achievable — via progress, not title.** Kiro v3 drives OSC
   progress `4;0` at idle and `4;3` while working (clean, repeatable cycle:
   idle 4;0 → working 4;3 → idle 4;0). This is a scroll-proof idle signal, so an
   `osc_progress_idle` rule (`^4;0`) is valid and useful. This resolves ticket
   016's open idle question (answer: yes, achievable).
2. **`osc_title` is NOT emitted by kiro-cli 2.29.0** — it stays at the shell
   title (`Windows PowerShell`) in every state, so the existing
   `osc_title_working` rule never matches on this version. Kept it anyway
   (append-safe; older/newer Kiro builds may set it), but it is not relied upon.
3. **Blocked already works** — the existing `tool_approval` rule classified the
   real approval screen as `blocked` (matched, `visible_blocker=true`). No new
   blocked rule needed for this shape. (Bonus: `osc_progress 4;4;100` appears
   during approval, distinct from working/idle, but screen detection suffices.)
4. **Sessions picker was reported as `idle` with no matched rule** (fell through
   to the known-agent idle fallback). A navigable list is not a live prompt, so
   a `skip_state_update` rule preserves prior state instead.

## Rules added (version 2026.09.19.1 → 2026.10.10.1)

- **`sessions_picker`** (`state=unknown`, `skip_state_update=true`, priority 1020):
  gates on the picker's list-navigation controls (`↑↓ to navigate` + `to select`
  AND one of `ctrl+p to preview` / `to view all controls` / `tab to switch`),
  with `not` guards excluding the approval footer (`requires approval`,
  `esc to cancel`) so a real tool-approval still wins on priority.
- **`osc_progress_idle`** (`state=idle`, `visible_idle=true`, priority 120):
  `regex ^4;0` on the `osc_progress` region. Lowest-priority idle backstop so any
  live working/blocked frame wins; provides idle that survives viewport scroll.
- Version/`updated_at` bumped; `min_engine_version` left at 2 (no new
  region/feature used). Bundled `src/detect/manifests/kiro.toml` and
  `distribution/agent-detection/kiro.toml` kept byte-identical (sha256 verified).

## Validation performed

- **Offline (`herdr agent explain --file` with the edited manifest as an
  isolated XDG override):**
  - sessions-picker frame → `unknown` + `skip_state_update=true` via
    `sessions_picker` (previously idle/no-match).
  - tool-approval frame → still `blocked` via `tool_approval` (new rule's `not`
    guards let approval win).
- **Live (temporary global override + `herdr server reload-agent-manifests`
  against the running session):**
  - Reload reported `active_version 2026.10.10.1`,
    `local_override_shadowing_remote=true`.
  - At idle, `osc_progress_idle` evaluated with region `4;0` and `matched=true`;
    overall state still `idle` (prompt-line rule wins at 1100, OSC rule is the
    scroll-proof backstop at 120 — intended layering).
  - Working classification unchanged (earlier in the same session: footer
    `Kiro is working` + `osc_progress 4;3` matched `osc_progress_working`).
  - Temporary global override **removed** afterwards; reload confirmed Kiro
    reverted to the remote manifest (`2026.09.19.1`, no override). No
    pre-existing override was present or clobbered.
- **Manifest parse:** `tomllib` parses the file; 10 rules; the live herdr engine
  compiled every gate/regex when it loaded the override (proves schema validity).

## Known gap in this run

- The full `herdr` **test binary** could not be compiled on this machine during
  this session: `rustc` crashed with `STATUS_STACK_BUFFER_OVERRUN` (0xc0000409)
  while codegen-linking the large test harness, after a prior memory-pressure
  event. This is an environment/toolchain resource limit, not a code defect: the
  production `cargo build` and earlier test suites (integration, endpoint
  contract, Kiro unit tests) compiled and passed. The manifest change is pure
  TOML and was validated live against the running engine. The bundled-manifest
  schema-validation unit test and `just ci` should be re-run in a healthy
  environment (CI / maintainer machine) before the PR — recorded as a carry-over.

## Cleanup

Temporary global override removed and reversion confirmed; disposable session
stopped/deleted; outer pane closed; repro dir removed; harmless probe file from
the rejected approval verified absent. Installed hooks untouched (015 asset not
involved; KIRO_HOME was a throwaway).
