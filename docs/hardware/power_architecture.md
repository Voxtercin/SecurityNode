# Power Architecture — SecurityNode

## Power Tree

```
USB 5V Input (J2)
    |
    +-- TVS (D2: SMF5.0CA) -- ESD / transient clamping
    |
    +-- Ferrite Bead (FB1) -- EMI filtering
    |
    +-- USBLC6-2P6 -- ESD protection for data lines
    |
    v
TPS2553DBVR (U4) -- Adjustable current-limited power distribution switch
    |  ILIMIT = 0.075–1.7A (set by external resistor)
    |
    +---> 5V Rail (local distribution)
    |        |
    |        +---> J3 pin (ESP32-CAM 5V supply)
    |        |
    |        +---> KPEG242-5V buzzer (LS1, switched by Q1)
    |        |
    |        +---> Other 5V loads
    |
    v
AP2112K-3.3TRG1 (U2) -- 600mA LDO
    |
    v
3.3V Rail (system logic, ESP32-C3, ESP32-CAM I/O)
    |
    +---> ESP32-C3-WROOM-02-N4 (U3)
    +---> SN74LVC2G66DCTR (U5)
    +---> LEDs, sensors, pull-ups
```

## Key Components

### Input Protection — D2 (SMF5.0CA)

- Bidirectional TVS diode in SOD-123FL package
- 5V working voltage, clamps transients up to ~9.2V
- Protects downstream circuitry from USB plug-in spikes and ESD

### EMI Filter — FB1 (TAI-TECH HCB1608KF-600T20)

- 60Ω @ 100MHz, 2A ferrite bead in 0603 package
- Provides high-frequency impedance to isolate USB noise from board

### ESD Protection — USBLC6-2P6

- SOT-666-6 ESD diode array
- Protects USB D+ and D- data lines from static discharge

### Current-Limited Switch — U4 (TPS2553DBVR)

- Adjustable current limit: 0.075A to 1.7A
- 85mΩ on-resistance at 5V
- Active-high enable, reverse blocking
- Provides soft-start and protection against USB over-current

### 3.3V LDO — U2 (AP2112K-3.3TRG1)

- Input: up to 6V (typically 5V from TPS2553 output)
- Output: fixed 3.3V ± 2%
- Maximum output current: 600mA
- Low dropout, low quiescent current — suitable for battery-friendly designs
- Thermal shutdown and current limit protection built-in

## Decoupling Strategy

| Capacitor | Value | Package | Location / Purpose |
|-----------|-------|---------|--------------------|
| C3 | 22uF | 0805 | Bulk input near USB / TPS2553 |
| C4 | 470nF | 0603 | High-frequency decoupling near TPS2553 |
| C5 | 100nF | 0603 | HF decoupling near AP2112 input |
| C6–C8 | 10uF | 0603 | Bulk decoupling on 3.3V rail (multiple locations) |
| C9–C16 | 100nF–1uF | 0603 | Local decoupling at IC power pins |

## Power Budget (Estimated)

| Load | Voltage | Current (typ) | Current (max) |
|------|---------|---------------|---------------|
| ESP32-C3-WROOM-02-N4 | 3.3V | 80mA | 300mA (Wi-Fi TX peak) |
| ESP32-CAM | 3.3V / 5V | 120mA | 500mA (Wi-Fi + camera peak) |
| Buzzer (LS1) | 5V | 30mA | 100mA |
| Alarm LED | 3.3V | 5mA | 20mA |
| Sensors (PIR, door contact) | 3.3V | 1mA | 10mA |
| SN74LVC2G66 | 3.3V | 1uA | 10uA |
| Other logic / pull-ups | 3.3V | 5mA | 15mA |
| **Total Estimated** | — | **~240mA** | **~950mA** |

> Note: The TPS2553 current limit must be set above the maximum expected 5V load but below the USB host capability (typically 500mA for standard USB 2.0). If total load exceeds 500mA, a higher-current USB source or external 5V supply is required.

## Design Notes

- The ferrite bead (FB1) + bulk capacitors (C3, C4) form a π-filter that reduces conducted EMI from the switching/digital loads back to the USB host.
- The AP2112K is rated for 600mA, which is sufficient for the ESP32-C3 and peripherals but may require thermal consideration if the ESP32-CAM draws heavily from the 3.3V rail.
- The ESP32-CAM receives both 5V (for its internal regulator) and 3.3V (for I/O reference) through connector J3.
- Reverse-voltage protection is provided inherently by the TPS2553 (reverse blocking feature) and the Schottky diode (D3) on the 5V path.
