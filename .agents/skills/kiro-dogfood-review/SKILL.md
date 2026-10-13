---
name: kiro-dogfood-review
description: Review and analyze the Kiro v3 integration dogfood data (hook event log + herdr state snapshots) and judge the integration's effectiveness. Use when the user says "review the dogfood", "analyze kiro effectiveness", "how is the kiro integration doing", or asks for a dogfood status/report before deciding to submit the Kiro integration.
---

# Kiro v3 integration dogfood review

Use this skill to turn the raw dogfood capture into an effectiveness judgment for
the Kiro v3 herdr integration (spike branch `spike/kiro-v3-hook-harness`), so we
can decide whether it is ready to propose upstream.

Data lives at `C:\Users\uosmi\.herdr-kiro-dogfood\` (local, not committed):
- `events.jsonl` — every Kiro v3 lifecycle hook fire (from the additive logging
  hook `~/.kiro/hooks/herdr-dogfood-log.*`).
- `snapshots/*.jsonl` — `herdr agent list --json` captures.
- `log.md` — the human journal of observed behavior.

## Steps

1. **Run the analyzer** and read its output:
   `python C:\Users\uosmi\.herdr-kiro-dogfood\tools\analyze-dogfood.py --json`
   (add `--since <ISO8601>` to scope to a recent window).
   It reports event counts by action/hook_event, turns completed,
   working→idle latency, duplicate/flapping reports, likely resumed sessions
   (expected to fall back to screen detection), snapshot kiro status counts, and
   flagged anomalies.

2. **Cross-check hook vs detected state.** For sampled windows, compare the
   event stream (what the hook reported) against the snapshot `agent_status`
   (what herdr showed). Call out any divergence and classify it:
   - missed transition (real state change, no event / no status change)
   - wrong state (hook said working but Kiro was idle, or vice versa)
   - stale (status lagged the real transition — note the latency)
   - flap (rapid back-and-forth)

3. **Check the known boundaries behave as designed:**
   - `/sessions` picker and history screens must NOT flip live state
     (skip_state_update) — look for idle/working churn during navigation.
   - idle must survive scrolling (OSC `4;0` backstop) — no false "working" when
     the prompt scrolls off.
   - resumed panes (`--resume-id`) fall back to screen detection (no
     SessionStart event) — the analyzer lists these; confirm they still showed
     correct state via the manifest.
   - `blocked` comes from screen detection, not the hook — confirm tool-approval
     prompts still surfaced as blocked.

4. **Read `log.md`** for human-observed issues the data can't show (spurious
   notifications/sounds, context pollution, hook stalls).

5. **Judge readiness** with a short verdict table:

   | Dimension | Result | Evidence |
   |---|---|---|
   | Coverage (events per real transition) | ✅/⚠️/❌ | counts, missed transitions |
   | Accuracy (hook vs truth) | ✅/⚠️/❌ | mismatch count + classes |
   | Latency (working→idle) | ✅/⚠️/❌ | median/max seconds |
   | Boundaries (picker/scroll/resume/blocked) | ✅/⚠️/❌ | observations |
   | Noise (notifications/flap) | ✅/⚠️/❌ | duplicate count, journal |

   Then state: ready to propose / needs fixes (list them, each tied to a file in
   the spike branch) / needs more dogfood data (say what's missing).

## Rules

- This reviews DATA; it does not modify the integration. If fixes are needed,
  name the exact file in the spike branch (`src/integration/assets/kiro/*`,
  `src/detect/manifests/kiro.toml`, `src/agent_resume.rs`, etc.) and hand off.
- Do not post anything to `herdrdev/herdr`. Dogfood is pre-submission only.
- Treat small sample sizes honestly — if there are few turns/sessions logged,
  say the data is insufficient rather than over-concluding.
- The logging hook is additive and separate from the shipped asset; never cite
  it as part of the integration being proposed.
