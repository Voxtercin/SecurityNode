# Bring-Up Checklist — SecurityNode

> **Status:** Design-stage / not-yet-fabricated board. This checklist describes the
> *planned* bring-up procedure verified against the schematic netlist. Pin numbers,
> designators, and chip roles below reflect the schematic ground truth.

## Pre-Power Checks

- [ ] Visually inspect PCB for solder bridges, missing components, or component orientation issues.
- [ ] Verify USBLC6-2P6 (U2) ESD array and input TVS diode (D1, SMF5.0CA) are correctly placed near USB connector J2.
- [ ] Confirm AP2112K-3.3TRG1 LDO (U1) and TPS2553DBVR CAM power gate (U4) orientations match the PCB silkscreen.
- [ ] Verify PTC resettable fuse F1 (Littelfuse 1206L075) is installed in the VBUS input path (whole-board over-current protection).
- [ ] Check that ferrite bead FB1 is installed between USB 5V and the main power rail.
- [ ] Verify decoupling capacitors are populated (C1 = 22µF VBUS bulk; per-designator value map is to be confirmed against the Altium source).

## Power-Up Sequence

- [ ] Connect a current-limited bench supply (5V, 500mA limit) to J2 or inject 5V at the test points.
- [ ] Measure 5V rail at the output of FB1 (should be ~4.9–5.0V). Note: the 5V rail feeds U1 (LDO) and U4 (CAM gate) in parallel, plus LS1 buzzer and J5 PIR VCC.
- [ ] Measure 3.3V rail at the output of AP2112K (U1): should be 3.28–3.32V. (The 3.3V rail feeds only U3 / the C3 and U5 plus logic pull-ups.)
- [ ] Check 3.3V ripple with an oscilloscope (< 50mVpp recommended).
- [ ] Confirm U4 (TPS2553) CAM_5V output is OFF by default at power-up — R11 100k pulldown holds CAM_EN low until the C3 drives IO7 (CAM_5V test is in the ESP32-CAM Interface section).

## Programming Interface

The ESP32-C3 (U3) is flashed over its **native USB** (USB-Serial/JTAG) on IO18/IO19 via J2 — there is **no** USB-UART bridge chip (no CP210x) on this board. J1 is an alternate UART0 debug/flash header (GPIO20/GPIO21 + BOOT/EN).

- [ ] Connect USB cable to J2.
- [ ] Verify PC enumerates the ESP32-C3's **native USB CDC (USB-Serial/JTAG)** device — NOT a CP210x or other USB-UART bridge.
- [ ] If using the alternate UART0 path instead, connect to header J1 (1=GND, 2=UART0_TX/GPIO21, 3=UART0_RX/GPIO20, 4=BOOT_GPIO9, 5=ESP_EN, 6=GND).
- [ ] Enter bootloader mode (manual): hold SW1 (BOOT, GPIO9), press SW2 (EN/reset), release SW2, then release SW1. (Native USB normally auto-resets into the bootloader without this.)
- [ ] Flash a simple "Hello World" / blink sketch to the ESP32-C3 (U3).

## Peripheral Tests

### Alarm Output
- [ ] Write a test sketch to toggle **IO10 (BUZZER)** — drives Q1 (DN2302S) gate via R12 100R.
- [ ] Verify buzzer (LS1, on 5V rail) sounds when IO10 is driven HIGH.
- [ ] Write a test sketch to toggle **IO6 (LED_ALERT)** — drives D5 (XL-2012SURC red LED) via R15 330R.
- [ ] Verify alarm LED (D5) lights when IO6 is driven HIGH.

### Sensor Inputs
- [ ] Connect a door contact switch to J4.
- [ ] Toggle the switch and verify the ESP32-C3 reads the correct logic level on **IO4 (DOOR_GPIO4)**. (Active polarity of the on-board debounce/clamp network — R7/R8/C10 + D2 Schottky — is to be confirmed against the schematic.)
- [ ] Connect a PIR sensor to J5 (PIR VCC on J5 is hard-wired to the **5V rail**).
- [ ] Verify PIR output triggers **IO5 (PIR_GPIO5)**. (Signal reaches IO5 through a clamp/conditioning network — D4 BAT54S + 100k bias resistors; exact polarity to be confirmed against the schematic.)

### ESP32-CAM Interface
- [ ] Seat ESP32-CAM module onto J3 connector.
- [ ] **CAM power-gate test:** drive **IO7 (CAM_EN)** HIGH and verify **CAM_5V** appears at U4 (TPS2553) output (J3 P$9 / J7 pin6). With IO7 LOW, confirm CAM_5V is OFF (R11 100k default-OFF).
- [ ] Verify the CAM module's own **CAM_3V3** rail comes up (the CAM's internal regulator output, J3 P$8) once CAM_5V is enabled.
- [ ] Force/simulate an over-current on the CAM branch and verify U4 asserts FAULT — confirm the C3 reads it on **IO3 (CAM_FAULT)** (R10 10k pull-up).
- [ ] Verify the UART1 link between ESP32-C3 and ESP32-CAM: C3 **IO0 = RX**, **IO1 = TX** (via R20/R21 1k and U5). Confirm with a logic analyzer. Note that U5 (SN74LVC2G66) auto-connects this link only when a powered CAM is seated (control tied to CAM_3V3) — verify continuity is present with the CAM powered and open with the CAM removed/off.
- [ ] Send a capture command from the ESP32-C3 and verify the CAM responds with status/ack.

## Communication Tests

- [ ] Test basic UART1 echo between ESP32-C3 (IO0=RX / IO1=TX) and ESP32-CAM over the U5-isolated link.
- [ ] Verify the U5 (SN74LVC2G66) **automatic presence gating**: with a powered CAM seated the UART1 link is connected; with the CAM absent/unpowered the link is isolated. U5 is NOT host/GPIO-selectable and is NOT a USB-to-UART routing mux.
- [ ] **Flash the ESP32-CAM out-of-band:** connect an external USB-UART adapter to header **J7** (6-pin: GND, CAM_TXD_RAW, CAM_RXD_RAW, B_GPIO0_CAM, CAM_3V3, CAM_5V) and fit the **J6** 2-pin CAM-GPIO0 boot jumper to enter the CAM bootloader. (No on-board USB path reaches the CAM; do NOT attempt to route the board's USB to the CAM.)

## Integration Test

> The event flow and vision/alarm behavior below describe the **proposed** firmware design
> (no firmware is committed yet — code directories hold only `.gitkeep`). Use as the planned
> acceptance test once firmware exists.

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
