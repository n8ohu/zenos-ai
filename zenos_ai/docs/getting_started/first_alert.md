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

To confirm it's on, ask your AI: *"Is alert_manager enabled?"* It will read the Dojo drawer and tell you.

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

To find your phone's notify service name: **Developer Tools → Services**, filter by `notify.mobile_app_` — it will be something like `notify.mobile_app_pixel_8` or `notify.mobile_app_my_iphone`. Use the part after `notify.` in the config above.

**Restart HA** after adding this (a config reload is not sufficient — `configuration.yaml` changes require a full restart). Confirm `notify.admin_devices` appears in Developer Tools → Services after restart.

---

## Step 3 — Tag your first entity

Pick one entity you want `alert_manager` to watch. A good first choice: a smoke detector, door sensor, or any binary sensor where "on" means something's wrong.

In HA's entity registry, add the label **`Alert Manager`** to that entity.

That's it. No code. The HyperIndex resolves the label automatically — Step 4 fires the first run manually so you don't have to wait for a scheduled sweep.

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

For a notification to fire, the entity needs to be in its alert state — door open, sensor on, whatever "bad" looks like for your entity. If it's currently healthy, `action_required` comes back false and nothing is sent. **Trigger the alert state now** if you want to test the full path.

Once the kata has `action_required: true`, the notification router fires on the next emission cycle (the scheduler runs at minimum every hour when enabled). To test it right now without waiting:

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
