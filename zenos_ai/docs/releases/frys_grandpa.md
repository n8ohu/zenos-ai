# Release Notes — 2026.5.0 "Fry's Grandpa"

**Released:** TBD
**Branch:** `feat/2026.5.0`
**Base:** 2026.4.3 'Lights, Camera, Action'
**UAT:** Nyx (H:\)

---

*Good news, everyone.*

---

## Summary

Five open loops. All of them closed.

**Priority inject** is the flagship. Cameras classify motion. Alertmanager routes error-severity alerts into `_zen_priority_inject`. The prompt now renders a NOTIFICATIONS block at the top of every context frame — active alerts, urgency, providers, time since. The AI enters every conversation already knowing what's wrong, without being told. The loop from "something happened" to "the AI carries that forward" is now closed end-to-end.

**Camera alert policy** closes the classification gap. Each camera can now hold a `_alert_policy` drawer that defines how its motion events get classified and where they route. `set_alert_policy` mode writes it; `look` and `scan` honour it; `info` surfaces it. Private keys (`_*`) are now preserved across all look/scan cycles — alert policy no longer gets clobbered on the next camera pass.

**`provision_member`** closes the onboarding loop. Before this release, adding a family member who had no HA `person.*` entity and no pre-provisioned cabinet required a manual four-step sequence that left the person as an orphan after restart (#135). One call now does the full job: find the first available expansion slot, provision it, wire it into the family, rebuild the manifest. OOBE routes external family members through it automatically.

**Profile editor write** closes the loop that was never actually closing. FC returns `confirmed` on success, not `success` — the check was wrong and every write was silently failing its own gate. Second writes also failed because `_u_cur` didn't parse the JSON string FC returns for existing drawers. Both fixed.

**ZQ-1 v4.6.0** closes the regex loop. `is regex()` is not a valid Jinja2 test in HA — `regex` filter was silently matching nothing. Now `regex_search()`. Also: `entity_id_regex` filter for matching against the entity ID string directly, and `stats_eligible` for filtering recorder-eligible entities.

The bootstrap paradox at the center of this release: the alert system feeds context into the AI that manages the alerts that feed the context. It caused itself. Good news, everyone.

---

## Priority Inject — New

The priority inject system is a persistent alert layer that survives summary cycles and lives in the prompt.

### Architecture

```
Alertmanager (error severity)
  → zen_event kind: priority_inject_write
    → zen_priority_inject_handler automation
      → _zen_priority_inject drawer (household cabinet)
        → zen_priority_context sensor (tracks active count, urgency, providers)
          → zen_os_1.jinja priority_inject() macro
            → NOTIFICATIONS block in prompt
```

### `_zen_priority_inject` Drawer

Stored in the household cabinet. Up to 5 entries, sorted by urgency then timestamp. Each entry:

```json
{
  "id": "uuid",
  "summary": "Cabinet health degraded — 2 slots offline",
  "urgency": "urgent",
  "provider": "zen_alertmanager",
  "since": "2026-05-03T10:30:00",
  "expires": "2026-05-03T11:30:00"
}
```

**Urgency values:** `critical` | `urgent` — validated on write, invalid entries are dropped.  
**Expires:** ISO timestamp, capped at 24 hours. Stale entries are GC'd by Core on every cycle.  
**Capacity:** Capped at 5 entries. Overflow evicts lowest-urgency oldest first.

### `zen_priority_inject_handler` Automation

Processes `zen_event kind: priority_inject_write` and `zen_event kind: priority_inject_clear`. Write path validates urgency, truncates summary to 255 chars, converts `expires_minutes` to ISO timestamp, merges into the existing slot (providers never overwrite each other). Clear path removes all entries matching the provider.

### `zen_priority_context` Sensor

Always-live sensor tracking the current inject state:

| Field | Description |
|---|---|
| `active_count` | Number of active (non-expired) entries |
| `providers` | List of providers with active entries |
| `highest_urgency` | `critical` \| `urgent` \| `none` |
| `oldest_since` | Timestamp of the oldest active entry |

### NOTIFICATIONS Block in Prompt

`priority_inject()` macro in `zen_os_1.jinja` renders a NOTIFICATIONS block when active entries exist. Not counted toward the prompt token budget (separate from the main context frame). If no active entries, the block is omitted.

### Core GC

`dojotools_core` GC now runs a priority inject eviction pass on every cycle. Validates urgency enum, clamps expires to 24-hour max, evicts stale/invalid entries, enforces 5-entry cap sorted by urgency + since.

---

## Alertmanager — v1.2.0

### Priority Inject Integration

Error-severity alerts now wire into the priority inject slot automatically. On alert raise, a `zen_event kind: priority_inject_write` fires with the alert summary, `urgency: urgent`, and a 60-minute expiry. On alert clear, `zen_event kind: priority_inject_clear` fires to scrub the provider's entries.

The AI now sees active error-severity alerts at the top of every context frame without being told.

### `notify_target: postman`

New routing target alongside existing `notify_target` options. Routes the alert notification through `zen_dojotools_postman` with `_channel_hint`, `_image_entity`, `_title`, and `_urgency` passed through. Camera motion alerts routed to `postman` get the snapshot attached automatically when `_image_entity` is set.

### `zen_priority_context` Sensor

New always-live sensor. Tracks active inject state across all providers — count, urgency ceiling, provider list, oldest timestamp. Readable at any time without touching the household cabinet directly.

---

## Camera — v1.3.0

### `set_alert_policy` Mode

Writes a `_alert_policy` drawer to the camera's cache entry in the household cabinet. Defines how motion events from this camera are classified and where they route.

```yaml
zen_dojotools_camera:
  mode: set_alert_policy
  camera_entity: camera.front_door
  policy_payload: >
    {
      "enabled": true,
      "classify_on": "motion",
      "notify_target": "postman",
      "urgency": "urgent",
      "channel_hint": "security"
    }
```

Policy is readable via `info` mode and is preserved across all look/scan cycles.

### Private Key Preservation

`look` and `scan` now preserve all `_*` private keys when updating the camera's cache drawer. Previously each look/scan replaced the full drawer — `_alert_policy`, `_default_ctx`, and any other private keys were silently clobbered. Fixed via `_look_existing_private` read before write.

### `sendto: sensor.*` Support

`sendto` now accepts a cabinet sensor entity ID directly in addition to `image.zen_image_*` and `person.*` targets. The caller pre-resolves the sensor to an entity ID at call time:

```yaml
zen_dojotools_camera:
  mode: look
  camera_entity: camera.front_door
  sendto: "{{ states('sensor.zen_default_user_cabinet_resolved') }}"
```

The tool resolves the cabinet's member entry to `person_entity_id` and routes through postman. Use this for dynamic routing without hardcoding an entity ID.

### `_alert_policy` in `info` Response

`info` mode now includes the stored `_alert_policy` if present.

---

## Identity — provision_member (closes #135)

### `provision_member` Mode

Single-call orchestration for adding a family member who has no HA `person.*` entity and no pre-provisioned cabinet.

```yaml
zen_dojotools_identity:
  mode: provision_member
  member_name: Marianne
  profile_payload: '{"first_name": "Marianne", "role": "extended_family", "pronouns": "she/her"}'
  # family_entity: sensor.<cabinet>   # optional — defaults to zen_default_family_cabinet_resolved
  # member_type: user                 # optional — default user
```

**What it does:**
1. Validates `member_name` (required)
2. Resolves `family_entity` (defaults to `zen_default_family_cabinet_resolved`)
3. Scans expansion slots 1–5 for the first in `init` state
4. Hard-stops with clear error if all 5 slots are occupied
5. Calls the provisioner — auto-stamps `init` → `online_unmounted` → provisions as `user` (or `member_type`) with `preferred_name` + `name` seeded from `member_name` plus any `profile_payload` fields
6. Calls `family_add_member` to wire the new cabinet into the target family
7. Returns combined response: `{status: ok, member_entity, member_name, family_entity, provisioner_result, family_result}`

**Before this release:** Adding Marianne required: find a stacks slot, call provisioner, write profile via profile editor, call family_add_member, rebuild manifest — and the person entity was an orphan until manifest rebuild. After a restart, she'd be gone from the manifest.

**After this release:** One call. Done.

### `member_name` and `profile_payload` Fields

Two new fields on `zen_dojotools_identity`, used only by `provision_member`:

| Field | Required | Description |
|---|---|---|
| `member_name` | Yes (for provision_member) | Display name — stored as `preferred_name` and `name` in the provisioned profile drawer |
| `profile_payload` | No | JSON string — additional profile fields merged with member_name at provision time |

### OOBE Updated

`flynn_oobe.yaml` step `3_people` now routes external family members (no HA person entity) through `provision_member` instead of `profile_editor mode: write target_type: family`, which could write profile data but could not create the entity, wire it into the family, or rebuild the manifest.

---

## ZQ-1 — v4.6.0

### `regex` Filter Corrected

`is regex()` is not a valid Jinja2 test in Home Assistant. The `regex` filter was calling `is regex()` and silently matching nothing. Fixed to `regex_search()`.

**If you were using `filter_type: regex` and getting no results** — this is why. No config change needed; the fix is in the template.

### `entity_id_regex` Filter (step 10b)

New filter that matches against the entity ID string directly, independent of state or attributes.

```yaml
zen_dojotools_index:
  mode: index
  label: security_camera
  entity_id_regex: "camera\\.front_.*"
```

Useful for targeting a naming-convention subset within a labeled group without requiring a sub-label.

### `stats_eligible` Filter

Filters for recorder-eligible entities: must have `state_class`, `unit_of_measurement`, a float-parseable state, and must not be disabled. Use before calling `recorder.get_statistics` to avoid passing non-numeric entities into the stats pipeline.

---

## Profile Editor — Write Bug Fixes

Two independent bugs in `zen_dojotools_profile_editor` write mode, both fixed:

**Bug 1 — Wrong success key:** FC returns `confirmed: true` on success, not `success: true`. The write gate was checking `success` and treating every write as a failure. The profile data landed, but the tool reported an error. Fixed: `_write_ok` now checks `confirmed`.

**Bug 2 — Second write merge failure:** On a second write, FC returns the drawer value as a JSON-encoded string inside a wrapper dict. `_u_cur` was not parsing this string, so `ns.d.get()` was operating on a string instead of a mapping — every second write clobbered instead of merging. Fixed: `_u_cur` now runs `from_json` when the unwrapped value is a string.

Both bugs were present since profile editor GA. The 5s timeout (also a write-path issue) was already fixed in the FileCabinet file in this release cycle.

---

## Cortex v38 — Kata First

Supersedes v37. AUDIO directive updated: `zen_dojotools_notification_router` replaced with `zen_dojotools_postman` for push-only notifications. All other v37 content unchanged.

Load: `zen_admintools_prompt_loader: cortex_version: latest, ship_zen_system: false`

---

## SystemTools — v1.7.0

### `cabinet_schema_upgrade` Tool

Wraps `zen_admintools_cabinetadmin flip_schema_version` for schema migration operations. Previously required calling cabinetadmin directly with the internal operation name. Now surfaced as a first-class SystemTools operation.

---

## Files Changed

| File | Change |
|---|---|
| `custom_templates/zenos_ai/zen_os_1.jinja` | `priority_inject()` macro; NOTIFICATIONS block; `cabinet_list()` adds writeable+status; `identity_manifest()` roster+tree shape; `kfc_slot()` description+instructions+dojo_cabinet; `render_prompt()` priority inject excluded from total count |
| `custom_templates/zenos_ai/zen_query.jinja` | v4.6.0: `entity_id_regex` filter (step 10b); `regex` → `regex_search()`; `stats_eligible` filter |
| `packages/zenos_ai/dojotools/dojotools_admintools.yaml` | Cortex v38 Kata First (v37 → v38: AUDIO directive postman fix); taskmaster v1.3.2 (declarative synthesis guidance); script renamed `zen_admintools_prompt_loader` |
| `packages/zenos_ai/dojotools/dojotools_alertmanager.yaml` | v1.2.0: `zen_priority_context` sensor; `notify_target: postman`; priority inject write/clear on error/clear; `zen_priority_inject_handler` automation |
| `packages/zenos_ai/dojotools/dojotools_camera.yaml` | v1.3.0: `set_alert_policy` mode; `_*` key preservation in look/scan; `sensor.*` sendto dispatch; `_alert_policy` in info response; sendto help example uses dynamic cabinet resolution |
| `packages/zenos_ai/dojotools/dojotools_core.yaml` | Priority inject GC: urgency validation, 24hr cap, 5-entry limit, eviction sort |
| `packages/zenos_ai/dojotools/dojotools_dispatcher.yaml` | Trapper Keeper rebuild triggers (HA start + 6/12/18/0 daily); `zen_flynn_health_resummarize` automation; alarm panel routing |
| `packages/zenos_ai/dojotools/dojotools_filecabinet.yaml` | Write wait timeout 5s → 10s; `fatal_response` corrected to YAML dict; `zen_admintools_prompt_loader` rename refs |
| `packages/zenos_ai/dojotools/dojotools_identity.yaml` | `provision_member` mode; `member_name` + `profile_payload` fields; Step 3m sequence; `person.*` resolution implemented (was stub) |
| `packages/zenos_ai/dojotools/dojotools_index.yaml` | v4.6.0 reference; `entity_id_regex` documented; `regex` description corrected; `log_result \| default(false)` fix; `ci_write` status propagated; `*` operator fallback fix |
| `packages/zenos_ai/dojotools/dojotools_kungfu_loader.yaml` | `zen_admintools_prompt_loader` rename refs; trapper_keeper target v2.5.0; taskmaster moved to conductor tier |
| `packages/zenos_ai/dojotools/dojotools_postman.yaml` | `pm_` tag prefix condition gate; `timestamp` in dispatch response |
| `packages/zenos_ai/dojotools/dojotools_profile.yaml` | `_write_ok` checks `confirmed` (not `success`); `_u_cur` parses JSON string on second write; jacket write uses combine; companion preserves role/verbosity/kfc_ref |
| `packages/zenos_ai/dojotools/dojotools_summarizers.yaml` | Code fence strip: single dotall regex |
| `packages/zenos_ai/dojotools/dojotools_systemtools.yaml` | v1.7.0: `cabinet_schema_upgrade` tool |
| `packages/zenos_ai/dojotools/dojotools_utilities.yaml` | `tts_sent` event after TTS dispatch; `final_response` + history entry handle both mapping and string defensively |
| `packages/zenos_ai/flynn.yaml` | `zen_admintools_prompt_loader` rename refs |
| `packages/zenos_ai/flynn_oobe.yaml` | Step 3_people: external family members now route through `provision_member` |
| `packages/zenos_ai/plugins/calderaspas/calderaspas_spa_manager.yaml` | v5.2.0: complete rewrite with full action router (status/lights/jets/audio/temperature/cover/chemistry/log). **Workshop before ship** — hardware definition needs to be made pluggable. |
| `zenos_ai/docs/getting_started/first_alert.md` | Postman routing description updated |
| `zenos_ai/docs/getting_started/install.md` | Updated for 5.0 |
| `zenos_ai/docs/getting_started/troubleshooting.md` | Step 2.5 cabinet_schema_upgrade; script rename; new entries |

---

## Upgrade Notes

No schema migrations. No cabinet changes required on existing installs.

**Priority inject:** New system, no existing data to migrate. `_zen_priority_inject` will be auto-created in the household cabinet by the handler on first write. `zen_priority_context` sensor goes live on HA reload.

**Profile editor users:** Write mode now correctly reports success. If you added retry logic or workarounds for the false-failure behavior — you can remove them. If you were using `force_action: true` direct FC writes as a workaround (#135 era) — profile editor write is the correct path again.

**Camera users:** `_alert_policy` and other `_*` private keys are now preserved across look/scan. If you were re-calling `set_alert_policy` before each scan to restore a clobbered policy — you can stop.

**`zen_admintools_zenos_prompt_loader`:** Renamed to `zen_admintools_prompt_loader`. The old name is gone — update any automations or scripts calling it directly. Flynn and all dojotools refs are updated in this release.

**Calderaspas SPA manager:** v5.2.0 is in the branch but flagged for workshop before ship. Hardware definition (device model, sensor IDs, capacity) needs to be pluggable via a YAML block in the help section. Not shipping in 5.0 GA — carry-forward to 5.1.
