# Firmware Architecture — SecurityNode

## Overview

The SecurityNode firmware runs on the **ESP32-C3-WROOM-02-N4** and is responsible for:

1. Sensor input monitoring (door contact, PIR)
2. Alarm output control (buzzer, LED)
3. Communication with the ESP32-CAM vision module
4. System state management and decision logic
5. UART programming / debug interface

## Main Firmware Loop

```
[Setup]
    |
    +-- Initialize GPIOs
    +-- Initialize UARTs (USB debug + CAM link)
    +-- Initialize Wi-Fi (optional, for remote monitoring)
    +-- Set initial state: ARMED
    |
    v
[Main Loop]
    |
    +-- Read sensor inputs (GPIO4, GPIO5)
    +-- Evaluate state machine
    +-- Handle UART commands from CAM
    +-- Handle UART commands from USB (debug / config)
    +-- Update alarm outputs
    +-- Sleep / yield to RTOS
    |
    v
[Repeat]
```

## State Machine

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
- Configure GPIO4 and GPIO5 with `GPIO_INTR_POSEDGE` or `GPIO_INTR_NEGEDGE`.
- ISR sets a flag; main loop processes the flag to avoid long ISRs.
- Low power: ESP32-C3 can enter light sleep and wake on GPIO interrupt.

**Polling**
- Read GPIO states every 50–100ms in the main loop.
- Simpler but higher power consumption and slightly slower response.

### Debounce

- Software timer: ignore state changes within 50ms of the first transition.
- Prevents false triggers from mechanical switch bounce (door contact) or PIR module instability.

## UART Communication

### USB Debug UART (via U5 switch)

| Parameter | Value |
|-----------|-------|
| Port | UART0 (GPIO1/2) |
| Baud Rate | 115200 |
| Purpose | Programming, debug logs, configuration commands |

### ESP32-CAM Link UART

| Parameter | Value |
|-----------|-------|
| Port | UART1 (GPIO8/9) |
| Baud Rate | 115200 |
| Purpose | Capture commands, result codes, status messages |

### UART Command Protocol

See `docs/firmware/cam_protocol.md` for detailed message format.

## Alarm Output Control

### LED (GPIO6)

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

### Buzzer (GPIO7 via Q1 MOSFET)

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
│   ├── debug_uart.cpp        # USB debug / CLI
│   ├── debug_uart.h
│   └── config.h              # Pin definitions, timeouts, constants
├── lib/                      # External libraries (if any)
├── platformio.ini            # PlatformIO configuration (if used)
└── README.md                 # Build and flash instructions
```

## Build and Flash

### Using PlatformIO (Recommended)

```bash
cd firmware/esp32c3
platformio run --target upload
```

### Using Arduino IDE

1. Install ESP32 board support (ESP32-C3 variant).
2. Select board: "ESP32C3 Dev Module".
3. Select port corresponding to USB-UART bridge.
4. Upload sketch.

### Using ESP-IDF

```bash
cd firmware/esp32c3
idf.py set-target esp32c3
idf.py build
idf.py flash
```

## Design Notes

- Keep ISRs short: set flags and defer processing to the main loop or a FreeRTOS task.
- Use a ring buffer for UART receive to handle asynchronous CAM responses.
- Implement a watchdog timer (ESP32-C3 has a hardware TWDT) to recover from firmware hangs.
- If using FreeRTOS, consider separate tasks for sensor polling, UART handling, and alarm control to improve responsiveness.
- Store configuration (Wi-Fi credentials, sensitivity thresholds) in NVS (Non-Volatile Storage) to persist across reboots.
