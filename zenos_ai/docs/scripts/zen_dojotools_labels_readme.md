# Zen DojoTools Labels — 4.2.1

*Create, read, delete, and assign labels in the Home Assistant label index*

---

## Overview

`zen_dojotools_labels` is the core primitive for label management in ZenOS-AI. It is how the system creates, inspects, and assigns the labels that power the HyperIndex, KFC components, and cabinet resolution.

Labels are the connective tissue of the system. The Scheduler, Ninja Summarizer, and HyperIndex all work by traversing labels — not by reading hardcoded entity lists. If a label isn't right, the index finds the wrong things (or nothing).

This script is **not MCP-exposed by default**. It is an admin tool. Friday can read the label index for context, but label creation and assignment are operator tasks.

---

## Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `action_type` | select | `read` | `create`, `read`, `delete`, `tag`, `untag`, `reset` |
| `label_list` | list of text | `[]` | Label names (case-insensitive). Required for create, delete, tag, untag |
| `target_entities` | list of entity_ids | `[]` | Entities to tag/untag. Required for tag and untag |
| `confirm` | boolean | `false` | Required `true` for create, delete, and reset |

---

## Actions

### `read`

Returns the label index. Three modes depending on inputs:

| Inputs | Returns |
|---|---|
| Nothing | Full sorted label index |
| `label_list` only | Label details (description) for each label |
| `target_entities` only | Labels assigned to each entity |
| Both | Both of the above |

```json
{
  "status": "ok",
  "action": "read_index",
  "target": "*",
  "index": ["zen_cabinet", "zen_dojo_cabinet", "zen_kata_cabinet", ...]
}
```

Use this first before any write operation to see what already exists.

---

### `create`

Creates new labels in the HA label index. Requires `confirm: true`.

Without confirmation, returns a `status: start` preview showing which labels would be created and which already exist — useful for dry-running before committing.

```yaml
action_type: create
label_list:
  - water
  - irrigation
  - pool
confirm: true
```

```json
{
  "status": "ok",
  "action": "create",
  "created_labels": ["water", "irrigation", "pool"],
  "already_existed": [],
  "message": "Create Action Complete."
}
```

---

### `delete`

Removes labels from the HA label index. Requires `confirm: true`.

> **Warning:** Deleting a label removes it from all entities that carry it. The HyperIndex will no longer find those entities via that label. KFC components that reference the label will stop returning data.

---

### `tag`

Applies one or more labels to one or more entities. No confirmation required.

HA handles the fan-out — all labels are applied to all entities in a single call.

```yaml
action_type: tag
label_list:
  - water
  - irrigation
target_entities:
  - sensor.garden_moisture_zone_1
  - sensor.garden_moisture_zone_2
  - sensor.irrigation_valve_front
  - binary_sensor.irrigation_running
```

---

### `untag`

Removes one or more labels from one or more entities. No confirmation required.

---

### `reset`

Wipes all entity assignments from every `zen_` label. Labels survive — only the assignments are cleared. Use this when label IDs are correct but entity tagging is wrong or missing (for example, after a partial install or a label migration gone sideways).

`confirm: true` is required. The script will refuse without it.

After reset, `zen_resolver_refresh` is fired automatically. Flynn re-engages on the next `zen_label_health` state change and re-assigns via Gate 1.

```yaml
action_type: reset
confirm: true
```

```json
{
  "status": "ok",
  "action": "reset",
  "labels_cleared": 16,
  "message": "Soft reset complete. 16 zen_ label(s) cleared. All assignments wiped. zen_resolver_refresh fired — Flynn will re-assign on next label health check."
}
```

> **This is the soft reset.** For the nuclear option (delete labels entirely and trigger full Flynn rebuild), use `script.zen_admintools_reset_labels`.

---

## Response Status Values

| Status | Meaning |
|---|---|
| `ok` | Action completed |
| `start` | Validation passed, awaiting confirmation (create/delete with `confirm: false`) |
| `partial` | Some labels were skipped (already existed on create, or not found on delete) |
| `error` | Validation failure or missing required input |

---

## Label Best Practices

### Cross-entity labels outperform single-use labels

The HyperIndex traverses the label graph — it finds all entities that share a label and assembles them into a structured snapshot. A label applied to 20 related entities produces a rich, contextualized Kata. A label applied to only one entity produces a thin one.

**Design labels around domains, not individual devices.**

```
# Good — water is a domain label shared across everything in that domain
water → sensor.water_main_flow
water → sensor.water_leak_kitchen
water → sensor.water_leak_basement
water → binary_sensor.irrigation_running
water → switch.irrigation_valve_zone_1

# Weak — single-entity labels add overhead without adding context
water_main_flow_sensor → sensor.water_main_flow
```

### Layer broad and specific labels together

Broad labels give the Ninja Summarizer its domain scope. Specific labels let the Index do targeted queries. Both serve different purposes.

```
# Broad domain label (feeds KFC component)
water → sensor.spa_water_temp
water → sensor.spa_ph

# Specific sub-label (targeted queries)
spa → sensor.spa_water_temp
spa → sensor.spa_ph
spa → sensor.spa_filter_state
```

The KFC drawer uses `label: water` to get the full water picture. A targeted `zen_dojotools_index` query can use `label: spa` to get spa-specific data. No conflicts, richer coverage.

### Prefix conventions

ZenOS-AI system labels use `zen_` prefixes (e.g., `zen_cabinet`, `zen_dojo_cabinet`). Your home labels should use a flat, readable namespace — `water`, `security`, `energy`, `spa`, `lighting`, `hvac`. Keep slugs short and lowercase.

### Labels are cheap — tag generously

Unlike entity exposure (which costs context window), labels cost nothing at rest. Tag every sensor that might be relevant to a domain. The Ninja Summarizer will evaluate relevance at summarization time. It's easier to remove a label later than to wonder why a sensor wasn't showing up in a Kata.

### The label-to-KFC connection

Every KFC component has a `label` field in its Dojo drawer. That label is what the HyperIndex traverses to build the component's entity snapshot. If you create a new label and forget to tag entities with it, the component runs with an empty dataset.

Checklist when adding a component:
1. Create the label (`create`)
2. Tag all relevant entities (`tag`)
3. Reference the label in the KFC drawer (`zen_dojotools_kungfu_writer`)
4. Dry-run with `force_summary` (post=false) and verify the Kata has data

---

## Dependencies

| Dependency | Purpose |
|---|---|
| HA native `labels()` | Read label index |
| HA native `label_description()` | Read label metadata |
| `homeassistant.create_label` | Create labels |
| `homeassistant.delete_label` | Delete labels |
| `homeassistant.add_label_to_entity` | Tag entities |
| `homeassistant.remove_label_from_entity` | Untag entities |
| `zen_resolver_refresh` event | Fired after reset to re-trigger resolver sensors |
