# ICEOxford channels in detail

Column = channel key in lower case with `:` → `_` (`COM4:A:K` → `com4_a_k`). Labels below are
the monitor's labels as of 2026-11; the live list is `select * from channels`.

## Instruments

| Port | Instrument | Columns |
|---|---|---|
| COM3 | Lake Shore 336 temperature controller | `com3_{a,b,c,d}_{k,r}` |
| COM4 | Lake Shore 336 temperature controller (runs the heater and the valve drive) | `com4_{a,b,c,d}_{k,r}`, `com4_out1_setp`, `com4_out1_htr`, `com4_out3_pct` |
| COM5 | INFICON VGC501 Pirani-type gauge on the He pump line | `com5_p` |
| Dev1/ai0 | NI USB-6008 reading the He dump pressure transducer (P = 242.3·V − 127) | `daq_ai0_p` |
| COM8 | Sumitomo F-70 He compressor (read only) | `com8_t1`, `com8_t2`, `com8_t3` |
| software | Needle-valve PID in the monitor | `valve_sp`, `valve_out` |
| USB | Thorlabs PM100D optical power meter (on demand) | `pm_w` |
| COM20 | ID Photonics CoBrite DX1 tunable laser (on demand) | `las_*` |

## Temperatures

`_k` = kelvin from the controller's curve, `_r` = raw sensor resistance in Ω (always available
even where the curve is poor).

| Column | Label | Notes |
|---|---|---|
| `com3_a_k` | COM3 A · Broken Cernox | Sensor broken; column is empty. |
| `com3_b_k` | COM3 B · LTP | Mounted on the same copper plate as the 1K-pot Cernox, wired separately from the loom: the clean reference for 1K-pot changes. |
| `com3_c_k` | COM3 C · 50K Stage | Generic curve; loom. ~52 K when cold. |
| `com3_d_k` | COM3 D · Sample Mount | |
| `com4_a_k` | COM4 A · 1K Pot Cernox | Heater control input and watchdog sensor; loom. ~0.94–0.97 K at base with the pot pumped. |
| `com4_b_k` | COM4 B · 4K plate | Loom. ~2.8–3.0 K when cold. A fast rise = 4K stage running dry. |
| `com4_c_k` | COM4 C · Cassette In | Generic curve; loom. |
| `com4_d_k` | COM4 D · Cassette Out | Generic curve; loom. |

Loom = COM3 C and COM4 A–D share one wiring loom; correlated smooth bumps across them (seen as
~30 min pulses on a ~3.85 h cycle in a warm overnight run) are pickup, not temperature.

## Heater loop (COM4 Output 1, heats the 1K pot, control input A)

- `com4_out1_setp` (K) and `com4_out1_htr` (% of heater power).
- Modes (Off / Closed Loop PID / Open Loop) and range are not columns; see markers.
- Model 336 units: P = gain; I = 1000 / Ti[s]; D in % where 100 % = Ti/4 s.
- In closed loop the manual output is added to the PID output in heater **current**:
  x = √(100·manual) + (P/6)·(e + I/1000·∫e dt) + retained integral; heater % = x²/100.
  The integral survives Open Loop, so closing the loop often gives a short heater kick.
- A watchdog switches the heater off above 310 K, faster than 5 K/min, or on invalid readings.

## Needle valve and He pressures

- `com4_out3_pct`: Lake Shore analog Output 3 (0–10 V) readback, which drives the He needle
  valve; higher % = more open. When cold, little or no flow below ~22 %; the cold-hold range
  is ~22–35 % (~25–27 % in Cooldown #1).
- `valve_out`: the software PID's command (%), `valve_sp`: its pressure setpoint (mbar). Both are
  logged whether the PID is in auto or manual; the mode and gains are only in markers.
- The real valve sticks (~1 %) and has ~0.75–2 % backlash; the output needed for a given
  pressure drifts up ~1 %/day.
- `com5_p` (mbar): VGC501 gauge. **Gas correction set to helium from 2026-10-30 21:40:54 UTC**;
  earlier values are N₂-equivalent, ×0.797 for helium below 1 mbar. Repeatability ±2 %,
  accuracy ±15 %. Typical: 1–7 mbar during cooldown, 0.03–0.3 mbar cold (boil-off floor
  ~0.03 mbar).
- `daq_ai0_p` (mbar): He dump/recovery pressure.
- Manual gauges the PC cannot read are in `manual_readings`: `MAN:CHAMBER:P` (cryostat chamber
  vacuum) and `MAN:FORE:P` (turbopump foreline), mbar, one row per typed value.

## Compressor (Sumitomo F-70, °C)

`com8_t1` He discharge, `com8_t2` cooling water out, `com8_t3` cooling water in. (T4 spare, not
logged.)

## Optics

- `pm_w` (W, as read). Negative = sensor needs zeroing. dBm = `10*log10(pm_w/1e-3)` for
  `pm_w > 0`; there is no dBm column. NULL = power meter not connected.
- Laser: `las_wl` output wavelength (nm, double), `las_f` coarse frequency (THz, double),
  `las_off` fine-tune offset (GHz), `las_p` set power (dBm), `las_on` output on, `las_busy`
  tuning in progress, `las_tchip` / `las_tbase` chip / base temperature (°C), `las_ild` diode
  current and `las_itec` TEC current (mA). NULL = laser not connected. A coarse tune switches the
  output off for ~1 s.
