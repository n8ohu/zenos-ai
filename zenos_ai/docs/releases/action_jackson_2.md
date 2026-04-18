# Release Notes — 2026.4.2 'Action Jackson 2'

**Released:** 2026-04-17  
**Branch:** `feat/2026.4.2`  
**Base:** 2026.4.1 'Action Jackson'  
**UAT:** Nyx (H:\)

---

*He didn't deadlock the first time. He's not going to start now.*

---

## Summary

One bug closed, one cortex shipped.

New installs were deadlocking on a cabinet state Flynn had no gate for. Present-but-uninitialized cabinets (state=`init`, no VolumeInfo) never appeared in `missing_cabinets`, so Gate 2 never saw them and nothing ever called cabinetadmin on them. The system just... waited. Forever.

Gate 2.1 closes the gap. Post-warmup, Flynn now reads `slot_to_default_entity` directly and auto-initializes any virgin cabinet it finds. Silent, safe, idempotent — live data is protected by the GUID gate inside `flynn_initialize_cabinets`.

Two other hardening items rode along: summarizer health sensors now wake on `kata_emit` events for tighter heartbeat coverage, and Gate 4's "system ready" check now verifies monastery and agent health are actually confirmed before firing the notification.

---

## Bug Fix

### Gate 2.1 — Virgin cabinet auto-init on new installs (closes #133)

**Root cause:** `zen_cabinet_health` rolls up optional slots as healthy even when their entity is still in `init`. New installs landed in a deadlock: cabinets existed, health said ok, `missing_cabinets` was empty, Gate 2 passed — but nothing had ever written VolumeInfo, so the cabinets were functionally empty and the system couldn't bootstrap.

**Fix:** Gate 2.1, inserted between Gate 2 and Gate 2.5. Reads `slot_to_default_entity` directly (bypasses health rollup), detects any `state=init` entity, calls `script.flynn_initialize_cabinets`.

**Boot safety:** Skipped during warmup and `ha_start`. In that window the recorder may not have restored variables yet — a live cabinet can transiently show `state=init` before its first evaluation. After warmup expires, `init` = genuinely virgin. No false positives on running installs.

**Data safety:** `flynn_initialize_cabinets` is GUID-gated. Any cabinet that already has VolumeInfo is skipped. Running installs are unaffected.

**Early exit updated:** The Flynn early-exit condition now also checks for init-state cabinets. A technically-green health rollup no longer lets Flynn step aside when virgin cabs are present.

**Gate 4 hardened:** The "system ready" notification now requires monastery and agent health to be confirmed before firing. Reaching Gate 4 was necessary but not sufficient — this closes the window where the notification could fire before the system was actually ready.

---

## Health Monitoring

### `zen_health_report` — init-state cabinet awareness

`zen_health_report` now surfaces init-state cabinets as a first-class field. New `health_sensors.init_cabinets` field lists any slots whose default entity is in `state=init`. Diagnosis includes a plain-English entry pointing at Gate 2.1 and the manual fallback if the auto-init stalls.

Previously, a new install deadlocked silently — health report showed `ok`, resolvers showed `all_ok`, and nothing in the output explained why Flynn wasn't bootstrapping.

### Summarizer health — `kata_emit` wakeup

`zen_summarizer_health` and `zen_supersummary_health` now trigger on `kata_emit` events (`component: ninja_summary` and `component: super_summary`) in addition to their existing triggers. Tighter heartbeat coverage — health sensors update immediately when a kata write lands rather than waiting for the next scheduled poll.

---

## Cortex v36 — Index First

`zen_admintools_zenos_prompt_loader` v36 "Index First" addresses token pressure from broad live-state queries.

- **INDEX FIRST** elevated to directive position 2, right after tone. Full explanation of why — GetLiveContext and GetLiveState dump the entire home as an unstructured blob, no scoping, massive token burn. Index first, always. Boot payload compact index is the starting point; `zen_dojotools_index` for live lookups; GetLiveContext is last resort with a required explanation.
- **5 noise directives removed:** truncation meta-notice, time context pointer (cortex is authoritative), grand index keywords (vague, superseded by cortex), follow-up pointer (cortex interaction_policy is authoritative), old TOOL SELECTION — INDEX FIRST (replaced by new position-2 directive).
- **`pipeline.tiers` removed** from cortex — tier membership is inferrable from boot payload buckets.
- **`pipeline.knowledge_tree` removed** from cortex — architecture doc, not operational guidance.
- **`advanced_intents.rules[0]` removed** — GetLiveContext duplicate, now consolidated in `index_selection.getlivecontext_rule`.
- **`programs_auto_exec.find`** reverts to short form after deploy: `"zen_dojotools_index IS live state — searchable, scoped, clean. GetLiveContext = last resort. See directives."` — cabinet write to `sensor.zenos_default_ai_user_cabinet`, version `2.1.2`. Run on H:\ post-deploy.

Load command: `zen_admintools_zenos_prompt_loader: cortex_version: latest, ship_zen_system: false`

---

## Files Changed

| File | Change |
|------|--------|
| `packages/zenos_ai/flynn.yaml` | Gate 2.1 added; warmup exit sentinel; early exit updated; Gate 4 monastery+agent check |
| `packages/zenos_ai/sensors/zenos_summarizer_system_health.yaml` | `kata_emit` wakeup triggers for ninja and supersummary health |
| `packages/zenos_ai/dojotools/dojotools_systemtools.yaml` | `zen_health_report`: init-state cabinet detection added (`init_cabinets` field + diagnosis) |
| `packages/zenos_ai/dojotools/dojotools_admintools.yaml` | Cortex v36 Index First: purpose, directives, cortex blocks added; selector and `_cv` normalization updated |
| `zenos_ai/docs/scripts/zen_flynn_readme.md` | Gate 2.1 documented; early exit updated; Gate 4 updated; Health Sensor → Gate Map updated |
| `zenos_ai/docs/getting_started/troubleshooting.md` | Gate 2.1 in Flynn Gate States table; new-install deadlock symptom added |

---

## Upgrade Notes

No schema migrations. No helper changes. No action required on running installs.

If you were on a new install that hit the deadlock described in #133 — update, run nuclear cabinet reset (Step 7 in the troubleshooting guide), and let Flynn come up clean.
