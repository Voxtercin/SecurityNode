# MCU Interfaces — SecurityNode

## Main Controller: ESP32-C3-WROOM-02-N4 (U3)

The ESP32-C3-WROOM-02-N4 is the central security controller. It manages sensor inputs, alarm outputs, and communicates with the ESP32-CAM vision module.

> **Design stage:** this is a not-yet-fabricated board. The hardware pin map and topology below are verified against the schematic netlist; any firmware/protocol behavior referenced is **proposed** design, not as-built.

## Pinout Mapping

> Note: This table is the authoritative ESP32-C3 (U3) pin map, taken from the schematic netlist (`assets/SecurityNode.pdf`) and the Altium BOM. It supersedes earlier speculative tables. Note the UART1 direction: **IO0 = RX, IO1 = TX**.

| ESP32-C3 Pin | Function | Connected To | Description |
|--------------|----------|--------------|-------------|
| EN (dedicated pin 2) | ESP_EN | SW2 + R2 (and Header J1) | Chip enable / reset — dedicated EN pin, not a GPIO |
| IO0 | UART1_RX | ← R20 (1k) ← U5 ← J3 (CAM TXD) | CAM-link UART RX (from ESP32-CAM) |
| IO1 | UART1_TX | → R21 (1k) → U5 → J3 (CAM RXD) | CAM-link UART TX (to ESP32-CAM) |
| IO2 | STRAP_GPIO2 | — | Boot strap pin |
| IO3 | CAM_FAULT | U4 (TPS2553) FAULT, R10 10k pull-up | Input: CAM power-gate fault flag (active-low) |
| IO4 | DOOR_GPIO4 | J4 (Door Contact) | Digital input: door open/close |
| IO5 | PIR_GPIO5 | J5 (PIR Motion, PIR powered from 5V) | Digital input: motion detected |
| IO6 | LED_ALERT | → R15 (330Ω) → D5 (LED) | Drives alarm status LED |
| IO7 | CAM_EN | → R9 (100Ω) → U4 (TPS2553) EN | Output: enables the ESP32-CAM power gate |
| IO8 | STRAP_GPIO8 | — | Boot strap pin |
| IO9 | BOOT_GPIO9 | SW1 + R1 (and Header J1) | BOOT button / boot mode strap |
| IO10 | BUZZER | → R12 (100Ω) → Q1 gate (DN2302S) → LS1 | N-MOSFET gate drive for buzzer |
| IO18 | USB_D_N | → R3 (22Ω) → U2 (USBLC6) → J2 | Native USB D− (USB-Serial/JTAG) |
| IO19 | USB_D_P | → R4 (22Ω) → U2 (USBLC6) → J2 | Native USB D+ (USB-Serial/JTAG) |
| GPIO20 (pin 11) | UART0_RX | Header J1 (pin 3), direct | Debug/console UART RX — not through U5 |
| GPIO21 (pin 12) | UART0_TX | Header J1 (pin 2), direct | Debug/console UART TX — not through U5 |
| 3.3V | Power | U1 (AP2112K-3.3) output | Logic rail supply (C3 + U5 only) |
| GND | Ground | Global ground | Common return |

## Programming Interface

The ESP32-C3 is programmed over its **native USB** (the built-in USB-Serial/JTAG peripheral) through the Mini USB connector (J2). There is **no USB-UART bridge chip** on the board (no CP210x / no FTDI): J2 connects through U2 (USBLC6 ESD protection) and R3/R4 (22Ω series) to IO18 (D−) / IO19 (D+). The PC enumerates the C3's built-in USB CDC device directly.

- **Native USB (primary)**: J2 → U2 → IO18/IO19. Used for flashing, debug output, and logs.
- **UART0 debug/flash header (J1, alternate)**: GPIO20 (RX) / GPIO21 (TX) are wired **direct to J1**, not through U5. This header, together with the BOOT/EN buttons, is an alternate path for `esptool.py` flashing or serial console.

> The SN74LVC2G66DCTR (U5) does **not** participate in programming. It only isolates the C3↔CAM UART1 link (see "UART Protocol to ESP32-CAM" below). The ESP32-CAM is programmed out-of-band via J7 with an external USB-UART adapter — no on-board USB path reaches the CAM.

### Entering Bootloader Mode

1. Hold **SW1 (BOOT, IO9)** pressed.
2. Press and release **SW2 (EN, dedicated EN pin)**.
3. Release **SW1**.
4. ESP32-C3 enters download mode, ready for `esptool.py` or Arduino IDE upload over native USB (or via the J1 UART0 header).

## Sensor Input Conditioning

Door contact (IO4) and PIR motion (IO5) sensors are connected through connector headers J4 and J5. The PIR is **powered from the 5V rail** (J5 VCC is hard-wired to 5V; the board cannot supply 3.3V to that connector), and its output reaches IO5 through an on-board conditioning network. The inputs are characterized as follows:

- **Pull-up / bias**: external resistors are present on the conditioning networks (10kΩ–100kΩ class); internal pull-ups can also be enabled in firmware.
- **Active polarity**: to be confirmed against the schematic — the exact pull-up-vs-divider role and active-low/active-high polarity of the door (R7/R8/C10/D2) and PIR (R16/R18/BAT54S D4) networks are inferred from net topology, not annotated. Confirm before writing ISR logic.
- **Debounce**: the door line has real hardware debounce on board (RC filter); additional debounce can be implemented in firmware (software timer).

## Alarm Output Control

### Buzzer (LS1) — Q1 (DN2302S) N-MOSFET Driver

| Parameter | Value |
|-----------|-------|
| MOSFET | DN2302S (20V, 3.2A, SOT-23) |
| Gate drive | ESP32-C3 IO10 (BUZZER) via series resistor R12 (100Ω) |
| Gate pulldown | R14 (10k, to ensure OFF state at boot) |
| Source | GND |
| Drain | Buzzer negative terminal |
| Buzzer positive | 5V rail (ungated — not via U4/TPS2553) |

When IO10 is HIGH, the MOSFET turns on, sinking current through the buzzer. The buzzer is a **KPEG242-5V** electromagnetic type (LS1), rated for 5V operation, and sits on the ungated 5V rail (U4/TPS2553 gates the CAM branch only, not the buzzer).

### Alarm LED (D5)

- Designator: **D5** (XL-2012SURC), red 0805 SMD LED (~2V forward voltage)
- Current-limiting resistor: **R15** (330Ω)
- Driven directly by ESP32-C3 IO6 (LED_ALERT, 3.3V logic)
- LED current: ~(3.3V – 2.0V) / 330Ω ≈ 4mA

## UART Protocol to ESP32-CAM

The ESP32-C3 communicates with the ESP32-CAM via a dedicated UART1 port on **IO0 (RX)** and **IO1 (TX)** (each via a 1kΩ series resistor, R20/R21), separate from the native-USB programming path. This link runs through header J3 and through the U5 isolation switch.

**U5 (SN74LVC2G66DCTR) — automatic CAM-presence-gated UART isolation:** U5 is a 2-channel analog switch wired as a **series isolation/disconnect switch on the C3↔CAM UART1 link only** (CAM_TXD/RXD_RAW ↔ the _ISO nets). Its control pins (1C/2C) are **hard-tied to CAM_3V3** — the ESP32-CAM module's own internal 3.3V regulator output. As a result the UART link **auto-connects only when a powered CAM is seated** in J3 and **auto-disconnects when the CAM is absent or off**. It is **not** a USB-to-UART programming multiplexer, is **not** host- or GPIO-selectable, and does not touch UART0/USB.

| Parameter | Value |
|-----------|-------|
| Baud Rate | 115200 (default) |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

> The CAM UART command structure and response codes are a **proposed (design-stage)** protocol documented in `docs/firmware/cam_protocol.md` — no firmware is implemented yet.

## Headers

### J1 — Alternate UART0 Debug / Flash Header

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | GND | Ground |
| 2 | UART0_TX (GPIO21) | UART0 TX, direct from C3 (not through U5) |
| 3 | UART0_RX (GPIO20) | UART0 RX, direct to C3 (not through U5) |
| 4 | BOOT_GPIO9 | Boot mode strap (SW1 / IO9) |
| 5 | ESP_EN | Chip enable / reset (SW2 / dedicated EN pin) |
| 6 | GND | Ground |

### J6 — ESP32-CAM Boot Jumper

A **2-pin jumper** that asserts the ESP32-CAM's GPIO0 (B_GPIO0_CAM) for download mode. No C3 GPIO drives this strap — it is a manual jumper (with J7 pin 4 and R17 10k). It is **not** an auxiliary GPIO expansion header.

### J7 — ESP32-CAM Programming Header

A **6-pin** out-of-band programming header for the ESP32-CAM, used with an **external USB-UART adapter** (plus the J6 boot jumper). No on-board USB path reaches the CAM through this header.

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | GND | Ground |
| 2 | CAM_TXD_RAW | ESP32-CAM U0TXD |
| 3 | CAM_RXD_RAW | ESP32-CAM U0RXD |
| 4 | B_GPIO0_CAM | ESP32-CAM GPIO0 boot strap |
| 5 | CAM_3V3 | ESP32-CAM 3.3V (module's own regulator output) |
| 6 | CAM_5V | ESP32-CAM gated 5V (U4 output) |

> The ESP32-C3 does **not** break out general-purpose aux GPIOs (e.g. GPIO11–17 are not available on the WROOM-02 module). There are no auxiliary expansion headers exposing spare C3 GPIOs.

## Design Notes

- All GPIOs operate at 3.3V logic levels. Do not connect 5V logic directly to ESP32-C3 pins. (The PIR runs on 5V; its output is conditioned on board before reaching IO5.)
- The ESP32-C3 has internal pull-up/pull-down resistors that can be enabled in firmware, reducing the need for external resistors on digital inputs.
- The alarm LED is driven directly by IO6 (via R15); the buzzer is driven through Q1 (DN2302S) from IO10 (via R12) — both are low-current loads.
- The U5 analog switch needs **no firmware management**: it is presence-gated by CAM_3V3 in hardware, so the C3↔CAM UART1 link connects/disconnects automatically with CAM presence and cannot create host-side bus contention.
- The 3.3V rail (U1 / AP2112K-3.3) supplies the C3 and U5 only; the buzzer and PIR are on the 5V rail; the CAM runs on the U4-gated CAM_5V plus its own CAM_3V3.
