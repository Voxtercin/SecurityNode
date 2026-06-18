# Firmware Architecture — SecurityNode

> **Status: PROPOSED / design-stage.** No firmware exists yet (the code directories
> currently hold only `.gitkeep`). Everything in this document — the FSM, the RTOS
> task layout, the proposed `src/` file structure, and the build/flash instructions —
> is a planned design, not as-built. Pin references have been corrected against the
> authoritative schematic netlist (see `docs/application_node/DESIGN-AUDIT.md`).

## Overview

The SecurityNode firmware runs on the **ESP32-C3-WROOM-02-N4** and is responsible for:

1. Sensor input monitoring (door contact, PIR)
2. Alarm output control (buzzer, LED)
3. Communication with the ESP32-CAM vision module
4. System state management and decision logic
5. UART0 debug/console interface (programming itself is over native USB — see *Build and Flash*)

## Main Firmware Loop

```
[Setup]
    |
    +-- Initialize GPIOs
    +-- Initialize UARTs (UART0 debug/console + UART1 CAM link)
    +-- Initialize Wi-Fi (optional, for remote monitoring)
    +-- Set initial state: ARMED
    |
    v
[Main Loop]
    |
    +-- Read sensor inputs (IO4 door, IO5 PIR)
    +-- Evaluate state machine
    +-- Handle UART1 commands from CAM
    +-- Handle UART0 / USB-CDC commands (debug / config)
    +-- Update alarm outputs
    +-- Sleep / yield to RTOS
    |
    v
[Repeat]
```

## State Machine (Proposed)

> Proposed FSM — no firmware implements this yet.

The core of the firmware is a finite-state machine (FSM) that governs system behavior:

```
                    +-----------+
                    |   INIT    |
                    +-----+-----+
                          |
                          v
                    +-----------+
        +---------->|   ARMED   |<----------+
        |           +----+------+           |
        |                |                  |
        |    [No trigger]|   [Sensor event]  |
        |                |                  |
        |                v                  |
        |           +-----------+           |
        |           | TRIGGERED |           |
        |           +----+------+           |
        |                |                  |
        |                v                  |
        |           +-----------+           |
        |           |  VISION   |           |
        |           |  CHECK    |           |
        |           +----+------+           |
        |                |                  |
        |   +------------+------------+      |
        |   |                         |      |
        |   v                         v      |
        | +-----------+           +-----------+ |
        | |  CLEAR    |           |  ALARM    | |
        | |  (safe)   |           |  (active) | |
        | +----+------+           +----+------+ |
        |      |                         |      |
        |      |   [Disarm / timeout]    |      |
        |      |                         |      |
        +------+-------------------------+------+
```

### State Descriptions

| State | Description | Entry Actions | Exit Actions |
|-------|-------------|---------------|--------------|
| **INIT** | System startup, self-test | Initialize hardware, test LED/buzzer, check CAM presence | Transition to ARMED |
| **ARMED** | Normal monitoring mode | LED slow blink (1Hz), buzzer OFF, sensors active | — |
| **TRIGGERED** | Sensor event detected | LED fast blink (4Hz), send CMD_CAPTURE to CAM | — |
| **VISION_CHECK** | Waiting for CAM analysis | Maintain fast blink, start 5s timeout timer | — |
| **CLEAR** | CAM reports visible face | Log event, return to ARMED | — |
| **ALARM** | CAM reports suspicious or timeout | LED solid ON, buzzer ON, log event | Wait for disarm or timeout |

## Sensor Input Handling

### Interrupt-Driven vs. Polling

Two approaches are supported:

**Interrupt-Driven (Recommended)**
- Configure IO4 (door, J4) and IO5 (PIR, J5) with `GPIO_INTR_POSEDGE` or `GPIO_INTR_NEGEDGE`.
- ISR sets a flag; main loop processes the flag to avoid long ISRs.
- Low power: ESP32-C3 can enter light sleep and wake on GPIO interrupt.

**Polling**
- Read GPIO states every 50–100ms in the main loop.
- Simpler but higher power consumption and slightly slower response.

### Debounce

- Software timer: ignore state changes within 50ms of the first transition.
- Prevents false triggers from mechanical switch bounce (door contact) or PIR module instability.

> Note: the door has on-board RC hardware debounce, and the PIR is powered from the
> **5V rail** (not 3.3V) with its output conditioned by a clamp/divider network before
> reaching IO5. The active polarity (active-high vs active-low) of both inputs is
> inferred from the conditioning network topology and is **to be confirmed** against the
> schematic before the ISR edge/level logic is finalized.

## UART Communication

### UART0 Debug Header (direct to J1)

| Parameter | Value |
|-----------|-------|
| Port | UART0 (RX = GPIO20 / pin11, TX = GPIO21 / pin12) |
| Routing | Wired **direct to header J1**, NOT through U5 |
| Baud Rate | 115200 |
| Purpose | Debug logs, console, configuration commands; alternate flash header |

> Note: this is a plain UART0 debug/console header. It is **not** the primary
> programming path — the C3 is flashed over its native USB (see *Build and Flash*).

### ESP32-CAM Link UART

| Parameter | Value |
|-----------|-------|
| Port | UART1 (RX = **IO0** / pin18 ← R20 1k ← U5, TX = **IO1** / pin17 → R21 1k → U5) |
| Baud Rate | 115200 |
| Purpose | Capture commands, result codes, status messages |

> Note the direction: **IO0 = UART1_RX, IO1 = UART1_TX**. The link passes through
> **U5 (SN74LVC2G66)**, a CAM-presence-gated series isolation switch whose control pins
> are hard-tied to the CAM's own 3.3V (`CAM_3V3`). U5 auto-connects the link only when a
> powered CAM is seated and auto-disconnects when the CAM is absent/off — firmware does
> **not** control U5, and it is **not** a programming multiplexer.

### UART Command Protocol

See `docs/firmware/cam_protocol.md` for detailed message format.

## Alarm Output Control

### LED (IO6 → R15 330R → D5)

```cpp
// Example (Arduino-style pseudocode)
digitalWrite(ALARM_LED_PIN, HIGH);   // Alarm ON
digitalWrite(ALARM_LED_PIN, LOW);    // Alarm OFF

// Blink pattern
void blinkLED(int freqHz, int durationMs) {
    int periodMs = 1000 / freqHz;
    int cycles = durationMs / periodMs;
    for (int i = 0; i < cycles; i++) {
        digitalWrite(ALARM_LED_PIN, HIGH);
        delay(periodMs / 2);
        digitalWrite(ALARM_LED_PIN, LOW);
        delay(periodMs / 2);
    }
}
```

### Buzzer (IO10 → R12 100R → Q1 DN2302S → LS1)

```cpp
digitalWrite(BUZZER_PIN, HIGH);   // Buzzer ON
digitalWrite(BUZZER_PIN, LOW);    // Buzzer OFF

// Pulsed tone
void pulseBuzzer(int onMs, int offMs, int cycles) {
    for (int i = 0; i < cycles; i++) {
        digitalWrite(BUZZER_PIN, HIGH);
        delay(onMs);
        digitalWrite(BUZZER_PIN, LOW);
        delay(offMs);
    }
}
```

## ESP32-CAM Power Gate Control (IO7 / IO3)

The ESP32-CAM's 5V supply (`CAM_5V`) is gated by **U4 (TPS2553)**, not powered
continuously. The C3 controls and monitors this gate:

| Signal | Pin | Direction | Role |
|--------|-----|-----------|------|
| CAM_EN | **IO7** (→ R9 100R → U4 EN) | Output | Drive HIGH to enable the CAM 5V gate; held OFF by default at boot (R11 100k pulldown on U4 EN) |
| CAM_FAULT | **IO3** (← U4 FAULT, R10 10k pull-up) | Input | Active-low fault flag from U4 (over-current / thermal); read to detect CAM power faults |

> Proposed sequencing: the gate is off at reset, so firmware should drive IO7 HIGH to
> power the CAM before attempting any UART1 capture, allow the CAM to boot, then watch
> IO3 for a fault. The U5 isolation switch will only connect the UART1 link once the
> powered CAM asserts its own `CAM_3V3`, so a brief settle delay after enabling the gate
> is expected. (Behavior is proposed; no firmware implements it yet.)

## Wi-Fi and Networking (Optional)

The ESP32-C3 includes Wi-Fi and Bluetooth LE. Potential features:

- **Remote monitoring**: Publish sensor events and alarm status to MQTT broker.
- **OTA updates**: Firmware updates over Wi-Fi.
- **Mobile app**: Simple web server or BLE interface for arming/disarming.
- **Cloud logging**: Send images or event logs to a remote server.

> Note: Wi-Fi operation increases power consumption. Consider deep-sleep strategies if battery operation is desired.

## File Structure (Proposed)

```
firmware/esp32c3/
├── src/
│   ├── main.cpp              # Entry point, setup(), loop()
│   ├── state_machine.cpp     # FSM implementation
│   ├── state_machine.h
│   ├── sensors.cpp           # Door contact, PIR input handling
│   ├── sensors.h
│   ├── alarm.cpp             # Buzzer and LED control
│   ├── alarm.h
│   ├── cam_uart.cpp          # ESP32-CAM communication
│   ├── cam_uart.h
│   ├── debug_uart.cpp        # UART0 debug / CLI (native-USB CDC + J1 header)
│   ├── debug_uart.h
│   └── config.h              # Pin definitions, timeouts, constants
├── lib/                      # External libraries (if any)
├── platformio.ini            # PlatformIO configuration (if used)
└── README.md                 # Build and flash instructions
```

## Build and Flash (Proposed)

> Proposed toolchain/workflow — no firmware project exists yet (the `firmware/esp32c3/`
> tree holds only a `.gitkeep`). The directory layout above and the commands below are
> the planned setup.
>
> **Programming path:** the ESP32-C3 is flashed over its **native USB
> (USB-Serial/JTAG)** — J2 Mini-USB → U2 USBLC6 ESD → R3/R4 22R → IO18 (D−) / IO19 (D+).
> There is **no USB-UART bridge chip (no CP210x/CH340)** on this board; the host
> enumerates the C3's built-in USB CDC/JTAG device directly. To force the bootloader
> manually, hold SW1 (BOOT, IO9) and tap SW2 (EN/reset). Header J1 is an **alternate**
> UART0 flash/debug path (GPIO20/21) if you prefer wiring an external adapter.

### Using PlatformIO (Recommended)

```bash
cd firmware/esp32c3
platformio run --target upload   # uploads over the native USB serial/JTAG port
```

### Using Arduino IDE

1. Install ESP32 board support (ESP32-C3 variant).
2. Select board: "ESP32C3 Dev Module".
3. Select the serial port that enumerates when J2 is plugged in — this is the C3's
   **native USB (USB-Serial/JTAG)** device, not a USB-UART bridge port.
4. Upload sketch (hold SW1 + tap SW2 first if the board does not auto-enter the
   bootloader).

### Using ESP-IDF

```bash
cd firmware/esp32c3
idf.py set-target esp32c3
idf.py build
idf.py flash
```

## Design Notes (Proposed)

> Proposed implementation guidance and RTOS task layout — design-stage, not yet built.

- Keep ISRs short: set flags and defer processing to the main loop or a FreeRTOS task.
- Use a ring buffer for UART1 receive to handle asynchronous CAM responses.
- Implement a watchdog timer (ESP32-C3 has a hardware TWDT) to recover from firmware hangs.
- **Proposed FreeRTOS task layout:** separate tasks for sensor polling (IO4/IO5),
  UART1 CAM-link handling (IO0/IO1), alarm output control (IO6 LED, IO10 buzzer), and
  CAM power-gate supervision (IO7 enable / IO3 fault). This is a proposed decomposition
  to improve responsiveness; the exact task count and priorities are to be decided when
  firmware is written.
- Store configuration (Wi-Fi credentials, sensitivity thresholds) in NVS (Non-Volatile Storage) to persist across reboots.
