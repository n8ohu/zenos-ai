# Release Notes — 2026.4.0 'Ectoplasm'

**Released:** 2026-04-04  
**Branch:** `feat/zen_dojotools_ectoplasm`  
**UAT:** Nyx (H:\)

---

## Summary

Ectoplasm is the extended surface release. The headline is a new DojoTool that wraps the full Spook/HA structural API. Supporting it: Index and Inspect both got major capability expansion (topology seeds, registry modes, pagination), Scribe gets authoring quality-of-life improvements, and the Ninja Summarizer gets a run governor that absorbs duplicate trigger firings.

---

## New: `zen_dojotools_ectoplasm`

New tool. Spook/HA extended surface wrapper. Manages structural HA state that the standard service layer does not expose through Jinja.

**Requires:** Spook integration (listed ZenOS prerequisite).

**Covers (25 action types):**

| Group | Actions |
|---|---|
| Reads | `repair_list`, `ghost_list` |
| Repairs | `repair_create`, `repair_remove`, `repair_ignore_all`, `repair_unignore_all` |
| Areas | `area_create`, `area_delete`, `area_assign_device`, `area_unassign_device`, `area_assign_entity`, `area_unassign_entity` |
| Floors | `floor_create`, `floor_delete`, `floor_assign_area`, `floor_unassign_area` |
| Entities | `entity_hide`, `entity_unhide`, `entity_disable`, `entity_enable`, `entity_rename` |
| Devices | `device_disable`, `device_enable` |
| Labels | `label_assign_area`, `label_unassign_area`, `label_assign_device`, `label_unassign_device` |
| Integrations | `integration_disable`, `integration_enable` |
| Housekeeping | `orphan_cleanup` |

All write actions are confirm-gated — omit `confirm_action` or set `false` for a preview (`status: start`); set `true` to execute. Read actions execute immediately.

Note: `label_create`/`delete` and `label_assign_entity`/`unassign_entity` remain in `zen_dojotools_labels`. Ectoplasm adds the area and device label surfaces only.

---

## Scribe 1.2.0

- **`enable` / `disable` modes** — shorthand aliases that patch `meta.enabled: true/false` without a full rewrite. Normalize to `patch` internally; `_changes` is set before normalization so both variables resolve correctly in the same step.
- **`repair` mode** — detect and flatten wrapper-accumulated drawer corruption. `dry_run: true` by default; set `false` to write.
- **`domain_inspect_domains` / `domain_inspect_limit` fields** — constrain step-4b domain inspect by HA domain (comma-separated) and max entity count (default 25). Wired into `_thought_payload.index_call` and `_kfc_payload.index_call`.
- **Bug fix: `meta.enabled` missing from `_thought_payload`** — thought artifacts now carry `meta.enabled` on write. Previously, `formalize_scroll` → `publish_kfc` still worked by fallback coincidence, but reads and audits showed no `enabled` field in the thought.
- **Bug fix: emission fields not written on `new_thought`** — `drift_threshold`, `emission_cooldown_minutes`, `suggested_act_event`, and `master_switch` were in the fields block but silently dropped from `_thought_payload`. Now wired. `_kfc_payload` already had them.
- **Mode description** updated to include `enable`, `disable`, and `repair`.

---

## Index 4.6.3

**Topology seeds** — each operand set now resolves via a full precedence chain:
```
entities_N > label_N > device_N > integration_N > area_N > floor_N
```
New fields: `device_1`, `integration_1`, `area_1`, `floor_1` (and `_2` equivalents). Topology seeds can return large sets — always `dry_run: true` first to check `total_count`.

**Pagination** — `limit` / `offset` fields. `dry_run` returns `total_count` for paging loops. `limit: 0` = no limit (default).

**Auto-cap** — when `expand_entities: true` with a topology or wildcard seed and no explicit `limit`, result is automatically capped at 50. Set `limit` explicitly to override.

**`+history` field** — request 24h hourly recorder stats per entity. Requires explicit `limit` — hard error without it on topology/wildcard seeds.

**Inspect registry modes** — Index now surfaces all Inspect registry modes directly: `area_info`, `floor_info`, `device_info`, `area_list`, `floor_list`, `label_list`, `zone_list`, `person_list`, `device_list`, `integration_entities`. Pass `mode: <mode>` with no entity inputs.

---

## Inspect 4.6.2

**Registry enum modes** — bypass the entity loop and query the HA registry directly:

| Mode | Input |
|---|---|
| `area_info` | `area_id` |
| `floor_info` | `floor_id` |
| `device_info` | `device_id` |
| `area_list` | — |
| `floor_list` | — |
| `label_list` | — |
| `zone_list` | — |
| `person_list` | — |
| `device_list` | — (tooling primitive) |
| `integration_entities` | `integration` domain slug |

Default mode remains `inspect` (entity loop). `device_list` is a tooling primitive — not intended for direct AI calls on large installs. Use `floor_list` → `area_info` → `device_info` drill-down instead.

Footgun note: `floor_id()` takes area **name** not area ID slug (`floor_id(area_name(area_id))` is the correct pattern). `area_attr()`/`floor_attr()` do not exist in HA or Spook — use `area_name()`, `floor_id()`, `floor_name()`, `labels()` instead.

---

## Ninja Summarizer — Run Governor

New step 3b in the Ninja Summarizer pipeline. Fires between `meta.enabled` check and HyperIndex call.

- Reads `burnout_seconds` from `zen_ninja_config` drawer in household cabinet. Default: 300s.
- Reads `zen_ninja_last_run_<component_slug>` from Kata cabinet for last-run timestamp.
- If elapsed < burnout, emits `summarizer_run_blocked` event (`reason: dedup_window`) and stops.
- New `force: true` field bypasses the governor for admin overrides or emergency on-demand runs.

Configure per-install: write `zen_ninja_config.burnout_seconds` to household cabinet via FileCabinet.

---

## AdminTools

- **`zen_admintools_run_repair`** — now documented. Human-confirmed passthrough to versioned maint/ scripts. Ships with three repair actions: `identity_family_repair_4_5_6`, `stamp_cab_guid_4_5_6`, `roster_guid_repair_4_5_6`.
- **`ship_alert_manager`** (boolean, default `false`) — ships `alert_manager` KFC v1.3.0 to Dojo. Action emission demo: monitors alerts, alert_when_off entities, persistent notifications.
- **`ship_taskmaster`** (boolean, default `false`) — ships `taskmaster` KFC v1.3.1 to Dojo. Conductor pattern demo: task load, calendar, bed/occupancy context. Uses `domain_inspect_domains: "todo,calendar"`.
- Both KFCs have independent if-gates — can be shipped without triggering each other or `ship_zen_system`.

---

## Dispatcher 1.1.0

- Two-level routing architecture
- All inter-tool calls visible on event bus
- Urgency signal architecture stub wired (`zen_dojotools_urgency_handler` v1 registered, handler not yet implemented)
- Fault isolation: unknown tools return structured error, not hard fault

---

## Bug Fixes

| Fix | File |
|---|---|
| Ninja summarizer: domain inspect OverflowError prevention + filter guard | `dojotools_summarizers.yaml` |
| Calendar: `strptime` empty-string guard for start/end fields | `dojotools_office.yaml` |
| Dispatcher: `_ver` coercion + office `from_json` guard | `dojotools_dispatcher.yaml` |
| todo_lists_data trailing artifact + wildcard lists coercion guard | various |
| Index: description `>-` block scalar + bool crash fix | `dojotools_index.yaml` |
| Summarizers: strip ` ```json``` ` fence from monk_response | `dojotools_summarizers.yaml` |
| `floor_id()` fix: pass area name, not area ID slug | `dojotools_index.yaml` |
| Replace `area_attr`/`floor_attr` with native HA functions | `dojotools_index.yaml` |

---

## Upgrade Notes

- **Ectoplasm requires Spook.** If Spook is not installed, `zen_dojotools_ectoplasm` will load but all write actions will fail at the Spook service call. Install Spook via HACS first.
- **Ninja burnout window is opt-in by default** (300s default is already set; no config required). To tighten or loosen, write `zen_ninja_config.burnout_seconds` to the household cabinet.
- **`ship_alert_manager` / `ship_taskmaster` default `false`** — run prompt loader with these set to `true` when ready to ship the in-box KFCs.
- **Scribe `enable`/`disable` aliases** are backward-compatible. Existing `patch` calls are unaffected.
