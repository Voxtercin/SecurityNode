# SecurityNode — Design Documentation Audit

> Audit of the SecurityNode docs against the authoritative hardware ground truth
> (schematic netlist `SecurityNode.pdf`, rendered schematic, and Altium LiveBOM
> `SecurityNode.BomDoc`). Design-stage project: PCB drawn in Altium but not built,
> no firmware written. This report separates what is **true** from what is **false**
> so the application note can be fixed against reality.

---

## 1. Verdict & headline

The **high-level architecture is sound and correctly documented** — two MCUs (ESP32-C3 controller + ESP32-CAM vision slave), door/PIR sensing, LED+buzzer alarm, USB power with 3.3 V regulation, and a CAM-absent fallback are all accurate. **The hardware-fact layer is broken**: a single wrong ESP32-C3 pin map and a wrong understanding of two key chips (U4 and U5) were authored speculatively and then propagated into nearly every hardware/firmware doc, the bring-up checklist, the application note, *and* `CLAUDE.md` itself. Firmware written against the current pin tables would not work on the real board, so these are contradictions/fabrications to fix — not acceptable design latitude.

The root causes are few but pervasive: (a) the CAM UART is on the wrong GPIOs, (b) the buzzer is on the wrong GPIO, (c) UART0/programming is mis-described as a "USB-UART through U5" path when the C3 actually uses **native USB**, (d) U5 is called a programming multiplexer when it is an automatic presence-gated isolation switch, (e) U4 is called a whole-board current limiter when it only gates the CAM, and (f) the LDO and a few diode/LED/resistor designators are swapped. Fix those six root facts once and most of the documentation set becomes correct.

---

## 2. Verified correct (keep trusting these)

These claims were checked and **match the schematic/BOM**. They are safe to keep and to build on:

**Concept & system level**
- Two-MCU architecture: **U3 = ESP32-C3-WROOM-02-N4** (controller/master) + **ESP32-CAM** vision module that mounts on header **J3** (UART1 slave). (README, system_overview, firmware_architecture)
- Event flow concept: door/motion → C3 asks CAM to capture+judge face coverage → buzzer+LED if suspicious → degrade to basic alarm if CAM absent/times out. (Correct as a *proposal* — see §5.)
- Power input is **J2 USB Mini-B** (KH-MINI-SMT-5P-Cu), VBUS 5 V, with a 3.3 V regulated rail. (README, system_overview, power_architecture)

**Correct pins (the few that are right)**
- **Door contact = GPIO4** (IO4 / DOOR_GPIO4, from J4). ✔
- **PIR motion = GPIO5** (IO5 / PIR_GPIO5, from J5). ✔
- **Alarm LED = GPIO6** (IO6 / LED_ALERT). ✔ *(pin is right; the LED's designator/resistor are wrong — see §4)*
- Both sensor inputs are inputs; door has real hardware debounce on board. ✔

**Correct parts / topology**
- **Q1 = DN2302S** N-MOSFET, low-side buzzer switch; gate from the buzzer GPIO via **R12 100R**, with **R14 10k** pulldown. ✔ (pin number is wrong, see §3)
- **LS1 = KPEG242-5V** buzzer, on the **5 V rail**, low-side switched by Q1. ✔
- **D5 = XL-2012SURC** red 0805 LED is the actual alarm-LED part. ✔
- **U2 = USBLC6-2P6** ESD array on USB D+/D−; series **R3/R4 22R** to the C3. ✔
- **U4 = TPS2553DBVR** adjustable-ILIM USB power switch (ILIM via R13 21k5); EN via R9 100R. ✔ *(part right; role overstated — see §3)*
- **U1 = AP2112K-3.3** LDO ratings (≤6 V in, 3.3 V/600 mA out, thermal+current limit). ✔ *(ratings right; designator wrong — see §4)*
- **FB1 = HCB1608KF-600T20** ferrite (60 Ω@100 MHz, 2 A, 0603) in the VBUS→5 V path. ✔
- **SW1 = BOOT** (GPIO9, R1 pull-up), **SW2 = EN/reset** (R2 pull-up); manual bootloader = hold SW1, tap SW2. ✔
- The C3↔CAM UART1 link physically runs through **J3** and through **U5**, and both directions pass U5. ✔ (the *pins* and U5's *role description* are wrong elsewhere)
- README contains **no** pin/designator errors (it has no pin numbers) — fully accurate at its level of detail.

---

## 3. Critical errors (must fix) — HIGH severity

These are hardware-fact contradictions or fabrications. Firmware or bring-up done against them will fail. **Recurring** errors are grouped; the same wrong fact appears across many files.

| Location (recurring across docs) | What the docs say | The truth (ground truth) |
|---|---|---|
| **CAM UART1 pins** — mcu_interfaces (GPIO8/9), esp32cam_interface, firmware_architecture, cam_protocol, application_node (A2, pinout, block diag), **CLAUDE.md** | C3↔CAM UART1 is on **GPIO8 / GPIO9** | UART1 is **IO0 = RX (pin18)** and **IO1 = TX (pin17)**, via R20/R21 1k and U5. GPIO8 = STRAP_GPIO8, GPIO9 = BOOT_GPIO9 (SW1 BOOT). Note **IO0=RX, IO1=TX**. |
| **Buzzer pin** — mcu_interfaces, alarm_subsystem, firmware_architecture, application_node (pinout + block diag), **CLAUDE.md** | Buzzer / Q1 gate is on **GPIO7** | Buzzer is **IO10 (BUZZER)** → R12 100R → Q1 gate. **IO7 = CAM_EN** (drives U4 TPS2553 enable via R9), not the buzzer. |
| **Programming / UART0** — mcu_interfaces, firmware_architecture, application_node, **CLAUDE.md** | UART0 debug/programming is **GPIO1/2**, "routed through U5", USB-UART for flashing | UART0 is **RXD=GPIO20 (pin11)** / **TXD=GPIO21 (pin12)**, wired **direct to J1**, **never through U5**. GPIO1=UART1_TX, GPIO2=STRAP_GPIO2. |
| **C3 programmed via "USB-UART bridge"** — mcu_interfaces, firmware_architecture (Arduino step), esp32cam_interface, host_tools, bringup (CP210x), application_node (A6) | C3 is flashed over a USB→UART bridge / CP210x through J2; U5 muxes that UART | **No bridge chip exists.** The C3 is programmed over its **NATIVE USB** (USB-Serial/JTAG): J2 → U2 USBLC6 → R3/R4 22R → IO18(D−)/IO19(D+). PC enumerates the C3's built-in USB CDC, never a CP210x. |
| **BOOT/EN on GPIO0/GPIO3** — mcu_interfaces pinout | GPIO0 = BOOT (SW1); GPIO3 = EN (SW2) | **BOOT is IO9** (BOOT_GPIO9, R1, SW1). **EN is the dedicated EN pin (pin2)**, not a GPIO (ESP_EN, R2, C9, SW2). IO0 = UART1_RX; IO3 = CAM_FAULT. |
| **U4 (TPS2553) role** — system_overview, mcu_interfaces (buzzer "via TPS2553"), power_architecture, alarm_subsystem, application_node, bringup framing | U4 is a whole-board / USB current limiter / "power distribution"; the 5 V rail and buzzer flow through it | U4 is a **dedicated power GATE for the ESP32-CAM only**: IN=5 V rail, OUT=**CAM_5V** (J3 P$9 / J7 pin6). EN via IO7/CAM_EN; ILIM R13 21k5; FAULT→IO3; **R11 100k pulldown holds it OFF by default**. It does **not** feed the 5 V rail, LDO, or buzzer, and does **not** protect the USB host — **F1 PTC fuse** does. |
| **U5 (SN74LVC2G66) role** — system_overview, mcu_interfaces, esp32cam_interface (whole "USB-UART mux" section), host_tools, bringup, application_node | U5 is a USB-to-UART **programming multiplexer** that switches the USB UART between C3 and CAM (CTRL LOW/HIGH, host-selectable) | U5 is a **series isolation/disconnect switch on the C3↔CAM UART1 link only**. Control pins 1C/2C are **hard-tied to CAM_3V3** (the CAM's own 3.3 V output) → **automatic presence-gating**: closes only when a powered CAM is seated, opens when absent. **Not** a USB mux, **not** on UART0/USB, **not** host-controllable. |
| **Omitted PTC fuse F1** — power_architecture power tree | Input chain is TVS → ferrite → USBLC6 (no fuse) | First element after VBUS is **F1 PTC fuse** (Littelfuse 1206L075/16V, 750 mA hold / 1.5 A trip), in series before the D1 clamp and FB1. Whole-board USB over-current = F1, **not** U4. |
| **5 V rail topology** — power_architecture (TPS2553 in series ahead of rail; CAM 5V + buzzer on one node) | TPS2553 feeds the whole 5 V rail and the LDO; CAM 5 V and buzzer share that node | VBUS → F1 → D1+C1 → FB1 → **5 V RAIL**, which feeds **U1 LDO and U4 in parallel** (plus LS1, J5 PIR VCC). **CAM_5V is the gated U4 output (separate node)**; buzzer/PIR are on the ungated 5 V rail. |
| **CAM IO0 driven by C3 GPIO10** — mcu_interfaces (GPIO10=CAM_BOOT), esp32cam_interface | C3 GPIO10 drives CAM IO0 boot strap | **No C3 GPIO drives CAM IO0.** CAM GPIO0 (B_GPIO0_CAM, J3 P$6) is asserted only by the **J6 2-pin jumper** (+ J7 pin4, R17 10k). **IO10 = BUZZER.** |
| **Fabricated aux GPIOs / headers** — mcu_interfaces (GPIO11/12 AUX on J6/J7), sensor_interfaces (J6/J7 expansion, GPIO11/12 spare) | J6/J7 are general-purpose expansion headers exposing spare GPIO11/GPIO12 + 3.3V/5V | **Fabricated.** GPIO11/12 are not broken out (GPIO11 is internal/flash on WROOM). **J6 = 2-pin CAM-GPIO0 download jumper.** **J7 = 6-pin ESP32-CAM programming header** (1=GND, 2=CAM_TXD_RAW, 3=CAM_RXD_RAW, 4=B_GPIO0_CAM, 5=CAM_3V3, 6=CAM_5V). |
| **CAM debug "through the board's USB"** — mcu_interfaces, esp32cam_interface, bringup (USB→CAM switchover) | The CAM is programmed/debugged through the board's USB by switching U5 | The CAM is programmed **out-of-band via J7** with an **external USB-UART adapter** + J6 jumper. No on-board USB path reaches the CAM. |
| **PIR VCC selectable 3.3 V/5 V** — sensor_interfaces | PIR VCC (J5 pin1) can be 3.3 V or 5 V | J5 pin1 (PIR VCC) is **hard-wired to the 5 V rail only**. The board cannot supply 3.3 V to the PIR connector. |
| **J3 pin map** — esp32cam_interface | Generic AI-Thinker order: pin1=3V3, pin3=5V, pin5=TXD, pin6=RXD, pin7=IO0, pins 8-14 = SD/PSRAM GPIOs | Schematic J3: **P$2=CAM_TXD_RAW, P$3=CAM_RXD_RAW, P$6=B_GPIO0_CAM, P$8=CAM_3V3 (CAM's own 3V3, gates U5), P$9=CAM_5V**. SD/PSRAM breakouts are fabricated; CAM_3V3 is **not** fed from the AP2112K. |

**Net pin map to author against (authoritative ESP32-C3 / U3):**

| GPIO | Net / function |
|---|---|
| EN (pin2) | ESP_EN (reset, SW2, R2) |
| IO0 | UART1_RX (CAM link RX, ← R20 1k ← U5) |
| IO1 | UART1_TX (CAM link TX, → R21 1k → U5) |
| IO2 | STRAP_GPIO2 |
| IO3 | CAM_FAULT (← U4 FAULT, R10 10k pull-up) |
| IO4 | DOOR_GPIO4 (door, J4) |
| IO5 | PIR_GPIO5 (PIR, J5) |
| IO6 | LED_ALERT (→ R15 330R → D5) |
| IO7 | CAM_EN (→ R9 100R → U4 EN) |
| IO8 | STRAP_GPIO8 |
| IO9 | BOOT_GPIO9 (BOOT, SW1, R1) |
| IO10 | BUZZER (→ R12 100R → Q1 → LS1) |
| IO18 | USB_D_N (→ R3 22R → U2 → J2) |
| IO19 | USB_D_P (→ R4 22R → U2 → J2) |
| GPIO20 (pin11) | UART0_RX → J1-3 |
| GPIO21 (pin12) | UART0_TX → J1-2 |

---

## 4. Secondary errors — MEDIUM severity

Wrong but lower-blast-radius: mostly designator swaps that mislead bring-up/BOM cross-reference but don't by themselves break firmware logic.

| Location | What the docs say | The truth |
|---|---|---|
| LDO designator — system_overview, esp32cam_interface, power_architecture, bringup, **(appears throughout)** | AP2112K LDO is **U2** | AP2112K-3.3 LDO is **U1**. **U2 = USBLC6** ESD array. (U3=C3, U4=TPS2553, U5=switch.) |
| Alarm LED designator — mcu_interfaces, alarm_subsystem | Alarm LED is **D4**, resistor **R16** | LED is **D5** (XL-2012SURC); series resistor is **R15 330R**. **D4 = BAT54S** PIR clamp; **R16 = 10k** PIR bias. |
| Input TVS designator — power_architecture, bringup | USB input TVS is **D2** | VBUS input TVS (SMF5.0CA) is **D1**. **D2 = PSBD1DF40V1H Schottky 40V 1A** door clamp. |
| J1 pin order — mcu_interfaces | J1: 1=GND, 2=EN, 3=BOOT, 4=RX, 5=TX, 6=GND | J1: 1=GND, **2=UART0_TX (GPIO21), 3=UART0_RX (GPIO20), 4=BOOT_GPIO9, 5=ESP_EN**, 6=GND. |
| Decoupling table — power_architecture | C3=22µF bulk, C4=470nF, specific C5–C16 value map presented as fact | 22µF VBUS bulk is **C1** (not C3; C3 is a 100nF USB-data decoupler); 470nF = C2/C10 (C10=door debounce). The whole designator→value map is inconsistent with the schematic. |
| "5 V unregulated rail" — sensor_interfaces | 5 V pin is an "unregulated rail" | 5 V rail is filtered/protected (F1 PTC + FB1 ferrite + clamps); CAM_5V is additionally **gated/current-limited by U4**. |
| Sensor ESD "not implemented" — sensor_interfaces | No sensor-connector ESD on the PCB | **Partly implemented**: PIR line has **D6** (SMF5.0CA TVS); door has **D2** clamp + RC. |
| GPIO13–21 "unused" — mcu_interfaces | GPIO13–21 free for expansion | GPIO18/19 = native-USB D−/D+; GPIO20/21 = UART0 to J1. In use. |
| Power budget framing — power_architecture, application_node (CAM 200mA@3.3V row) | TPS2553 limit sized as whole-board USB limiter; CAM modeled at 3.3 V 200 mA | U4 gates only the CAM branch (ILIM R13); CAM is fed from **CAM_5V** (gated 5 V), peaks ~500 mA — not 200 mA on 3.3 V. F1 handles whole-board over-current. |
| A4 power rails — application_node | AP2112K 3.3 V rail supplies C3, CAM, sensors, indicators | 3.3 V rail supplies **C3 + U5 only**. CAM runs on **CAM_5V** + its own **CAM_3V3**; PIR + buzzer are on the **5 V rail**. |
| A3 / PIR "3.3 V logic" — application_node | PIR output is a clean 3.3 V logic signal | PIR is **5 V-powered**; signal reaches IO5 through a clamp/divider (R16/R18 100k + BAT54S D4). Qualify accordingly. |
| cam_protocol checksum example — cam_protocol.md line 152 | RESULT example "0xAA 0x02 0x82 0x01 0x80", checksum = 0x80 | `0x02 ^ 0x82 ^ 0x01 = **0x81**`. Frame should be `0xAA 0x02 0x82 0x01 **0x81**`. (Confirmed in file: line 152 prints `= 0x80`.) |

---

## 5. Design proposals (not wrong — but unverifiable, label them "proposed")

These are forward-looking firmware/host/protocol designs. **No firmware or host code exists** (every code dir holds only `.gitkeep`). Nothing in the hardware contradicts them; they are legitimate for a theory-stage note **as long as they are explicitly framed as proposed, not as-built.**

- **CAM UART wire protocol** (cam_protocol.md): frame `0xAA | LENGTH | CMD_ID | PAYLOAD | XOR-checksum`; commands 0x01/0x02/0x03; responses 0x81–0x84; result/status/error code tables; 115200 8N1. Internally consistent **except the one checksum-example arithmetic slip** (§4) and the leftover "let me recalculate" editorial stumble in the CMD_CAPTURE example (~lines 62–82) — clean both up so an implementer doesn't copy a bad literal.
- **FSM**: INIT→ARMED→TRIGGERED→VISION_CHECK→{CLEAR/ALARM}→ARMED, 5 s VISION_CHECK timeout → fallback ALARM, basic-alarm-without-CAM. Consistent across system_overview, firmware_architecture, alarm_subsystem, application_node.
- **Firmware structure / RTOS task layout / build-flash** (PlatformIO primary, ESP-IDF/Arduino alt). Explicitly "Proposed" — fine; only the "USB-UART bridge" wording in the Arduino step is wrong (§3).
- **Host tools** (config util, log viewer, firmware updater, serial monitor, dashboard). Fine — except the two U5 "toggle/select a module" claims, which are hardware-false (§3).
- **Power budget** magnitudes, debounce timings, baud/framing, Wi-Fi/BLE/MQTT/OTA options, vision workflow command names — all acceptable estimates/choices; relabel as proposed and align the CAM rail/load with reality.

---

## 6. Application note (`application_node.tex`) — remediation plan

**Good news first:** the note's **structure is exactly right** for a design deliverable — explicit assumptions table, requirements with traceability matrix, FSM, UART protocol (frame/checksum rule correct), and a theoretical verification plan. The *form* is publishable. The problem is that its **hardware facts were inherited from the wrong source docs** (`mcu_interfaces.md` + the U4/U5 misconceptions), so the technically-checkable content is wrong in the same places as everything else.

**Exact fixes to make in the `.tex`:**

1. **Pinout table (`tab:pinout`, lines ~293–302)** — replace with the authoritative map in §3:
   - BUZZER_DRV: GPIO7 → **GPIO10**.
   - CAM_TX/CAM_RX: GPIO8/GPIO9 → **IO1 (TX) / IO0 (RX)** (and fix the swap: IO0=RX, IO1=TX).
   - USB_TX/USB_RX: GPIO1/GPIO2 "through U5" → **GPIO21/GPIO20, direct to J1, not through U5**.
   - VBUS row: "current-limited by TPS2553" → **"over-current protected by PTC fuse F1; TPS2553 gates CAM_5V only."**
   - Keep DOOR_IN=GPIO4, PIR_IN=GPIO5, ALARM_LED=GPIO6 (correct).
2. **Assumption A2 (line ~158)** — "ESP32-CAM via UART1 (GPIO8/9)" → **"UART1 on IO0=RX / IO1=TX, isolated by U5 (CAM-presence-gated)."** Keep 115200.
3. **Assumption A4 (line ~160) + Power section (lines ~337–339)** — rewrite the rail tree: VBUS→**F1**→D1/C1→**FB1**→5 V rail → (U1 LDO ∥ U4 in parallel). State that the 3.3 V rail feeds **C3 + U5 only**; the CAM is on **CAM_5V (U4-gated)** + its own **CAM_3V3**; PIR + buzzer are on the **5 V rail**. Remove "TPS2553 distributes power [board-wide]."
4. **Assumption A6 (line ~162) + R6** — "USB-UART channel for programming" → **"native USB (USB-Serial/JTAG) on IO18/IO19; separate UART0 debug header J1 on GPIO20/21."** No bridge chip.
5. **U4 / U5 descriptions (power section + PCB checklist line ~396)** — U4 = **CAM-only power gate** (EN via IO7/CAM_EN, ILIM R13 21k5, default-OFF via R11). U5 = **presence-gated series isolation switch on the C3↔CAM UART1 link** (not a programming-path mux). Delete "verify U5 routing for programming-path selection."
6. **Block diagram (`fig:appnode-block`, lines ~258–262)** — buzzer node GPIO7 → **GPIO10**; CAM node GPIO8/9 → **IO0/IO1**; Host node "USB UART0" → **"native USB (IO18/19)"** with UART0/J1 shown as a separate debug header.
7. **Output drive (line ~372)** — "GPIO drives the MOSFET gate directly" → **"via R12 100R series, R14 10k pulldown."**
8. **Power budget (`tab:power`, line ~356)** — move CAM load to **CAM_5V (gated 5 V)** and use the ~500 mA peak the system docs cite, not 200 mA@3.3 V.
9. **Designators** — fix LDO **U1** (not U2), input TVS **D1**, alarm LED **D5**/**R15** anywhere they appear.
10. **Framing** — add an explicit banner: **"Design-stage / not yet fabricated; pin map verified against the schematic netlist."** The note (line ~181) lists `hardware/altium/` as populated source — it currently holds only a `.gitkeep`; relabel artifacts as planned where they don't yet exist.

**Important meta-fix:** `CLAUDE.md` itself encodes the same wrong canonical pin map (GPIO1/2 UART0 "through U5", GPIO7 buzzer, GPIO8/9 CAM UART, U5 "multiplexes the USB UART", TPS2553 as the 5 V switch) and points to `mcu_interfaces.md` as authoritative. **Correct `CLAUDE.md` and `mcu_interfaces.md` first** — they are the upstream that re-seeds every other doc and any future AI edits.

---

## 7. Open questions for the author

1. **Vision approach feasibility (scope this explicitly).** The docs assume the ESP32-CAM can "judge whether a face is covered (mask/helmet/balaclava)" and return a result over UART. If you intend a **pretrained model like YOLO** on the ESP32-CAM, flag the footprint reality: the ESP32 (OV2640 module) is heavily RAM/flash/compute constrained — full YOLO does not fit. Realistic options to evaluate and document as a **decision**: a tiny quantized classifier (e.g. TFLite-Micro / ESP-DL face-or-coverage classifier on small frames), the built-in face-detection primitives, or **offloading inference off-device** (CAM streams a frame; a host/edge does YOLO). The current note treats vision as a black box; pick an approach and write down its memory/latency budget. *(This is the biggest unscoped technical risk in the project.)*

2. **Diode designator split (D1/D3/D6).** BOM has exactly 2× PSBD1DF40V1H (Schottky 40 V 1 A) and 2× SMF5.0CA (TVS 5 V). Only **D2 = Schottky** (schematic prints "40 V 1 A") and **D5 = LED** are certain. The sensible-topology inference is **D1 = VBUS TVS, D3 = 5 V-rail Schottky, D6 = PIR-line TVS**, but per-designator package isn't printed. Confirm from a high-res render or the Altium source before publishing the diode table. **D4 = BAT54S is certain.**

3. **Per-designator R/C → MPN assignment.** The BOM is a value **catalog with no designator column**, so individual R#/C# → exact MPN is inferred from value class + net context. Load-bearing **values** are confirmed (R3/R4=22R, R8/R20/R21=1k, R9/R12=100R, 10k pull-ups, R11/R18/R19=100k, R13=21k5, R15=330R). The exact one-to-one split among same-value parts (e.g. which 100nF is which) is inference — say so, or add a designator column to the BOM.

4. **Sensor conditioning network polarity.** The door (R7 10k / R8 1k / C10 470nF / D2) and PIR (R16/R18 100k, C13/C17, D4, D6) networks' exact pull-up-vs-divider roles and active polarity are inferred from net topology, not annotated. Confirm active-low vs active-high before writing the ISR logic. (The GPIO↔net assignment IO4=door, IO5=PIR is certain.)

5. **BOM hygiene cleanup** (catalogued separately): the SMF5.0CA TVS is mislabeled "TVS 12V" (it's 5 V); the 100nF cap description says "0201" but the footprint is 0603; F1 has a stray "YTL " prefix on the MPN; J2 lists manufacturer EDAC but the MPN is Kinghelm; several parts use non-standard `MF=`/`MP=` keys or corrupted `… Pad Number` keys that drop out of standard exports; the YAGEO 100k links to the wrong (84.5 Ω) datasheet. Worth a pass before any real fabrication/quote.

6. **Connector MPNs.** Only J2 (Mini-USB) and J3 (ESP32-CAM) are catalogued. J1/J4/J5/J6/J7 pin counts/roles are read from schematic symbols, not a catalogued part — assign real connector parts before fabrication.

---

*Audit basis: schematic netlist (authoritative on disagreement), rendered schematic, and Altium LiveBOM. Where three independent passes disagreed, the netlist won.*
