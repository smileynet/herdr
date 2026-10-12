# DRAFT — GitHub Discussion (Ideas category) — NOT POSTED

> Internal draft for review. Do not post until (a) we've dogfooded the build for
> a few weeks and (b) the acting account's contributor status is confirmed.
> Target category: Ideas. Keep ~150-200 words, casual/lowercase, problem-first.
> Matches `.github/DISCUSSION_TEMPLATE/ideas.yml` (three required sections).

---

**Title:** `[idea] deepen kiro cli v3 support: hook-based working/idle state`

### idea / problem

i run kiro cli v3 in herdr panes alongside claude, codex, and opencode. herdr
already ships a `kiro.toml` detection manifest, so panes get labeled and screen
detection mostly works — but kiro's live state is still inferred only from the
visible prompt. idle is lost when the prompt scrolls off, and there's no
authoritative working/idle signal the way the hook-integrated agents have. so
kiro is a second-class citizen in the sidebar when i'm triaging several agents
at once.

### requested change

deepen kiro to the same tier as kimi/claude/codex:

- an optional `herdr integration install kiro` hook that reports authoritative
  working/idle (kiro v3 exposes standalone `.kiro/hooks` events —
  SessionStart / UserPromptSubmit / PreToolUse / PostToolUse / Stop)
- keep screen detection as the no-hook fallback, and enrich `kiro.toml` with a
  scroll-proof idle signal (kiro v3 drives OSC progress `4;0` idle / `4;3`
  working) and a skip-state rule for the `/sessions` picker

### why you want this

the whole reason i use herdr is to see which of many agents needs me without
polling each pane. right now kiro's idle/working state is less reliable than the
other agents', so i miss when a kiro pane is actually done or waiting. a hook
integration (same model herdr already uses) makes kiro's state trustworthy in
the sidebar.

---

happy to open a PR if a maintainer approves the direction and the contributor
gate — i've prototyped it locally against kiro-cli 2.29.0 and am dogfooding it.

*(drafted with an AI agent that i direct and review.)*
