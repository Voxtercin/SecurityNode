# Power Architecture — SecurityNode

> **Design-stage / not-yet-fabricated.** Power topology below is verified against the
> schematic netlist (`assets/SecurityNode.pdf`) and Altium BOM (`assets/SecurityNode.BomDoc`).

## Power Tree

```
USB 5V Input — VBUS (J2)
    |
    v
F1 -- PTC resettable fuse (Littelfuse 1206L075, 750mA hold / 1.5A trip)
    |     <-- whole-board USB over-current protection lives HERE
    v
D1 (SMF5.0CA TVS) -- input transient clamp   +   C1 22uF -- bulk hold-up
    |
    v
FB1 -- Ferrite Bead (60Ω@100MHz) -- EMI filtering
    |
    v
5V RAIL  (filtered + protected; ungated)
    |
    +---> U1 (AP2112K-3.3) LDO  ----> 3.3V RAIL
    |                                   |
    |                                   +---> ESP32-C3-WROOM-02-N4 (U3)
    |                                   +---> SN74LVC2G66DCTR (U5)
    |                                   +---> logic pull-ups
    |
    +---> U4 (TPS2553DBVR) CAM power GATE ----> CAM_5V  (gated node)
    |        EN <- IO7/CAM_EN (via R9 100R)        |
    |        ILIM set by R13 21k5                   +---> ESP32-CAM 5V (J3 P$9 / J7 pin6)
    |        FAULT -> IO3/CAM_FAULT                 |
    |        default OFF (R11 100k pulldown)        (CAM also runs its own CAM_3V3,
    |                                                the CAM module's internal regulator)
    +---> LS1 KPEG242-5V buzzer (low-side switched by Q1)
    |
    +---> J5 PIR VCC (PIR is powered from 5V only)

USB data lines (D+/D-) are separately ESD-protected by U2 (USBLC6-2P6)
on the J2 → R3/R4 22R → IO18/IO19 native-USB path (not in the power rail above).
```

## Key Components

### Whole-Board Over-Current — F1 (Littelfuse 1206L075 PTC fuse)

- Resettable PTC fuse, **first element after VBUS** (1206 package)
- 750 mA hold / 1.5 A trip; protects the USB host and the whole board from over-current
- This — not U4 — is the board's over-current limiter on the USB input

### Input Protection — D1 (SMF5.0CA)

- Bidirectional TVS diode clamping VBUS transients (designator is **D1**, not D2; D2 is the door-line Schottky)
- 5V working voltage, clamps USB plug-in spikes / ESD on the 5V input
- Paired with C1 22µF bulk capacitor for input hold-up and decoupling

### EMI Filter — FB1 (TAI-TECH HCB1608KF-600T20)

- 60Ω @ 100MHz, 2A ferrite bead in 0603 package
- Sits between the input clamp and the 5V rail; isolates board switching/digital noise from the USB host

### ESD Protection — U2 (USBLC6-2P6)

- SOT-666-6 ESD diode array
- Protects USB D+ and D- data lines (IO19/IO18 native USB) from static discharge
- On the USB **data** path only (J2 → R3/R4 22R → U2 → IO18/IO19); not in the power rail

### ESP32-CAM Power Gate — U4 (TPS2553DBVR)

- **Dedicated power gate for the ESP32-CAM branch only** — IN = 5V rail, OUT = CAM_5V (J3 P$9 / J7 pin6)
- Enabled by the C3 via **IO7/CAM_EN** (through R9 100R); held **OFF by default** by R11 100k pulldown during C3 boot
- Adjustable current limit set by **R13 (21k5)**; on-resistance ~85mΩ; soft-start; reverse blocking
- FAULT output fed back to the C3 on **IO3/CAM_FAULT** (R10 10k pull-up)
- Does **not** feed the 5V rail, the LDO, or the buzzer, and does **not** protect the USB host (F1 does)

### 3.3V LDO — U1 (AP2112K-3.3TRG1)

- Designator is **U1** (not U2 — U2 is the USBLC6 ESD array)
- Input: up to 6V (here ~5V directly from the ungated 5V rail)
- Output: fixed 3.3V ± 2%
- Maximum output current: 600mA
- Low dropout, low quiescent current — suitable for battery-friendly designs
- Thermal shutdown and current limit protection built-in
- Feeds the **3.3V rail, which powers only U3 (the C3), U5, and logic pull-ups**

## Decoupling Strategy

The schematic uses a conventional bulk + local-decoupling scheme; exact per-designator
capacitor assignments are **to be confirmed against the Altium source** (the BOM is a value
catalog without a designator column, so individual C# → value/MPN is inferred from net context).
Qualitatively:

- **C1 = 22µF** bulk hold-up on the **VBUS input** (this is the input bulk cap — *not* C3; C3 is a
  100nF decoupler on the USB data domain).
- **470nF** decoupling appears on the analog/sensor and rail nodes (e.g. C2; C10 is the door-line
  debounce cap, not a power decoupler).
- The **3.3V rail** carries bulk decoupling (µF-class) plus local 100nF caps at each IC power pin
  (U3, U5).
- Each IC supply pin (U1 in/out, U4 in/out, U3, U5) has a local 100nF (and where appropriate a
  µF-class) decoupling capacitor near its power pin.

> The previously published per-designator value table (C3=22µF, C4=470nF, C5–C16 = fixed map)
> did not match the schematic and has been removed; treat the above as the design intent until the
> Altium designator map is confirmed.

## Power Budget (Estimated — proposed/design-stage figures)

> Magnitudes below are design-stage estimates, not measured. Note the **rail each load sits on**:
> the CAM is fed from the **gated CAM_5V** node (via U4), the buzzer and PIR from the **ungated 5V
> rail**, and only the C3 / U5 / logic from the **3.3V rail**.

| Load | Rail | Current (typ) | Current (max) |
|------|------|---------------|---------------|
| ESP32-C3-WROOM-02-N4 (U3) | 3.3V | 80mA | 300mA (Wi-Fi TX peak) |
| ESP32-CAM | CAM_5V (gated by U4) + own CAM_3V3 | 180mA | ~500mA (Wi-Fi + camera peak) |
| Buzzer (LS1) | 5V | 30mA | 100mA |
| Alarm LED (D5) | 3.3V (via IO6 → R15 330R) | 5mA | 20mA |
| PIR sensor (J5) | 5V | 1mA | 10mA |
| Door contact (J4) | 3.3V logic | <1mA | <1mA |
| SN74LVC2G66 (U5) | 3.3V | 1uA | 10uA |
| Other logic / pull-ups | 3.3V | 5mA | 15mA |
| **Total Estimated** | — | **~300mA** | **~950mA** |

> Notes:
> - **Whole-board USB over-current is bounded by F1** (PTC, 750 mA hold / 1.5 A trip), not by U4.
> - **U4's adjustable current limit (R13 = 21k5) bounds only the CAM_5V branch** — it must be set
>   above the CAM's ~500 mA peak yet within what the upstream 5V rail / USB host can deliver.
> - The CAM is on the **gated 5V** node (CAM_5V), not on the 3.3V rail — earlier "200 mA @ 3.3V"
>   figures for the CAM were incorrect.
> - If total board load approaches USB-host capability (typically 500 mA for standard USB 2.0), a
>   higher-current source or external 5V supply may be required.

## Design Notes

- The ferrite bead (FB1) + input bulk capacitor (C1) form a filter that reduces conducted EMI from the switching/digital loads back to the USB host.
- The AP2112K (U1, 600mA) only has to supply the **C3, U5, and logic pull-ups** on the 3.3V rail — the ESP32-CAM does **not** draw from this rail, so the LDO is comfortably sized.
- The ESP32-CAM receives **CAM_5V** (gated by U4, J3 P$9) for its internal regulator, and supplies its **own** CAM_3V3 (the CAM module's internal regulator output, J3 P$8) — the board's AP2112K does **not** feed the CAM's 3.3V.
- The CAM branch is **off by default** at power-up (R11 100k holds U4 disabled) and is enabled by the C3 asserting IO7/CAM_EN, giving the controller explicit power control and over-current sensing (FAULT → IO3) of the camera.
- Reverse-voltage protection: the TPS2553 (U4) provides reverse blocking on the gated CAM branch. A 40V/1A Schottky (PSBD1DF40V1H) is present on the board's diode complement; the exact designator on the 5V path (vs. the door clamp D2) is **to be confirmed against the Altium source** — only D2 = Schottky and D5 = LED are certain from the current data.
