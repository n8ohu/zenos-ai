# Getting Started — ZenOS-AI 2026.5.0 'Fry's Grandpa'

*From fresh install to live agent*

---

New to ZenOS-AI? This folder is your onboarding path. Read in order for a clean first-run experience.

---

## Documents in This Folder

### 1. `install.md` — Installation

Start here. Covers:

* File drop: `packages/zenos_ai/` and `custom_templates/zenos_ai/` into your HA config
* `configuration.yaml` setup
* Pasting the conversation agent system prompt
* Setting your conversation agent before restart
* Restarting HA and verifying Flynn initializes cleanly
* Checking `sensor.zen_agent_health` for first-boot confirmation

**Time:** ~15 minutes on a clean install.

---

### 2. `first_run.md` — First Boot & OOBE

After installation. Covers:

* What Flynn does on first boot (the stepgate sequence)
* The OOBE conversation flow — naming your AI, seeding the household profile
* Using the persona selector and input helpers
* Editing profiles after setup
* Common first-run issues and fixes

---

### 3. `first_alert.md` — Your First Alert

After first boot. Covers:

* Enabling `alert_manager` via your AI or prompt_loader
* Creating a `notify.admin_devices` service
* Tagging your first entity for monitoring
* Firing a test summary and checking the kata
* Getting your first notification

**Do this next — it's the fastest way to see the pipeline working end to end.**

---

### 4. `entity_exposure.md` — What to Expose to Your Agent

Before you hand Friday the keys. Covers:

* The three-tier model: Actionable vs Contextable vs Invisible
* What always gets exposed (DojoTools scripts)
* What never gets exposed (AdminTools, cabinet sensors, credentials)
* How to use labels instead of direct exposure for high-cardinality data
* Token efficiency: one label on 50 sensors beats 50 direct reads

**Read this before configuring your conversation agent's entity list.**

---

### 5. `cabinet_placement.md` — Where Things Go and Why

After entity exposure. Covers:

* Dojo vs Kata vs System cabinet — what lives where
* Drawer vs KFC — when to use each
* The quick-reference placement table
* Common misplacement patterns and how to fix them

---

### 6. `oobe.md` — OOBE Walkthrough

The six-step first-boot configuration protocol in detail. Covers:

* What OOBE is and when it fires
* The conversational path (chatting with your AI to configure it)
* The fast path (setting helpers directly in Settings → Helpers)
* How Flynn detects OOBE completion
* What to do if the OOBE notification won't dismiss

---

### 7. `troubleshooting.md` — Troubleshooting

When something isn't right. Covers:

* Reading health sensors at a glance
* Summarizer kill switches — checking and resetting
* Seven-step graduated repair sequence:
  1. Check `sensor.zen_agent_health` → `roster` attribute
  2. Fire `zen_resolver_refresh`
  3. Reseed templates (`zen_admintools_reset_template`)
  4. Label reset (soft — `reset` mode)
  5. Cabinet repair (`cabinetadmin`)
  6. Nuclear label reset (`zen_admintools_reset_labels`)
  7. `reset_all` cabinet sequence

---

### 8. `user_management.md` — User and AI User Management

For adding, removing, and moving identities after initial setup. Covers:

* Provisioning a new AI user or human user from a stacks cabinet
* Deprovisioning (removing) an identity and returning the cabinet to stacks
* Swapping an identity to a different cabinet (replace mode)
* Transferring the default label before a swap
* Targeted identity-layer repairs: single cabinet reset, label fix, manifest rebuild
* Full identity-layer nuke and restart

---

## Recommended Order

```
install.md → first_run.md → first_alert.md → entity_exposure.md → cabinet_placement.md → oobe.md
```

Keep `troubleshooting.md` and `user_management.md` open in a tab. You might need them.

---

→ **[Full Documentation Hub](../readme.md)**
→ **[Understanding KF4 — Adding Components](../kung_fu/understanding_kf4.md)**
→ **[Health Sensor Reference](../sensors/readme.md)**
