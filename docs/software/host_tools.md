# Host Tools — SecurityNode

## Overview

This directory is reserved for PC-side software that supports the SecurityNode system. While no host application currently exists, the following tools are planned for future development to enhance configuration, monitoring, and diagnostics.

## Planned Tools

### 1. Configuration Utility

A Python or C++ desktop application to configure SecurityNode settings over USB UART:

- Set sensor sensitivity thresholds
- Configure alarm timeout durations
- Set Wi-Fi credentials (if Wi-Fi is enabled)
- Update the ESP32-CAM detection algorithm parameters

### 2. Event Log Viewer

A simple GUI or CLI tool to read and display event logs from the ESP32-C3:

- Connect via USB and request log dump
- Display events in a timeline (sensor triggers, CAM results, alarm activations)
- Export logs to CSV or JSON for analysis

### 3. Firmware Update Tool

A helper script to flash both ESP32-C3 and ESP32-CAM firmware:

- Automatically toggle the U5 analog switch to target the correct module
- Enter bootloader mode automatically (assert BOOT/EN pins via serial control lines if available)
- Wrap `esptool.py` with user-friendly prompts

### 4. Serial Monitor / Debug Console

An enhanced serial terminal with protocol awareness:

- Decode and display CAM protocol frames in human-readable format
- Color-coded output for different message types (commands, responses, errors)
- Send manual commands to the ESP32-CAM for testing

### 5. Network Dashboard (Optional)

If Wi-Fi remote monitoring is implemented:

- Web-based dashboard showing current system status
- Live sensor state indicators
- Alarm history and statistics
- Remote arm/disarm capability (with authentication)

## Development Environment

### Python (Recommended for Rapid Prototyping)

```bash
# Example dependencies
pip install pyserial          # UART communication
pip install click             # CLI framework
pip install flask             # Web dashboard (optional)
pip install pyqt5             # GUI framework (optional)
```

### C++ / Qt

For a more polished cross-platform GUI, Qt (PyQt or C++ Qt) is a solid choice.

## File Structure (Planned)

```
software/host/
├── config_tool/
│   ├── main.py
│   ├── serial_interface.py
│   └── settings.yaml
├── log_viewer/
│   ├── main.py
│   └── log_parser.py
├── firmware_updater/
│   └── flash.py
├── serial_monitor/
│   └── monitor.py
├── dashboard/              # Optional web dashboard
│   ├── app.py
│   └── templates/
└── README.md               # Build and usage instructions
```

## Contributing

If you develop a host-side tool for SecurityNode, please:

1. Place it in a new subdirectory under `software/host/`.
2. Include a `README.md` with setup and usage instructions.
3. List dependencies in a `requirements.txt` (Python) or equivalent.
4. Consider cross-platform compatibility (Windows, macOS, Linux).

## Design Notes

- All UART communication should follow the protocol defined in `docs/firmware/cam_protocol.md`.
- Respect the U5 analog switch state when connecting to either ESP32-C3 or ESP32-CAM.
- Avoid flooding the ESP32-C3 with commands during alarm events to prevent missed sensor triggers.
- If implementing remote access over Wi-Fi, ensure proper authentication and encryption to prevent unauthorized control of the security system.
