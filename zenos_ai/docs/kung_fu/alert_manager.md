# Alert Manager — KFC Guide

> **KF4 1.3.0** | ZenOS-AI 2026.4.1

Alert Manager is ZenOS-AI's built-in alert component. It monitors entities tagged with the **`Alert Manager`** label, interprets their states, and dispatches notifications when something needs attention.

It ships disabled. Enable it when you're ready to start monitoring.

---

## Enabling alert_manager

`alert_manager` ships with `meta.enabled: false`. To enable:

- Ask your AI: *"Enable alert manager"*
- Or set `meta.enabled: true` in the `alert_manager` Dojo drawer via `zen_dojotools_scribe`

New to this? See **[Your First Alert](../getting_started/first_alert.md)** for a full zero-to-notification walkthrough.

---

## What to Wire vs Skip

Not every alert needs instant dispatch. Match trigger timing to what you're actually monitoring:

| Type | Examples | When to dispatch |
|------|----------|-----------------|
| Life safety | Smoke, CO, gas, flood sensor | Immediately — wire a per-KFC trigger file |
| Security | Door/window open, motion | Immediately or on mode change — depends on context |
| Equipment | Appliance fault, pump off | Immediately for failure states |
| Chemistry / maintenance | Pool/spa chemistry, filter | Hourly sweep is fine — no trigger file needed |
| Informational | Energy, temperature drift | Scheduled sweep — no trigger file needed |

The Scheduler's built-in `hourly_trigger` handles most passive monitoring. Reserve per-KFC trigger files for things that matter in the moment.

---

## Wiring Instant Dispatch

For entities that need immediate alert on state change, create a dedicated trigger file in your installer's custom packages directory:

```yaml
# packages/your_family/kfc_trigger_alert_manager.yaml
# ⚠️ PERSONAL FILE — DO NOT COMMIT TO REPO

automation:
  - id: 'YOUR_UNIQUE_ID'
    alias: KFC Trigger — Alert Manager
    description: >-
      Fires a targeted force_summary for alert_manager on meaningful state changes.
    triggers:
      - trigger: state
        entity_id:
          - binary_sensor.your_smoke_detector   # REPLACE
          - binary_sensor.your_co_detector      # REPLACE
        to: 'on'
        not_from: [unknown, unavailable]
        for:
          seconds: 5

    actions:
      - event: zen_event
        event_data:
          event:
            kind: summary_force
            component: alert_manager

    mode: queued
    max: 3
```

This file fires `zen_event` with `kind: summary_force`, which the core Scheduler handles as a targeted dispatch for `alert_manager`. No Scheduler changes needed.

---

## Trigger Direction — alert_when_* Labels

The AI needs to know which direction is bad for each entity. Two conventions — you can use them as HA labels on individual entities, or document them in the `component_summary`:

| Entity label | Meaning |
|-------------|---------|
| `alert_when_on` | ON is the concerning state. Tag: leak sensors, smoke detectors, fault sensors. |
| `alert_when_off` | OFF is the concerning state. Tag: pumps that should always be running, valves that should always be open. |

Without these, the AI guesses — and sometimes gets it wrong. A shutoff valve "on" (open) is healthy. Without the label, it might look alarming.

---

## Debounce by Condition Type

| Condition | Recommended debounce |
|-----------|---------------------|
| Smoke / CO | 5 seconds or less |
| Flood / leak | 5–15 seconds |
| Door / window | 30–60 seconds |
| Motion | 60+ seconds, or skip instant dispatch entirely |
| Equipment fault | 30–60 seconds |
| Chemistry / maintenance | None — hourly sweep only |

Set `for: seconds: N` in your trigger block. Prevents false dispatches from transient sensor noise.

---

## Setting Up Notifications

`alert_manager` dispatches via `zen_dojotools_notification_router` to `notify.admin_devices`.

You need a `notify.admin_devices` group in your HA config pointing at your phone:

```yaml
notify:
  - name: admin_devices
    platform: group
    services:
      - service: mobile_app_your_phone  # replace with your actual notify service
```

The notification router respects quiet hours and work hours. To override during an emergency, use the `breakthrough` flag — but alert_manager only sets it for genuine life-safety events.

For more on notification targets and quiet hours: [HA Notify Docs](https://www.home-assistant.io/integrations/notify/).

---

## The component_summary Field

The `component_summary` in the Dojo drawer is what the Ninja Summarizer reads when interpreting your entities. Be specific:

- What does "normal" look like for this subsystem?
- What does "concerning" look like?
- Which specific entities need special interpretation?

Good example:

```
Monitors household safety sensors. Key entities:
- smoke_detector_kitchen (alert_when_on = smoke detected)
- co_detector_basement (alert_when_on = CO detected)
- water_sensor_basement (alert_when_on = water present)
- valve_main_shutoff (alert_when_off = valve closed when it shouldn't be)
Flag any on-state for smoke/CO/water as urgent. Flag shutoff valve off-state as urgent.
```

The more specific you are, the less the AI has to guess.

---

## Adding More Entities

Tag any entity with the **`Alert Manager`** HA label. It's automatically included on the next Summarizer run. No drawer edit needed.

---

*ZenOS-AI KF4 1.3.0 — 2026-04-06*
*Source: Nyx (live system observation), Cayt (dev)*
