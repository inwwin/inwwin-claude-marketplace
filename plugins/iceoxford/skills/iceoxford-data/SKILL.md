---
name: iceoxford-data
description: How to find, query and correctly interpret the ICEOxford cryostat recording sessions archived in MotherDuck (database `iceoxford`) - temperatures of the 1K pot, 4K plate, 50K stage, sample mount and cassettes, He gauge and dump pressures, needle valve, heater loop, F-70 compressor, CoBrite laser and Thorlabs power meter, plus event markers. Use whenever someone asks about a cooldown, a session, a step test, cryostat temperatures or pressures, "the run where ...", or wants a plot, dashboard or analysis of ICEOxford data through the MotherDuck MCP.
---

# ICEOxford session data in MotherDuck

The ICEOxford monitor (a PySide6 app on the lab PC, repo `inwwin/ICEOxford`) samples a
1 K-pot helium-flow cryostat at ~1 Hz. While the user has **"Keep in cloud"** switched on, the
data belongs to a numbered **session**; session rows go to Supabase (7-day buffer) and are
copied to MotherDuck for good by an archiver in the monitor (at start, local midnight, and when a
session ends or is edited). You read the MotherDuck copy.

## Ground rules

- **Read only.** Use the MotherDuck `query` tool, never `query_rw` or any write. The monitor is
  the only writer; a stray write corrupts the archive's de-duplication.
- **Database `iceoxford`** = the real cryostat. `iceoxford_sim` holds simulator data from the
  same schema: never mix or join the two, and never report sim data as real.
- **All timestamps are UTC** (`timestamptz`). The lab is in the UK; say "UTC" when you quote
  times, or convert with `ts AT TIME ZONE 'Europe/London'` when the user wants local times.
- **Aggregate in SQL.** A 10-day session is ~930 k rows × 40 columns. Never pull raw rows over
  MCP beyond a few hundred; use `time_bucket`, `min/max/avg`, `arg_min/arg_max`, window functions.
- Before writing SQL for a new kind of question, it is worth calling `get_query_guide` once
  (MotherDuck's own guidance).

## Tables (schema `main`)

| Table | Grain | Key columns |
|---|---|---|
| `sessions` | one row per session | `id`, `started_at`, `ended_at` (NULL = still running), `name`, `notes`, `tags varchar[]` |
| `session_summary` (view) | one row per session | sessions columns + `duration`, `seconds_recorded`, `first_ts`, `last_ts`, `min_1k_pot_k` (COM4 A), `min_4k_plate_k` (COM4 B), `max_heater_pct`, `laser_used`, `power_meter_used`, `markers`, `marker_labels` (list, in time order) |
| `readings` | one row per **second** of a session | `session_id`, `ts` (whole second), `uploaded_at`, then one column per channel |
| `markers` | one row per event | `session_id`, `ts`, `label` (free text) |
| `manual_readings` | one row per typed value | `session_id`, `ts`, `channel` (`MAN:CHAMBER:P` cryostat chamber, `MAN:FORE:P` turbo foreline), `value` (mbar) |
| `channels` | one row per channel | `key` (e.g. `COM4:A:K`), `column_name` (`com4_a_k`), `source`, `label`, `quantity`, `unit` |

There are no primary keys or indexes; filter on `session_id` and a `ts` range.

**`readings` values:** each is the **mean of that second's samples**; NULL means the channel was
not sampled that second (instrument disconnected, on-demand instrument off, read error). Values
are never carried forward. `las_on`/`las_busy` are booleans (last sample of the second). A gap of
missing `ts` rows means the monitor was not running or not uploading; gaps over ~30 s are real
gaps, not zeros. New channels appear as new columns automatically, so check
`information_schema.columns` or `channels` rather than assuming the list below is complete.

## Channels: what each column is

Full table with labels, units and physics notes: [references/channels.md](references/channels.md).
The short version:

| Column | Meaning | Unit |
|---|---|---|
| `com4_a_k` / `_r` | **1K pot** Cernox (also the heater's control input and watchdog sensor) | K / Ω |
| `com4_b_k` | **4K plate** | K |
| `com4_c_k`, `com4_d_k` | Cassette in / out | K |
| `com3_b_k` / `_r` | **LTP** sensor, on the same copper plate as the 1K-pot Cernox (clean reference) | K / Ω |
| `com3_c_k` | 50K stage | K |
| `com3_d_k` | Sample mount | K |
| `com3_a_k` | "Broken Cernox": no readings, ignore | K |
| `com4_out1_setp`, `com4_out1_htr` | Heater loop (Output 1, heats the 1K pot) setpoint and output | K, % |
| `com5_p` | He gauge (INFICON VGC501, pump line after the needle valve) | mbar |
| `daq_ai0_p` | He dump (recovery) pressure | mbar |
| `com4_out3_pct` | Needle-valve drive readback (higher = more open) | % |
| `valve_out`, `valve_sp` | Software valve PID: command and pressure setpoint (logged in manual mode too) | %, mbar |
| `com8_t1`, `t2`, `t3` | Sumitomo F-70 compressor: He discharge, water out, water in | °C |
| `pm_w` | Thorlabs power meter, W as read (negative = needs zeroing). dBm = `10*log10(pm_w/1e-3)`, only for `pm_w > 0` | W |
| `las_*` | CoBrite DX1 tunable laser: `las_wl` nm, `las_f` THz (coarse), `las_off` GHz (fine tune), `las_p` set power dBm, `las_on`, `las_busy` (tuning), `las_tchip`/`las_tbase` °C, `las_ild`/`las_itec` mA | |

## Caveats that change conclusions

1. **He gauge units changed on 2026-10-30 21:40:54 UTC.** From then on `com5_p` is
   helium-corrected. Before it, the gauge was in N₂-equivalent units: multiply by **0.797** to
   compare with later data (valid below 1 mbar only; above 1 mbar there is no simple factor).
   Gauge repeatability ±2 %, accuracy ±15 %.
2. **Loom pickup.** COM3 C, COM4 A, B, C, D share one wiring loom. Smooth, simultaneous bumps on
   those sensors but not on the LTP (`com3_b_*`) are a measurement artifact, not temperature.
   Check against `com3_b_r` before calling a bump thermal.
3. **Generic curves.** The 50K stage and cassette sensors use generic curves; their kelvin values
   near room temperature (and anywhere warm) are only approximate.
4. **Labels are current, not historical.** `channels` is overwritten with the monitor's current
   labels on each archive run. `session_summary` hard-codes COM4 A = 1K pot, COM4 B = 4K plate
   (wiring as of 2026-10). If the user mentions rewiring, trust the user.
5. **4K stage running dry:** a fast rise of `com4_b_k` (e.g. +0.05 → +0.3 K/min) followed by a
   collapse of `com5_p` at an unchanged valve means the 4K stage dried up, not a valve fault.
   Pot behaviour and suitable PID gains change after it.
6. **Heater (Model 336 Output 1):** in closed loop the manual % is *added* to the PID output in
   current, not power (heater % = x²/100). `max_heater_pct` = 100 can be a brief kick when the loop
   was closed, not sustained heating; look at the time series.
7. **Valve mode is not a column.** Auto/manual, gains, presets and limits are only in `markers`
   (labels like `valve manual`, `valve auto log sp=… kp=…`, `valve preset '…'`).
8. **A running session is incomplete here.** If `ended_at` is NULL, MotherDuck lags up to ~1 day.
   The newest part is in Supabase project `dekweyfdktiozjwdgwab`, table `iceoxford.readings`
   (same column names) where `session_id` = X and `ts` > MotherDuck's `max(ts)` for that session;
   say so if you cannot reach Supabase. Data outside sessions is never archived (it lives only
   7 days in Supabase and in the lab PC's local CSVs).

## Markers

Markers are the lab notebook: free text written by the user, by MCP clients (other Claude
sessions) and by scripts. Common families (numbers vary):

- `Step test i/n: setpoint X mbar`, `Step test start (…)`, `Step test end: done; restored …`:
  valve step-test scripts (`scripts/valve_step_test.py`).
- `backlash sweep: backlash_pct=…, i_deadband_pct=…, kp_log=…`, `valve test: …`: valve PID tuning
  grid runs.
- `valve integral dead band …`, `valve auto log sp=…`, `valve manual`, `valve output limits …`,
  `valve preset '<name>': …`, `valve presets edited '<name>'`: valve settings changes.
- `heater loop: setpoint X K, ramp Y K/min, PID p/i/d`, `T step: SP a -> b K, P… I… D…`,
  `Loop: setpoint a -> b K (… overshoot … mK)`: heater-loop changes and their measured response.
- `power meter connected …`, `laser connected COM20 (MCP)`, `remote control allowed …`,
  `cloud session #N '…' resumed after M min`: housekeeping.

Search with `label ilike '%…%'`; use `regexp_extract` to pull numbers out of labels. Marker
`ts` has sub-second precision while `readings.ts` is whole seconds, so join on ranges
(`r.ts >= m.ts`), never on equality.

## Workflow

1. **Find the session.** `session_summary` by name, tag, date, content or marker. If the user's
   description is vague, list sessions first and confirm which one.
2. **Check what was recorded**: `count(col)` per column for that session shows which instruments
   were connected (e.g. the laser and power meter are on-demand).
3. **Downsample** to a bucket that gives at most a few hundred rows (`time_bucket`), or locate
   events (thresholds, `arg_min`, markers) and then zoom into a window around them.
4. **Interpret** with the caveats above and quote times in UTC.

## Query recipes

Find sessions:
```sql
select id, name, tags, started_at, ended_at, duration, min_1k_pot_k, min_4k_plate_k,
       max_heater_pct, laser_used, power_meter_used, markers
from session_summary order by started_at desc;

-- by content / marker / tag
select id, name from session_summary where min_4k_plate_k < 3.5;
select id, name from session_summary where list_contains(tags, 'cooldown');
select distinct session_id from markers where label ilike '%He transfer%';
```

Overview of a session at 15-minute resolution:
```sql
select time_bucket(interval '15 minutes', ts) as t,
       avg(com4_a_k) as pot_k, avg(com4_b_k) as plate_k, avg(com3_c_k) as stage50_k,
       avg(com5_p) as he_mbar, avg(com4_out3_pct) as valve_pct, max(com4_out1_htr) as htr_max
from readings where session_id = 2 group by all order by t;
```

Cooldown milestones (first time below a temperature):
```sql
select min(ts) filter (where com4_b_k < 10) as plate_below_10k,
       min(ts) filter (where com4_a_k < 4.2) as pot_below_4k2,
       min(ts) filter (where com4_a_k < 1.5) as pot_below_1k5,
       arg_min(ts, com4_a_k) as coldest_at, min(com4_a_k) as coldest_k
from readings where session_id = 2;
```

Readings around a marker (±10 min, 30 s buckets):
```sql
with m as (select ts from markers where session_id = 2 and label ilike '%dry%' limit 1)
select time_bucket(interval '30 seconds', r.ts) as t, avg(com4_b_k) as plate_k, avg(com5_p) as he_mbar
from readings r, m
where r.session_id = 2 and r.ts between m.ts - interval '10 minutes' and m.ts + interval '10 minutes'
group by all order by t;
```

Stability of a hold between two markers (e.g. a step-test step):
```sql
-- lead() over ALL markers, so a step ends at the next marker of any kind (e.g. "Step test end")
with s as (select * from (select ts, lead(ts) over (order by ts) as next_ts, label from markers
                          where session_id = 2)
           where label like 'Step test %/%: setpoint%')
select s.label, s.ts, count(*) as n, avg(r.com5_p) as mean_mbar,
       stddev(r.com5_p) / avg(r.com5_p) * 100 as sd_pct, avg(r.com4_out3_pct) as valve_pct
from s join readings r on r.session_id = 2 and r.ts >= s.ts and r.ts < s.next_ts
group by all order by s.ts;
```

He gauge on one scale across the gas-correction change:
```sql
select ts, case when ts < timestamptz '2026-10-30 21:40:54+00' and com5_p < 1
                then com5_p * 0.797 else com5_p end as he_mbar_helium
from readings where session_id = 2 and com5_p is not null order by ts;  -- then bucket it
```

Gaps (monitor off or not uploading):
```sql
select ts - gap as gap_start, ts as gap_end, gap from (
  select ts, ts - lag(ts) over (order by ts) as gap from readings where session_id = 2)
where gap > interval '30 seconds' order by gap_start;
```

Manual pressures (chamber vacuum, foreline):
```sql
select ts, channel, value from manual_readings where session_id = 2 order by ts;
```

For charts or dashboards, follow the MotherDuck MCP's own dive guide (`get_dive_guide`); keep
the SQL above as the data layer and plot temperatures on a log axis when they span 300 K → 1 K.
