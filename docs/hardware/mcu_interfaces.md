# MCU Interfaces — SecurityNode

## Main Controller: ESP32-C3-WROOM-02-N4 (U3)

The ESP32-C3-WROOM-02-N4 is the central security controller. It manages sensor inputs, alarm outputs, and communicates with the ESP32-CAM vision module.

## Pinout Mapping

> Note: Exact GPIO assignments should be verified against the Altium schematic. The table below reflects the typical connections visible in the PCB render and BOM.

| ESP32-C3 Pin | Function | Connected To | Description |
|--------------|----------|--------------|-------------|
| GPIO0 | BOOT | SW1 / Header J1 | Boot mode strap / user button |
| GPIO1 | TXD0 | U5 (switch) → USB | UART TX for programming / debug |
| GPIO2 | RXD0 | U5 (switch) → USB | UART RX for programming / debug |
| GPIO3 | EN | SW2 / Header J1 | Chip enable / reset button |
| GPIO4 | SENSOR_IN_1 | J4 (Door Contact) | Digital input: door open/close |
| GPIO5 | SENSOR_IN_2 | J5 (PIR Motion) | Digital input: motion detected |
| GPIO6 | ALARM_LED | LED (D4 via R16) | Drives alarm status LED |
| GPIO7 | BUZZER_CTRL | Q1 Gate (via R12, R14) | N-MOSFET gate for buzzer |
| GPIO8 | CAM_TX | J3 (ESP32-CAM RX) | UART TX to ESP32-CAM |
| GPIO9 | CAM_RX | J3 (ESP32-CAM TX) | UART RX from ESP32-CAM |
| GPIO10 | CAM_BOOT | J3 / Header | ESP32-CAM boot mode control |
| GPIO11 | AUX_1 | J6 / J7 | Auxiliary digital I/O |
| GPIO12 | AUX_2 | J6 / J7 | Auxiliary digital I/O |
| GPIO13–GPIO21 | — | — | Available for future expansion |
| 3.3V | Power | AP2112K output | Logic rail supply |
| GND | Ground | Global ground | Common return |

## Programming Interface

The ESP32-C3 is programmed via UART through the Mini USB connector (J2). The SN74LVC2G66DCTR (U5) analog switch multiplexes the UART lines:

- **Normal Operation**: USB UART ↔ ESP32-C3 (programming, debug output, logs).
- **CAM Debug Mode**: USB UART ↔ ESP32-CAM (direct programming of the vision module without removing it from J3).

### Entering Bootloader Mode

1. Hold **SW1 (BOOT)** pressed.
2. Press and release **SW2 (EN)**.
3. Release **SW1**.
4. ESP32-C3 enters UART download mode, ready for `esptool.py` or Arduino IDE upload.

## Sensor Input Conditioning

Door contact and PIR sensors are connected to GPIO4 and GPIO5 through connector headers J4 and J5. The inputs are typically:

- **Pull-up enabled** on ESP32-C3 GPIOs (internal or external 10kΩ–100kΩ).
- **Active-low logic**: Sensor closure pulls GPIO to GND; open circuit = HIGH.
- **Optional debounce**: Can be implemented in firmware (software timer) or with a small RC filter on the PCB.

## Alarm Output Control

### Buzzer (LS1) — Q1 (DN2302S) N-MOSFET Driver

| Parameter | Value |
|-----------|-------|
| MOSFET | DN2302S (20V, 3.2A, SOT-23) |
| Gate drive | ESP32-C3 GPIO7 via series resistor R12 |
| Gate pulldown | R14 (to ensure OFF state at boot) |
| Source | GND |
| Drain | Buzzer negative terminal |
| Buzzer positive | 5V rail (via TPS2553) |

When GPIO7 is HIGH, the MOSFET turns on, sinking current through the buzzer. The buzzer is a **KPEG242-5V** electromagnetic type, rated for 5V operation.

### Alarm LED (D4)

- Package: 0805 SMD LED (RED, 2V forward voltage)
- Current-limiting resistor: R16 (330Ω typical)
- Driven directly by ESP32-C3 GPIO6 (3.3V logic)
- LED current: ~(3.3V – 2.0V) / 330Ω ≈ 4mA

## UART Protocol to ESP32-CAM

The ESP32-C3 communicates with the ESP32-CAM via a dedicated UART port (GPIO8/9) separate from the programming UART. This avoids contention during CAM debug sessions.

| Parameter | Value |
|-----------|-------|
| Baud Rate | 115200 (default) |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

Command structure and response codes are documented in `docs/firmware/cam_protocol.md`.

## Expansion Headers

### J1 — Main Debug / Breakout

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | GND | Ground |
| 2 | EN | Chip enable (active HIGH) |
| 3 | BOOT | Boot mode strap (LOW = download) |
| 4 | RX | UART RX (from USB switch) |
| 5 | TX | UART TX (to USB switch) |
| 6 | GND | Ground |

### J6 / J7 — Auxiliary I/O

These headers bring out additional ESP32-C3 GPIOs and power rails for:

- External sensor expansion
- Relay modules
- Status LEDs
- Debug probes

## Design Notes

- All GPIOs operate at 3.3V logic levels. Do not connect 5V logic directly to ESP32-C3 pins.
- The ESP32-C3 has internal pull-up/pull-down resistors that can be enabled in firmware, reducing the need for external resistors on digital inputs.
- The alarm LED and buzzer are low-current loads and can be driven directly by ESP32-C3 GPIOs (via MOSFET for the buzzer).
- Ensure the analog switch (U5) control signal is correctly managed in firmware or hardware to avoid UART bus contention.
