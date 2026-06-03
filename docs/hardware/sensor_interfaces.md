# Sensor Interfaces — SecurityNode

## Supported Sensors

The SecurityNode board provides dedicated connectors for two primary sensor types:

1. **Door Contact Switch** — Magnetic or mechanical reed switch
2. **PIR Motion Sensor** — Passive infrared motion detector

Additional auxiliary connectors (J6, J7) allow expansion to other digital sensors.

## Door Contact (J4)

### Electrical Interface

| Parameter | Specification |
|-----------|--------------|
| Connector | 2-pin header (J4) |
| Switch type | Normally-closed (NC) or normally-open (NO) reed/magnetic contact |
| Logic level | 3.3V (ESP32-C3 GPIO) |
| Input conditioning | Internal/external pull-up to 3.3V |
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

### Firmware Handling

- Enable internal pull-up on GPIO4 (or rely on external pull-up).
- Configure GPIO4 as digital input.
- Use **interrupt-on-change** or **polling** to detect state transitions.
- Implement **software debounce** (e.g., 50ms timer) to avoid false triggers from contact bounce.

## PIR Motion Sensor (J5)

### Electrical Interface

| Parameter | Specification |
|-----------|--------------|
| Connector | 3-pin header (J5): VCC, OUT, GND |
| Supply voltage | 3.3V or 5V (depending on PIR module; verify module datasheet) |
| Output type | Digital TTL/CMOS logic |
| Logic level | 3.3V (verify PIR module compatibility) |
| Trigger condition | Motion detected → output HIGH (or LOW, depending on module) |

### Typical Wiring

```
J5 Pin 1 (VCC) ── 3.3V or 5V rail
J5 Pin 2 (OUT) ── ESP32-C3 GPIO5
J5 Pin 3 (GND) ── GND
```

Most common HC-SR501 or similar PIR modules:
- Output goes **HIGH** for a configurable duration (typically 2–200 seconds) when motion is detected.
- Output is **LOW** when no motion is present.
- Some modules have a retriggerable mode (continuous HIGH while motion persists) and a single-shot mode.

### Firmware Handling

- Configure GPIO5 as digital input with internal pull-down (if module output is open-drain) or no pull (if module drives active HIGH/LOW).
- Use **interrupt-on-rising-edge** to wake the system or trigger event processing.
- Respect the PIR module's minimum trigger interval to avoid false re-triggers.

## Sensor Event Logic

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
- ESD protection is recommended at the connector if sensors are mounted remotely with long cables (not implemented on current PCB; consider external protection for production).

## Auxiliary Connectors (J6 / J7)

These headers provide additional GPIOs, power, and ground for future sensors or modules:

| Pin | J6 Signal | J7 Signal | Notes |
|-----|-----------|-----------|-------|
| 1 | GND | GND | Common ground |
| 2 | 3.3V | 3.3V | Logic rail |
| 3 | 5V | 5V | Unregulated rail |
| 4 | GPIO11 | GPIO12 | Digital I/O |
| 5 | — | — | Reserved |

## Design Notes

- Verify the PIR module voltage requirement before connecting. Some modules require 5V and may not reliably trigger at 3.3V.
- If using 5V PIR modules, ensure the output signal is 3.3V-compatible or use a level-shifter/voltage divider.
- Door contact switches are passive and polarity-agnostic; however, consistent wiring (common to GND, signal to GPIO) simplifies firmware logic.
- Consider adding series resistors (e.g., 100Ω) and small capacitors (e.g., 100nF) at the connector for EMI suppression on long sensor cables.
