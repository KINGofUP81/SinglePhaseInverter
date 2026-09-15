# Single-Phase H-Bridge Inverter

A **380 V DC → single-phase AC** full-bridge inverter power board designed in Altium Designer. It combines a 650 V MOSFET H-bridge, bootstrap gate drive, an LC output filter and a complete sensing suite (DC bus voltage and current, AC output voltage and current, heatsink temperature). Everything is broken out to header connectors for an external STM32 controller.

> Designed at **IIT Ropar** by **Afraaz Khan, Zeeshan and Kratika**, under the guidance of **Prof. Saifullah Payami**.

---

## Highlights

| | |
|---|---|
| ⚡ **DC bus** | 380 V input, 2 × 470 µF / 450 V bulk capacitors with bleeder resistors, film decoupling and snubbers |
| 🔀 **Power stage** | Full H-bridge of 4 × **STW69N65M5** 650 V MDmesh™ M5 MOSFETs (TO-247) on heatsinks |
| 🎛️ **Gate drive** | 2 × **IR2110** high/low-side drivers with MUR160G bootstrap diodes |
| 🌊 **Output filter** | 2.2 mH inductor + 3 × 2.2 µF film capacitors for a clean sinusoidal output |
| 📏 **Sensing** | DC current (ACS758), AC current (ZMCT103C CT), DC bus and AC output voltage dividers, LM35 temperature |
| 🛡️ **Protection** | MOV, TVS, Zener gate clamps, Schottky ADC clamps, shutdown/fault line |
| 🧱 **PCB** | 4-layer stack-up with separate power and signal grounds |

## System Overview

```
                  ┌───────────────────── H-bridge ─────────────────────┐
 +380 V DC ──┬──► │  Q1 ─┐                                   ┌─ Q2      │
             │    │      ├── NODE_A ──► L1 2.2 mH ──► C 3×2.2µF ──┼──► AC_LINE
  C5/C6 470µF│    │  Q3 ─┘                                   └─ Q4      │──► AC_NEUTRAL
             │    └──────▲────────────────────────────────────▲────────┘
            PGND     IR2110 (A)                           IR2110 (B)
                          ▲                                    ▲
                    HIN_A / LIN_A                        HIN_B / LIN_B
                          └──────────── STM32 controller ──────┘
                                           ▲
       VBUS_FB · CURRENT_FB · VOUT_FB · COUT_FB · TEMP_FB · SD_FAULT
```

An external STM32 generates sinusoidal PWM (e.g. SPWM) for the two legs. The IR2110s level-shift that PWM to drive the four MOSFETs, the LC filter recovers the fundamental, and the feedback signals close the voltage and current loops.

## Bill of Materials (key parts)

| Block | Part | Qty | Role |
|---|---|---|---|
| Power switches | STMicro **STW69N65M5** | 4 | 650 V N-channel MOSFETs, H-bridge |
| Gate drivers | Infineon **IR2110** | 2 | High- and low-side driver per leg |
| Bootstrap | MUR160G + 1 µF | 2 | Floating high-side supply |
| Gate network | 22 Ω, 1N4148W, BZT52C18, 1 kΩ | 4 each | Turn-on/off shaping, 18 V clamp, pull-down |
| DC link | ELG477M450AT6AA 470 µF / 450 V | 2 | Bulk bus capacitance |
| DC link | ECW-FE2J105QD 1 µF / 630 V film, ECW-FD2J333JB 33 nF film | 1 + 3 | High-frequency decoupling |
| Output filter | RLB0914-222KL 2.2 mH | 1 | Filter inductor |
| Output filter | ECW-FE2W225KA 2.2 µF film | 3 | Filter capacitors |
| DC current | Allegro **ACS758LCB-050B** | 1 | ±50 A isolated Hall-effect sensor |
| AC current | **ZMCT103C** + 49.9 Ω burden | 1 | Output current transformer |
| Voltage sense | PTF65 1 MΩ precision resistors | 7 | High-voltage dividers |
| Temperature | **LM35DZ** | 1 | Heatsink / board temperature |
| Regulator | LM7805 | 1 | 12 V → 5 V for sensors |
| Protection | B72220S0421K101 MOV, SMAJ15A TVS, BAT54C | 1 each | Surge, rail and ADC-input clamping |
| Noise | BLM21PG221SN1D ferrite bead | 1 | Couples power ground to signal ground |
| Connectors | KF301-2P screw terminals, Harwin M20 headers | 3 + 3 | Power I/O and controller interface |
| Thermal | Wakefield 694-50 heatsinks | 4 | MOSFET cooling |

The full BOM is in [`Singlephaseinverter/SinglephaseinverterBOM.xlsx`](Singlephaseinverter/SinglephaseinverterBOM.xlsx).

## Design Details

<details>
<summary><b>H-bridge & gate drive</b></summary>

Each leg (NODE_A, NODE_B) uses a high-side and a low-side STW69N65M5. With 650 V ratings they give comfortable margin on a 380 V bus, and their low R<sub>DS(on)</sub> keeps conduction loss down.

Each MOSFET gate has:
- a **22 Ω** series resistor to control dv/dt and damp ringing,
- a **1N4148W** diode across it for fast turn-off,
- an **18 V Zener** (BZT52C18) to clamp gate-source spikes,
- a **1 kΩ** gate-source pull-down that holds the device off while the driver is unpowered.

The IR2110s run from a 12 V rail. Their high-side outputs are powered by MUR160G ultrafast bootstrap diodes and local 1 µF capacitors, recharged whenever the low-side switch conducts. The shutdown inputs are tied to a filtered `SD_FAULT` line so the controller can disable all four gates at once.
</details>

<details>
<summary><b>DC link</b></summary>

Two 470 µF / 450 V electrolytics provide bulk energy storage, with a string of 1 MΩ resistors to bleed the bus down safely after power-off. A 1 µF / 630 V film capacitor and three 33 nF film capacitors sit close to the bridge to supply the high-frequency switching current that electrolytics can't.
</details>

<details>
<summary><b>Output LC filter</b></summary>

The switched bridge voltage passes through a 2.2 mH inductor into three parallel 2.2 µF film capacitors (6.6 µF total). This low-pass filter removes the PWM carrier and its harmonics, leaving the fundamental at `AC_LINE` / `AC_NEUTRAL`.
</details>

<details>
<summary><b>Sensing & feedback</b></summary>

| Signal | Sensor | Conditioning |
|---|---|---|
| `VBUS_FB` | 1 MΩ precision divider | Scales the 380 V bus into the ADC range |
| `CURRENT_FB` | ACS758LCB-050B | Isolated Hall-effect DC-link current, ±50 A |
| `VOUT_FB` | 1 MΩ precision divider | AC output voltage, biased with a 10 kΩ network |
| `COUT_FB` | ZMCT103C CT | 49.9 Ω burden, 10 kΩ / 10 kΩ mid-rail bias |
| `TEMP_FB` | LM35DZ | 10 mV/°C analog output |

A BAT54C dual Schottky provides input clamping to protect the MCU's ADC pins, and small filter capacitors on the feedback paths reduce switching noise.
</details>

<details>
<summary><b>Protection</b></summary>

- **MOV** (420 V<sub>RMS</sub>) absorbs surge energy.
- **SMAJ15A TVS** protects the 12 V auxiliary rail.
- **Zener gate clamps** protect each MOSFET gate.
- **Schottky clamps** protect the ADC inputs.
- **`SD_FAULT`** gives the controller a hardware path to shut down all four gates.
</details>

<details>
<summary><b>PCB layout</b></summary>

4-layer FR-4 stack-up:

| Layer | Name | Use |
|---|---|---|
| L1 | `POWER` | High-current bus and bridge routing |
| L2 | `PGND` | Power ground plane |
| L3 | `LOW_POWER` | Gate drive and auxiliary supplies |
| L4 | `SIGNAL/SGND` | Sensing, controller interface, signal ground |

- **Zoning:** the high-voltage section (bus capacitors, bridge, heatsinks, output filter) is kept physically apart from the low-voltage sensing and controller interface.
- **Grounds:** the power and signal grounds are separate, joined through a ferrite bead.
- **Iterations:** the design went through several layout revisions (`PCB1` → `PCB2 (ITERATION)` → `PCBFINAL`), with DRC reports and Gerber/ODB++ outputs generated for each.
</details>

## Controller Interface

| Header | Pins | Signals |
|---|---|---|
| **P1** (4-pin) | 1–4 | `HIN_A`, `LIN_A`, `HIN_B`, `LIN_B` (PWM inputs) |
| **P5** (2-pin) | 1–2 | `CURRENT_FB`, `VBUS_FB` |
| **P6** (6-pin) | 1–6 | `SD_FAULT`, `TEMP_FB`, `SGND`, `COUT_FB`, `VOUT_FB`, `3.3V` |

| Terminal | Connection |
|---|---|
| **P3** | +380 V DC bus input / PGND |
| **P2** | AC output (`AC_LINE` / `AC_NEUTRAL`) |
| **P4** | +12 V auxiliary supply / PGND |

## Repository Contents

```
Singlephaseinverter/
├── Singlephaseinverter.PrjPcb        # Altium project (open this)
├── Sheet2.SchDoc                     # Main schematic
├── PCBFINAL.PcbDoc                   # Final PCB layout
├── PCB1.PcbDoc, PCB2(ITERATION).PcbDoc  # Earlier layout iterations
├── Schlib2.SchLib, PcbLib2.PcbLib    # Project libraries
├── <part folders>/                   # Vendor symbols & footprints (SchLib/PcbLib)
├── *.stp / *.step                    # 3D models
├── SinglephaseinverterBOM.xlsx       # Bill of materials
├── Inverter(FINAL SCHEM).pdf         # Schematic + PCB prints
├── Singlephaseinverter(*).pdf        # Review / submission prints
└── Project Outputs for Singlephaseinverter/
    ├── PCBFINAL.G*                   # Gerbers + drill
    ├── odb/                          # ODB++ fabrication data
    └── Design Rule Check - *.html    # DRC reports
```

📄 **Start here:** [`Inverter(FINAL SCHEM).pdf`](Singlephaseinverter/Inverter(FINAL%20SCHEM).pdf) has the full schematic and PCB layout without needing Altium.

## Getting Started

1. Open `Singlephaseinverter/Singlephaseinverter.PrjPcb` in **Altium Designer**.
2. Review `PCBFINAL.PcbDoc` and re-run DRC.
3. Fabrication files (Gerber, drill, ODB++) are in `Project Outputs for Singlephaseinverter/`.

> ⚠️ **High voltage.** This board runs from a 380 V DC bus and its bulk capacitors store lethal energy. Bring it up with an isolated, current-limited supply, confirm the bus has bled down before touching it, and always use proper isolation and PPE.

## Roadmap

- [ ] Assemble and bring up the power stage
- [ ] Closed-loop SPWM control on STM32 (voltage and current loops)
- [ ] Dead-time and switching-loss optimisation
- [ ] THD measurement of the filtered output
- [ ] Thermal characterisation under load
