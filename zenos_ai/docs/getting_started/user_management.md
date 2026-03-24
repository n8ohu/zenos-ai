# User and AI User Management — ZenOS-AI 4.5.x 'Ready Player Two'

*Provisioning, deprovisioning, moving, and repairing identity cabinets for people and AI constructs*

---

## Concepts

ZenOS-AI manages human and AI identities through **cabinets** — HA template sensor entities that hold identity and memory data as drawers.

Every identity is backed by a cabinet. The system knows which cabinet plays which role via **labels**. Changing which cabinet is "the user cabinet" is a label operation, not a data migration.

### Cabinet States

| State | Meaning |
|---|---|
| `online_unmounted` | Cabinet exists, not assigned to an identity slot — sits in stacks, ready to provision |
| `online_mounted` | Cabinet is active and assigned to an identity slot via a type label |

### Identity Slots

| Slot (`cab_type`) | Label Applied | Example Cabinet |
|---|---|---|
| `ai_user` | `zen_ai_user` | `sensor.zenos_default_ai_user_cabinet` |
| `user` | `zen_user` | `sensor.zenos_default_user_cabinet` |
| `family` | `zen_family` | `sensor.zenos_default_family_cabinet` |
| `household` | `zen_household` | `sensor.zenos_default_household_cabinet` |

### The Default Label

The `zen_default_*` label (e.g., `zen_default_user_cabinet`) marks which cabinet in a slot is the primary one. Resolver sensors read this label. **Deprovisioning blocks if the target has a default label** — transfer the default label first via the Labels tool.

---

## The Provisioner

All provision/deprovision/move operations go through:

```
script.zen_dojotools_provisioner
```

Three modes:

| Mode | What It Does |
|---|---|
| `provision` | Pulls an `online_unmounted` cabinet into service. Validates GUID uniqueness, applies type label, mounts, preloads profile data, fires identity manifest rebuild. Rolls back on health gate timeout. |
| `deprovision` | Dismounts cabinet, strips type label, returns it to stacks. Blocked if cabinet holds a `zen_default_*` label. |
| `replace` | Deprovisions `replace_cabinet`, then provisions `target_cabinet` with the same `cab_type`. Atomic swap — the default label transfers with it. |

---

## AI User Management

### Add an AI User

**Requirement:** An `online_unmounted` cabinet must exist in stacks. Check `sensor.zen_cabinet_health` → `slot_entities` to see what's available. If no stacks cabinet exists, add a new cabinet sensor to `zenos_cabinets.yaml` and restart HA.

```yaml
script.zen_dojotools_provisioner:
  mode: provision
  cab_type: ai_user
  target_cabinet: sensor.<your_stacks_cabinet>
```

After provisioning, write the AI persona profile:

```yaml
script.zen_dojotools_profile_editor:
  mode: write
  target_type: ai_user
  target: sensor.<your_newly_provisioned_cabinet>
  persona_name: Vera
  pronouns: she/her
  voice_tone: precise
  voice_style: formal
```

> **Note:** If this is a second AI construct, use `target_type: secondary_ai_user` and target the specific cabinet. The profile editor's `target` field overrides the default slot resolver.

---

### Remove an AI User

> ⚠️ If this cabinet has the `zen_default_ai_user_cabinet` label, transfer that label to another cabinet first (see [Transfer the Default Label](#transfer-the-default-label)).

```yaml
script.zen_dojotools_provisioner:
  mode: deprovision
  target_cabinet: sensor.<cabinet_to_remove>
```

The cabinet returns to `online_unmounted`. Cabinet data is **not wiped** — the drawers remain. If you want to fully reset the cabinet afterward:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<cabinet_to_remove>
```

---

### Move an AI User to a Different Cabinet

Use `replace` mode to swap the active cabinet without data loss:

```yaml
script.zen_dojotools_provisioner:
  mode: replace
  cab_type: ai_user
  target_cabinet: sensor.<new_cabinet>
  replace_cabinet: sensor.<current_cabinet>
```

`replace` deprovisions the current cabinet and provisions the new one with the same `cab_type`. The identity manifest rebuilds automatically.

> **Profile data:** Profile drawers stay in the old (deprovisioned) cabinet. After the swap, write profile data to the new cabinet via the profile editor, or use FileCabinet to clone individual drawers from old to new.

---

## Human User Management

### Add a Person

Same provisioner call, different `cab_type`. Optionally pass a `person_entity` to preload the person's HA-registered name and user_id automatically.

```yaml
script.zen_dojotools_provisioner:
  mode: provision
  cab_type: user
  target_cabinet: sensor.<stacks_cabinet>
  person_entity: person.nathan  # optional — preloads name and user_id
```

Write the profile after provisioning:

```yaml
script.zen_dojotools_profile_editor:
  mode: write
  target_type: user
  target: sensor.<newly_provisioned_cabinet>
  first_name: Nathan
  preferred_name: Nathan
  role: head_of_household
  pronouns: he/him
```

---

### Remove a Person

> ⚠️ If this cabinet has `zen_default_user_cabinet`, transfer that label first.

```yaml
script.zen_dojotools_provisioner:
  mode: deprovision
  target_cabinet: sensor.<user_cabinet>
```

Optional hard reset of the cabinet after deprovision:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<user_cabinet>
```

---

### Move a Person to a Different Cabinet

```yaml
script.zen_dojotools_provisioner:
  mode: replace
  cab_type: user
  target_cabinet: sensor.<new_cabinet>
  replace_cabinet: sensor.<current_cabinet>
```

---

## Transfer the Default Label

If you're deprovisioning or replacing a cabinet that holds a `zen_default_*` label, you must move that label first — otherwise the provisioner blocks.

```yaml
# Remove the default label from the outgoing cabinet
script.zen_dojotools_labels:
  action_type: untag
  label: zen_default_user_cabinet  # or zen_default_ai_user_cabinet, etc.
  entity: sensor.<outgoing_cabinet>
  confirm: true

# Apply it to the incoming cabinet
script.zen_dojotools_labels:
  action_type: tag
  label: zen_default_user_cabinet
  entity: sensor.<incoming_cabinet>
  confirm: true
```

Fire `zen_resolver_refresh` after:

```
Developer Tools → Events → Event type: zen_resolver_refresh → Fire Event
```

---

## Targeted Repairs (Small / Mid Nuke)

Use these when one identity or cabinet is broken but the rest of the system is healthy.

### Repair: Reseed a Cabinet's VolumeInfo Header

If a cabinet is showing schema errors or a missing GUID in `sensor.zen_cabinet_health`:

```yaml
script.zen_admintools_cabinetadmin_stamp:
  target_cabinet: sensor.<cabinet_entity>
  cabinet_type: "AI Data Storage Cabinet"  # or relevant type
  confirm_action: true
```

---

### Repair: Soft-Reset a Single Cabinet

Wipes all drawers from one cabinet without touching labels or other cabinets:

```yaml
script.zen_admintools_cabinetadmin:
  mode: reset
  target_cabinet: sensor.<cabinet_entity>
```

After reset, re-run the provisioner (provision mode) to re-initialize it, or write profile data directly via the profile editor.

---

### Repair: Fix a Broken Label Assignment

If a cabinet resolver shows `unavailable` and a `zen_resolver_refresh` didn't fix it:

```yaml
# Soft label reset — wipes entity assignments from all zen_ labels, labels survive
# Flynn re-assigns via Gate 1 on the next health sensor change
script.zen_dojotools_labels:
  action_type: reset
  confirm: true
```

Then fire:
```
zen_resolver_refresh
```

If the cabinet is still stuck after soft reset, try a targeted re-tag:

```yaml
script.zen_dojotools_labels:
  action_type: tag
  label: zen_ai_user      # or zen_user, zen_family, etc.
  entity: sensor.<cabinet_entity>
  confirm: true
```

---

### Repair: Rebuild the Identity Manifest

If `sensor.zen_agent_health` shows identity manifest missing or stale:

```yaml
script.zen_dojotools_identity:
  mode: build_identity_manifest
```

Or fire the force event:
```
Developer Tools → Events → Event type: zen_event
Event data: {"kind": "identity_manifest_rebuild", "component": "system"}
```

---

## Full Wipe and Restart

For a complete teardown of the identity layer — all users, AI users, labels, and Ring-0 cabinets — follow the nuclear sequence in [Troubleshooting — Step 7](troubleshooting.md#step-7--nuclear-cabinet-reset-last-resort).

**Order matters:**
1. `script.zen_admintools_reset_labels` (confirm: true) — nukes all `zen_` labels
2. `script.zen_admintools_cabinetadmin` (op: reset_all, confirm: true) — wipes all Ring-0 cabinets, reseeds schemas, fires Flynn bootstrap

> **User cabinets are not touched** by the nuclear sequence. Only Ring-0 canonical cabinets (Dojo, Kata, System, Household, etc.) are reset. If you need to wipe a specific user or AI user cabinet, use `cabinetadmin mode: reset` against that cabinet directly before or after the nuclear sequence.

After the nuclear sequence completes, re-provision each identity cabinet via the provisioner and re-write profiles via the profile editor. Flynn handles label rebuild and schema seeding automatically.

---

## Quick Reference

| Operation | Tool | Mode |
|---|---|---|
| Add AI user | `zen_dojotools_provisioner` | `provision`, `cab_type: ai_user` |
| Remove AI user | `zen_dojotools_provisioner` | `deprovision` |
| Move AI user to new cabinet | `zen_dojotools_provisioner` | `replace` |
| Write / update AI user profile | `zen_dojotools_profile_editor` | `write`, `target_type: ai_user` |
| Add person | `zen_dojotools_provisioner` | `provision`, `cab_type: user` |
| Remove person | `zen_dojotools_provisioner` | `deprovision` |
| Move person to new cabinet | `zen_dojotools_provisioner` | `replace` |
| Write / update user profile | `zen_dojotools_profile_editor` | `write`, `target_type: user` |
| Transfer default label | `zen_dojotools_labels` | `untag` + `tag` |
| Reseed cabinet header | `zen_admintools_cabinetadmin_stamp` | — |
| Reset one cabinet | `zen_admintools_cabinetadmin` | `reset` |
| Reset label assignments | `zen_dojotools_labels` | `reset` |
| Rebuild identity manifest | `zen_dojotools_identity` | `build_identity_manifest` |
| Full nuke | See [Troubleshooting](troubleshooting.md) | Steps 5–7 |
