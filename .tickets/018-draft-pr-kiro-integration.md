---
id: "018"
title: "Draft the Kiro integration PR (research prior art + write description)"
status: open
priority: medium
blocked_by: ["015", "016", "017"]
---

# Draft the Kiro integration PR (research prior art + write description)

## Goal

Prepare a submission-ready pull request for the Kiro CLI v3 integration from the
`smileynet/herdr` fork against `herdrdev/herdr`, following Herdr's exact PR and
code conventions — AND respecting Herdr's contribution policy. This ticket
produces a DRAFT and the supporting research; it does NOT authorize pushing a PR
to the canonical repo on its own.

## CRITICAL: contribution-policy gate (read before anything else)

Per `CONTRIBUTING.md` and `AGENTS.md` "External contributor guardrail":

- Herdr does **not** accept unsolicited implementation PRs. Only verified
  maintainers (`.github/MAINTAINERS`) or accounts in
  `.github/APPROVED_CONTRIBUTORS` may open an implementation PR; everything else
  is auto-closed.
- The acting account here is `smileynet`. Before drafting-for-submission,
  **verify**: `gh auth status`, whether `smileynet` is in
  `.github/APPROVED_CONTRIBUTORS` or `.github/MAINTAINERS`, and whether this is a
  private/custom fork. If NOT approved:
  - Do **not** open an implementation PR to `herdrdev/herdr`.
  - This is a feature/new-surface, so the allowed path is a **GitHub Discussion**
    proposing the integration (short, human-written, problem-first), not an issue
    and not a PR. An agent may open an *issue* only for a reproduced bug — this is
    not that.
  - The draft PR can still live on the fork (`smileynet/herdr`) for the
    maintainer-controlled process to pick up, and as the artifact behind a
    Discussion.
- Do not manufacture an issue to justify this work. Do not claim passing tests or
  a small diff creates an exception.

## Research to do (prior art for code + PR writing)

1. **Merged integration PRs in this repo.** Study how previous agent
   integrations landed to copy structure, commit shape, and review handling:
   ```
   gh pr list --repo herdrdev/herdr --state merged --search "integration" --limit 30
   gh pr list --repo herdrdev/herdr --state merged --search "kimi OR droid OR grok OR mastracode OR hook" --limit 30
   ```
   For the closest matches (adding a NEW agent integration), read the diff and PR
   body: `gh pr view <n> --repo herdrdev/herdr`, `gh pr diff <n> --repo herdrdev/herdr`.
   Note: which files a new integration touches, how `IntegrationTarget` additions
   were framed re the frozen-enum contract, how `*_INTEGRATION_VERSION` was set,
   test expectations, and how the title/body were written.
2. **Any prior Kiro-specific PRs/branches.** The fork had a `kiro-integration-pr`
   branch referenced in ticket 014. Check:
   `gh pr list --repo herdrdev/herdr --search "kiro" --state all` and
   `git branch -a | Select-String kiro`. Learn from what was tried/closed.
3. **Feature-discussion research (the approved path for a new surface).**
   This integration is a new product surface, so the primary allowed channel for
   an unapproved contributor is a **GitHub Discussion**, not an issue or PR
   (`.github/ISSUE_TEMPLATE/config.yml` disables blank issues and routes feature
   requests/ideas/contribution proposals to Discussions; the bug issue template
   is only for reproduced bugs). Research before writing it:
   - Read `.github/DISCUSSION_TEMPLATE/ideas.yml` — the `[idea]` template has
     exactly three required fields: **idea / problem**, **requested change**,
     **why you want this**. Draft the Discussion to those fields, short and
     human-written, problem-first; do NOT paste the implementation, diffs, or
     agent investigation dumps (CONTRIBUTING.md "Start a Discussion").
   - Search existing Discussions for prior Kiro / integration / agent-detection
     proposals so this does not duplicate and can reference prior interest:
     ```
     gh search prs --repo herdrdev/herdr kiro   # cross-check closed attempts
     gh api 'repos/herdrdev/herdr/discussions?per_page=100' --jq '.[] | {num:.number,title:.title,cat:.category.name,state:.state}' 2>&1
     # or browse https://github.com/herdrdev/herdr/discussions and filter: kiro, integration, hook, detection
     ```
     If `gh` lacks a discussions subcommand in this environment, use the GraphQL
     API via `gh api graphql` to list `repository.discussions`, or fetch the
     Discussions web page. Record discussion numbers/titles/categories found.
   - Study how prior accepted integrations were proposed and discussed (what
     framing got maintainer buy-in, what scope questions were asked, how
     product-fit was argued) so the Kiro Discussion mirrors that tone. Note:
     upvotes/comments show interest but do not guarantee implementation,
     priority, or permission to open a PR (CONTRIBUTING.md).
   - Capture findings in the draft (below) as a "prior discussion" subsection:
     what exists, what was accepted/declined, and how the Kiro proposal relates.

4. **Project conventions** (already in-repo, cite in the PR body):
   - `CONTRIBUTING.md`: one PR = one accepted problem, no bundled cleanup,
     lowercase conventional title (`feat: ...`), `refs #<issue>` in body (NOT
     `fixes/closes/resolves`), `just ci` must pass, don't touch root README/
     CHANGELOG/`docs/preview`/`docs/versions`/`distribution/` release payloads.
   - `AGENTS.md` Commit Style: lowercase conventional, no emojis, no AI
     co-author lines, descriptive subject.
   - `AGENTS.md` Testing + Agent Detection Updates: engine tests + live smoke
     validation; `just check` includes Windows cross-compile.

## Draft artifacts to produce

- `.tickets/pr/kiro-integration-discussion-draft.md` containing a GitHub
  Discussion (`[idea]` category) written to the three required template fields
  from `.github/DISCUSSION_TEMPLATE/ideas.yml`:
  - **idea / problem**: Kiro CLI panes only get state from screen scraping, so
    idle is lost on scroll, history/menu screens misclassify, and there is no
    authoritative session identity/resume or reliable attention notification —
    unlike claude/codex which have hook integrations.
  - **requested change**: a first-party Kiro v3 hook integration (like
    kimi/droid/claude) plus enriched `kiro.toml` fallback rules.
  - **why you want this**: concrete user impact (accurate idle/working/blocked,
    attention sounds that fire correctly, resume support when managing many Kiro
    panes).
  Keep it short and human-written; no diffs, no agent dumps. Include a "prior
  discussion" note summarizing any existing related Discussions found.
- `.tickets/pr/kiro-integration-pr-draft.md` containing:
  - **Title** (lowercase conventional): e.g. `feat: add kiro cli v3 hook integration`.
  - **Summary**: the problem (Kiro was screen-only), the two-part fix (hook
    integration 015 + manifest enrichment 016), and the user-visible result
    (accurate idle/working/blocked, session identity/resume, notifications).
  - **What changed**: bulleted file-level inventory (assets, `integration/mod.rs`,
    `registry.rs`, `detect/mod.rs` authority, both `kiro.toml` manifests).
  - **Contract note**: how the `IntegrationTarget` addition respects the frozen
    client endpoint enum (append-only / or experimental-letta path), citing
    AGENTS.md.
  - **Testing**: engine/plumbing unit tests + the live validation from 017
    (Kiro CLI version, signals, transitions, resume), confirmation `just ci`
    passed, and the Windows cross-compile note for maintainers.
  - **Scope/alignment**: explicit statement that this is a new surface requiring
    maintainer alignment, and that it was prepared through the approved path.
  - A short "how to verify" section for a reviewer.
- A matching **commit plan**: the sequence of focused commits (asset+wiring;
  manifest; tests), each with a proposed lowercase conventional message. Per
  AGENTS.md/CONTRIBUTING, **propose commit messages and get alignment before
  committing**. Keep tickets/`.scratch` research OUT of the final PR (strip
  `.tickets/` and `.scratch/research/` before the submission commit — the user
  asked to remove them before final PR).

## Pre-submission checklist (only if approved to submit)

- [ ] `smileynet` confirmed in `.github/APPROVED_CONTRIBUTORS` (or maintainer); else STOP, use Discussion
- [ ] Branch rebased on current `upstream/master`; `just ci` green on the new head
- [ ] `.tickets/` and `.scratch/research/` removed from the PR diff
- [ ] One focused problem; no unrelated cleanup; lowercase conventional title
- [ ] No edits to root README/CHANGELOG/docs/preview/docs/versions/distribution release payloads
- [ ] Draft PR body + commit plan reviewed and approved by the user
- [ ] (If applicable) Discussion opened proposing the integration

## Acceptance criteria

- [ ] Prior merged integration PRs researched; findings summarized with PR numbers
- [ ] Any prior Kiro PR/branch history reviewed
- [ ] Existing related Discussions searched; relevant discussion numbers/titles recorded
- [ ] `.tickets/pr/kiro-integration-discussion-draft.md` written to the `ideas.yml` three-field template, short and human-written
- [ ] `.tickets/pr/kiro-integration-pr-draft.md` written (title, summary, changes, contract note, testing, verify steps)
- [ ] Commit plan with proposed messages produced and left for user alignment
- [ ] Contribution-policy gate evaluated and the correct path (Discussion vs approved PR) chosen and documented
- [ ] Plan to strip tickets/scratch before final PR documented

## Resolution

Pending. Blocked by 015, 016, 017.
