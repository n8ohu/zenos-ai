# Zen DojoTools AdminTools — 4.2.1

*Ring-2 administrative tools: component registration, cabinet repair, template management, and prompt configuration*

---

## Overview

AdminTools is the **Ring-2 administrative layer** of ZenOS-AI. It handles tasks that fall outside normal runtime behavior: registering new Kung Fu components, repairing cabinets, pressing schema templates, and loading the AI's identity substrate.

Most tools in this module are **admin-only** — they are not exposed to the AI agent and should not be called by Friday during normal operation. The exception is `zen_dojotools_kungfu_writer`, which is fully MCP-exposed and is the correct way for Friday or any operator to register a new KFC component.

---

## Scripts in This Module

| Script | Version | MCP-Exposed | Purpose |
|---|---|---|---|
| `zen_dojotools_kungfu_writer` | 4.2.0 | **Yes** | Register or update a Kung Fu component in the Dojo |
| `zen_admintools_reset_template` | 1.1.0 | No | Press zen_template and kfc_template into cabinets |
| `zen_admintools_reset_labels` | 4.2.1 | No | Nuclear: delete all zen_ labels and assignments, trigger Flynn rebuild |
| `zen_admintools_cabinetadmin` | 4.2.1 | No | Inspect, restore, reset, hammer, init, or reset_all Ring-0 cabinets |
| `zen_admintools_cabinetadmin_backup` | 1.x | No | Factory-stamp or repair a cabinet's VolumeInfo drawer |
| `zen_admintools_kfc_migration_press` | 1.1.0 | No | One-time migration: seed scheduling fields into KFC drawers |
| `zen_admintools_zenos_prompt_loader` | 4.2.1 | No | Load versioned Cortex, Directives, and Purpose (v27 = RC2, v28/latest = GA) |

---

## zen_dojotools_kungfu_writer

The primary tool for registering a new Kung Fu Component (KFC) into the Zen Dojo Cabinet. This is the only AdminTools script accessible to AI agents.

Each call creates or updates a drawer in the Dojo cabinet. That drawer becomes the component's canonical spec: it tells the Scheduler when to run, the Ninja Summarizer what data to gather, and the AI how to interpret what it finds.

### Input Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `action` | select | Yes | `help` | `write` or `help` |
| `kata_key` | text | write | — | Unique component ID, slugified (e.g., `hot_tub_manager`) |
| `friendly_name` | text | write | — | Human-readable component title |
| `component_summary` | text | No | — | One-sentence description of what this component monitors |
| `label` | text | No | — | HyperIndex label used to discover relevant entities |
| `command` | text | No | — | Library macro token (e.g., `~ALERTS~`) |
| `trigger_subscriptions` | text | No | — | Comma-separated trigger IDs this component subscribes to |
| `delay_seconds` | number | No | `0` | Stagger delay (0–300s, step 5) to spread inference load |
| `master_switch` | entity | No | — | `input_boolean` gate — component skips if off |
| `tool` | text | No | — | Associated tool name |
| `version` | text | No | — | Semantic version string |
| `component_instructions` | text | No | — | Operational notes passed to the monk during summarization |
| `more_info` | text | No | — | Extended context for the monk |
| `confirm_action` | boolean | Yes | `false` | Must be `true` to execute a write |

### Valid Trigger IDs

```
ha_start              daily_midnight        daily_noon
hourly_trigger        quarter_hour          every_10_minutes
every_5_minutes       home_mode_updates     alarm_panel
door_opens_or_closes  lock_changes          window_opens_or_closes
garage_door           home_occupancy_change force_summary
force_gc
```

> Hardware triggers (`nathan_bed_changes`, `kim_bed_changes`, `water_flow_stop`, `hot_tub_state`, `elec_panel_door_change`, `mb_wake`, `test_button`, `start_home_wake`, `start_home_evening`) are available in the trigger ID list but only active if you have added them to a custom scheduler. See `docs/examples/zen_dojotools_scheduler_custom.yaml`.

### Actions

**`help`** — Returns the live `kfc_template` schema from the Dojo cabinet. Safe to call anytime. Use this to see the current schema before writing a new component.

**`write`** — Upserts the component drawer in the Dojo cabinet. Requires `confirm_action: true`. Idempotent — re-running with the same `kata_key` updates the existing drawer in place.

### Response Format

```json
{
  "status": "success",
  "message": "KFC drawer written: hot_tub_manager",
  "drawer": "hot_tub_manager",
  "cabinet": "zen_dojo_cabinet"
}
```

### Example: Register a New Component

```yaml
action: write
kata_key: hot_tub_manager
friendly_name: Hot Tub Manager
component_summary: Monitors hot tub temperature, chemistry, and filter status
label: hot_tub
command: ~HOT_TUB~
trigger_subscriptions: quarter_hour, hot_tub_state
delay_seconds: 30
confirm_action: true
```

### Example: Inspect the KFC Schema

```yaml
action: help
```

Returns the live `kfc_template` — the canonical field reference for all KFC drawers.

---

## zen_admintools_reset_template

Presses the `zen_template` seed into the Kata cabinet and the `kfc_template` seed into the Dojo cabinet.

Called automatically by **Flynn gate-3** if the templates are missing at boot. Can be re-run manually if templates become corrupted — it is fully idempotent.

**Not MCP-exposed.** Run via HA Developer Tools → Services, or trigger Flynn's gate-3 check.

### Behavior

- Reads the Kata cabinet and Dojo cabinet (resolved via labels `zen_summary,zen,summary` and `kfc,dojo,zen`)
- If `zen_template` is absent from the Kata cabinet → writes seed template
- If `kfc_template` is absent from the Dojo cabinet → writes seed template
- Uses `force_action: true` — will overwrite if explicitly needed

### When to Use

- After a fresh install if `sensor.zen_agent_health` reports a schema-related error
- If the `kfc_template` drawer in the Dojo appears corrupted or empty
- If Flynn's gate-3 fires a `dojo_loaded: warn` event

---

## zen_admintools_reset_labels

⚠️ **NUCLEAR** — deletes all `zen_` labels and all their entity assignments. No undo.

Use this when label IDs are corrupt, have wrong names, or you need a full label reinstall. For wiping assignments only (labels survive), use `zen_dojotools_labels` with `action_type: reset` instead.

**Not MCP-exposed.** Admin use only.

### What It Does

1. Untags all entities from every label with a `zen_` ID
2. Deletes every `zen_` label (`zen_*` IDs only — non-zen labels are never touched)
3. Fires `zen_resolver_refresh` so resolver sensors re-evaluate immediately

Flynn re-engages automatically: `zen_label_health` flips to `critical` → Stepgate Sentinel fires → Gate 0 recreates labels → Gate 1 re-assigns. No HA restart required (HA 2024.x+ propagates label creation live).

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `confirm` | boolean | `false` | Must be `true` — refuses without it |

### Full Nuclear Reset Sequence

To completely rebuild from scratch (labels + cabinets):

```
1. zen_admintools_reset_labels          — nuke + rebuild all zen_ labels
2. zen_admintools_cabinetadmin          — op: reset_all (wipe cabinets + reseed + Flynn)
```

Two tool calls. Order matters — labels first, then cabinets.

---

## zen_admintools_cabinetadmin

Ring-2 cabinet maintenance tool. Supports inspecting, restoring, resetting, and hammering the 14 Ring-0 system cabinets.

**Not MCP-exposed.** Admin use only. Destructive modes require explicit confirmation.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | `inspect` | `inspect`, `restore`, `reset`, `hammer`, `init`, `reset_all` |
| `target_cabinet` | entity (sensor) | — | Single cabinet to target; leave empty to target all Ring-0 cabinets |
| `hammer_ok` | boolean | `false` | Required `true` for `hammer` mode |
| `confirm` | boolean | `false` | Required `true` for `reset_all` |

### Modes

| Mode | Destructive | Description |
|---|---|---|
| `inspect` | No | List all drawers in targeted cabinet(s) with counts |
| `restore` | No | Stamp a `_restored_marker` drawer if cabinet is empty |
| `reset` | **Yes** | Clear all drawers in targeted cabinet(s) |
| `hammer` | **Yes** | Clear all drawers + stamp `_hammered` marker. Requires `hammer_ok: true` |
| `init` | **Yes** | Initialize cabinet with `AI_Cabinet_VolumeInfo` drawer; clears if targeting all |
| `reset_all` | **NUCLEAR** | Wipe all Ring-0 cabinets + reseed schemas + trigger Flynn bootstrap. `confirm: true` required |

### Ring-0 Cabinets

When `target_cabinet` is empty, all 14 Ring-0 cabinets are targeted:

```
sensor.zenos_system_cabinet
sensor.zenos_system_history_cabinet
sensor.zenos_dojo_cabinet
sensor.zenos_kata_cabinet
sensor.zenos_history_cabinet
sensor.zenos_scratchpad_cabinet
sensor.zenos_default_household_cabinet
sensor.zenos_default_household_history_cabinet
sensor.zenos_default_family_cabinet
sensor.zenos_default_family_history_cabinet
sensor.zenos_default_user_cabinet
sensor.zenos_default_user_history_cabinet
sensor.zenos_default_ai_user_cabinet
sensor.zenos_default_ai_user_history_cabinet
```

### When to Use

- **inspect** — routine health check; see what's in a cabinet without touching it
- **restore** — cabinet appears empty after a crash; stamp a marker to confirm it's mounted
- **reset** — nuke a cabinet's contents cleanly (e.g., clear scratchpad, reset history)
- **hammer** — full wipe with audit trail; last resort before re-init
- **init** — fresh cabinet initialization; sets VolumeInfo metadata
- **reset_all** — full nuclear cabinet reset + reseed. Calls `reset_template` and fires Flynn via `zen_cabinet_health` state change. If you want to customize the sequence (skip a cabinet, change order), run the steps individually — that is exactly what `reset_all` orchestrates under the hood.

---

## zen_admintools_cabinetadmin_backup

Factory tool for stamping or repairing a cabinet's `AI_Cabinet_VolumeInfo`, `_label_index`, and `_zen_relationships` drawers.

Use this when a cabinet is missing its metadata header — for example, after creating a new cabinet entity that has never been initialized, or after a cabinet loses its VolumeInfo drawer.

**Not MCP-exposed.** Admin use only.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `confirm_action` | boolean | `false` | Must be `true` — script exits otherwise |
| `cabinet_entity` | entity (sensor) | — | Target cabinet |
| `cabinet_type` | select | `AI Data Storage Cabinet` | Cabinet class (determines flag auto-profile) |
| `cabinet_guid` | text | — | Optional; generates new GUID if blank |
| `force_new_guid` | boolean | `false` | Force GUID regeneration even if one exists |
| `auto_profile` | boolean | `true` | Auto-detect security flag profile from cabinet type |
| `explicit_flag_profile` | select | `standard` | Override flag profile: `system`, `secure`, `public`, `standard` |
| `mount_scan` | boolean | `true` | Scan existing drawers for `[mount:GUID]` links |
| `owner_person` | entity (person) | — | Owner reference (HA person entity) |
| `partner_person` | entity (person) | — | Partner reference |
| `family_guid` | text | — | Family GUID for ACL |

### Flag Profiles (Auto-Detected)

| Cabinet Type | Profile |
|---|---|
| Household / Family | `system` |
| User / AI User | `public` |
| Kata / Archive | `system` |
| Chat History | `secure` |
| Default | `standard` |

### When to Use

- Newly created cabinet entity that needs its VolumeInfo drawer stamped
- Cabinet metadata is corrupt or missing after an upgrade
- GUID needs to be regenerated (e.g., cloned from another install)

---

## zen_admintools_kfc_migration_press

One-time migration tool. Seeds `trigger_subscriptions`, `delay_seconds`, and `kata_key` into the 12 existing KFC drawers that predated the Dojo-driven scheduler (KF4 RC2).

This script has already been run on all production installs. It exists for reference and for fresh installs that import pre-KF4 drawer exports.

**Not MCP-exposed.**

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `dry_run` | boolean | `true` | If true, logs intended writes without executing them |

### Behavior

- Iterates all 12 component drawers in the Dojo cabinet
- Merges `trigger_subscriptions`, `delay_seconds`, and `kata_key` into each using `combine()` — **does not overwrite existing fields**
- Idempotent — safe to re-run

### When to Use

Only needed if you are importing KFC drawers that were written before KF4 RC2. New installs and components written via `zen_dojotools_kungfu_writer` already include these fields.

---

## zen_admintools_zenos_prompt_loader

Loads the AI's identity substrate: **Cortex**, **Directives**, and **Purpose**. These three variables define how Friday reasons, what she prioritizes, and how she accesses the knowledge graph.

**Not MCP-exposed.** This is a configuration and prompt-engineering tool, not a runtime script. The loaded values persist in the AI cabinet and are read by the prompt engine at inference time.

### What Gets Loaded

| Variable | Purpose |
|---|---|
| `Purpose` | Role definition — what ZenOS-AI is and what it manages |
| `Directives` | 14 behavioral rules: tone, safety, confirmation patterns, tool preferences |
| `Cortex` | Full reasoning substrate — schema references, DojoTools index, behavior rules, error policy, library access patterns |

### When to Use

- After a fresh install, if the Cortex is empty or missing
- When upgrading the Cortex schema (new version of the reasoning contract)
- When adjusting behavioral rules for a specific deployment

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `cortex_version` | select | `latest` | `latest` or `28` = 2026 GA (4.3.x). `27` = 2026 RC2 (4.2.x) |

Use the `cortex_version` field to select which version to load. The three primitives (Purpose, Directives, Cortex) are versioned together as a set. v27 = 2026 RC2 (4.2.x series). v28 / latest = 2026 GA (4.3.x series — includes Context Resolution directive and Cortex 28.0.0 with scope-aware context stack resolution).

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `script.zen_dojotools_filecabinet` | Drawer reads and writes |
| Zen Dojo Cabinet | KFC component registry |
| Zen Kata Cabinet | Summary template storage |
| `sensor.zen_*_cabinet` (Ring-0 set) | cabinetadmin targets |
