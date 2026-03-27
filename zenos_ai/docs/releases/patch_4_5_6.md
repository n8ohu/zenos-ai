# ZenOS-AI Patch 4.5.6

*Post-GA identity and alertmanager correctness patch*

**Branch:** `patch/post-ga-identity-fixes`
**Base:** 4.5.5 'Ready Player Two'
**UAT:** In progress — Nyx on live H:\ install

---

## Summary

Five correctness fixes and one admin improvement. No architecture changes. All fixes are idempotent — existing installs safe.

---

## Changes

### Identity — `unlink_partners` variable name fix

**File:** `dojotools/dojotools_identity.yaml`

`unlink_partners` computed `_a_vi_updated` / `_b_vi_updated` but the two cabinet write events referenced `_user_vi_updated` / `_ai_vi_updated` — names that never existed in that block. Both writes were silent no-ops. ACL removal had never worked.

**Fix:** Variable references in the two write events corrected to match the computed names.

---

### Identity — Bootstrap family graph wiring

**File:** `flynn.yaml`

Cold bootstrap (`flynn_bootstrap_content`) wired default user and AI into the household and linked them as partners, but never called `household_add_family` or `family_add_member`. The default family cabinet existed but was an orphan — not connected to the household graph. The identity manifest correctly reflected this gap, which caused the security roster to drop family-contained entities as invalid.

**Fix:** Flynn bootstrap now calls `household_add_family` + `family_add_member` (user + ai_user) after `link_partners`, before the final `build_identity_manifest`. All three calls are guarded (skips if cabinet unavailable) and idempotent.

**Existing installs:** Run `zen_dojotools_systemtools` with `tool: run_repair`, `repair_action: identity_family_repair_4_5_6`. Or run `script.zen_maint_4_5_6_identity_family_repair` directly from Developer Tools.

---

### Identity — Name resolution fallback (#110)

**File:** `dojotools/dojotools_identity.yaml`

The identity tree builder and manifest used a two-step name resolution chain (`friendly_name → entity_id`). Pre-RP2 VolumeInfo entries stored the display name in a `name` field, not `friendly_name` — these resolved to bare entity IDs in the membership tree and manifest output.

**Fix:** Resolution chain extended to `friendly_name → name → entity_id` at 8 locations: `build_identity_manifest`, membership tree renderer, and `member_lookup`/`is_member` paths.

---

### Home Mode — Timer seeding without `initial:` (#112)

**Files:** `dojotools/zen_home_mode.yaml`, `flynn.yaml`

The 10 `input_datetime` schedule and quiet/work-hours helpers carried `initial:` values. HA evaluates `initial:` on every restart — any time you changed a helper time, the next restart would reset it to the `initial:` default. User-configured schedules were silently discarded.

**Fix (zen_home_mode.yaml):** All 10 `input_datetime` helpers — `initial:` removed.

**Fix (flynn.yaml):** Flynn bootstrap now seeds each timer individually on first boot (guard: `in ['', 'unknown', 'unavailable']`). Each timer has its own `if` block — partially-configured installs are not disrupted.

Default times seeded:

| Helper | Default |
|---|---|
| `zen_am_start` | 06:00 |
| `zen_morning_start` | 08:00 |
| `zen_daytime_start` | 10:00 |
| `zen_evening_start` | 17:00 |
| `zen_night_start` | 21:00 |
| `zen_late_night_start` | 23:00 |
| `zen_quiet_hours_start` | 23:00 |
| `zen_quiet_hours_end` | 06:00 |
| `zen_work_hours_start` | 09:00 |
| `zen_work_hours_end` | 17:00 |

---

### AlertManager — Persistence fix (#113)

**File:** `dojotools/dojotools_alertmanager.yaml`

Both write paths (alert fire and alert clear) used `action_type: write`, which is not a valid FileCabinet action type. FileCabinet returned a help response on every call — `_zen_active_alerts` was never persisted. Every AlertManager run treated the active-alerts dict as empty, so every `alert_fire` was treated as a new alert and fired the notification again. The intended fire-once deduplication never worked.

**Fix:** Both paths updated to `action_type: create` + `force_action: true` (create-or-overwrite). Deduplication is now functional.

---

### SystemTools — `run_repair` admin entry point

**Files:** `dojotools/dojotools_systemtools.yaml`, `maint/maint_4_5_6.yaml` *(new)*

Adds a `run_repair` tool to SystemTools with a `repair_action` picklist. Dispatches to versioned one-time repair scripts in `packages/zenos_ai/maint/`.

Repair scripts are **not AI-accessible** — they are operator-run via SystemTools or directly from Developer Tools → Services. Each script includes a version declaration and idempotency guarantee.

**4.5.6 repair script:** `zen_maint_4_5_6_identity_family_repair` — wires default family into household graph for installs bootstrapped before 4.5.6.

---

### Cabinet Health — Legacy warn triage by slot presence

**File:** `custom_templates/zenos_ai/zenos_health.jinja`, `sensors/zenos_cabinet_health.yaml`

`zen_cabinet_health` emitted `warn` for every cabinet with state `Variables` in `all_cabs` (parent `Zen Cabinet` label), including retired stubs whose slot-specific labels had been intentionally pulled. On installs with many decommissioned legacy cabs this produced persistent noise warn state unrelated to anything needing migration attention.

**Fix:** `legacy_schema` detection in both `zen_cabinet_health_state()` and `zen_cabinet_health_reason()` now builds a slotted set first (entities assigned to any required or optional slot label) and only flags legacy-schema cabs that are in that set. Unslotted cabs — intentionally retired, parent label only — are silent.

Behavior unchanged for any cabinet that currently holds a slot assignment.

---

## PRs Merged

| PR | Author | Summary |
|---|---|---|
| #115 | teskanoo | Scribe `label_description` → `label_description_generic`, `version` → `version_semantic`, person selector fix |
| #116 | teskanoo | AdminTools/filecabinet caller_token normalize, is_relabel/is_write_action moved earlier |
| #117 | teskanoo | FileCabinet normalize-at-top refactor |

---

## Files Changed

| File | Change |
|---|---|
| `dojotools/dojotools_identity.yaml` | unlink_partners fix, name fallback fix |
| `dojotools/dojotools_alertmanager.yaml` | action_type fix (fire + clear paths) |
| `dojotools/dojotools_systemtools.yaml` | run_repair tool + repair_action dispatch |
| `dojotools/dojotools_filecabinet.yaml` | PR #117 normalize-at-top refactor |
| `dojotools/dojotools_admintools.yaml` | PR #116 caller_token normalize |
| `dojotools/zen_home_mode.yaml` | Strip initial: from all 10 input_datetime helpers |
| `dojotools/dojotools_scribe.yaml` | PR #115 parameter renames + person selector fix |
| `flynn.yaml` | Bootstrap family graph wiring + timer seeding |
| `maint/maint_4_5_6.yaml` | New — one-time identity family repair script |
| `custom_templates/zenos_ai/zenos_health.jinja` | Legacy warn triage — slot presence gate |
| `sensors/zenos_cabinet_health.yaml` | Version bump |

---

## Verification

1. **unlink_partners** — call with linked pair, confirm both VolumeInfo `acls.partner` lists updated
2. **Bootstrap wiring** — fresh install or manual bootstrap: `sensor.zen_default_family_cabinet_resolved` should appear in household `members.families`; user/AI VolumeInfo should have `acls.family` entries
3. **Timer seeding** — fresh install: helpers should seed to defaults; restart: custom values should persist unchanged
4. **AlertManager** — call `alert_fire` twice with same key: second call should be a no-op (no duplicate notification)
5. **run_repair** — call with `repair_action: identity_family_repair_4_5_6` on a broken install, confirm `status: ok` and family graph wired
