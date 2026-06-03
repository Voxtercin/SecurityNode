# Bring-Up Checklist — SecurityNode

## Pre-Power Checks

- [ ] Visually inspect PCB for solder bridges, missing components, or component orientation issues.
- [ ] Verify USBLC6-2P6 and TVS diode (D2) are correctly placed near USB connector J2.
- [ ] Confirm AP2112K-3.3TRG1 (U2) and TPS2553DBVR (U4) orientations match the PCB silkscreen.
- [ ] Check that ferrite bead FB1 is installed between USB 5V and the main power rail.
- [ ] Verify all decoupling capacitors (C3–C16) are populated.

## Power-Up Sequence

- [ ] Connect a current-limited bench supply (5V, 500mA limit) to J2 or inject 5V at the test points.
- [ ] Measure 5V rail at the output of FB1 (should be ~4.9–5.0V).
- [ ] Verify TPS2553 output: measure voltage at its output pin; check for current-limit status if load is high.
- [ ] Measure 3.3V rail at the output of AP2112K (U2): should be 3.28–3.32V.
- [ ] Check 3.3V ripple with an oscilloscope (< 50mVpp recommended).

## Programming Interface

- [ ] Connect USB cable to J2.
- [ ] Verify PC enumerates a CP210x / UART bridge or native USB CDC (depends on bootloader).
- [ ] Test UART communication through SW1/SW2 (BOOT and EN switches).
- [ ] Enter bootloader mode: hold SW1 (BOOT), press SW2 (EN), release SW1.
- [ ] Flash a simple "Hello World" / blink sketch to ESP32-C3.

## Peripheral Tests

### Alarm Output
- [ ] Write a test sketch to toggle the GPIO connected to Q1 (MOSFET gate).
- [ ] Verify buzzer (LS1) sounds when GPIO is driven HIGH.
- [ ] Verify alarm LED lights when its GPIO is driven HIGH.

### Sensor Inputs
- [ ] Connect a door contact switch to J4 (or designated header).
- [ ] Toggle the switch and verify ESP32-C3 reads the correct logic level.
- [ ] Connect a PIR sensor to J5 (or designated header).
- [ ] Verify PIR output triggers a digital input on ESP32-C3.

### ESP32-CAM Interface
- [ ] Seat ESP32-CAM module onto J3 connector.
- [ ] Verify 3.3V and 5V reach the CAM module power pins.
- [ ] Verify UART TX/RX lines between ESP32-C3 and ESP32-CAM via loopback or logic analyzer.
- [ ] Send a capture command from ESP32-C3 and verify CAM responds with status/ack.

## Communication Tests

- [ ] Test basic UART echo between ESP32-C3 and ESP32-CAM.
- [ ] Verify analog switch (U5) correctly routes UART to either ESP32-C3 or ESP32-CAM from the USB port.
- [ ] Test switchover between programming mode (USB → ESP32-C3) and CAM debug mode (USB → ESP32-CAM).

## Integration Test

- [ ] Trigger a door-open event.
- [ ] Confirm ESP32-C3 requests image capture from ESP32-CAM.
- [ ] Confirm alarm sounds if CAM reports a covered face (simulate with test command if vision algorithm is not yet ready).
- [ ] Confirm alarm does NOT sound if CAM reports a visible face.
- [ ] Disconnect ESP32-CAM and verify basic alarm still functions (fallback mode).

## Notes / Issues

| Date | Issue | Resolution |
|------|-------|------------|
|      |       |            |
|      |       |            |

## Sign-Off

| Test Phase | Pass | Fail | N/A |
|------------|------|------|-----|
| Pre-Power  | [ ]  | [ ]  | [ ] |
| Power-Up   | [ ]  | [ ]  | [ ] |
| Programming| [ ]  | [ ]  | [ ] |
| Peripherals| [ ]  | [ ]  | [ ] |
| Integration| [ ]  | [ ]  | [ ] |

---

**Tester:** ________________________  **Date:** ________________________
