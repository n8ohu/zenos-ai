# Alert Manager — KFC Guide

> **KF4 1.4.0** | ZenOS-AI 2026.4.1

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

The AI needs to know which direction is bad for each entity. Apply one of the `alert_when_*` modifier labels alongside `alert_manager`. The label name encodes the alert condition — no additional configuration needed.

| Label | Nominal | Alert when | Use for |
|-------|---------|------------|---------|
| `alert_when_on` | off | state is `on`, `problem`, or `detected` | Leak sensors, smoke detectors, fault flags |
| `alert_when_off` | on | state is `off` or `unavailable` | Pumps, valves, line power, services that must stay running |
| `alert_when_not_off` | off | any non-`off` state (`on`, `open`, `triggered`, etc.) | Multi-state sensors where any non-idle reading is a problem |
| `alert_when_under_N` | ≥ N | numeric state drops below N (extract N from slug) | Salt level, battery %, filter life, days remaining |

Each label carries a description that the Ninja Summarizer reads as authoritative context — you don't need to re-explain the condition in `component_summary`. The label IS the rule.

Without a modifier label, an entity with `alert_manager` is treated as informational context — the AI reads it but won't flag it as a deviation unless the state is obviously wrong.

### ZQ query shortcut

To find all entities with any `alert_when_*` modifier label across all domains:

```python
{"label": "alert_manager", "label_regex": "^alert_when_"}
```

This returns every alert_manager entity that carries at least one modifier label — useful for auditing coverage.

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

The `component_summary` in the Dojo drawer describes the overall scope of what this component monitors. Keep it high-level — you don't need to re-explain each entity's alert condition here. The `alert_when_*` labels carry that context directly in their descriptions, and the Ninja Summarizer reads them per-entity.

Focus `component_summary` on:
- What subsystems are covered
- Severity tier groupings
- Any cross-entity rules or escalation context

Good example:

```
Monitors household safety and infrastructure sensors across all subsystems.
Severity tiers: critical (smoke/CO/leak/alarm), urgent (water valve, line power),
warning (filter, salt, maintenance), information (grid state, appliance status).
Surfaces active deviations; escalates via notification_router at urgency >= 5.
```

Per-entity semantics live in the `alert_when_*` labels — not here.

---

## Adding More Entities

Tag any entity with the **`Alert Manager`** HA label. It's automatically included on the next Summarizer run. No drawer edit needed.

---

*ZenOS-AI KF4 1.4.0 — 2026-04-07*
*Source: Nyx (live system observation), Cayt (dev)*
