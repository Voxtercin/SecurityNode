# SecurityNode

SecurityNode is an embedded security system based on an **ESP32-C3** microcontroller and an **ESP32-CAM** vision module. The platform monitors physical access events using external sensors and augments a conventional alarm with event-triggered visual verification.

## Overview

The system is designed around a compact PCB and integrates:

- **ESP32-C3** as the central security controller
- **ESP32-CAM** as an auxiliary vision-processing module
- **Door contact** and **PIR motion sensor** for event detection
- **Alarm LED** and **buzzer** (MOSFET-driven) for local indication
- **USB power input** with **3.3 V regulation**

When a door-opening or motion event is detected, the ESP32-C3 can request the ESP32-CAM to capture and analyze an image. The vision module determines whether the person entering has a visible or covered face (e.g., helmet, mask, or balaclava). If a suspicious condition is detected, the ESP32-C3 activates the buzzer and alarm LED.

## Repository Structure

```
SecurityNode/
├── assets/              # PCB renders, schematic blocks, and source PDFs
├── docs/
│   ├── hardware/        # Hardware interface and design documentation
│   ├── firmware/        # Firmware architecture and protocols
│   ├── presentation/    # Final presentation (HTML + PDF)
│   ├── software/        # Host-side tools and utilities
│   ├── system/          # System architecture and requirements
│   └── teacher_notes/   # Presentation guidelines
├── hardware/
│   └── altium/          # Altium Designer project files
├── firmware/
│   ├── esp32c3/         # Main controller firmware
│   └── esp32cam/        # Vision module firmware
├── software/
│   └── host/            # PC-side configuration and monitoring tools
├── tests/               # Validation procedures and bring-up checklists
└── tools/               # Helper scripts, export helpers, and generators
```

## Modular Architecture

- The ESP32-C3 acts as the **central security controller**, managing sensor inputs and actuators.
- The ESP32-CAM functions as an **auxiliary vision-processing module**.
- The system remains operational as a **basic alarm** even if the vision module is not used.
- Provides an **intelligent upgrade path** for advanced threat detection.

## Academic Context

This project integrates:

- Power regulation
- Digital input conditioning
- Embedded control
- Actuator driving
- Sensor-based event detection
- Computer vision assistance

into a compact PCB-based security platform. It is developed as a **Final Project of Electronic Design**.

## License

This project is licensed under the [MIT License](LICENSE).
