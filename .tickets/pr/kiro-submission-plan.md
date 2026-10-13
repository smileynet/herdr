# Kiro integration — submission plan (NOT YET EXECUTED)

Status: drafts ready; **dogfooding first** (a few weeks), then re-evaluate.
Nothing is posted or pushed to herdrdev/herdr until the user says so.

## Contribution-policy gate (re-verify at submission time)

- Acting account: `smileynet`. Currently NOT in `.github/MAINTAINERS` nor
  `.github/APPROVED_CONTRIBUTORS` — BUT `smileynet` authored merged PR #2301
  (kiro idle detection), so may be eligible. Re-check both files at submission
  time (they change).
- If NOT approved: the sanctioned path for this new surface is the **Ideas
  Discussion** (`kiro-integration-discussion-draft.md`), not an implementation
  PR. Unsolicited implementation PRs from non-approved accounts are auto-closed
  (research confirmed #4499/#4363 closed this way regardless of quality).
- If approved: open the PR using `kiro-integration-pr-draft.md`, referencing the
  Discussion. The PR gate must apply the `ai-review`/`greptile-review` label or
  the bots show "Review skipped".

## Dogfooding plan (now → a few weeks)

**Status (2026-10-13): dogfood instrumentation is LIVE.**
- v3 integration installed into `~/.kiro/hooks` (release build); prior v1 hook
  backed up to `~/.kiro/hooks/.herdr-dogfood-backup/`.
- Additive logging hook `~/.kiro/hooks/herdr-dogfood-log.{json,ps1}` writes every
  Kiro v3 lifecycle event to `~/.herdr-kiro-dogfood/events.jsonl`.
- Analysis tooling in `~/.herdr-kiro-dogfood/tools/`
  (`analyze-dogfood.py`, `dogfood-snapshot.ps1`, `dogfood-teardown.ps1`) and the
  `kiro-dogfood-review` agent skill.
- Day-one finding already logged (`~/.herdr-kiro-dogfood/log.md`): the hook maps
  both PreToolUse and PostToolUse to `working`, producing dozens of identical
  working reports per turn — evaluate trimming `KIRO_HOOK_EVENTS` to
  `[SessionStart, UserPromptSubmit→working, Stop→idle]`.

**OPEN DECISION — which server runs the integration during dogfood:**
- A: swap the daily server to a build with the integration (full fidelity, but
  restarts the live session and makes the branch the daily driver). The daily
  binary is a versioned standalone release selected by PATH at
  `~/.herdr/packages/standalone/releases/<ver>` — reversible but invasive.
- B: dedicated long-lived `herdr-dev` session on the release build; do Kiro work
  there (non-destructive; partial coverage).
- C: manifest-only on stable (zero risk; exercises screen detection + captures
  the event log, but NOT hook authority — the stable server rejects
  `herdr:kiro` reports).
The event log is captured under all three; only A/B exercise hook authority.

Run the integration on real daily Kiro usage to shake out issues the harness
couldn't observe:

1. Install the built debug/release integration into the real `~/.kiro/hooks`
   (currently a prior v1 identity-only hook is there — back it up first; our v3
   asset supersedes it via the version bump).
2. Use Kiro v3 normally across sessions and watch the herdr sidebar for:
   - idle/working transitions tracking the real agent state,
   - the `/sessions` picker not flipping state,
   - idle surviving scroll (OSC `4;0`),
   - resumed panes behaving acceptably (screen-detection fallback),
   - no stray notifications / no hook stalls / no context pollution.
3. Note any misclassification or hook misfire; fix on the spike branch.
4. Keep a short dogfood log in `.scratch/research/kiro-dogfood-log.md`.

## Pre-submission checklist (when ready)

- [ ] Dogfood period complete; issues found are fixed and re-validated.
- [ ] Contributor gate re-checked; path (Discussion vs PR) chosen.
- [ ] Branch rebased on current `origin/master` (= `upstream/master`); conflicts
      resolved; our manifest changes still additive to the then-current #4372
      manifest.
- [ ] `just ci` green on a healthy machine (not the memory-pressured one);
      `just check` if acting as maintainer (Windows cross-compile).
- [ ] `.tickets/` and `.scratch/research/` removed from the PR diff. (They live
      on the spike branch for our workflow; strip before the submission branch —
      the user asked for tickets removed pre-PR.)
- [ ] Squash into focused, lowercase conventional commits (see plan below).
- [ ] PR body finalized from `kiro-integration-pr-draft.md`; `refs #<n>` set to
      the Discussion/issue number; Validation section re-run numbers filled in.
- [ ] No root README/CHANGELOG / docs/next/CHANGELOG edits; add
      `docs/next/website/...` user docs if direction approved.

## Proposed final commit structure (strip tickets/scratch first)

Keep it reviewable as focused commits (propose messages, get alignment before
committing — per AGENTS.md/CONTRIBUTING.md):

1. `feat: add kiro cli v3 hook integration`
   - assets/kiro/*, integration/{mod,registry,targets,actions,env,types}.rs,
     cli/integration.rs, detect/mod.rs authority, agent_resume.rs official-source,
     integration/tests.rs.
2. `feat: enrich kiro screen-detection manifest`
   - src/detect/manifests/kiro.toml + distribution/agent-detection/kiro.toml.
3. (fold the 017 session-anchor fix into commit 1 when squashing — it's part of
   the same feature, not a separate fix once history is cleaned for the PR.)

## Known follow-ups to mention in the PR, not block on

- Resume: add launch-args resume parse + `is_official_agent_source` arm so
  resumed Kiro panes re-anchor to hook authority (currently screen fallback).
- Ticket 019: make the resume trust/auto-accept flag (`-a`) configurable.
