---
title: "Add Kiro variant to IntegrationTarget enum"
status: done
priority: high
blocked_by: []
---

# Add Kiro variant to IntegrationTarget enum

## What to build

Add `Kiro` to the `IntegrationTarget` enum and update all match arms that exhaustively match it.

## Details

- Add `Kiro` variant to `src/api/schema/integrations.rs` enum
- Bump `ALL` array size from 15 to 16 and include `Self::Kiro`
- Add placeholder match arms in all exhaustive matches (registry, actions, etc.) — these will be filled in by subsequent tickets

## Acceptance criteria

- [ ] `Kiro` variant exists in `IntegrationTarget`
- [ ] `IntegrationTarget::ALL` includes `Kiro`
- [ ] Code compiles with `cargo check` (all match arms covered, even if placeholder)
