# Your First Alert

> **Version:** 2026.4.1 | **Last Updated:** April 2026

*The fastest way to see ZenOS-AI's summarizer pipeline working end to end.*

---

You've done first-run setup. Your AI knows your home. Now let's get it to tell you something.

This walkthrough takes you from zero to your first notification using `alert_manager` — ZenOS-AI's built-in alert component. Six steps.

---

## Step 1 — Enable alert_manager

`alert_manager` ships disabled. Enable it by asking your AI:

> "Enable alert manager"

Or via prompt_loader if you're an admin: set `meta.enabled: true` in the `alert_manager` Dojo drawer using `zen_dojotools_scribe`.

Either way, confirm it's on before continuing.

---

## Step 2 — Create a notify.admin_devices service

`alert_manager` sends notifications to `notify.admin_devices`. You need to create this as a notify group in HA pointing at your phone.

In your HA `configuration.yaml` (or a package file):

```yaml
notify:
  - name: admin_devices
    platform: group
    services:
      - service: mobile_app_your_phone  # replace with your actual notify service
```

Reload HA after adding this. Confirm `notify.admin_devices` appears in Developer Tools → Services.

---

## Step 3 — Tag your first entity

Pick one entity you want `alert_manager` to watch. A good first choice: a smoke detector, door sensor, or any binary sensor where "on" means something's wrong.

In HA's entity registry, add the label **`Alert Manager`** to that entity.

That's it. No code. The HyperIndex finds it automatically on the next summarizer run.

---

## Step 4 — Fire a test summary

Ask your AI to run a summary now:

> "Run alert manager summary"

Or fire a `zen_event` directly from Developer Tools:

```yaml
event: zen_event
event_data:
  event:
    kind: summary_force
    component: alert_manager
```

The Ninja Summarizer runs, reads your labeled entity, interprets the state against the `alert_manager` component spec, and writes a Kata to the Kata Cabinet.

---

## Step 5 — Check the kata

Ask your AI:

> "What did alert manager find?"

Or read the `alert_manager` drawer in the Kata Cabinet directly. You should see a compact, structured summary of the entity you tagged — state, attention flag, urgency score.

If the kata is empty or the entity isn't mentioned, double-check the label name. It must match exactly: `Alert Manager` (case-sensitive).

---

## Step 6 — Get the notification

If the entity you tagged is in a concerning state (e.g. the door is open, the sensor is on), `alert_manager` should have set `action_required: true` in the kata. On the next emission window, the notification router fires automatically.

To test it right now without waiting:

> "Send me the alert manager summary"

Your AI will call `zen_dojotools_notification_router` and push a notification to `notify.admin_devices`.

Check your phone. That's the pipeline — all the way through.

---

## What's Next

You've just seen the full KF4 summarizer loop: entity → label → Ninja → kata → notification.

Every other Kung Fu Component works the same way. To add more:

- **[Understanding KF4](../kung_fu/understanding_kf4.md)** — how to write a component from scratch
- **[Alert Manager guide](../kung_fu/alert_manager.md)** — severity triage, debounce, wiring more entities
- **[Entity Exposure](entity_exposure.md)** — what to show your AI and why

---

→ **[Back to Getting Started](readme.md)**
