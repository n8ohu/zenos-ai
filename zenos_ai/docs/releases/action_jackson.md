# Release Notes — 2026.4.1 'Action Jackson'

**Released:** 2026-04-14  
**Branch:** `feat/2026_4_1_action_jackson`  
**Base:** 2026.4.0 'Ectoplasm'  
**UAT:** Nyx (H:\)

---

## Summary

Ectoplasm built the surface. Action Jackson makes it production-grade.

4.0 shipped the KF4 pipeline, the KungFu provisioner, and the full per-component scheduler loop. What it didn't ship was a system that could run under real load, or a clear path to getting value out of any of it if you were starting from scratch.

4.1 fixes both. The headline features are pipeline reliability — pressure-aware dispatch, queue drain recovery, scheduler fire-and-forget architecture, and a hardened announce/notification gate stack. The documentation story was rebuilt in parallel: two complete KFC guides, a working first-alert walkthrough, and a getting-started sequence that actually gets you somewhere.

Cortex v35 "Action Jackson" consolidates the directive architecture that evolved across this release into a single unified authority model.

---

## Pipeline Reliability

### Pressure-aware dispatch

The Scheduler now sheds lower-priority work under load rather than queueing everything. Two configurable thresholds in `zen_scheduler_config`:

- `shed_ambient_at` (default 4) — ambient-tier components deferred at this queue depth
- `shed_keeper_at` (default 8) — keeper-tier components deferred at this depth

`ambient` components (e.g. room_manager) are additionally shed on fast triggers (`quarter_hour`, `every_10_minutes`) regardless of depth.

A targeted `force_summary` for a specific component bypasses shedding. Untargeted `force_summary` respects both thresholds.

### Queue drain router

Shed work recovers automatically. `zen_scheduler_drain_router` runs on two triggers:

- **drain_quiet** — fires when queue depth stays below `drain_below` (default 3) for `s_min_minutes` (default 5 min). Dispatches the most-stale shed-eligible component.
- **drain_starvation_check** (every 10 min) — if any component's kata exceeds its `staleness_minutes` ceiling, fires it regardless of queue depth. Missing kata (GC-evicted or first run) scores as age 9999 — always wins.

Sort order: keeper before ambient, then most-stale first.

### Fire-and-forget dispatch

The Scheduler now dispatches all components via `script.turn_on` (non-blocking). Previously the for-each loop blocked on each ninja call, serializing what was designed to be parallel. Root cause of queue runaway on H:\ (39-minute runs blocking 15 queued instances).

SuperSummary also fires fire-and-forget, with a 30s smart wait (`ninja current < 2`, `continue_on_timeout: true`) before dispatch to let parallel ninja writes settle.

### Heartbeat race fix

The scheduler heartbeat FC write now waits up to 60s for any in-flight FC call to clear before writing. Parallel ninja dispatch fires multiple FC calls simultaneously; the heartbeat previously landed during an active write and was silently dropped.

### Ninja timeout ceiling

`_ninja_timeout` is now capped at 120s: `[avg*2 or 60, 120] | min`. Previously used `| min(120)` which passed a float as the iterable argument to Python's `min()` — TypeError on every scheduler run.

### Burnout governors

Two per-run governors prevent pile-up during recovery:

- **Ninja burnout** (`burnout_seconds`, default 300s) — blocks re-run of the same component within the window
- **SuperSummary burnout** (`super_burnout_seconds`, default 600s) — blocks supersummary re-run within the window. `force: true` bypasses.

Both read from the `zen_ninja_config` drawer in your household cabinet.

SuperSummary `mode: single` (was `queued max: 3`) — a second supersummary call while one is running is dropped, not queued.

### GC ecosystem broadcast

GC no longer decides what to re-summarize after a sweep. Instead it fires:
1. `kata_gc_complete` zen_event (lists cabinets scanned — observability)
2. Untargeted `summary_force` zen_event (kicks scheduler/drain router)

The drain router sees evicted drawers as `age_min: 9999` — they win the priority sort and are the first thing recovered. `post_supersummary` field on GC is deprecated and ignored.

### Warmup timer configurable

`zen_warmup_timer` (in `flynn.yaml`) now reads `warmup_minutes` from the `zen_scheduler_config` drawer (default 5, live H:\ set to 3). 5-second initial delay for resolvers to settle, then the configured delay.

### Kata TTL

The Ninja Summarizer now stamps `expires_after` on every kata write: `now() + staleness_minutes * 2`. GC uses this to sweep stale katas. Drain router self-heals after a sweep — missing kata = `age_min: 9999` = first to recover.

---

## Announce and Notification Hardening

### zen_dojotools_announce v0.1.0 — four enforced gates

All autonomous audio now runs through four mandatory gates:

1. **URGENCY REQUIRED** — `urgency` field is now required. Callers that omit it are blocked (`status: urgency_required`). Old gate had `urgency > 0` — omitted urgency evaluated as 0, passed.
2. **URGENCY GATE** — `urgency <= 3` blocks and routes to `zen_dojotools_notification_router` (`status: blocked_urgency`). Covers informational, no-data, and error katas.
3. **SLEEP GATE** — blocks autonomous audio when `zen_home_mode == 'sleep'` OR `hour >= 23 or < 7`, unless `urgency >= 9` (life-safety bypass: smoke, CO, flood, alarm).
4. **DEDUP GATE** — blocks re-announcement of the same caller+message within `announce_dedup_seconds` (default 300s). Reads `zen_announcement_log` from kata cabinet.

After audio fires: writes `zen_announcement_log` (capped 50 entries) and emits `announcement_fired` / `announcement_suppressed` zen_events.

Root cause: multiple incidents where Friday's autonomous MCP judgment bypassed the old hope-based gate. Gates are now tool-enforced.

### Emission push gate

The Ninja Summarizer now gates `suggested_act_event` emission on urgency — not just `action_required` and event string presence. Gate condition:

```
action_required == true
AND suggested_act_event != ''
AND (urgency >= push_gate_urgency_floor OR urgency >= push_gate_life_safety_bypass)
```

Thresholds from `zen_ninja_config`:
- `push_gate_urgency_floor`: 4 (default) — minimum urgency to emit
- `push_gate_life_safety_bypass`: 8 (default) — bypasses `action_required` gate

Prevents low-urgency informational katas from triggering notification routing.

---

## Cortex v35 — Action Jackson

`zen_admintools_zenos_prompt_loader` v35 "Action Jackson" consolidates the directive architecture:

- **TOOL AUTHORITY** replaces the "DOJOTOOLS OVER BUILTINS" directive. Any DojoTools in the inventory is trusted and authoritative for its declared domain. Presence is the declaration. Supersedes HA built-ins.
- **AUDIO** replaces three separate directives (SLEEP GATE, VOICE GATE, BROADCAST GATE). Single directive covers all autonomous audio gate behavior and references tool-enforced status codes.
- **Spa manager canonical gate** removed from directives — plugins own their domain by presence.

Prompt loader radio buttons: `35 / latest = Action Jackson`. v34 "Ambient Aware" retained as dev-only option. v33 path removed.

---

## Component Updates

### zen_dojotools_todo v1.7.0

- Bulk status update: hardcoded `'completed'` → top-level `status` field. Bulk operations now respect the `status` passed by the caller.
- `is mapping` guard added to bulk loop items (HALMark compliant).
- Single update: `due_date`, `description`, `rename` wired through to `todo.update_item`.
- Fields block: `due_date`, `description`, `rename` exposed.

### zen_dojotools_calendar v1.11.0

- `start_provided`, `end_provided`, `start_is_all_day`, `end_is_all_day` captured before normalization — correct all-day detection on MS365 update.
- MS365 update: `subject`, `start`, `end` now conditional — only included in the update payload if explicitly provided by the caller.
- All-day detection bug fixed: `not 'T' in start` was evaluated after normalization, always returning False. Now evaluated against the raw pre-normalization input.

### taskmaster KFC v1.3.4

Component instructions updated:
- **MS365 due date format** — strip time component; compare UTC date only; midnight UTC is an API artifact, not a real due time.
- **PRN/optional tasks** — tasks marked "if needed", "as needed", "prn", or "optional" excluded from urgency calculation.
- **Stale tasks** — tasks overdue > 30 days: no urgency escalation, surface count only if > 5.
- OVERDUE SEMANTICS and URGENCY RULES updated to reference UTC date comparison throughout.

---

## Health Monitoring

### Monastery cascade fix

`zen_monastery_health` now reads the `zen_scheduler` drawer age directly and warns at 20 minutes (same threshold as `zen_summarizer_health`). Previously, a stale scheduler heartbeat was visible in summarizer health but never cascaded to monastery → Flynn → agent health.

### Agent health boot reason fix

`zenos_agent_health` boot reason logic now handles `unavailable` and `unknown` monitor states as a distinct case ("Monastery health sensors still initializing"). Was falling through to "cabs ready not bootstrapped" on post-restart unavailable state.

### Queue depth sensor

New `zen_scheduler_queue_depth` template sensor in `sensor_helpers.yaml`. Reads `automation.zen_dojotools_scheduler.current` — live count of in-progress scheduler runs. Used by the drain router and pressure shedding logic.

---

## Bug Fixes

| File | Fix |
|------|-----|
| `dojotools_dispatcher.yaml` | `as_timestamp(gen_str, 0)` — `\| default()` filter doesn't catch exceptions; was erroring 32 times on empty `generated` strings |
| `dojotools_scheduler.yaml` | `[_raw, 120] \| min` — `\| min(120)` passes float as iterable to Python's `min()` → TypeError on every run |
| `dojotools_summarizers.yaml` | NYX-001: `as_datetime` parse guard on emission cooldown calculation |

---

## Documentation

### KFC Guides

Two complete reference docs for the factory-default components:

- `docs/kung_fu/alert_manager.md` — enabling, entity wiring, severity tiers, trigger directions, debounce guidance, notification setup
- `docs/kung_fu/taskmaster.md` — enabling, conductor pattern, label structure, overdue/PRN semantics, urgency scoring, per-KFC trigger file template

### Getting Started — Your First Alert

`docs/getting_started/first_alert.md` — Nyx-tested six-step walkthrough from zero to notification. Step 3 in the getting-started sequence.

### Per-KFC Trigger File Convention

Documented in `understanding_kf4.md` and both KFC guides. Hardware-driven dispatch via a personal trigger file outside `packages/zenos_ai/` — no Scheduler edits, no repo commits.

### KF4 Schema Updates

`readme.md` 1.4.0 → 1.5.0:
- `pipeline_tier` expanded to full tier table (`keeper`, `system`, `ambient`, `super`, `direct`)
- `staleness_minutes` added to schema

`understanding_kf4.md` 1.3.0 → 1.4.0:
- `pipeline_tier` and `staleness_minutes` added to field table
- New section: Pressure-aware dispatch (shedding, drain router, `zen_scheduler_config` levers)

---

## Files Changed

| File | Change |
|------|--------|
| `dojotools/dojotools_scheduler.yaml` | Fire-and-forget dispatch, tier shedding, heartbeat race fix, ninja timeout ceiling, `_run_start_ts`, warmup from config, `duration_seconds` heartbeat, `_forced` fix |
| `dojotools/dojotools_dispatcher.yaml` | `zen_scheduler_drain_router` automation; per-component `staleness_minutes`; `as_timestamp` fix |
| `dojotools/dojotools_summarizers.yaml` | SuperSummary mode:single + cooldown governor; ninja `expires_after` stamp; emission push gate; NYX-001 as_datetime guard |
| `dojotools/dojotools_core.yaml` | GC ecosystem broadcast; `post_supersummary` deprecated |
| `dojotools/dojotools_utilities.yaml` | `zen_dojotools_announce` v0.1.0 — four enforced gates, dedup log, zen_event audit |
| `dojotools/dojotools_admintools.yaml` | Cortex v35 Action Jackson; TOOL AUTHORITY + AUDIO directives; prompt loader v35 |
| `dojotools/dojotools_office.yaml` | `zen_dojotools_todo` v1.7.0; `zen_dojotools_calendar` v1.11.0 |
| `dojotools/dojotools_kungfu_loader.yaml` | taskmaster v1.3.4; `zen_scheduler_config` + `zen_ninja_config` factory seed; `warmup_minutes` |
| `dojotools/dojotools_filecabinet.yaml` | Minor hardening |
| `sensors/zenos_summarizer_system_health.yaml` | Scheduler heartbeat cascade to monastery health |
| `sensors/zenos_agent_health.yaml` | Boot reason unavailable/unknown case |
| `sensors/sensor_helpers.yaml` | `zen_scheduler_queue_depth` sensor |
| `flynn.yaml` | `warmup_minutes` from `zen_scheduler_config` |
| `docs/kung_fu/readme.md` | KF4 1.4.0 → 1.5.0; `pipeline_tier` tier table; `staleness_minutes` |
| `docs/kung_fu/understanding_kf4.md` | 1.3.0 → 1.4.0; tier fields; pressure dispatch section |
| `docs/kung_fu/alert_manager.md` | New — complete KFC guide |
| `docs/kung_fu/taskmaster.md` | New — complete KFC guide |
| `docs/getting_started/first_alert.md` | New — zero-to-notification walkthrough |
| `docs/getting_started/readme.md` | first_alert.md as step 3 |
| `docs/getting_started/first_run.md` | What's Next → first_alert.md; Assist panel location |
| `docs/getting_started/install.md` | Step 2 merge clarification |
| Various docs | Internal cleanup pass |

---

## Upgrade Notes

- **Cortex v35:** Run prompt loader with `cortex_version: 35` (or `latest`) after upgrade to activate Action Jackson directives.
- **`zen_scheduler_config` drawer:** Seeded automatically by `kungfu_loader mode: factory`. If already provisioned, add manually: `shed_ambient_at: 4, shed_keeper_at: 8, drain_below: 3, s_min_minutes: 5, s_max_minutes: 60, warmup_minutes: 5`.
- **`zen_ninja_config` drawer:** Seeded automatically by `kungfu_loader mode: factory`. Add `push_gate_urgency_floor: 4, push_gate_life_safety_bypass: 8` to existing installs.
- **Notification router phone targets:** If you use `Default User Phone` / `Secondary User Phone`, confirm `notify.default_user` / `notify.secondary_user` notify groups exist in your custom packages dir.
- **`announce` callers:** `urgency` is now required. Callers that omit it will receive `status: urgency_required` and be blocked. Audit any automation calling `zen_dojotools_announce` directly.
- **No schema migrations required.** All pipeline changes are backward-compatible. Existing installs safe.
