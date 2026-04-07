# Release Notes — 2026.4.1 'Action Jackson'

**Released:** 2026-04-06  
**Branch:** `feat/2026_4_1_action_jackson`  
**Base:** 2026.4.0 'Ectoplasm'  
**UAT:** Nyx (H:\)

---

## Summary

Ectoplasm built the surface. Action Jackson puts it to work.

4.0 shipped the KF4 summarizer pipeline, the AdminTools provisioner, the full per-KFC scheduler loop, and two starter components (`alert_manager`, `taskmaster`) — all fully wired, all tested, both disabled by default. What it didn't ship was a clear path to getting value out of any of it if you were setting up ZenOS-AI for the first time.

4.1 fixes that. The headline is documentation: two complete KFC guides, a getting-started walkthrough that takes you from zero to your first notification, and a per-KFC trigger file convention that makes hardware-driven dispatch portable and install-safe. Supporting it: the summarizer's tone model gets two new directives, and the notification router becomes install-agnostic.

The system itself is largely unchanged. This is a `.1` because it is not changing how anything works — it's making sure what works is usable.

---

## KFC Guides — alert_manager and taskmaster

Two new KFC reference docs. Both components shipped as code in Ectoplasm. This release ships them as something you can actually configure, wire, and reason about.

### `alert_manager` — `docs/kung_fu/alert_manager.md`

Complete guide for ZenOS-AI's built-in alert component. Covers:

- Enabling from AI conversation or AdminTools
- **What to wire vs skip** — severity-based triage. Life safety and equipment fault states warrant instant dispatch and a per-KFC trigger file. Chemistry, maintenance, and informational entities are fine on an hourly sweep. One table, no ambiguity.
- **Trigger direction table** — `alert_when_on` vs `alert_when_off` explained with entity-type examples. Without this, the AI guesses state semantics — and sometimes gets it wrong.
- **Debounce guidance by condition type** — smoke/CO at 5 seconds; doors at 30–60; chemistry not at all.
- **Notification setup** — what `notify.admin_devices` needs to be and how to wire it.

### `taskmaster` — `docs/kung_fu/taskmaster.md`

Complete guide for the task and calendar management component. Covers:

- Enabling from AI conversation or AdminTools
- **The conductor pattern** — how taskmaster uses a structured todo list as a time-block schedule driver, and how domain lists feed into it
- **Data sources and label structure** — `daily`, `weekly`, `tasks`, `calendar` label domains explained
- **Overdue semantics and PRN items** — what "overdue" means, when PRN items surface, how urgency escalates
- **Morning and evening cycles** — what fires when, what gets read, what gets surfaced
- **Per-KFC trigger file template** — full bed-sensor example for wake/sleep dispatch
- Urgency scoring table — 5 levels, clear thresholds, clear dispatch rules

Both guides include "Enabling" sections at the top per the same pattern: ships disabled, one sentence to turn it on, link to the walkthrough.

---

## Getting Started — Your First Alert

New doc: `docs/getting_started/first_alert.md`

Nyx ran this on H:\ and it works. Six steps. Zero to notification.

The path: enable `alert_manager` → wire `notify.admin_devices` → tag one entity → fire a manual summary → check the kata → get the notification. Each step is one thing. The last step requires the entity to be in its alert state — the doc says so.

This is the recommended next step after first boot. It's now step 3 in the getting-started sequence, between `first_run.md` and `entity_exposure.md`. Reason: it's the fastest path to seeing the full KF4 loop — entity → label → Ninja → kata → notification — working end to end on your actual hardware.

---

## Per-KFC Trigger File Convention

Documented in `understanding_kf4.md` and both KFC guides.

KFCs that need instant dispatch on hardware state changes get a dedicated trigger file per component. The file lives in the installer's custom packages directory — outside `packages/zenos_ai/` — fires a `zen_event` with `kind: summary_force`, and the core Scheduler handles it. No Scheduler edits. No dispatch loop duplication.

This is the same pattern as install-specific notify group config: shared package uses portable names, personal wiring lives in the installer's own space and never touches the repo.

KFCs that are fine with scheduled sweeps need no trigger file. One file per KFC that actually needs it.

---

## KF4 1.3.0 — Three New Schema Fields

`docs/kung_fu/readme.md` bumped to 1.3.0. Three new optional fields added to the drawer schema table:

| Field | What it does |
|---|---|
| `emission_cooldown_minutes` | Minimum minutes between emissions for this component. Prevents notification spam on rapid-fire triggers. |
| `drift_threshold` | Minimum urgency delta required to consider a kata change "significant" for emission gating. |
| `suggested_act_event` | Slug of the action the AI should suggest in kata output. `null` = no action. |

These fields ship in both new KFCs. They were already wired in the Scribe/Ninja pipeline (Ectoplasm). Documentation was the gap.

---

## Tone Directives — v32 Prompt

Two new directives added to the v32 `directives` string in `zen_admintools_zenos_prompt_loader`:

**Urgency is a dispatch signal, not a voice register.** Urgency scores in component kata tell the pipeline whether to act, not how dramatically to speak. Maintenance items are everyday home management — present them that way. Reserve alarm language for genuine life-safety events.

**Cruise director model.** Lead with the conclusion. Omit the reasoning chain unless the user asks. One sentence if something needs attention. Stop when there's nothing to say.

Both directives were validated on H:\ during the Ectoplasm test period (hot tub manager and taskmaster were the loud offenders). They are now part of the base prompt rather than per-KFC patches.

---

## Notification Router — Generic Service Targets

`zen_dojotools_notification_router` in `dojotools_utilities.yaml` no longer calls install-specific notify services directly. The `Default User Phone` and `Secondary User Phone` targets now route to `notify.default_user` and `notify.secondary_user` respectively.

Installers wrap their real device services in notify groups under those names in their own custom packages directory — the same pattern as `notify.admin_devices`. The shared package never sees device names.

**Upgrade note:** If you use the named phone targets, add a notify group file to your custom packages dir:

```yaml
notify:
  - name: default_user
    platform: group
    services:
      - service: mobile_app_your_device   # replace

  - name: secondary_user
    platform: group
    services:
      - service: mobile_app_their_device  # replace
```

`Admin Devices` and `Family Devices` targets are unaffected.

---

## Getting Started — Functional Fixes

Six issues found and fixed in a doc review pass:

| Doc | Issue | Fix |
|---|---|---|
| `first_alert.md` | Step 1: no method given to confirm enabled state | Added: ask your AI "Is alert_manager enabled?" |
| `first_alert.md` | Step 2: "Reload HA" — wrong for configuration.yaml changes | Changed to "Restart HA" with note explaining why |
| `first_alert.md` | Step 2: no guidance on finding your notify service name | Added: Developer Tools → Services, filter by `notify.mobile_app_` |
| `first_alert.md` | Step 3: "next summarizer run" misleading when summarizers are off | Rewritten to point forward to Step 4's manual trigger |
| `first_alert.md` | Step 6: "emission window" undefined | Defined inline: scheduler fires at minimum every hour when enabled |
| `first_alert.md` | Step 6: no instruction to put entity in alert state | Added: trigger the alert state before testing |
| `first_run.md` | "Open a conversation" — no WHERE | Added: Assist panel, chat bubble icon top-right |
| `install.md` | "merge this in" — no example of what that looks like | Added YAML example showing merge into existing homeassistant: block |

---

## Files Changed

| File | Change |
|---|---|
| `docs/kung_fu/alert_manager.md` | New — complete KFC guide |
| `docs/kung_fu/taskmaster.md` | New — complete KFC guide |
| `docs/getting_started/first_alert.md` | New — zero-to-notification walkthrough |
| `docs/kung_fu/readme.md` | KF4 1.2.0 → 1.3.0, three new schema fields |
| `docs/kung_fu/understanding_kf4.md` | Per-KFC trigger file convention, version bump |
| `docs/getting_started/readme.md` | first_alert.md as step 3, numbering cascaded |
| `docs/getting_started/first_run.md` | What's Next → first_alert.md; Assist panel added |
| `docs/getting_started/install.md` | Step 2 merge clarification |
| `dojotools/dojotools_admintools.yaml` | v32 tone directives; taskmaster template updated |
| `dojotools/dojotools_utilities.yaml` | Notification router generic service targets |
| `docs/examples/zen_dojotools_scheduler_custom.yaml` | Generic bed sensor trigger IDs |
| Various docs and templates | Internal cleanup pass |

---

## Upgrade Notes

- **Notification router phone targets:** If you use `Default User Phone` / `Secondary User Phone` in notification calls, add a `notify.default_user` / `notify.secondary_user` group to your custom packages dir before upgrading (see above). Calls that don't resolve will fail silently.
- **alert_manager and taskmaster ship disabled.** Run prompt loader with `ship_alert_manager: true` / `ship_taskmaster: true` to provision. Enable via AI conversation after provisioning.
- **No schema migrations required.** All changes are additive or doc-only. Existing installs safe.
