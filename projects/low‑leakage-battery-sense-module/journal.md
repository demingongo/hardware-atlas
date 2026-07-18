# Journal — Low‑Leakage Battery Sense Module

---

## 2026-07-18 — ADC Noise Analysis and MAX17048 Evaluation

### readBatteryPercent() cost (32 samples, 3 ms settle)

#### Time per call

| Step                                      | Cost        |
| ----------------------------------------- | ----------- |
| `delay(3)` settle                         | 3,000 µs    |
| 32 × `analogReadMilliVolts` (~50 µs each) | ~1,600 µs   |
| `pinMode`, `digitalWrite`, math           | ~5 µs       |
| **Total**                                 | **~4.6 ms** |

#### Current per call (extra above normal loop baseline)

| Rail    | Consumer                      | Current   | Duration  |
| ------- | ----------------------------- | --------- | --------- |
| 3.3 V   | Opto LED (R4 + PC817C)        | ~1 mA     | ~4.6 ms   |
| BAT     | Divider + R3                  | ~8.4 µA   | ~4.6 ms   |

Average extra drain at a 60 s reporting interval:

    1 mA × 4.6 ms / 60 000 ms ≈ 77 nA

Negligible for a BLE gamepad that already draws ~30–80 mA during active use.

---

### Is ADC-based battery monitoring practical on ESP32-C3?

**Short answer: no, not reliably under BLE.**

The observed ±3–5% scatter (±36–60 mV at ADC pin, ±88–145 mV on V_BAT) persists even
with 32-sample averaging. Three structural problems make it very hard to fix in
software alone:

1. **BLE-correlated noise — not random.**
   The 2.4 GHz transceiver fires bursts timed to BLE connection events. More
   samples average out thermal and quantisation noise but do nothing for noise
   that is time-locked to the BLE schedule. The bad samples cluster together
   rather than cancel out.

2. **Ratiometric ADC with a noisy reference.**
   `analogReadMilliVolts()` uses VDD (3.3 V) as its reference. Any droop on that
   rail — from BLE transients, LDO load regulation, USB vs. battery supply
   impedance — scales all readings proportionally. A 1% VDD error gives a 1%
   reading error with no way to distinguish it from a real battery change.

3. **Flat OCV curve in the mid-range.**
   A 1S LiPo has a nearly flat open-circuit voltage between 20–80% charge.
   A ±60 mV voltage uncertainty maps to roughly ±5% SOC in that flat region —
   the measurement error is larger than the signal.

**Conclusion:** The ±5% scatter is structural. Decoupling caps and more samples
help, but the fundamental accuracy ceiling for an ESP32-C3 ADC under active BLE
is around ±3–5% SOC — good enough for a rough bar indicator, not a reliable
fuel gauge.

---

### MAX17048 evaluation

The MAX17048 (Analog Devices / Maxim) is a dedicated 1S/2S LiPo fuel gauge with
I2C interface (Adafruit breakout: https://adafru.it/5580).

#### How it works

- Measures cell voltage with an **internal precision reference** (~±1.25 mV).
- Runs the **ModelGauge** algorithm on-chip: models the cell's OCV–SOC curve,
  compensates for temperature and cell impedance, tracks charge/discharge cycles.
- Reports SOC with 1/256 % resolution over I2C at up to 400 kHz.
- Provides a configurable **ALRT interrupt** (open-drain, active-low) that fires
  when SOC drops below a programmable threshold — no polling required.

#### Key specifications (from MAX17048 datasheet)

| Parameter             | Value                              |
| --------------------- | ---------------------------------- |
| SOC accuracy          | ±1% typical, ±7.5% max (no cal.)  |
| Voltage resolution    | 78.125 µV / cell                   |
| Voltage accuracy      | ±1.25 mV                           |
| Supply voltage        | 2.5 V – 5.5 V (powered from BAT)  |
| Active current        | ~50 µA                             |
| Sleep current         | ~3 µA                              |
| Interface             | I2C, 100/400 kHz                   |
| Package               | SOT-23-5, DFN-6                    |

#### Does ESP32 ADC noise affect it?

**No.** The MAX17048 is completely immune to ESP32 supply noise because:

- It communicates over **digital I2C** — immune to analog noise on the ESP32 VDD rail.
- It has its **own internal reference voltage**, decoupled from anything the ESP32 does.
- It connects **directly to the battery terminals**, not through the ESP32 supply domain.

The ESP32 only reads a register over I2C; the measurement itself happens entirely
inside the MAX17048.

#### Comparison

| Criterion              | Discrete divider (this circuit) | MAX17048             |
| ---------------------- | ------------------------------- | -------------------- |
| SOC accuracy (BLE on)  | ±3–5% (measured)                | ±1% typical          |
| Noise susceptibility   | High (VDD-referenced ADC)       | None (own reference) |
| Continuous BAT drain   | < 1 µA standby                  | ~50 µA always-on     |
| Per-measurement drain  | ~77 nA avg (60 s interval)      | included in 50 µA    |
| Component count        | 6 resistors + MOSFET + opto     | 1 IC + 1–2 caps      |
| SOC algorithm          | Linear voltage map (crude)      | ModelGauge (good)    |
| Low-battery alert      | No                              | Yes (ALRT pin)       |
| Unit cost (approx.)    | < $0.50                         | ~$2–4 (bare IC)      |

#### Verdict

For a **BLE gamepad** that is always powered during use, the MAX17048 is the
right choice. It gives reliable ±1% readings regardless of BLE activity, and
50 µA continuous is acceptable for a device that already draws 30–80 mA.

The discrete divider circuit remains useful in **deep-sleep designs** where every
µA matters and a ±5% coarse indicator is acceptable — or as a low-cost option
if decoupling caps are added and the host-side UI only shows a bar rather than
a number.
