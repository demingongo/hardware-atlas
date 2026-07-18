# Design Notes — Low‑Leakage Battery Sense Module

## Why a switched divider?

A permanently connected 470 kΩ + 330 kΩ divider across a 4.2 V LiPo draws:

    I = V_BAT / (R1 + R2) = 4.2 V / 800 kΩ ≈ 5.25 µA continuously

On a 500 mAh cell that is ~126 µAh per day — small, but in a deep-sleep design
where other consumers are in the nA–µA range, the divider becomes the dominant
drain.

The switched design reduces standby draw from the divider to **< 1 µA**
(dominated by MOSFET off-state leakage Ids), with no impact on measurement
accuracy.

---

## Voltage divider

### Goal

Map the 1S LiPo range (3.0 V – 4.2 V) into the ESP32-C3 ADC input range (0 V – 3.3 V).

### Values chosen: R1 = 470 kΩ, R2 = 330 kΩ

    V_ADC = V_BAT × R2 / (R1 + R2)
          = V_BAT × 330 / 800
          = V_BAT × 0.4125

| V_BAT  | V_ADC  | Within ADC range (0–3.3 V)? |
| ------ | ------ | --------------------------- |
| 4.20 V | 1.73 V | ✓                           |
| 3.70 V | 1.53 V | ✓                           |
| 3.00 V | 1.24 V | ✓                           |

### Inverse formula (ADC to battery voltage)

    V_BAT = V_ADC × (R1 + R2) / R2
          = V_ADC × 800 / 330
          ≈ V_ADC × 2.424

### Alternative higher-ratio divider (better ADC resolution)

Replacing R2 with 560 kΩ gives a ratio of 0.544×, spreading the 3.0 V – 4.2 V
range across 1.63 V – 2.28 V instead of 1.24 V – 1.73 V. This increases
resolution by ~32% on a 12-bit ADC but is not necessary for most battery
monitoring use cases.

---

## Gate pull-up resistor (R3)

### Function

Holds Q1 (AO3401) gate at BAT when U1 (PC817C) is off, ensuring Vgs = 0 V and
the MOSFET is fully off.

### Value: 100 kΩ

- **When MOSFET is OFF (U1 off):** both ends of R3 sit at BAT → **zero current
  through R3**. No standby leakage contribution from R3.
- **When MOSFET is ON (U1 saturated):** gate is pulled toward GND, so current
  flows through R3. Theoretical estimate at 4.2 V: I = 4.2 V / 100 kΩ = 42 µA.
  **In practice, measured total active current is ~8.4 µA at 4.0 V** — much
  lower than theory predicts. The likely cause is that the PC817C Vce at the
  very low collector current required here (~5–42 µA) is higher than the
  datasheet Vce_sat (measured at IF = 10 mA). A higher Vce leaves GATE elevated
  above GND, reducing the voltage across R3 and cutting R3 current
  significantly. The net result is better than expected: lower active current
  while Q1 still turns on fully (Vgs is still well below the threshold).

### Why not smaller (e.g. 10 kΩ)?

Smaller R3 would require more collector current from U1 to pull the gate down.
100 kΩ only needs a few µA (see measured value above), which PC817C handles
easily. There is no benefit to forcing more current through R3.

### Can R3 be larger (e.g. 470 kΩ)?

Yes. At 470 kΩ the theoretical pull-up current when ON drops to ~9 µA and the
real current would be even lower. U1 still has enough margin. Use 470 kΩ if
minimising active current matters more than worst-case switching margin.

---

## LED series resistor (R4)

### Function

Sets LED current through U1 (PC817C) from the ESP32-C3 GPIO (3.3 V logic high).

### Value: 2.2 kΩ

    I_LED = (V_GPIO − V_F) / R4
          = (3.3 V − 1.2 V) / 2200 Ω
          ≈ 0.95 mA

PC817C minimum CTR at IF = 1 mA (grade C): 80%.

    I_collector_min = I_LED × CTR_min = 0.95 mA × 0.8 ≈ 0.76 mA

Current required to pull gate down through R3:

    I_required = V_BAT / R3 = 4.2 V / 100 kΩ = 42 µA  (theoretical worst case)

In practice the measured R3 current is much lower (see R3 section), so the
actual margin is far greater than the worst-case figure below.

Margin (worst case): 0.76 mA / 42 µA ≈ **18×**. U1 will fully saturate.

### Acceptable range for R4

| R4 value | I_LED   | I_collector_min | Margin over 42 µA (worst case) |
| -------- | ------- | --------------- | ------------------------------ |
| 1.0 kΩ   | 2.1 mA  | 1.68 mA         | 40×                            |
| 2.2 kΩ   | 0.95 mA | 0.76 mA         | 18× ← chosen                  |
| 4.7 kΩ   | 0.45 mA | 0.18 mA         | 4.3×                           |

All values in the 1 kΩ – 4.7 kΩ range provide adequate margin. 2.2 kΩ balances
GPIO loading and switching margin.

---

## Q1 — AO3401A verification

| Parameter       | Datasheet value          | In this circuit                   |
| --------------- | ------------------------ | --------------------------------- |
| Vds max         | −30 V                    | Max ≈ −4.2 V (1S LiPo) ✓         |
| Vgs(th)         | −0.5 V to −1.5 V         | Vgs = −4.2 V when ON ✓            |
| Id max          | −4 A continuous          | I ≈ 5.25 µA ✓                     |
| Rds(on) typical | 60 mΩ at Vgs = −4.5 V   | V_drop = 5.25 µA × 60 mΩ ≈ 0 V ✓ |
| Ids off-state   | < 1 µA typical           | Primary standby leakage source    |

**Gate drive margin:** at full battery (BAT = 4.2 V, Gate pulled to GND by U1):
Vgs = GND − BAT = 0 − 4.2 = −4.2 V, well beyond the worst-case threshold of
−1.5 V.

---

## U1 — PC817C verification

| Parameter         | Datasheet value   | In this circuit                          |
| ----------------- | ----------------- | ---------------------------------------- |
| Vceo (transistor) | 35 V              | Vce_max ≈ 4.2 V (MOSFET off) ✓          |
| Ic max            | 50 mA             | I_collector ≈ 0.76 mA ✓                 |
| IF max            | 50 mA             | I_LED ≈ 0.95 mA ✓                       |
| CTR (IF = 1 mA)   | 80–300% (grade C) | Worst case 80% → 18× gate margin ✓      |

**Vce when MOSFET is OFF:**
- U1 collector is at GATE = BAT ≈ 4.2 V
- U1 emitter is at GND
- Vce = 4.2 V — well within 35 V rating ✓

---

## ADC Noise and Supply Decoupling

### Observed behaviour

The ESP32-C3 ADC is susceptible to two noise sources when this module is in use:

1. **BLE-correlated noise (USB power):** The RF transceiver injects switching
   noise onto the 3.3 V rail during advertising and connection intervals.
   Manifests as a ±25 mV spread at ADC_BAT (±60 mV on V_BAT, ±2–3%) even
   when the battery voltage is perfectly stable.

2. **Supply bias error (5 V pin power):** When the ESP32 is powered from a
   battery management circuit via its 5 V pin rather than a PC USB port, the
   higher-impedance supply allows BLE switching transients to momentarily pull
   the 3.3 V LDO output below nominal. Because the ADC uses VDD as its
   reference, every reading is biased proportionally low — a systematic
   under-read of ~15–20% was observed in practice.

### Software mitigation

- Oversample: take ≥32 samples per measurement and average them.
- Only report a new BLE battery level when the rounded percentage changes
  (avoids tray-icon flicker from ±1% jitter).

### Hardware fix (recommended)

Add two decoupling capacitors in parallel between the ESP32’s 3.3 V rail and
GND, placed as close to the VDD/3.3 V pin as possible:

| Ref | Value  | Dielectric | Package | Voltage | Purpose                        |
| --- | ------ | ---------- | ------- | ------- | ------------------------------ |
| C1  | 100 nF | X7R MLCC   | 0805    | ≥10 V   | High-frequency bypass (BLE RF) |
| C2  | 10 µF  | X5R MLCC   | 0805    | ≥10 V   | Bulk charge reservoir          |

The 100 nF handles fast (MHz-range) transients from the radio; the 10 µF
provides charge during slower supply variations.

### Candidate parts

| Ref | Manufacturer | Part number        | Package | Dielectric | Voltage |
| --- | ------------ | ------------------ | ------- | ---------- | ------- |
| C1  | Murata       | GRM21BR71E104KA01L | 0805    | X7R        | 25 V    |
| C1  | Samsung      | CL21B104KBCNNNC    | 0805    | X7R        | 25 V    |
| C2  | Murata       | GRM21BC81A106KE18L | 0805    | X5R        | 10 V    |
| C2  | Samsung      | CL21A106KAYNNNE    | 0805    | X5R        | 10 V    |

---

## SMD optocoupler alternatives

If DIP-4 through-hole is unsuitable, the following SO-4 parts are direct
replacements with identical pinout. No connection changes required.

| Part       | Package | Typical CTR at 1 mA |
| ---------- | ------- | ------------------- |
| TLP291     | SO-4    | 100–300%            |
| EL817SMD   | SO-4    | 80–300%             |
| LTV-817SMD | SO-4    | 80–300%             |
