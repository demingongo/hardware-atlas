# Inventory — Low‑Leakage Battery Sense Module

## Bill of Materials

| Ref | Description         | Value      | Package | Candidate Parts                     | Qty |
| --- | ------------------- | ---------- | ------- | ----------------------------------- | --- |
| Q1  | P-channel MOSFET    | —          | SOT-23  | AO3401A, AO3407A, DMG2305UX         |  1  |
| U1  | Optocoupler         | —          | DIP-4   | PC817C                              |  1  |
| R1  | Resistor — Rtop     | 470 kΩ, 1% | 0805    | Any 1% metal-film 0805              |  1  |
| R2  | Resistor — Rbottom  | 330 kΩ, 1% | 0805    | Any 1% metal-film 0805              |  1  |
| R3  | Resistor — Gate p/u | 100 kΩ     | 0805    | Any standard 0805                   |  1  |
| R4  | Resistor — Rled     | 2.2 kΩ     | 0805    | Any standard 0805                   |  1  |
| C1  | Capacitor — bypass  | 100 nF, X7R | 0805  | Murata GRM21BR71E104KA01L, Samsung CL21B104KBCNNNC | 1 (opt) |
| C2  | Capacitor — bulk    | 10 µF, X5R  | 0805  | Murata GRM21BC81A106KE18L, Samsung CL21A106KAYNNNE | 1 (opt) |

**Total unique parts: 6 (+ 2 optional decoupling capacitors)**

> C1 and C2 are strongly recommended when the ESP32 is powered from a battery
> circuit rather than USB. See `design-notes.md` — ADC Noise and Supply
> Decoupling section for details and placement guidance.

---

## Notes

### Q1 — P-channel MOSFET

- **AO3401A** (preferred): Vgs(th) = −0.5 V to −1.5 V, Rds(on) = 60 mΩ typical. SOT-23. Widely available on Mouser, LCSC, Digikey.
- **AO3407A**: Higher Id rating (−4.2 A vs −4 A), same SOT-23 footprint. Drop-in replacement.
- **DMG2305UX**: DFN2×2 footprint (2 mm × 2 mm). Use only if board space is extremely limited; requires a different PCB footprint.

### U1 — Optocoupler

- **PC817C** (DIP-4, through-hole): baseline part. Electrically correct, but adds ~5 mm board height.
- SMD alternatives with the **same pinout** (SO-4 footprint, drop-in connections):

  | Part       | Package | Notes                   |
  | ---------- | ------- | ----------------------- |
  | TLP291     | SO-4    | Recommended SMD option  |
  | EL817SMD   | SO-4    | Common, widely stocked  |
  | LTV-817SMD | SO-4    | Same footprint as above |

### R1, R2 — Voltage divider

- **1% tolerance required** for accurate battery readings across the full temperature range.
- 0.1% available if tighter accuracy is needed.
- Do not substitute values without recalculating V_ADC — see `design-notes.md`.

### R3 — Gate pull-up

- 5% tolerance is sufficient.
- Value may be increased to 470 kΩ to reduce operating current; see `design-notes.md` for tradeoffs.

### R4 — LED series resistor

- 5% tolerance is sufficient.
- Value range 1 kΩ – 4.7 kΩ acceptable; see `design-notes.md` for margin analysis.
