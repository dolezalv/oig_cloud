# Plan: Fix oig_cloud recorder-executor log flood

> Internal review artifact (not for upstream merge). Reviewed at tag
> `v2.0.6-pre.13-HF`; cited line numbers refer to that ref.

## Problem

On a live Home Assistant install (HA core on Python 3.14, oig_cloud
`v2.0.6-pre.13-HF`), `home-assistant.log` floods with HA-core warnings:

```
WARNING [homeassistant.helpers.frame] Detected code that accesses the database
without the database executor; Use
homeassistant.components.recorder.get_instance(hass).async_add_executor_job()
for faster database operations. Please report this issue
```

Each warning carries a full `stack_info=True` traceback (~18 lines). Observed
rate: a startup burst then a steady **~4/min (≈240/hr)**; the log had reached
**102 MB** before truncation. Functionally harmless (the integration keeps
working), but it is an unbounded log-growth / SD-card-wear risk and buries real
log lines.

## Root cause (evidence)

- `oig_cloud` is the **only** custom integration on the box that touches the
  recorder (verified by grep across all `custom_components/`). ge_spot et al.
  produce a *different* warning class (`homeassistant.util.loop` blocking-call),
  not this one.
- The warning is logged via `_report_usage_no_integration` — HA walked the
  stack and found no integration frame, because the offending recorder query
  runs inside a **worker thread** (the executor), so the integration that
  scheduled it is not on the thread's stack. That is why the log stack trace is
  undiagnosable and source attribution required reading the code.
- HA core emits this whenever a recorder function (`history.*`,
  `statistics.*`, `session_scope`) runs on a thread that is **not** the
  recorder's dedicated executor. oig_cloud is **inconsistent**: most sites use
  the **general** executor (`hass.async_add_executor_job`); two already use the
  correct recorder executor (`get_instance(hass).async_add_executor_job`) —
  i.e. a half-finished migration.

### Sites (read at `v2.0.6-pre.13-HF`)

Wrong — general executor, **each warns**:

| File:line | recorder fn | hass handle |
|---|---|---|
| `custom_components/oig_cloud/service_shield.py:2222` | `recorder.history.state_changes_during_period` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:5141` | `get_significant_states` | `self._hass` |
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:9278` | `history.state_changes_during_period` | `self._hass` |
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:12903` | `statistics_during_period` | `hass` (param) |
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:13131` | `history.get_significant_states` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:14673` | `get_significant_states` | `self.hass` |
| `custom_components/oig_cloud/balancing/core.py:429` | `statistics_during_period` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_statistics.py:621` | `history.state_changes_during_period` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_statistics.py:741` | `history.state_changes_during_period` | `self.hass` |

Already correct — the template for the fix:

| File:line | pattern |
|---|---|
| `custom_components/oig_cloud/boiler/profiler.py:96,104` | `instance = get_instance(self.hass)` then `instance.async_add_executor_job(state_changes_during_period, …)` |
| `custom_components/oig_cloud/oig_cloud_battery_health.py:113,127` | `get_instance(self._hass).async_add_executor_job(get_significant_states, …)` |

Also to verify in review (uses recorder, but appears already routed via
`get_instance`/`session_scope`): `oig_cloud_adaptive_load_profiles.py`
(`get_instance` + `session_scope`), and the dead-looking
`oig_cloud_battery_health_old.py` (confirm it is not imported/loaded — if dead,
leave it, do not "fix" unused code).

## Fix

For each wrong site, replace the general executor with the recorder's dedicated
executor, mirroring the two correct sites:

```python
from homeassistant.components.recorder import get_instance
...
states = await get_instance(<hass>).async_add_executor_job(<fn>, <hass>, ...)
```

- Add the `get_instance` import in each scope that lacks it (these are local,
  in-function imports today; keep that style).
- Use the **exact** hass handle at each site (`self.hass`, `self._hass`, or the
  `hass` parameter) — they differ per call.
- No behavioural/semantic change: same recorder function, same args, same
  result type. The only change is *which thread pool* runs it.

## Why this is the correct fix (not a mask)

This is precisely what the HA warning instructs and what
`recorder.get_instance(hass).async_add_executor_job` exists for. The recorder
executor serialises DB access on the recorder's own connection; using the
general executor pool risks SQLite contention with recorder writes. So the fix
removes the warning **and** improves DB-access correctness.

## Risks / edge cases (attack these in review)

1. **Import path** — `get_instance` lives in `homeassistant.components.recorder`
   (top-level), while some sites import `history` from
   `homeassistant.components.recorder.history`. Ensure the added import is the
   correct module, not under `.history`.
2. **Recorder readiness** — `get_instance(hass)` returns the recorder instance;
   if recorder is not yet set up, it can return `None`/raise. Are any of these
   call sites reachable before recorder setup (e.g. during early
   `async_setup_entry` / startup burst)? The current code already runs these in
   an executor *after* await points, so recorder should be up — but confirm,
   especially the statistics "first calc after start" path.
3. **Single-threaded recorder executor** — long history/statistics queries now
   serialise against recorder writes. For a home-scale DB this is fine and is
   the intended model, but confirm none of these queries are pathologically
   long (e.g. 30-day `get_significant_states` over many entities) such that they
   would stall recording. (The same query already ran in a thread; only the pool
   changes.)
4. **Per-site hass handle correctness** — a wrong handle (`self.hass` vs
   `self._hass`) would raise `AttributeError`. Each patch must match the file.
5. **`statistics_during_period` signature** — confirm arg order is preserved
   when only the executor wrapper changes.
6. **Pre-release churn** — file/line may move upstream between now and the PR;
   the PR must be rebased on upstream HEAD, re-locating sites by pattern not
   line number.

## Verification (executed in step 3, on the box)

1. Apply the patch to the on-box copy; restart HA.
2. Confirm oig_cloud loads: 0 new `ERROR`, forecast/statistics/balancing
   sensors still update, boiler control + LG SmartThinQ still healthy.
3. Measure the frame-warning rate over 5–10 min → expect **~0** (allow a small
   one-time startup residue from any unconverted/early path).
4. If residue remains, re-attribute (debug-log a window) before claiming done —
   do not declare success on a partial drop.

## Delivery

- **On-box patch** — immediate relief; will be reverted by the next HACS
  redownload (accepted; documented).
- **Upstream PR** to `psimsa/oig_cloud` — the durable fix (step 4), rebased on
  upstream default branch.

## Out of scope (tracked separately)

- oig_cloud `battery_health` **false SoH** (reports ~57% vs BMS ~89%) — a
  calculation bug, unrelated to the executor.
- ge_spot blocking-call (`tzdata`) warning — different integration/class.
- OIG local-telemetry proxy migration — shelved (no DNS-redirect hardware).
- EOL Raspbian 10 OS migration — separate backlog item.

## Rollback

Restore the pre-change backup (`oig_cloud-backup-<ts>.tar.gz` taken on the box),
or trigger a HACS redownload — either restores upstream files.
