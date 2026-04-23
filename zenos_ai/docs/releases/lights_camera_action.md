# Release Notes — 2026.4.3 'Lights, Camera, Action'

**Released:** 2026-04-23  
**Branch:** `feat/2026.4.3`  
**Base:** 2026.4.2 'Action Jackson 2'  
**UAT:** Nyx (H:\)

---

*Lights on. Camera rolling. Postman delivers the frame.*

---

## Summary

Three systems got completed in this release. Not started — completed.

Camera had the visual analysis and the cache. What it didn't have was a way to pass that frame downstream, or a way to hold context for a specific camera across runs. Both are now done. `sendto` delivers the last look directly to a dashboard slot or fires a postman call with the image attached. `set_default_ctx` stores a per-camera default context call; `look` and `scan` both honour it instead of overwriting it.

Index had label-based filtering. What it didn't have was a clean way to exclude — to run a broad sweep and then strip out the entities you know don't belong. The ZQ-1 exclusion suite adds five post-filter fields that apply after all inclusion logic, safe for ACL injection patterns and targeted suppression. Index also gained the compound/recursive DSL so a single `index_command` can span multiple independent label sets without the caller orchestrating multiple round-trips.

Postman was v0.3.0-alpha. It's 1.0.0 now. The pipeline emission story is complete: `kata_input` derives title and message directly from a ninja output dict, `breakthrough` bypasses the house sleep gate without requiring a high urgency level, and the response router automation converts button taps into `zen_event(kind: postman_response)` for ack correlation. `zen_dojotools_notification_router` is deprecated — postman covers everything it did and more.

One security item also closed: the log viewer's search mode was passing the `pattern` field directly into a shell command. That's fixed.

---

## Index — 4.7.1

### ZQ-1 Exclusion Suite (stages 8b–8f)

Five new post-filter fields added to the Index Command DSL, applied in order after all inclusion filters:

| Field | Excludes |
|---|---|
| `exclude_entity_ids` | Explicit entity ID list |
| `exclude_domain` | All entities in a domain |
| `exclude_label` | All entities carrying a label |
| `exclude_integration` | All entities from an integration |
| `exclude_device` | All entities belonging to a device |

Exclusions apply after the full inclusion set is resolved. Safe to use as an ACL injection layer — append exclusion fields to any existing Index Command call without restructuring it. Pattern 10 "The ACL Stripper" added to `zq1_patterns.md`.

### Compound/Recursive Index Command DSL

`index_command` now accepts a nested dict in addition to the existing string DSL:

```yaml
index_command:
  operator: union
  index_1:
    label: security_camera
  index_2:
    label: motion_sensor
```

Supports all four set operators (`union`, `intersection`, `difference`, `symmetric_difference`) with arbitrary nesting depth. Timeout auto-scales to `min(timeout × 3, 15)` seconds when a compound dict is detected. Ninja Summarizer routes `index_command` dict fields through the compound indexer automatically — no caller changes needed.

### Camera Enrichment Inline

Camera entities in Index results now return a structured inline block:

```json
{
  "analysis": "...",
  "cached_at": "2026-04-23T09:15:00",
  "cache_age_minutes": 12,
  "snapshot_path": "/config/www/zen_snapshots/camera_front_door.jpg",
  "found": true
}
```

Also available under `domain_context.camera` in the full result.

---

## Camera — v1.2.0

### `info` Mode

Returns entity attributes, cached analysis metadata, and the stored `_default_ctx` for the camera — everything needed to audit cache state without triggering a new look.

### `sendto` Field

Available on `look` results. Dispatches the captured frame to a destination without a separate call:

- `image.zen_image_<slot>` — writes to a dashboard snapshot slot (`canvas`, `portrait`, `wallpaper`, `id_pic`, `dashboard_snap`, `home_state`)
- `person.*` — fires `zen_dojotools_postman` with the image attached directly to the push notification

### `_default_ctx` Preservation

`look` and `scan` modes were clobbering the context index call stored by `set_default_ctx`. Fixed. Both modes now read the existing drawer before writing and conditionally merge `_default_ctx` back into the updated cache entry. `set_default_ctx` survives repeated look/scan cycles.

---

## Postman — v1.0.0

**Supersedes `zen_dojotools_notification_router`** (deprecated, retained for backwards compatibility).

### `kata_input` Field

Pass a kata payload dict directly from ninja pipeline output. Postman derives `title` and `message` automatically when explicit values are not set:

- `title` → `{component | title} — {period | title}`
- `message` → `attention` field, falling back to `suggested_act_desc`, then `'Action required.'`

### `breakthrough` Field

`breakthrough: true` bypasses the house sleep gate regardless of urgency level. Equivalent to urgency >= `life_safety_bypass` for gate evaluation only — push still goes out at the declared urgency, no tier change.

### `zen_postman_response_router` Automation

Listens for `mobile_app_notification_action` events and re-emits as `zen_event(kind: postman_response, tag, action, device_id)`. Runs `mode: parallel, max: 10` — concurrent taps from multiple household members are handled independently. Postman's step 8.5 push-ack wait and any independent subscriber can correlate by `tag`.

### `notification_router` Deprecation

`zen_dojotools_notification_router` is deprecated. All capabilities are now in postman:

- `kata_input` → `kata_input` field (same dict shape)
- `breakthrough` → `breakthrough` field
- Android notification fields (`channel`, `importance`, `color`, `vibrationPattern`, `ledColor`, `sticky`, `persistent`, `clickAction`, `car_ui`, etc.) → `notification_data` dict passthrough
- `notification_targets` → `push_targets` in user `postman_profile` drawer

The script remains in place. It will be removed in a future release.

---

## Security

### Log Viewer — Pattern Field Sanitization

The `pattern` field in `zen_log_search` mode was interpolated directly into a shell command via `grep`. A crafted input could be used to read files outside the intended log scope. Fixed by stripping `'`, `"`, `` ` ``, and `;` in a dedicated sanitization step before the field reaches the shell template.

`lines` minimum lowered from 50 to 25 while in there.

---

## Files Changed

| File | Change |
|---|---|
| `packages/zenos_ai/dojotools/dojotools_index.yaml` | ZQ-1 exclusion suite (8b–8f); compound/recursive index_command DSL; camera enrichment inline |
| `packages/zenos_ai/dojotools/dojotools_camera.yaml` | `info` mode; `sendto` field; `_default_ctx` preservation in look + scan |
| `packages/zenos_ai/dojotools/dojotools_postman.yaml` | v1.0.0; `kata_input`; `breakthrough`; `zen_postman_response_router` automation |
| `packages/zenos_ai/dojotools/dojotools_utilities.yaml` | `notification_router` deprecated; announce refs updated to postman |
| `packages/zenos_ai/dojotools/dojotools_systemtools.yaml` | Log viewer pattern sanitization; lines min → 25 |
| `packages/zenos_ai/dojotools/dojotools_summarizers.yaml` | Ninja: `index_command` dict routing to compound indexer |
| `zenos_ai/docs/scripts/zen_dojotools_postman_readme.md` | New — full postman reference doc |
| `zenos_ai/docs/scripts/zen_dojotools_index_readme.md` | v4.7.1; compound DSL; camera enrichment; exclusion suite |
| `zenos_ai/docs/scripts/zen_dojotools_summarizers_readme.md` | v4.7.0; Ninja index_command routing |
| `zenos_ai/docs/scripts/readme.md` | All 4.3 entries updated |
| `zenos_ai/docs/custom_templates/zen_query_jinja.md` | Exclusion suite (8b–8f) documented |
| `zenos_ai/docs/zen_hyperindex/zq1_patterns.md` | Pattern 10 "The ACL Stripper"; exclusion field reference table |

---

## Upgrade Notes

No schema migrations. No cabinet changes. No action required on running installs.

**Postman users:** If you're calling `zen_dojotools_notification_router` from automations, no immediate action needed — the script still runs. Migrate to `zen_dojotools_postman` at your own pace. Map `notification_targets` to `push_targets` in the user `postman_profile` drawer; everything else is a field rename or moves into `notification_data`.

**Camera users:** `_default_ctx` is now preserved across look/scan cycles. If you were working around the clobber by re-calling `set_default_ctx` before each look — you can stop.
