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
| `custom_components/oig_cloud/oig_cloud_battery_forecast.py:14739` | `get_significant_states` | `self.hass` |
| `custom_components/oig_cloud/balancing/core.py:429` | `statistics_during_period` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_statistics.py:621` | `history.state_changes_during_period` | `self.hass` |
| `custom_components/oig_cloud/oig_cloud_statistics.py:741` | `history.state_changes_during_period` | `self.hass` |

**10 wrong sites total.** Note `12903`'s handle is a local
`hass = self.hass or self._hass` (line 12889), not a parameter — preserve that
fallback when wrapping: `get_instance(hass)`. Note `14673` and `14739` are both
inside `_try_load_last_month_from_history` (a rare month-boundary path) — see
verification M1 handling below.

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
recorder_instance = get_instance(<hass>)
if recorder_instance is None:      # mirror adaptive_load_profiles.py's guard
    return  # or skip the read, per the site's existing error path
states = await recorder_instance.async_add_executor_job(<fn>, <hass>, ...)
```

- Add the `get_instance` import in each scope that lacks it (these are local,
  in-function imports today; keep that style).
- `get_instance` is re-exported from **both** `homeassistant.components.recorder`
  and `homeassistant.helpers.recorder`; the repo's two correct templates use
  *different* ones (`profiler.py` → `helpers.recorder`; `battery_health.py` →
  `components.recorder`). Either works — just do **not** import it from
  `…recorder.history`. Match whichever the surrounding code already uses.
- Use the **exact** hass handle at each site (`self.hass`, `self._hass`, or at
  `12903` the local `hass = self.hass or self._hass`) — they differ per call;
  a wrong handle raises `AttributeError`.
- Mirror the existing `None`/readiness guard (`adaptive_load_profiles.py`
  already does `if not recorder_instance`) so a not-yet-ready recorder degrades
  gracefully instead of raising.
- No behavioural/semantic change: same recorder function, same args, same
  result type. The only change is *which thread pool* runs it.

## Why this is the correct fix (not a mask)

This is precisely what the HA warning instructs and what
`recorder.get_instance(hass).async_add_executor_job` exists for. The recorder
executor serialises DB access on the recorder's own connection; using the
general executor pool risks SQLite contention with recorder writes. So the fix
removes the warning **and** improves DB-access correctness.

## Risks / edge cases (attack these in review)

1. **Import path** — `get_instance` is re-exported from both
   `homeassistant.components.recorder` and `homeassistant.helpers.recorder`
   (the two correct templates in this repo use different ones). Either is fine;
   just do not import it from `…recorder.history`.
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

A plain elapsed-time window is **insufficient** and risks a false pass: the
steady ~4/min flood comes from a *frequently-polled* site, but several wrong
sites run only rarely (`14673`/`14739` month-boundary; the statistics
"first calc after start"). A 5–10 min sample would never exercise those, so it
could read ~0 while a rare path is still unconverted. So verification has two
parts:

1. **Apply + sanity.** Patch the on-box copy; restart HA. Confirm oig_cloud
   loads: 0 new `ERROR`, forecast/statistics/balancing sensors still update,
   boiler control + LG SmartThinQ still healthy.
2. **Attribute the steady source FIRST (before patching is ideal).** Identify
   which frequently-polled site produces the ~4/min steady stream — briefly
   enable `custom_components.oig_cloud` debug logging and correlate the debug
   line preceding each frame warning, or bisect by feature toggle. This pins the
   dominant culprit so the steady-rate drop is attributable, not coincidental.
3. **Steady-rate check.** After patching, measure the frame-warning rate over
   ≥10 min → the steady stream must go to **0**.
4. **Exercise the RARE paths explicitly** (do not wait for elapsed time):
   trigger `_try_load_last_month_from_history` (`14673`/`14739`) and the
   statistics daily/first-calc path (`621`/`741`) — e.g. reload the integration
   / call the relevant service / temporarily shorten the schedule in a scratch
   copy — and confirm **no** frame warning is emitted from those paths after the
   fix. A site that cannot be triggered in test is called out explicitly as
   "fixed-by-inspection, not exercised."
5. If any residue remains, re-attribute before claiming done. Conversely, a ~0
   steady reading is **not** sufficient on its own — the rare paths in step 4
   must be checked, or the result is a false pass (per review M1).

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
