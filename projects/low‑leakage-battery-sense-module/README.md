# Low‑Leakage Battery Sense Module

## Purpose

A switchable voltage divider for reading a 1S LiPo/LiIon battery voltage (3.0 V – 4.2 V) on an ESP32-C3 ADC. The divider is disconnected from BAT when idle, reducing standby current draw to near zero.

Controlled by a single ESP32-C3 GPIO. Designed to pair with a bq25185 charger or any circuit that exposes a direct BAT rail.

---

## Specifications

| Parameter           | Value                           |
| ------------------- | ------------------------------- |
| Input voltage (BAT) | 3.0 V – 4.2 V (1S LiPo/LiIon)  |
| ADC output range    | 1.24 V – 1.73 V                 |
| Standby leakage     | < 1 µA (MOSFET off-state)       |
| Active current draw | ~5.3 µA at 4.2 V                |
| Control logic       | 3.3 V GPIO, active high         |
| ADC input           | 1× analog input (0 V – 3.3 V)  |

---

## Schematic

### Nodes

| Node        | Description                                     |
| ----------- | ----------------------------------------------- |
| `BAT`       | Battery positive (bq25185 BAT pin or LiPo+)    |
| `GATE`      | AO3401 gate — driven by R3 and U1 collector     |
| `ADC_BAT`   | Divider midpoint — to ESP32-C3 ADC input        |
| `GPIO_MEAS` | ESP32-C3 GPIO output — drive HIGH to measure    |
| `GND`       | System ground (shared by ESP32, charger, BAT−)  |

### Netlist

Every connection is listed explicitly. No implicit connections.

```
R3  (Rg_pullup, 100 kΩ, 0805)
    pin 1  →  BAT
    pin 2  →  GATE

Q1  (AO3401, P-channel MOSFET, SOT-23)
    S  (Source)       →  BAT
    G  (Gate)         →  GATE
    D  (Drain)        →  R1 pin 1

R1  (Rtop, 470 kΩ, 0805)
    pin 1  →  Q1 Drain
    pin 2  →  ADC_BAT

R2  (Rbottom, 330 kΩ, 0805)
    pin 1  →  ADC_BAT
    pin 2  →  GND

U1  (PC817C, Optocoupler, DIP-4)
    pin 1  (Anode)      →  R4 pin 1
    pin 2  (Cathode)    →  GND
    pin 3  (Emitter)    →  GND
    pin 4  (Collector)  →  GATE

R4  (Rled, 2.2 kΩ, 0805)
    pin 1  →  U1 pin 1 (Anode)
    pin 2  →  GPIO_MEAS
```

### ASCII Diagram

```
                     R3 (100 kΩ)
BAT ──────┬──────────────────────────────────── [GATE]
          │                                         │
          │                             ┌───────────┘
          │                             │
         (S)                           (G)   ← Q1 AO3401, P-ch MOSFET, SOT-23
          └──────────── Q1 ───────────(D)
                                        │
                                       R1 (Rtop, 470 kΩ)
                                        │
                     [ADC_BAT] ─────────┤
                          │             │
                   ESP32-C3 ADC        R2 (Rbottom, 330 kΩ)
                                        │
                                       GND


U1 PC817C (DIP-4):
  pin 4  (Collector)  ──── [GATE]
  pin 3  (Emitter)    ──── GND
  pin 1  (Anode)      ──── R4 (2.2 kΩ) ──── GPIO_MEAS (ESP32-C3 GPIO)
  pin 2  (Cathode)    ──── GND
```

The `[GATE]` node ties the top diagram and the PC817C section together.

---

## External Interface

| Signal      | Direction | Connects to       | Notes                          |
| ----------- | --------- | ----------------- | ------------------------------ |
| `BAT`       | Input     | Charger BAT pin   | Battery positive, up to 4.2 V  |
| `GND`       | Power     | System ground     | Must be common with ESP32      |
| `GPIO_MEAS` | Input     | ESP32-C3 GPIO (digital out) | **Turns the module ON/OFF.** Drive HIGH to connect the divider to BAT; drive LOW to disconnect it. Do not read voltage here. |
| `ADC_BAT`   | Output    | ESP32-C3 ADC pin (analog in) | **Read battery voltage here.** Scaled output: V_BAT × 0.4125. Only valid while GPIO_MEAS is HIGH. |

---

## Operation

> **GPIO_MEAS is the on/off switch for the entire module.**
> ADC_BAT is where you read the voltage — but only while GPIO_MEAS is HIGH.

| GPIO_MEAS  | U1 LED | U1 transistor | Q1 Vgs          | Q1 state | Divider      |
| ---------- | ------ | ------------- | --------------- | -------- | ------------ |
| LOW (0 V)  | OFF    | Open (high-Z) | 0 V (Gate=BAT)  | **OFF**  | Disconnected |
| HIGH (3.3 V) | ON   | Saturated     | ≈ −4.2 V        | **ON**   | Active       |

### Converting ADC reading to battery voltage

$$V_{BAT} = V_{ADC} \times \frac{R1 + R2}{R2} = V_{ADC} \times \frac{800\,k\Omega}{330\,k\Omega} \approx V_{ADC} \times 2.424$$

| V_BAT  | V_ADC  |
| ------ | ------ |
| 4.20 V | 1.73 V |
| 3.70 V | 1.53 V |
| 3.00 V | 1.24 V |

---

## Files

| File              | Description                          |
| ----------------- | ------------------------------------ |
| `README.md`       | Circuit description and connections  |
| `inventory.md`    | Bill of materials                    |
| `design-notes.md` | Design rationale and calculations    |
