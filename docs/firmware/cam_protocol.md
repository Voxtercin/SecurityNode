# CAM Protocol — SecurityNode

> **Status: proposed design — no firmware exists yet.** This is a design-stage specification for a not-yet-fabricated board; nothing here is as-built.

## Overview

This document defines the UART command protocol between the **ESP32-C3** (master) and the **ESP32-CAM** (slave). The protocol enables the ESP32-C3 to request image capture and receive face-coverage analysis results from the ESP32-CAM.

## Communication Parameters

| Parameter | Value |
|-----------|-------|
| Physical Interface | UART1 (ESP32-C3 IO0 = RX, IO1 = TX) |
| Baud Rate | 115200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |
| Message Format | Binary frame with header, length, payload, and checksum |

## Message Frame Format

All messages use a lightweight binary frame:

```
+--------+--------+----------+----------+----------+
| HEADER | LENGTH |  CMD_ID  | PAYLOAD  | CHECKSUM |
| 0xAA   |  N     |  1 byte  |  N bytes |  1 byte  |
+--------+--------+----------+----------+----------+
```

| Field | Size | Description |
|-------|------|-------------|
| HEADER | 1 byte | Always `0xAA` (frame start delimiter) |
| LENGTH | 1 byte | Length of payload (CMD_ID + PAYLOAD bytes), max 255 |
| CMD_ID | 1 byte | Command or response identifier |
| PAYLOAD | N bytes | Command-specific data (optional, can be 0 bytes) |
| CHECKSUM | 1 byte | XOR of all bytes from LENGTH to end of PAYLOAD |

### Checksum Calculation

```cpp
uint8_t calculateChecksum(uint8_t *data, uint8_t length) {
    uint8_t checksum = 0;
    for (uint8_t i = 0; i < length; i++) {
        checksum ^= data[i];
    }
    return checksum;
}
```

## Command Set (ESP32-C3 → ESP32-CAM)

### CMD_CAPTURE (0x01)

Requests the ESP32-CAM to capture an image and perform face-coverage analysis.

| Field | Value |
|-------|-------|
| CMD_ID | `0x01` |
| PAYLOAD | None (0 bytes) |

**Frame:**
```
0xAA 0x01 0x01 0x00
```

- `0xAA` — Header
- `0x01` — LENGTH (CMD_ID only, no payload)
- `0x01` — CMD_CAPTURE
- `0x00` — CHECKSUM (`0x01` ^ `0x01` = `0x00`)

### CMD_STATUS (0x02)

Requests the current status of the ESP32-CAM (e.g., ready, busy, error).

| Field | Value |
|-------|-------|
| CMD_ID | `0x02` |
| PAYLOAD | None |

**Frame:** `0xAA 0x01 0x02 0x03` (checksum: 0x01 ^ 0x02 = 0x03)

### CMD_RESET (0x03)

Requests a soft reset of the ESP32-CAM vision module.

| Field | Value |
|-------|-------|
| CMD_ID | `0x03` |
| PAYLOAD | None |

**Frame:** `0xAA 0x01 0x03 0x02` (checksum: 0x01 ^ 0x03 = 0x02)

## Response Set (ESP32-CAM → ESP32-C3)

### RSP_ACK (0x81)

Acknowledges receipt of a command.

| Field | Value |
|-------|-------|
| CMD_ID | `0x81` |
| PAYLOAD | 1 byte: echoed command ID |

**Example (ACK for CMD_CAPTURE):**
```
0xAA 0x02 0x81 0x01 0x82
```
- LENGTH = 2 (CMD_ID + 1 payload byte)
- CMD_ID = 0x81
- PAYLOAD = 0x01 (echo of CMD_CAPTURE)
- CHECKSUM = 0x02 ^ 0x81 ^ 0x01 = 0x82

### RSP_RESULT (0x82)

Returns the result of a capture and analysis operation.

| Field | Value |
|-------|-------|
| CMD_ID | `0x82` |
| PAYLOAD | 1 byte: result code |

**Result Codes:**

| Code | Meaning | ESP32-C3 Action |
|------|---------|-----------------|
| `0x00` | RESULT_CLEAR | Face visible, no threat. Log event, return to ARMED. |
| `0x01` | RESULT_SUSPICIOUS | Face covered (mask/helmet/balaclava). Activate alarm. |
| `0x02` | RESULT_NO_FACE | No face detected in image. Log event, return to ARMED. |
| `0x03` | RESULT_ERROR | Analysis failed (algorithm error, timeout). Fallback to basic alarm. |
| `0x04` | RESULT_BUSY | CAM is already processing a previous request. Retry later. |

**Example (RESULT_SUSPICIOUS):**
```
0xAA 0x02 0x82 0x01 0x81
```
- LENGTH = 2
- CMD_ID = 0x82
- PAYLOAD = 0x01 (SUSPICIOUS)
- CHECKSUM = 0x02 ^ 0x82 ^ 0x01 = 0x81

### RSP_STATUS (0x83)

Returns the current module status.

| Field | Value |
|-------|-------|
| CMD_ID | `0x83` |
| PAYLOAD | 1 byte: status code |

**Status Codes:**

| Code | Meaning |
|------|---------|
| `0x00` | STATUS_READY |
| `0x01` | STATUS_BUSY |
| `0x02` | STATUS_ERROR |
| `0x03` | STATUS_INIT |

### RSP_ERROR (0x84)

Reports a protocol or hardware error.

| Field | Value |
|-------|-------|
| CMD_ID | `0x84` |
| PAYLOAD | 1 byte: error code |

**Error Codes:**

| Code | Meaning |
|------|---------|
| `0x00` | ERR_NONE |
| `0x01` | ERR_CHECKSUM |
| `0x02` | ERR_UNKNOWN_CMD |
| `0x03` | ERR_TIMEOUT |
| `0x04` | ERR_CAM_FAIL |
| `0x05` | ERR_ANALYSIS_FAIL |

## Communication Sequence

### Normal Capture Flow

```
ESP32-C3                          ESP32-CAM
   |                                  |
   |--- CMD_CAPTURE ----------------->|
   |                                  |
   |<-- RSP_ACK (CMD_CAPTURE) --------|
   |                                  |
   |                                  | [Capture image]
   |                                  | [Run face coverage analysis]
   |                                  |
   |<-- RSP_RESULT (CLEAR/SUSPICIOUS)|
   |                                  |
```

### Status Query Flow

```
ESP32-C3                          ESP32-CAM
   |                                  |
   |--- CMD_STATUS ------------------>|
   |                                  |
   |<-- RSP_ACK (CMD_STATUS) --------|
   |                                  |
   |<-- RSP_STATUS (READY/BUSY) ------|
   |                                  |
```

### Error Recovery Flow

```
ESP32-C3                          ESP32-CAM
   |                                  |
   |--- CMD_CAPTURE ----------------->|
   |                                  |
   |<-- RSP_ERROR (ERR_CHECKSUM) -----|
   |                                  |
   |--- CMD_CAPTURE (retry) -------->|
   |                                  |
   |<-- RSP_ACK ----------------------|
   |                                  |
```

## Timeout Rules

| Operation | Timeout | Action on Timeout |
|-----------|---------|-------------------|
| ACK after command | 500ms | Retry command (max 3 retries) |
| RESULT after capture | 5000ms | Assume CAM failure, fallback to basic alarm |
| STATUS response | 500ms | Mark CAM as unresponsive |

## Implementation Notes

### ESP32-C3 Side

```cpp
// Pseudocode for sending a command
void sendCamCommand(uint8_t cmdId, uint8_t *payload, uint8_t payloadLen) {
    uint8_t frame[64];
    uint8_t idx = 0;

    frame[idx++] = 0xAA;          // Header
    frame[idx++] = 1 + payloadLen;  // LENGTH
    frame[idx++] = cmdId;         // CMD_ID

    for (uint8_t i = 0; i < payloadLen; i++) {
        frame[idx++] = payload[i];
    }

    // Calculate checksum (LENGTH ^ CMD_ID ^ payload...)
    uint8_t checksum = frame[1]; // start with LENGTH
    for (uint8_t i = 2; i < idx; i++) {
        checksum ^= frame[i];
    }
    frame[idx++] = checksum;

    uartWrite(frame, idx);
}
```

### ESP32-CAM Side

- Run a UART receive task that parses incoming frames.
- Validate header, length, and checksum before processing.
- Send ACK immediately upon receiving a valid command.
- Perform image capture and analysis asynchronously.
- Send RESULT when analysis completes.

## Future Extensions

- **Image streaming**: Transfer JPEG frames over UART for remote viewing (higher baud rate or SPI recommended).
- **Sensitivity adjustment**: Add payload to CMD_CAPTURE to set detection threshold.
- **Multi-face support**: Return count of detected faces and coverage status for each.
- **OTA for CAM**: Extend protocol to support firmware updates for the ESP32-CAM module.
