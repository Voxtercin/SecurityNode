# ESP32-CAM Interface — SecurityNode

## Overview

The **ESP32-CAM** module is connected to the SecurityNode main board via a dedicated header (J3). It serves as the auxiliary vision-processing module, capturing images and performing face-coverage analysis upon request from the ESP32-C3 controller.

## Physical Connection — J3 Header

The ESP32-CAM module (commonly the AI-Thinker variant with OV2640 camera) mounts onto the board through a 2-row header.

### Pinout (Typical ESP32-CAM AI-Thinker)

| J3 Pin | ESP32-CAM Pin | Function | Connected To |
|--------|---------------|----------|--------------|
| 1 | 3.3V | 3.3V power supply | AP2112K 3.3V output |
| 2 | GND | Ground | Global GND |
| 3 | 5V | 5V power supply | TPS2553 5V output |
| 4 | GND | Ground | Global GND |
| 5 | U0TXD | UART TX (CAM → C3) | ESP32-C3 GPIO9 (RX) |
| 6 | U0RXD | UART RX (CAM ← C3) | ESP32-C3 GPIO8 (TX) |
| 7 | IO0 | Boot / Strapping | ESP32-C3 GPIO10 or pulled HIGH |
| 8 | IO2 | SD Card / GPIO | Optional / unused |
| 9 | IO4 | SD Card / GPIO | Optional / unused |
| 10 | IO12 | SD Card / GPIO | Optional / unused |
| 11 | IO13 | SD Card / GPIO | Optional / unused |
| 12 | IO14 | SD Card / GPIO | Optional / unused |
| 13 | IO15 | SD Card / GPIO | Optional / unused |
| 14 | IO16 | PSRAM / GPIO | Optional / unused |

> **Note:** Verify exact J3 pin mapping against the Altium schematic, as pin assignments may vary depending on the specific ESP32-CAM module footprint used.

## Power Supply

The ESP32-CAM module requires **both 3.3V and 5V**:

| Rail | Source | Purpose |
|------|--------|---------|
| 3.3V | AP2112K (U2) | I/O logic, camera sensor digital supply |
| 5V | TPS2553 (U4) | Internal 5V→3.3V regulator on ESP32-CAM module, Wi-Fi RF amplifier |

### Power Considerations

- The ESP32-CAM can draw up to **500mA peak** during Wi-Fi transmission + camera operation.
- Ensure the TPS2553 current limit is set high enough (> 500mA) or bypass current limiting for the 5V CAM rail if a high-current USB adapter is used.
- The AP2112K (600mA max) may be insufficient if the ESP32-CAM draws heavily from the 3.3V rail. Monitor total 3.3V load.
- Place decoupling capacitors close to J3 pins for both 3.3V and 5V rails.

## UART Communication

A dedicated UART channel connects ESP32-C3 and ESP32-CAM, independent from the programming UART routed through U5.

| Parameter | Value |
|-----------|-------|
| UART Port | ESP32-C3 UART1 (GPIO8/9) |
| Baud Rate | 115200 (default) |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

### Message Flow

```
ESP32-C3                      ESP32-CAM
   |                               |
   |--- CMD_CAPTURE --------------->|
   |                               |
   |<-- ACK_CAPTURE ---------------|
   |                               |
   |                               | [Capture image]
   |                               | [Run face analysis]
   |                               |
   |<-- RESULT (CLEAR/SUSPICIOUS) --|
   |                               |
```

Detailed command structure is documented in `docs/firmware/cam_protocol.md`.

## Boot and Programming

The ESP32-CAM module can be programmed in two ways:

### 1. Via ESP32-C3 Bridge (Indirect)

- ESP32-C3 acts as a UART bridge, forwarding data from USB to the ESP32-CAM.
- Useful for remote firmware updates but adds complexity.

### 2. Direct Programming via U5 Analog Switch (Recommended)

- The **SN74LVC2G66DCTR (U5)** analog switch routes the USB UART lines to either:
  - **Position A**: USB ↔ ESP32-C3 (normal programming / debug)
  - **Position B**: USB ↔ ESP32-CAM (direct programming of the vision module)

### Entering ESP32-CAM Bootloader Mode

1. Set U5 to route USB UART to ESP32-CAM.
2. Pull ESP32-CAM IO0 LOW (via ESP32-C3 GPIO10 or a manual jumper).
3. Reset ESP32-CAM (power cycle or reset pin if available).
4. Release IO0.
5. ESP32-CAM enters UART download mode.

## Analog Switch — SN74LVC2G66DCTR (U5)

### Function

U5 is a **dual SPST analog switch** that allows the single USB UART interface to serve two masters:

| Control Signal | USB UART Connected To | Use Case |
|----------------|------------------------|----------|
| CTRL = LOW | ESP32-C3 | Normal operation, C3 programming |
| CTRL = HIGH | ESP32-CAM | CAM programming / direct debug |

### Control Logic

- The control pin of U5 is driven by an ESP32-C3 GPIO or a manual jumper/switch.
- Ensure both ESP32-C3 and ESP32-CAM do not attempt to drive the UART lines simultaneously.
- The switch has low on-state resistance (~3Ω), minimizing signal degradation at 115200 baud.

## Vision Processing Workflow

1. **Trigger**: Door open or PIR motion event detected by ESP32-C3.
2. **Command**: ESP32-C3 sends `CMD_CAPTURE` to ESP32-CAM via UART.
3. **Capture**: ESP32-CAM captures a frame from the OV2640 camera.
4. **Analysis**: ESP32-CAM runs face detection / coverage algorithm (e.g., Haar cascade, lightweight CNN, or simple skin-tone analysis).
5. **Result**: ESP32-CAM returns one of:
   - `RESULT_CLEAR` — face is visible, no threat detected.
   - `RESULT_SUSPICIOUS` — face is covered (mask, helmet, balaclava), threat detected.
   - `RESULT_ERROR` — capture or analysis failed.
6. **Action**: ESP32-C3 evaluates result and activates alarm if necessary.

## Module Isolation

The modular architecture allows the system to operate without the ESP32-CAM:

| Scenario | ESP32-CAM | ESP32-C3 Behavior |
|----------|-----------|-----------------|
| CAM connected and responsive | Active | Full vision-assisted alarm mode |
| CAM connected but not responding | Unresponsive | Timeout after 5s → fallback to basic alarm |
| CAM not installed | Absent | Basic alarm mode (sensor → immediate alarm) |

## Design Notes

- The ESP32-CAM AI-Thinker module includes an on-board PCB antenna; ensure no copper pours or traces block the antenna area on the main PCB.
- The camera ribbon cable (OV2640) connects to the ESP32-CAM module, not the main board. Ensure adequate clearance for the camera module.
- Keep UART traces between ESP32-C3 and J3 short and matched in length to minimize skew.
- If the ESP32-CAM draws excessive 3.3V current, consider adding a dedicated 3.3V regulator (e.g., another AP2112K or a switching regulator) for the CAM module.
- The AI-Thinker ESP32-CAM does not have an on-board USB-to-UART bridge; therefore, external programming (via U5 switch or FTDI adapter) is required.
