---
title: "Implement install_kiro and uninstall_kiro"
status: done
priority: high
blocked_by: ["002", "003"]
---

# Implement install_kiro and uninstall_kiro

## What was built

Install writes two files to `~/.kiro/hooks/`:
1. **Hook script** (`herdr-agent-state.sh` / `.ps1`) — the executable that reports session ID to herdr
2. **Hook config** (`herdr-agent-state.json`) — v3 standalone hook definition that tells kiro-cli when to run the script

Uninstall removes both files.

## Key design decisions

- **v3 global hooks**: The hook config uses kiro-cli v3's standalone format (`~/.kiro/hooks/*.json`). This requires `kiro-cli --v3` mode.
- **v2 cannot use global hooks**: Global hooks are v3-only (confirmed in 2.13 changelog). v2 hooks are embedded in agent configs, but shadowing `kiro_default` breaks tool dispatch entirely.
- **Install message**: Explicitly states "session reporting requires kiro-cli v3 mode" so users know.
- **Detection still works in v2**: The screen detection manifest (`kiro.toml`) handles working/blocked/idle without hooks. Only session ID reporting needs v3.

Write the `install_kiro()` and `uninstall_kiro()` functions in `targets.rs` and wire them into `actions.rs`.

## Details

### Installation mechanism (v2 hooks via agent config)

kiro-cli v2 hooks are embedded in agent configuration files at `~/.kiro/agents/<name>.json`. The install function will:

1. Resolve kiro config dir via `kiro_dir()`
2. Verify `~/.kiro` exists (error if kiro-cli not installed)
3. Create `~/.kiro/agents/` if it doesn't exist
4. Write the hook script to `~/.kiro/hooks/herdr-agent-state.sh` (or `.ps1` on Windows)
5. Make the script executable (Unix)
6. Write/update `~/.kiro/agents/herdr-agent.json` with an agent config that registers the hook

### Agent config file (`herdr-agent.json`)

The minimal agent config that registers the herdr hook:

```json
{
  "name": "herdr-agent",
  "description": "Herdr integration — reports session state to herdr server",
  "hooks": {
    "agentSpawn": [
      {
        "command": "~/.kiro/hooks/herdr-agent-state.sh"
      }
    ]
  }
}
```

Wait — this approach has a problem: kiro-cli agent configs are for *custom agents*, not for global hooks on the default agent. The v2 hook format requires hooks to be defined per-agent.

### Revised approach: Global agent hooks

Looking at how other integrations work with similar tools: the correct approach is to register the hook in kiro-cli's **default agent config** or use the global hooks directory.

Actually, re-reading the kiro-cli docs: v2 hooks ARE embedded in agent configs, but the default agent (`kiro_default`) uses the same format. The hook needs to be in the agent config that's actually used.

**Better approach**: Install the hook script and write a dedicated hooks config that the user's agent configs can reference. OR: modify the user's existing agent config to include the hook.

**Simplest viable approach (following Claude's pattern)**: 
- Write the hook script to `~/.kiro/hooks/herdr-agent-state.sh`
- Since kiro-cli v2 doesn't have a standalone hooks directory for auto-loading, the integration will need to inject the hook into the default agent config OR wait for global hooks support.

**Recommended approach for v2**: Install to a well-known path and output instructions for the user to add the hook to their agent config. Many other herdr integrations (Claude, Codex) modify a settings file — we should do the same.

**Final approach**: Look for `~/.kiro/settings/cli.json` or create a global agent that includes just the hook. Since kiro-cli v2.13+ supports global hooks (`~/.kiro/hooks/`), and our minimum version target is v2.0, we need a fallback.

**Decision**: Target kiro-cli ≥ 2.13 (global hooks support). The installer:
1. Writes script to `~/.kiro/hooks/herdr-agent-state.sh`
2. That's it — global hooks auto-fire on all agents

If targeting earlier v2, the installer would need to modify agent configs. Since 2.13 is recent and the global hooks dir is the clean path, target that.

### Revised plan

Install:
1. Resolve `kiro_dir()` → `~/.kiro`
2. Verify directory exists
3. Create `hooks/` subdirectory
4. Write hook script to `~/.kiro/hooks/herdr-agent-state.sh` (or `.ps1`)
5. Make executable (Unix)
6. No settings file modification needed — global hooks auto-load

Uninstall:
1. Remove `~/.kiro/hooks/herdr-agent-state.sh` (or `.ps1`)

### Types (`types.rs`)

```rust
pub(crate) struct KiroInstallPaths {
    pub hook_path: PathBuf,
}

pub(crate) struct KiroUninstallResult {
    pub hook_path: PathBuf,
    pub removed_hook_file: bool,
}
```

### Actions (`actions.rs`)

Add `Kiro` match arms to `install_target_inner` and `uninstall_target`. Import `install_kiro` and `uninstall_kiro`.

## Acceptance criteria

- [ ] `install_kiro()` writes hook script to `~/.kiro/hooks/`
- [ ] `install_kiro()` makes script executable on Unix
- [ ] `install_kiro()` errors clearly if `~/.kiro` doesn't exist
- [ ] `uninstall_kiro()` removes the hook script
- [ ] `KiroInstallPaths` and `KiroUninstallResult` structs exist in `types.rs`
- [ ] `actions.rs` dispatches to install/uninstall for `Kiro` target
- [ ] `cargo check` passes
- [ ] Integration version parsed correctly from installed script
