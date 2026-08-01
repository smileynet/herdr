---
title: "Add tests for kiro integration"
status: done
priority: high
blocked_by: ["004"]
---

# Add tests for kiro integration

## What to build

Add unit tests for the kiro integration covering install, uninstall, version detection, and registry behavior. Follow the existing test patterns in `src/integration/tests.rs`.

## Details

### Tests to add

1. **Install test** — verify hook script is written to correct path, is executable, contains version marker
2. **Uninstall test** — verify hook file is removed, result struct reports correctly
3. **Version detection** — verify `parse_integration_version` correctly reads the HERDR_INTEGRATION_VERSION from the kiro hook script
4. **Registry** — verify kiro appears in `integration_recommendations()` when `kiro-cli` is available, and `integration_status_at` works correctly
5. **Directory resolution** — verify `kiro_dir()` respects `KIRO_HOME` env var override
6. **Not-installed state** — verify correct behavior when `~/.kiro` doesn't exist

### Test patterns to follow

Look at existing tests in `src/integration/tests.rs` — they use:
- `integration_env_lock()` for env var serialization
- Temporary directories for isolated file operations
- The `IntegrationStatus` / `IntegrationStatusKind` assertions

## Acceptance criteria

- [ ] Install test passes
- [ ] Uninstall test passes
- [ ] Version parsing test passes
- [ ] Directory resolution with KIRO_HOME override test passes
- [ ] All tests pass with `cargo test`
- [ ] No test flakiness from env var races (uses lock)
