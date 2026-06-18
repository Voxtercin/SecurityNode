# Sensor Interfaces — SecurityNode

## Supported Sensors

The SecurityNode board provides dedicated connectors for two primary sensor types:

1. **Door Contact Switch** — Magnetic or mechanical reed switch
2. **PIR Motion Sensor** — Passive infrared motion detector

> Note: J6 and J7 are **not** general-purpose sensor-expansion headers. J6 is the 2-pin ESP32-CAM GPIO0 boot/download jumper and J7 is the 6-pin ESP32-CAM programming header (see `esp32cam_interface.md`). The board exposes no spare digital-sensor expansion connectors.

## Door Contact (J4)

### Electrical Interface

| Parameter | Specification |
|-----------|--------------|
| Connector | 2-pin header (J4) |
| Switch type | Normally-closed (NC) or normally-open (NO) reed/magnetic contact |
| Logic level | 3.3V (ESP32-C3 GPIO4) |
| Input conditioning | On-board RC debounce network (around R7 10kΩ / R8 1kΩ / C10 470nF) plus a Schottky clamp (D2, PSBD1DF40V1H) on the door line — hardware debounce is present on the PCB, not firmware-only. Exact pull-up-vs-divider polarity is inferred from the net layout and to be confirmed against the Altium source. |
| Trigger condition | Contact open (HIGH → LOW transition, or LOW → HIGH depending on wiring) |

### Typical Wiring

```
ESP32-C3 GPIO4 ──[10kΩ pull-up]── 3.3V
                 |
                 +── J4 Pin 1 (Switch terminal A)
                 |
J4 Pin 2 (Switch terminal B) ── GND
```

When the door is **closed**, the switch is closed and GPIO4 reads LOW (pulled to GND).
When the door is **open**, the switch opens and GPIO4 reads HIGH (pulled up to 3.3V).

> Note: The active logic can be inverted in firmware if a normally-open switch is used instead.

### Firmware Handling (proposed)

> Design-stage guidance; no firmware exists yet.

- Configure GPIO4 as a digital input (internal pull-up may be redundant given the on-board RC network — confirm against the schematic before relying on it).
- Use **interrupt-on-change** or **polling** to detect state transitions.
- On-board hardware debounce (C10 RC network) already filters most contact bounce; an optional software debounce (e.g., 50ms timer) can add margin.

## PIR Motion Sensor (J5)

### Electrical Interface

| Parameter | Specification |
|-----------|--------------|
| Connector | 3-pin header (J5): VCC, OUT, GND |
| Supply voltage | **5V only** — J5 pin 1 (VCC) is hard-wired to the 5V rail. The board cannot supply 3.3V to this connector. |
| Output type | Digital TTL/CMOS logic (5V-referenced at the module) |
| On-board conditioning | PIR output reaches GPIO5 through a clamp/divider network (R16/R18 100kΩ + BAT54S dual Schottky D4) — the GPIO does **not** see a raw clean 3.3V logic level; the network clamps/limits the 5V-referenced output before the C3 pin. |
| Trigger condition | Motion detected → output HIGH (or LOW, depending on module) |

### Typical Wiring

```
J5 Pin 1 (VCC) ── 5V rail (hard-wired; not selectable)
J5 Pin 2 (OUT) ── [on-board clamp/divider: R16/R18 100k + BAT54S D4] ── ESP32-C3 GPIO5
J5 Pin 3 (GND) ── GND
```

> The on-board conditioning network (R16/R18 100kΩ + BAT54S dual Schottky D4) clamps and limits the 5V-referenced PIR output so it is safe to present to the 3.3V GPIO5 input. The exact pull-up-vs-divider topology and active polarity are inferred from the schematic net layout and should be confirmed against the Altium source before finalizing firmware ISR logic.

Most common HC-SR501 or similar PIR modules:
- Output goes **HIGH** for a configurable duration (typically 2–200 seconds) when motion is detected.
- Output is **LOW** when no motion is present.
- Some modules have a retriggerable mode (continuous HIGH while motion persists) and a single-shot mode.

### Firmware Handling (proposed)

> Design-stage guidance; no firmware exists yet.

- Configure GPIO5 as a digital input. The on-board R16/R18 + BAT54S D4 network conditions the line; select internal pull/no-pull to match the network's resting state once the conditioning polarity is confirmed against the schematic.
- Use **interrupt-on-rising-edge** to wake the system or trigger event processing.
- Respect the PIR module's minimum trigger interval to avoid false re-triggers.

## Sensor Event Logic

> **Proposed / design-stage:** the event-priority table and alarm sequence below describe planned firmware behavior. No firmware exists yet, so treat this as the intended design rather than as-built logic.

The firmware evaluates sensor inputs with the following priority:

| Priority | Event | Action |
|----------|-------|--------|
| 1 | Door open detected | Trigger alarm sequence |
| 2 | PIR motion detected | Trigger alarm sequence |
| 3 | Both simultaneously | Trigger alarm sequence (same path) |

The alarm sequence is:

1. Log timestamp and sensor type.
2. Request image capture from ESP32-CAM.
3. Wait for vision analysis result (timeout: e.g., 5 seconds).
4. If result = **suspicious / covered face** → activate buzzer + LED.
5. If result = **visible face** or **timeout** → log event, return to idle.

## Input Protection

- All sensor connectors are routed to ESP32-C3 GPIOs which tolerate 0–3.3V.
- No high-voltage or analog interfaces are present on the sensor headers.
- On-board clamping/ESD is **partly implemented** on both sensor lines:
  - **Door (J4):** Schottky clamp **D2** (PSBD1DF40V1H, 40V/1A) plus an RC conditioning/debounce network.
  - **PIR (J5):** TVS clamp (SMF5.0CA, the diode in the PIR-line position — designator to be confirmed against the Altium source) plus the R16/R18 100kΩ + BAT54S D4 clamp/divider.
- For sensors mounted remotely on long cables, additional external protection may still be worth considering for production, but the board is **not** unprotected at the connectors.

## J6 / J7 — Not Sensor Connectors

J6 and J7 are **not** auxiliary sensor-expansion headers and do **not** break out spare GPIOs. GPIO11–GPIO17 are not available on the ESP32-C3-WROOM-02 module (GPIO11 and adjacent pins are used internally for flash), so no such expansion exists. For completeness:

| Header | Role |
|--------|------|
| **J6** | 2-pin ESP32-CAM GPIO0 boot/download jumper |
| **J7** | 6-pin ESP32-CAM programming header (GND, CAM_TXD_RAW, CAM_RXD_RAW, B_GPIO0_CAM, CAM_3V3, CAM_5V) |

Both relate to programming the ESP32-CAM out-of-band with an external USB-UART adapter; see `esp32cam_interface.md`.

## Design Notes

- J5 supplies the PIR with **5V** (hard-wired), which suits common 5V PIR modules (e.g. HC-SR501). There is no 3.3V option on this connector, so use a PIR module that runs on 5V.
- The PIR output does not reach GPIO5 directly: the on-board R16/R18 100kΩ + BAT54S D4 network clamps/limits the 5V-referenced output before the 3.3V GPIO. No external level-shifter is required for this board.
- Door contact switches are passive and polarity-agnostic; however, consistent wiring (common to GND, signal to GPIO) simplifies firmware logic.
- Series-R / small-C EMI conditioning is already present on the door line (RC network around D2) and a clamp/divider on the PIR line; for unusually long remote cable runs, additional external filtering can still be considered.
