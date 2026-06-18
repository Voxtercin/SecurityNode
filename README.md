# SecurityNode

SecurityNode is an embedded security system based on an **ESP32-C3** microcontroller and an **ESP32-CAM** vision module. The platform monitors physical access events using external sensors and augments a conventional alarm with event-triggered visual verification.

## Overview

The system is designed around a compact PCB and integrates:

- **ESP32-C3** as the central security controller
- **ESP32-CAM** as an auxiliary vision-processing module
- **Door contact** and **PIR motion sensor** for event detection
- **Alarm LED** and **buzzer** (MOSFET-driven) for local indication
- **USB power input** with **3.3 V regulation**

In the planned operating flow (design-stage; firmware not yet written), when a door-opening or motion event is detected, the ESP32-C3 requests the ESP32-CAM to capture and analyze an image. The vision module is intended to determine whether the person entering has a visible or covered face (e.g., helmet, mask, or balaclava). If a suspicious condition is detected, the ESP32-C3 activates the buzzer and alarm LED.

## Repository Structure

> **Project stage:** this is a **design-stage** project (PCB drawn in Altium but not yet
> fabricated; no firmware written). The documentation and design assets are the current
> deliverable. The code/source directories below — `hardware/altium/`, `firmware/esp32c3/`,
> `firmware/esp32cam/`, `software/host/`, `tests/`, and `tools/` — are currently **skeleton
> placeholders (each holds only a `.gitkeep`)**; the substance lives in `docs/` and `assets/`.

```
SecurityNode/
├── assets/              # PCB renders, schematic blocks, and source PDFs
├── docs/
│   ├── hardware/        # Hardware interface and design documentation
│   ├── firmware/        # Firmware architecture and protocols (proposed design)
│   ├── presentation/    # Final presentation (HTML + PDF)
│   ├── software/        # Host-side tools and utilities (proposed design)
│   ├── system/          # System architecture and requirements
│   └── teacher_notes/   # Presentation guidelines
├── hardware/
│   └── altium/          # Altium Designer project files (skeleton — not yet committed)
├── firmware/
│   ├── esp32c3/         # Main controller firmware (skeleton — not yet written)
│   └── esp32cam/        # Vision module firmware (skeleton — not yet written)
├── software/
│   └── host/            # PC-side configuration and monitoring tools (skeleton)
├── tests/               # Validation procedures and bring-up checklists (skeleton)
└── tools/               # Helper scripts, export helpers, and generators (skeleton)
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
