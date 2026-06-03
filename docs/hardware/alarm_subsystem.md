# Alarm Subsystem — SecurityNode

## Overview

The alarm subsystem provides local visual and audible indication when the SecurityNode detects a suspicious event. It consists of:

1. **Alarm LED** (D4) — Red 0805 SMD LED
2. **Buzzer** (LS1) — KPEG242-5V electromagnetic buzzer

Both indicators are controlled by the ESP32-C3 (U3) through dedicated GPIO pins.

## Alarm LED (D4)

### Circuit

```
ESP32-C3 GPIO6 ──[R16: 330Ω]──|>|── GND
                              D4
                              (RED LED)
```

### Parameters

| Parameter | Value |
|-----------|-------|
| LED | XINGLIGHT XL-2012SURC (RED, 0805 package) |
| Forward voltage (Vf) | ~2.0V |
| Forward current (If) | ~4mA (typical) |
| Current-limiting resistor (R16) | 330Ω |
| Current calculation | (3.3V – 2.0V) / 330Ω ≈ 3.9mA |

### Behavior

- GPIO6 **HIGH** → LED ON (alarm active / triggered).
- GPIO6 **LOW** → LED OFF (idle / standby).
- The LED can be blinked in firmware to indicate different states (e.g., slow blink = armed, fast blink = alarm active).

## Buzzer (LS1)

### Circuit

```
5V Rail ──[+] LS1 [-]── Q1 Drain
                         |
                         Q1 Source ── GND
                         |
                     Q1 Gate ──[R12]── ESP32-C3 GPIO7
                                  |
                                  [R14]── GND (pulldown)
```

### Components

| Component | Part Number | Function |
|-----------|-------------|----------|
| Buzzer | KPEG242-5V | Electromagnetic buzzer, 5V rated |
| MOSFET | DN2302S (Q1) | N-channel MOSFET, 20V / 3.2A, SOT-23 |
| Gate resistor | R12 | Limits inrush current to gate, reduces ringing |
| Gate pulldown | R14 | Ensures MOSFET is OFF during ESP32-C3 boot / GPIO floating |

### Parameters

| Parameter | Value |
|-----------|-------|
| Gate threshold voltage (Vgs(th)) | ~1.0–1.5V (logic-level compatible) |
| Rds(on) at Vgs=3.3V | Low enough for buzzer current (< 100mA) |
| Buzzer current | ~30–100mA depending on type and sound level |
| Gate pulldown resistor (R14) | 10kΩ–100kΩ typical |

### Behavior

- GPIO7 **HIGH** (3.3V) → Q1 turns ON → buzzer sounds.
- GPIO7 **LOW** (0V) → Q1 turns OFF → buzzer silent.
- The buzzer can be pulsed in firmware to generate different tones or patterns (e.g., continuous = intrusion, intermittent = warning).

### MOSFET Selection Rationale

The **DN2302S** was chosen because:

- **Low Vgs(th)** — Fully enhanced at 3.3V gate drive from ESP32-C3.
- **Low Rds(on)** — Minimizes voltage drop and heat dissipation at buzzer current levels.
- **Small package** — SOT-23 fits compact PCB layout.
- **Adequate current rating** — 3.2A continuous is well above the ~100mA buzzer requirement, providing margin.

## Alarm Activation Logic

The firmware controls the alarm outputs based on the following state machine:

```
[ARMED / IDLE]
    |
    +-- Sensor Triggered (door / PIR)
    |       |
    |       v
    |   [VISION CHECK]
    |       |
    |       +-- ESP32-CAM reports "suspicious" (covered face)
    |       |       |
    |       |       v
    |       |   [ALARM ON]
    |       |       |
    |       |       +-- Buzzer ON (continuous or pulsed)
    |       |       +-- LED ON (continuous or fast blink)
    |       |       |
    |       |       +-- Timeout / User Disarm
    |       |               |
    |       |               v
    |       |           [ALARM OFF]
    |       |               |
    |       |               v
    |       |           [Return to ARMED]
    |       |
    |       +-- ESP32-CAM reports "clear" (visible face)
    |       |       |
    |       |       v
    |       |   [LOG EVENT]
    |       |       |
    |       |       v
    |       |   [Return to ARMED]
    |       |
    |       +-- ESP32-CAM timeout / not connected
    |               |
    |               v
    |           [Fallback: ALARM ON]
    |               |
    |               v
    |           [Return to ARMED]
    |
    +-- No trigger
            |
            v
        [Remain in ARMED]
```

## Alarm Patterns (Suggested Firmware Implementation)

| State | LED Pattern | Buzzer Pattern |
|-------|-------------|----------------|
| Armed / Idle | Slow blink (1Hz) | Off |
| Sensor Triggered | Fast blink (4Hz) | Off |
| Vision Analysis | Fast blink (4Hz) | Off |
| Alarm Active | Solid ON | Continuous tone |
| Disarmed / Safe | Solid OFF | Off |
| Error / CAM disconnected | Double blink | Single chirp every 10s |

## Safety and Reliability

- The gate pulldown resistor (R14) ensures the buzzer cannot accidentally activate during ESP32-C3 boot when GPIOs are in high-impedance state.
- The current-limiting resistor (R16) protects the LED from overcurrent if the GPIO drive strength is higher than expected.
- The TPS2553 current-limited switch (U4) protects the USB host from overcurrent if the buzzer or any 5V load fails short.
- If the buzzer is too loud for the intended environment, consider PWM drive at lower duty cycle or a series resistor.

## Design Notes

- The buzzer draws from the 5V rail directly, not the 3.3V LDO, to avoid overloading the AP2112K regulator.
- Verify buzzer polarity if using a polarized electromagnetic type; reverse connection will not damage the device but may reduce sound output.
- The KPEG242-5V part in the BOM appears to be an electromagnetic buzzer; confirm the exact pinout and sound level before finalizing the firmware alarm patterns.
