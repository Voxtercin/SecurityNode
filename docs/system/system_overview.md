# System Overview — SecurityNode

## Project Description

SecurityNode is an embedded security system based on an **ESP32-C3-WROOM-02-N4** microcontroller and an **ESP32-CAM** vision module. The platform monitors physical access events using external sensors and augments a conventional alarm with event-triggered visual verification.

## Functional Requirements

1. **Sensor Monitoring**: Continuously monitor door contact and PIR motion sensor inputs.
2. **Event Detection**: Trigger an alarm sequence when a door-opening or motion event is detected.
3. **Visual Verification**: Request the ESP32-CAM module to capture and analyze an image of the detected event.
4. **Threat Assessment**: Determine if the detected person has a visible or covered face (e.g., helmet, mask, or balaclava).
5. **Alarm Activation**: Activate local alarm indicators (buzzer and LED) if a suspicious condition is detected.
6. **Modular Operation**: Remain operational as a basic alarm even if the vision module is not connected or active.

## Use Cases

> **Design-stage note:** The behavior below describes the *proposed* firmware/FSM. This is a design-stage, not-yet-fabricated project — no firmware exists yet. The flow is the intended design, not as-built behavior.

### Primary Use Case: Intrusion Detection with Visual Verification

```
[Idle State]
    |
    v
[Door Open / Motion Detected]
    |
    v
[Trigger ESP32-CAM Capture]
    |
    v
[Analyze Image — Face Visible?]
    |-- NO / Covered --> [Activate Buzzer + LED]
    |
    |-- YES / Clear ----> [Log Event, Return to Idle]
    |
    v
[Return to Idle]
```

### Fallback Use Case: Basic Alarm Mode

If the ESP32-CAM module is not available or vision analysis fails, the ESP32-C3 acts as a standalone alarm controller, activating outputs on any sensor event.

## System Block Diagram

```
+------------------+   UART1 (IO0=RX/IO1=TX)    +------------------+
|                  |<------------------------->|                  |
|   ESP32-C3       |  via U5 isolation -> J3    |   ESP32-CAM      |
|   (Controller)   |                            |   (Vision)       |
|                  |                            |                  |
+--------+---------+                            +--------+---------+
         |                                               |
         | GPIO                                          | Camera
         |                                               | Sensor
    +----+----+                                    +-----+-----+
    |         |                                    |           |
+---v---+ +---v----+                          +----v----+ +----v----+
| Buzzer| | Alarm  |                          |  Lens   | |  Wi-Fi  |
| (LS1) | | LED    |                          +---------+ | Antenna |
+-------+ +--------+                                      +---------+
         |
         | GPIO
    +----+----+
    |         |
+---v---+ +---v----+
| Door  | |  PIR   |
|Contact| |Motion  |
+-------+ +--------+
         |
         | Native USB (IO18/IO19) + 5V power in
    +----v----+
    | Mini USB|
    |   (J2)  |
    +---------+
```

## Key Components Summary

| Module | Component | Role |
|--------|-----------|------|
| Main Controller | ESP32-C3-WROOM-02-N4 | Security logic, sensor polling, alarm control, UART master |
| Vision Module | ESP32-CAM (via J3) | Image capture, face coverage analysis, UART slave |
| Power Input | Mini USB B (J2) | 5V DC supply |
| 3.3V Regulator | AP2112K-3.3TRG1 (U1) | LDO supplying the 3.3V rail (ESP32-C3 + U5 + logic pull-ups) |
| ESP32-CAM Power Gate | TPS2553DBVR (U4) | Current-limited power switch that gates the CAM 5V branch (CAM_5V) only; enabled by CAM_EN, OFF by default |
| CAM-UART Isolation Switch | SN74LVC2G66DCTR (U5) | Analog switch on the C3↔CAM UART1 link; auto-connects/isolates based on CAM presence (control tied to CAM_3V3) |
| Buzzer Driver | DN2302S (Q1) | N-channel MOSFET for buzzer activation |
| Alarm LED | LED_0805_RED | Visual alarm indicator |
| ESD Protection | USBLC6-2P6, SMF5.0CA | Transient voltage suppression on USB and power lines |

## Academic Context

This project integrates:

- Power regulation and distribution
- Digital input conditioning
- Embedded control and state machines
- Actuator driving (MOSFET)
- Sensor-based event detection
- Computer vision assistance
- Inter-module communication protocols

into a compact, modular PCB-based security platform.
