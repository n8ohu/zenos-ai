# Release Notes — Codename Ready Player Two (RP2)

**Branch:** `feat/ready-player-2 → main`
**Follows:** Meridian (`8fbf10c`, 2026-03-21)

---

## Summary

Ready Player Two is the identity and lifecycle release. Three major feature areas:

1. **Cabinet provisioning system** — mount-aware state machine, RP2 provisioner, expansion slots, scroll ceremony. Cabinets now have a proper online_unmounted / online_mounted lifecycle instead of raw HA template state.
2. **Warmup as first-class sensor state** — boot noise eliminated. The 5-minute post-restart window is now a named state (`warmup`) with its own sensor values, UX messaging, and explicit expiry signal.
3. **Identity model v4.3.0** — full family/household group management. Eleven new modes on `zen_dojotools_identity`. Household/family/member graph with principal slots, delegation links, depth-2 security resolution, and an enriched identity manifest with tree view.

---

## Cabinet Provisioning System

### Mount-Aware State Machine

Cabinets now track lifecycle as a first-class sensor state:

| State | Meaning |
|---|---|
| `online_unmounted` | Cabinet exists, not assigned to an identity slot — sits in stacks, ready to provision |
| `online_mounted` | Cabinet is active and assigned to an identity slot via a type label |

The manifest and resolver sensors gate on `online_mounted` only — stacks cabinets are invisible to the identity layer until provisioned.

### RP2 Provisioner (`zen_dojotools_provisioner`)

Three-mode provisioner for full identity lifecycle management:

| Mode | What It Does |
|---|---|
| `provision` | Pulls an `online_unmounted` cabinet into service. Validates GUID uniqueness, applies type label, mounts, preloads profile data, fires identity manifest rebuild. Rolls back on health gate timeout. |
| `deprovision` | Dismounts cabinet, strips type label, returns it to stacks. Blocked if cabinet holds a `zen_default_*` label. |
| `replace` | Deprovisions `replace_cabinet`, then provisions `target_cabinet` with the same `cab_type`. Atomic swap. |

Auto-stamp on `init` state — new cabinet sensors with no VolumeInfo are detected and stamped automatically before provision.

### Expansion Cabinet Slots

Five expansion cabinet sensor definitions added (`v4.5.1`). All 20 cabinet entity IDs aligned with friendly names. Vault intentionally excluded from auto-init.

### Scroll Ceremony (`zen_scroll`)

Read-only drawer access pattern for mounted cabinets. Formalizes the distinction between read and write access at the tool level.

---

## Warmup as First-Class Sensor State

### Problem

During the 5-minute post-restart warmup window, users saw a wall of red with no indication it was normal boot behavior. `sensor.zen_flynn_health` hit `warn`, `sensor.zen_agent_health` hit `warn`, and `binary_sensor.flynn_system_ready` went `off`.

### Solution

`warmup` is now a named sensor state, sitting between `error` and `warn` in the priority chain:

```
Priority: critical > error > warmup > warn > disabled > ok
```

During the boot window, if any health sensor is in a degraded state that is consistent with normal startup, the affected sensors surface `warmup` instead of `warn`. Dashboard shows **ZenOS — WARMUP** with a "settling after restart, no action needed" message. No false alerts.

### `zen_warmup_timer`

Automation added to `flynn.yaml`. Fires `zen_event kind: warmup_expired` exactly 5 minutes after `ha_start`. Flynn re-evaluates all gates on this event. If real issues remain after warmup expires, `warn`/`error` surfaces with the correct actionable message. If all clear, transitions to `ok`.

### `binary_sensor.flynn_system_ready`

Monastery `unavailable` during the warmup window is now non-blocking. Trigger-based health sensors start `unavailable` at boot — this is not a real failure. After warmup expires, `unavailable` on monastery is a real problem and gates `system_ready` off as expected.

### UX After These Changes

| Boot phase | Dashboard header | System Ready | Next step message |
|---|---|---|---|
| 0–5 min post-restart | ZenOS — WARMUP | on | "Settling after restart — no action needed" |
| 5+ min, all ok | ZenOS — OK | on | All gates green |
| 5+ min, cab warn | ZenOS — WARN | on | "Cabinet upgrade available — not urgent" |
| 5+ min, real error | ZenOS — ERROR | off | Actionable message |

---

## Identity Model v4.3.0

### Overview

`zen_dojotools_identity` grew from a 3-mode resolver into a full identity and group management tool. Eleven new modes added; the original `resolve`, `prompt`, and `build_identity_manifest` modes are unchanged.

See [zen_dojotools_identity_readme.md](zen_dojotools_identity_readme.md) for full mode reference and [user_management.md](../getting_started/user_management.md) for operational procedures.

### New Modes

| Mode | What It Does |
|---|---|
| `household_add_family` | Wires a family into the household members list |
| `household_remove_family` | Removes a family from the household members list |
| `household_add_member` | Adds user or AI; fills HoH/prime principal slot on first add |
| `household_remove_member` | Removes user or AI; warns if principal slot is now stale |
| `set_principal` | Replaces the HoH or prime AI slot for any container cabinet |
| `family_add_member` | Adds user, AI, or sub-family; sets default_family_guid on first join |
| `family_remove_member` | Removes member; clears default_family_guid if was default |
| `link_partners` | Bidirectional delegation link — `acls.partner[]` on both entities |
| `unlink_partners` | Symmetric removal from `acls.partner[]` on both sides |
| `set_default_family` | Explicitly patches `default_family_guid` on a member's VolumeInfo |
| `membership` | Tree view for containers; reverse lookup for members |
| `is_member` | Depth-2 boolean membership check `{is_member, depth: 1|2|null}` |

### Principal Slots

Each container has two privileged occupant slots:
- `acls.owner` — Head of Household (filled by first `user` added)
- `acls.partner[role=prime]` — Prime AI partner (filled by first `ai_user` added)

Slots block re-entry once filled. Use `set_principal` to transfer. Remove operations warn via `principal_warning` field rather than blocking — operator may need emergency removal.

### Delegation Model

`acls.partner[]` is the delegation authority record. It works for any entity pair (user↔user, user↔AI, AI↔AI). "Partner" in this model means: authorized to delegate on your behalf. This is a governance relationship — no token is issued without an explicit allow. The link records who has delegation authority; it does not grant it automatically.

### Multi-Household

All household operations accept an explicit `household_entity` parameter. When omitted, operations target `zen_default_household_cabinet_resolved`. A system can host multiple households; each manages its own membership independently.

### Enriched Identity Manifest

`build_identity_manifest` now writes `{roster, tree}` to the `zen_identity_manifest` drawer. The `tree` field is the depth-2 household membership structure including principal slots, member lists, and nested families. `identity_manifest_loader()` in `zen_os_1.jinja` updated to handle both the new `{roster, tree}` shape and the legacy roster-only format.

### Depth Rule

Families nest arbitrarily deep in the data model. Security resolution chases 2 levels only — `is_member(A, B)` checks direct membership (depth 1) and membership-via-sub-family (depth 2). Anything deeper is not resolved for security purposes.

### Flynn Bootstrap Wiring

`flynn_bootstrap_content` now wires household membership at bootstrap:
- `household_add_member` for the primary user (fills HoH slot)
- `household_add_member` for the AI user (fills prime slot)
- `link_partners` between user and AI (bidirectional delegation)
- `build_identity_manifest` to write the initial manifest

All calls are idempotent — guarded against unavailable resolvers and blocked on slot-occupied for existing installs.

---

## Flynn

### Gate 2.5 — Mount-Aware Mode

Flynn now triggers on `cabinet_mounted` and `cabinet_dismounted` events in addition to health sensor changes. Gate 2.5 runs after cabinet initialization — checks that required cabinets are `online_mounted` before proceeding. Stacks cabinets (online_unmounted) do not block.

### Silent OK Handoff (`input_boolean.zen_silent_ok_handoff`)

Added to `flynn.yaml`. When `on`, suppresses the Gate 4 "System Ready" notification on clean restarts. Useful once the system is established — the notification is meaningful on first boot, noise on every subsequent restart.

### Warmup Indicator

Flynn fires a `flynn_warmup` persistent notification during the warmup window. Dismissed automatically when warmup expires and all gates are green.

---

## Cabinet Health

### `warn` = Non-Blocking Across the Stack

Cabinet `warn` (legacy schema detected) is non-blocking throughout the health stack. The system is fully operational with a `warn` cabinet — schema upgrade is available but not urgent. `binary_sensor.flynn_system_ready` stays `on` with a `warn` cabinet.

Gate 2 behavior on `warn`:
- Inside warmup window or `ha_start` → logged only, no notification, continues
- Outside warmup window → non-urgent "Cabinet Upgrade Available" notification, continues

### `reason` Attribute

All health sensors now carry a `reason` attribute describing the current state in plain language. Surfaces the same text as `next_step` for the active condition.

### Health Report

`sensor.zen_cabinet_health` gains `cabinet_states` attribute — per-cabinet init/online state map, readable at runtime without a FileCabinet call.

---

## Infrastructure

### FileCabinet — Wildcard Read

`action_type: read` with `key: '*'` returns all drawer values for a cabinet. Previously required knowing the drawer key in advance.

### Dispatcher (`zen_event` dispatch layer)

Event-driven tool dispatch layer added. Fires the appropriate DojoTool script in response to structured `zen_event` payloads. Reduces automation boilerplate for event-triggered operations.

### Inspect + Index — Linked Help Contract

`zen_dojotools_inspect` now has a `mode: help` that returns a structured capability contract. `zen_dojotools_index` passthrough added — index callers can reach inspect help through the index tool.

### Boot Wipe Guard

Three-layer guard prevents cabinet variable wipe on HA restart. `ha_start` trigger removed from cabinet state template sensors — the boot-touch event pattern replaced with a safer GUID-gated init sequence.

### Volume Redirector Mode

Volume Redirector automation mode corrected to `queued` with `max: 40` (2× active cabinet count). Default `single` mode silently dropped concurrent write events on installs with multiple active cabinets.

---

## Docs

- **Getting Started docs pass** — all 4.5.x docs updated for RP2 terminology and procedure
- **`user_management.md`** — new: full Household and Family Management section with all 8 group management operations, onboarding sequence, delegation model framing
- **`zen_dojotools_identity_readme.md`** — full rewrite for v4.3.0 (15 modes documented)
- **RC2 kill** — RC2 legacy docs removed

---

## HALMark

All code on this branch was swept against the full ZenOS footgun registry and HALMark Ratified + Candidate FG set prior to the UAT sign-off at `d3e0783`. The identity model additions (commits after UAT) follow the same FG-38 normalization pattern used throughout the codebase:

- VolumeInfo reads use the canonical `_v → _vi → .get('value', _vi) → is mapping` chain
- `state_attr` in `variables:` blocks guarded with `| default({}, true)` + `is mapping` checks
- `link_partners` / `unlink_partners` use `from_json if is string else` normalization on existing partner arrays

---

## Compatibility Notes

- **Existing installs:** Flynn bootstrap membership wiring (`household_add_member`, `link_partners`) is idempotent — `household_add_member` blocks on slot-occupied for installs that already have principals set. Re-running bootstrap is safe.
- **Identity manifest shape change:** `zen_identity_manifest` drawer format changed from `{roster, count}` to `{roster, tree}`. `identity_manifest_loader()` handles both shapes. Old manifests load as `status: ok_legacy` until rebuilt.
- **Expansion cabinet entity IDs:** Installs with data in old expansion slot entity IDs (secondary/tertiary/quaternary/secondary_ai) require a migration script before provision. See open items.

---

## Open / Deferred

- Migration script for installs with data in old expansion entity IDs
- SP1 — `caller_token` enforcement (plumbing complete, activation deferred)
- KFC 1.1 — `meta.enabled` full enforcement, `requires.cert`/`requires.level` (SP1)
- Security architecture — zen_auth, TGT, SekretSQRL (post-GA)
- Cabinet import/export — file I/O arch decision unresolved
