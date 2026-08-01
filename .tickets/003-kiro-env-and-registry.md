---
title: "Wire kiro-cli directory resolution and registry"
status: done
priority: high
blocked_by: ["001"]
---

# Wire kiro-cli directory resolution and registry

## What to build

Add directory resolution for kiro-cli's config path and register Kiro in the integration registry (labels, command names, platform support, specs array).

## Details

### Directory resolution (`env.rs`)

Add `kiro_dir()`:
- Check env var `KIRO_HOME` first (kiro-cli's documented override)
- Fall back to `~/.kiro`
- Integration hook installs to: `{kiro_dir}/agents/herdr-agent.json` (the agent config file that registers the hook)

```rust
pub(crate) const KIRO_HOME_ENV_VAR: &str = "KIRO_HOME";

pub(crate) fn kiro_dir() -> io::Result<PathBuf> {
    config_dir_from_env_or_home(KIRO_HOME_ENV_VAR, &[".kiro"])
}
```

### Registry (`registry.rs`)

Add match arms for `IntegrationTarget::Kiro`:
- `integration_target_label` → `"kiro"`
- `integration_target_command_names` → `&["kiro-cli"]`
- `integration_target_supported` → supported on all platforms (Windows + Unix)
- `integration_target_available` → check if `kiro-cli` is on PATH

Add entry to `integration_specs()` array (bump size from 15 to 16):
```rust
(
    crate::api::schema::IntegrationTarget::Kiro,
    kiro_dir().map(|dir| dir.join("agents").join(super::KIRO_AGENT_CONFIG_INSTALL_NAME)),
    super::KIRO_INTEGRATION_VERSION,
)
```

Note: version detection is based on parsing `HERDR_INTEGRATION_VERSION=` from the installed agent config file. Since the hook script is referenced from the agent config, we store the version marker in the agent config or in the hook script itself. Need to decide which file the spec entry points to — likely the hook script file for version checking consistency.

### Constants (`mod.rs`)

```rust
const KIRO_HOOK_INSTALL_NAME: &str = if cfg!(windows) {
    "herdr-agent-state.ps1"
} else {
    "herdr-agent-state.sh"
};
const KIRO_HOOK_ASSET: &str = if cfg!(windows) {
    include_str!("assets/kiro/herdr-agent-state.ps1")
} else {
    include_str!("assets/kiro/herdr-agent-state.sh")
};
const KIRO_INTEGRATION_VERSION: u32 = 1;
```

## Acceptance criteria

- [ ] `kiro_dir()` resolves `KIRO_HOME` or falls back to `~/.kiro`
- [ ] `KIRO_HOME_ENV_VAR` constant defined
- [ ] All registry match arms for `Kiro` filled in
- [ ] `integration_specs()` array includes Kiro entry
- [ ] Constants for hook name, asset, and version defined in `mod.rs`
- [ ] `cargo check` passes
