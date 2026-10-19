# compactPi — Design Review (Schematic + Preliminary Layout)

**Project:** compactPi — CM5 carrier, now scoped as a general-purpose data-acquisition board
**Review date:** 2026-10-19
**Files reviewed:** `Design Files/*.kicad_sch` (rev 0, saved 2026-10-18), `compactPi.kicad_pcb` (saved 2026-10-19), `compactPi.kicad_pro`, README, JLCPCB stackup screenshot
**Scope change acknowledged:** autopilot sensors and the PCA9685 PWM block are removed (`sensors.kicad_sch` / `ap_io.kicad_sch` are now empty). This review treats the board as a CM5 DAQ carrier: power, USB-C PD, CAN, 2× UART, 2× I2C, 10/100 Ethernet, microSD, HDMI, USB host.

## Method

- Extracted the full netlist from the PCB (pad→net) and independently rebuilt connectivity from the schematic S-expressions (wires, junctions, labels, sheet pins, symbol pin transforms). The two agree pin-for-pin except for FB1/FB2 (no footprint, so absent from the PCB) and two connector shield pins (see M-08). **The PCB is in sync with the schematic — every finding below reflects the actual design state.**
- Checked key device behaviour against current datasheets: TLVM13630, TPS25751, LM74700-Q1, CM5 datasheet (RP-008180-DS-7), RP1 peripherals, MCP25625, WE 82400102.
- Layout is roughly 25 % routed (427 segments, 180 vias, 14 footprints still parked off-board), so layout comments are about floorplan, stackup and rules rather than finished routing.

## Bottom line

As drawn, the board would not power up correctly and several blocks could not work even if it did. Ten items are show-stoppers; most are net-naming / sheet-pin wiring slips left over from the restart, plus two component-value errors. None of them are architectural — the power architecture, part selection and connector scheme are sound and worth keeping. Fix the 🔴 list, re-run ERC, and the design is in good shape to finish layout.

**Highest-risk single item:** both 5 V bucks are set to **6.36 V** (C-01). The CM5's absolute-maximum 5 V input is 6.0 V. This one would destroy modules on first power-up.

---

## Finding summary

| ID | Area | Finding | Severity |
|---|---|---|---|
| C-01 | Power | TLVM13630 feedback dividers (53.6k/10k) give **6.36 V**, not 5 V (VFB = 1.0 V) | 🔴 Critical |
| C-02 | Power / top | Peripheral 5 V rail has **no source**: three spellings of the net (`5V0_PERIPHERAL`, `5V_PERIPHERAL`, `5V_Peripheral`) + FB2 dead-ends | 🔴 Critical |
| C-03 | Power | Core 5 V reaches the CM5 only through FB1, which has no footprint/value and is absent from the PCB | 🔴 Critical |
| C-04 | USB-C | CC1/CC2 are **not connected** to the TPS25751 (sheet pins marked No-Connect) — no PD, and a compliant source never enables VBUS | 🔴 Critical |
| C-05 | CAN | MCP25625 **VDD is on 5 V** → SPI/INT outputs drive 5 V into CM5 3.3 V GPIO; 3.3 V inputs are below VIH | 🔴 Critical |
| C-06 | CAN | MCP25625 ~RESET is pulled **down** (R23 → GND) — device permanently in reset | 🔴 Critical |
| C-07 | CM5 GPIO | SPI1 **MOSI/MISO swapped** (COPI on GPIO19, CIPO on GPIO20); same swap on the SPI0 labels | 🔴 Critical |
| C-08 | CAN | MCP25625 ~INT not connected to any GPIO (Linux `mcp251x` requires it) | 🔴 Critical |
| C-09 | CM5 | nRPIBOOT (pin 93) is wired to a label named `GND` — forces USB-boot mode every power-up | 🔴 Critical |
| C-10 | UART | UART1/UART2 connectors are not connected to any CM5 UART; pin 2/3 order is reversed vs Pixhawk convention | 🔴 Critical |
| M-01 | USB-C | TPS25751 ADCIN1/ADCIN2 floating (must be resistor-strapped to LDO_3V3); nI2Cc_IRQ needs pull-up; R1/R2 unvalued | 🟠 Major |
| M-02 | USB-C | TPS25751 VIN_3V3 unconnected; LDO_3V3/LDO_1V5/PPHV bypass far below datasheet minimums; PP5V on a dead net | 🟠 Major |
| M-03 | USB | ESD arrays U22/U23: GND (pin 2) and VDD (pin 5) unconnected — arrays are inert | 🟠 Major |
| M-04 | USB-C | R12 2.2 kΩ VBUS bleeder in 0402: 182 mW at 20 V PD (rated 63 mW) | 🟠 Major |
| M-05 | Power input | TVS clamp (~42 V) exceeds TLVM13630/FET 40 V abs-max; no fuse; unidirectional TVS conducts on reverse polarity | 🟠 Major |
| M-06 | Power | Peripheral rail enable is CM5 `VBUS_EN` (RP1 GPIO42, firmware-controlled USB-3 port power); no pull-down | 🟠 Major |
| M-07 | USB host | J11 VBUS has no current-limit switch | 🟠 Major |
| M-08 | Library | J15 and J14 shield pins don't map to footprint pads (S1/11 vs SH) — shields floating on the PCB | 🟠 Major |
| M-09 | CAN | Crystal Y1 has no value; C52/C53 = 22 pF likely wrong for a 2016 crystal; 47 pF on CLKOUT | 🟠 Major |
| M-10 | Layout | 14 parts unplaced, no thermal vias under U13/U14/U21, V_COMMON pour cut into the In4 GND reference, 0.3/0.2 vias, stackup ≠ JLC | 🟠 Major |
| m-01…m-16 | Various | BOM/value hygiene, cap voltage ratings, Ethernet notes, OTG_ID, J1 pin usage, docs | 🟡 Minor |
| S-01…S-07 | DAQ pivot | RTC battery, free UART/GPIO breakout, ADC/RS-485, LEDs, sink-only PD, always-on 3V3 | 🔵 Suggestion |
| G-01…G-10 | — | Things verified correct | ✅ Good |

---

## 🔴 Critical findings

### C-01 — Both 5 V rails are set to 6.36 V
**Sheet:** Power — R13/R8 (core), R10/R11 (peripheral)
TLVM13630 regulates FB to **1.0 V** (datasheet §7, VFB = 1.0 V typ). With RFBT = 53.6 kΩ and RFBB = 10 kΩ: VOUT = 1.0 × (1 + 53.6/10) = **6.36 V**. The 53.6k/10k pair is what you'd pick for a 0.8 V-reference part, so this looks like a carry-over from an earlier converter choice.
CM5 +5 V input: 4.75–5.25 V operating, **6.0 V absolute maximum**. The output caps (C11/C14/C20/C22, GRT32ER70J476 = 6.3 V) would also be over their rating.
**Fix:** RFBT = 40.2 kΩ, RFBB = 10 kΩ (TI Table 7-1 → 5.02 V). If you keep a series bead/trace drop to the CM5, 41.2 kΩ gives 5.12 V. Move the 47 µF output caps to 10 V parts (e.g. GRM32ER71A476KE15).

### C-02 — Peripheral 5 V rail has no source
**Sheets:** Power, compactPi (top), CM5_HighSpeed, autopilot_conn
Top-level sheet-pin wiring, as it actually resolves:

| Power-sheet pin | Top-level net it lands on | Consumers on that net |
|---|---|---|
| `5V0_PERIPHERAL` (U14 → FB2 → this pin) | `/5V_PERIPHERAL` | **nothing** |
| `5V_PERIPHERAL` (TPS25751 PP5V, via a *local* label of the same name) | not brought out | `/Power/5V_PERIPHERAL` island: U3 PP5V only |
| — | `/5V0_PERIPHERAL` | MCP25625 VDD/VDDA, J4, J6–J10 pin 1, C64–C67 — **no source** |
| — | `/5V_Peripheral` | J11 VBUS, U17 (HDMI 5 V switch) IN — **no source** |

KiCad labels are case-sensitive. *(Correction, re-review: a local label and a hierarchical label with the same text on one sheet **are** merged by KiCad — the original statement here was wrong; it did not change the conclusion for the wiring as it stood.)*
**Fix:** pick one name (`5V0_PERIPH`), use it for the Power sheet pin, both consumer-sheet pins and the top-level wires; delete the other two. Decide what PP5V should be (see M-02) rather than leaving it on a look-alike net.

### C-03 — Core 5 V path to the CM5 is open in the PCB
**Sheet:** Power — FB1 (and FB2)
`5V0_C_RAW` (U13 output) → FB1 → `5V0_CORE`. FB1/FB2 are `Device:FerriteBead_Small` with **no footprint and no value**, so "Update PCB from Schematic" silently dropped them: in the layout the CM5's six +5 V pins are on a net with nothing else on it.
Beyond the missing footprint: a ferrite bead in series with a 3 A rail feeding a module with its own bulk capacitance is a bad trade (DCR drop, saturation, LC resonance with the CM5 input caps). **Recommend deleting FB1/FB2** and connecting the rails directly. If you want them, specify a ≥ 6 A, < 5 mΩ bead and keep ≥ 47 µF on both sides.

### C-04 — USB-C CC lines are not connected to the PD controller
**Sheets:** compactPi (top), Power
Power-sheet pins `USB_CC1`/`USB_CC2` carry **No-Connect flags** at the top level; J15 CC1/CC2 sit on dangling labels `CC1`/`CC2`. With no Rd presented on CC, a USB-PD or Type-C-compliant source leaves VBUS at 0 V, so the USB-C input delivers nothing — not even 5 V — and no PD contract is possible.
**Fix:** wire J15.A5 → U3 CC1, J15.B5 → U3 CC2 (no external components needed; Rd/Rp are inside the TPS25751). Optionally J15 D± → U3 USB_P/USB_N too (BC1.2 detection; not required).

### C-05 — MCP25625 controller supply is 5 V; CM5 GPIO is 3.3 V
**Sheet:** autopilot_conn — U21 pin 3 (VDD) on `5V0_PERIPHERAL`
On the MCP25625, **VDD powers the CAN controller and its SPI/INT/RTS/BF pins** (2.7–5.5 V); **VDDA** powers the transceiver (4.5–5.5 V); **VIO** only sets the level of the TXD/RXD link between the two halves. With VDD = 5 V: SO, ~INT, CLKOUT drive 5 V into GPIO20/GPIO25-class pins (the CM5 GPIO is not 5 V tolerant), the CM5's 3.3 V SCK/SI/CS are below VIH (0.7 × VDD = 3.5 V), and the 3V3 pull-up on the RTS/BF pins is undefined-level.
**Fix:** VDD (pin 3) → 3V3, VIO (pin 11) → 3V3, VDDA (pin 19) → 5V0_PERIPH. Move C64/C66 (VDD decoupling) to the 3V3 node. This is exactly the Microchip MCP25625 click reference hookup.

### C-06 — ~RESET held low
**Sheet:** autopilot_conn — R23 10 kΩ from U21.2 (~RESET) to **GND**
Microchip's recommendation is a 10 kΩ pull-**up** to VDD (optionally an RC for POR). As drawn the controller never leaves reset. The hierarchical label `CAN_RST` isn't connected at the top level either, so nothing else drives the pin.
**Fix:** R23 to 3V3 (the new VDD). The Linux driver resets the device over SPI, so a GPIO is optional; if you want one, keep the pull-up and add a 1–10 kΩ series resistor per the datasheet.

### C-07 — SPI MOSI/MISO swapped at the CM5
**Sheet:** CM5_GPIO — hierarchical labels on GPIO19/20 (and GPIO9/10)
RP1 pin functions: **GPIO19 = SPI1_SIO[1] (MISO), GPIO20 = SPI1_SIO[0] (MOSI)**, GPIO21 = SCLK, GPIO18 = CE0, GPIO17 = CE1. The design puts `SPI_COPI1` on GPIO19 and `SPI_CIPO1` on GPIO20, so the MCP25625's SI pin is on the Pi's MISO and SO on the Pi's MOSI. The spare SPI0 labels have the same swap (`SPI_COPI0` on GPIO9 = MISO, `SPI_CIPO0` on GPIO10 = MOSI).
**Fix:** swap the two hierarchical labels on each bus in `CM5_GPIO.kicad_sch`. CE assignments are correct.

### C-08 — MCP25625 interrupt goes nowhere
**Sheets:** autopilot_conn, top
`SPI_CAN_INT_3V3` exists as a sheet pin but is not wired to anything at the top level; U21.25 (~INT) is a single-pin net. `mcp251x` will not probe without an interrupt line.
**Fix:** route to a free GPIO — GPIO25 is unused and is the default `interrupt=` in the stock `mcp2515` overlays. Example overlay once wired: `dtoverlay=mcp2515,spi1-0,oscillator=<Y1 Hz>,interrupt=25` (use `spi1-1cs` for the bus).

### C-09 — nRPIBOOT tied to ground
**Sheet:** CM5_GPIO — Module1 pin 93
The CM5IO reference's "Force RPIBOOT" jumper was deleted but its wire survived: pin 93 is wired to a *local* label named `GND` (and the PCB netlist has pin 93 on the GND net). nRPIBOOT has a 10 kΩ internal pull-up; low at boot forces USB-device boot and bypasses eMMC/SD, so the module would never boot normally.
**Fix:** delete the wire/label. If you want the function, add a 2-pin 2.54 mm header or a tactile switch to GND (the note "Jumper pins 1-2" on the sheet refers to a part that no longer exists).

### C-10 — UART connectors are not connected to the CM5
**Sheets:** autopilot_conn, top, CM5_GPIO
`UART1_RX_In/TX_Out` and `UART2_RX_In/TX_Out` are sheet pins with nothing attached at the top level; J4/J10 pins 2–3 are single-pin nets. The CM5_GPIO sheet has labels for GPIO14/15 (UART0), GPIO4/5 (UART2) and GPIO12/13 (UART4) ready to use — they just aren't brought out.
Also check the connector convention: on the Pixhawk/Dronecode 6-pin UART standard, pin 2 is **TX (out of the host)** and pin 3 is **RX (into the host)**; this design has pin 2 = RX_In, pin 3 = TX_Out, which crosses straight cables to any Pixhawk-style peripheral (GPS, telemetry radios).
**Fix:** UART0 (GPIO14/15) → J4, UART2 (GPIO4/5) or UART4 (GPIO12/13) → J10, with pin 2 = TX_Out, pin 3 = RX_In. Consider 33–100 Ω series resistors and ESD on the lines.

---

## 🟠 Major findings

### M-01 — TPS25751 configuration straps and I2C housekeeping
- **ADCIN1/ADCIN2 (pins 2, 3) are floating.** The datasheet requires both to be tied to LDO_3V3 through a resistive divider (Table 8-2 decodes the ratio: ~0 → tie to GND, ~0.54 → tie to LDO_1V5, ~1.0 → tie to LDO_3V3). The decoded values select I2C target address, dead-battery sink-path behaviour and default configuration. Generate the required values with TI's Application Customization Tool for your EEPROM image and strap accordingly.
- **nI2Cc_IRQ (pin 18)** should have a pull-up; the datasheet explicitly says not to ground it, and floating isn't documented as safe.
- **R1/R2 (I2Cc pull-ups) are still unvalued** (`R_US`). 4.7 kΩ to LDO_3V3 is typical.
- U3 has an empty Value field (BOM will show a blank line).

### M-02 — TPS25751 supply/bypass network
- **VIN_3V3 (pin 38) has only C8 on it.** TI's intended configuration is an always-present 3.3 V. Without it the device can only start in dead-battery mode from VBUS via its internal LDO, and the datasheet says the host must clear the DB flag over I2C — but I2Ct (pins 8/9) isn't connected to the CM5 either, so it stays in DB mode. Recommend a small always-on 3.3 V LDO from V_COMMON (also useful for S-06), or at minimum confirm with TI that EEPROM-based PD negotiation is complete in DB mode for your config.
- **LDO_3V3** needs 5–25 µF (C25+C34 = 0.2 µF). **LDO_1V5** needs 4.5–12 µF (C27 = 0.1 µF). **PPHV** recommends 47–100 µF (V_USB has 0.3 µF before the ideal diode).
- **PP5V (pins 34/35)** is the 5 V *source* supply for VBUS/VCONN. If the port is sink-only, terminate it per TI's unused-pin guidance (not on a dangling net). If you want DRP (compactPi powering a device), feed it from 5V0_PERIPH through its own ≥ 10 µF and accept that a plugged-in laptop may negotiate to be powered *by* compactPi.
- I2Ct unconnected means the CM5 can't read contract/voltage or control the port. For a DAQ board, an I2C link to I2C0/SCL0 (pins 80/82, currently unused) is cheap and useful (also lets you log input power).

### M-03 — USB ESD arrays have no ground
**Sheets:** top (U22), CM5_HighSpeed (U23)
WE-TVS 82400102 in SOT-23-6: pins 1↔6 and 3↔4 are the flow-through I/O pairs (wired correctly), **pin 2 = GND, pin 5 = VDD**. Both GND and VDD pins are marked No-Connect on U22 and U23, so neither array can clamp anything.
**Fix:** pin 2 → GND; pin 5 → the port's 5 V (VBUS for U22 — note VBUS can be 20 V under PD, which exceeds the array's 6 V VDD rating, so for U22 leave pin 5 open and rely on the GND path, or use a part rated for the data lines only) or open per WE guidance for U23.

### M-04 — VBUS bleeder resistor
**Sheet:** top — R12 2.2 kΩ 0402, VBUS → GND ("needed by some Type-C PSUs")
At 20 V PD: 9 mA, **182 mW** in a 63 mW package; at 15 V: 102 mW. The TPS25751 implements VBUS discharge internally; the connector-side capacitance is only C5+C7 ≈ 4.8 µF. Delete R12, or if you have evidence a specific PSU needs it, use ≥ 10 kΩ in 0805/1206.

### M-05 — Input protection chain doesn't close
**Sheet:** Power — D1/D2 SMAJ26A, U4–U9 LM74700/CSD18511Q5A, U13/U14
- SMAJ26A: 26 V stand-off, 28.9 V min breakdown, **~42 V clamp** at rated surge. Downstream: TLVM13630 (36 V operating / **40 V abs max**), CSD18511Q5A (**40 V**). A real surge is clamped above the converters' ratings; only the LM74700 (65 V) survives on paper.
- A fully charged 6S pack is 25.2 V against a 26 V stand-off — leakage will rise warm; 7S is out.
- No fuse anywhere. A reversed battery forward-biases the unidirectional TVS; without a fuse the TVS and trace are the fuse. The LM74700/FET pair blocks reverse voltage fine on its own — the TVS is the weak link here.
- The schematic's own ToDo list still has "Power input protection / simulate input protection" open.
**Options:** (a) fuse per input (SMD or specify harness fuse in docs) + keep TVS; (b) replace the ideal-diode controller on the battery inputs with an eFuse that has OVP (TPS2663, LM5069) and set OVP ≈ 30 V; (c) keep the architecture and choose a buck with ≥ 60 V abs-max. The bulk input caps (CGA5L1X7R2A475K, 100 V) are fine.

### M-06 — Peripheral rail enable is `VBUS_EN`
**Sheets:** top, Power — U14 EN ← Module1 pin 111
Per the CM5 datasheet, pin 111 (VBUS_EN) is a 3.3 V output from **RP1 GPIO42** meant for the USB-3 port power switch, and software decides when it asserts. The schematic note says the intent is "peripherals boot once 5V0_CORE is stable & CM5 enables", which is reasonable sequencing — but it ties *all* of CAN, I2C, UART VCC, HDMI 5 V and USB host power to a pin whose default, shutdown and reboot behaviour is owned by the bootloader/OS, and that goes low in halt (which may or may not be what you want for a logger). There is also no pull-down, so EN floats until RP1 drives it.
**Recommend:** 100 kΩ pull-down on PERIPHERAL_EN; either keep VBUS_EN deliberately (and document the config.txt/overlay requirement), or use a dedicated GPIO (GPIO23/24/26 are free) with a pull-up for default-on, or simply derive it from U13's PG with an RC delay.

### M-07 — USB host port power
J11 pin 1 is wired straight to the peripheral rail (once C-02 is fixed). USB host ports should have a current-limited switch (another RT9742 1 A, TPS2553, or similar) so a shorted cable doesn't take the CAN transceiver and every JST peripheral down with it. U17 already does this for HDMI 5 V; replicate it.

### M-08 — Connector shields floating in the PCB
`Connector:USB_C_Receptacle_USB2.0_16P` has shield pin **S1**; the GCT USB4105 footprint names its shell pads **SH** → no net on the four shell pads in the PCB. Same for the microSD symbol (pin **11**) vs Molex 104031 footprint (pad **SH**). The schematic shows them on GND; the board doesn't. Fix by editing the symbol pin numbers (or footprint pad names) to match; HDMI J2 is already correct.

### M-09 — CAN oscillator
- Y1 has no value or MPN. Choose 16 MHz or 20 MHz (both standard for `mcp251x`).
- C52/C53 = 22 pF came from the ceramic-resonator table. A 2016 crystal is typically CL = 8–12 pF → load caps ≈ 2·CL − C_stray ≈ 12–18 pF. Pick from the crystal datasheet.
- C68 (47 pF on CLKOUT) does nothing useful; CLKOUT is active at /8 after reset — leave it open or disable in software.
- ~RX0BF/~RX1BF are push-pull outputs when enabled; tying them to the RTS pull-up net is harmless only while they stay disabled. Leave them open.

### M-10 — Layout state and rules
- **Unplaced (parked at x > 150 mm):** R5, R6, R12, R19, R20, R21, C19, C21, C26, C54, C56, D4, NT1, U23. C19/C21/C26 are the 100 nF VIN bypasses for U13/U14 and must sit at the VIN pins; U23 must sit at J11.
- **No thermal vias** yet under U13/U14 (30-pin module, ~2 W loss each at 25 V in/3 A) or U21's EP.
- **V_COMMON pour on In4** (113–137 × 71–90 mm) sits on the layer that is the GND reference for B.Cu and In3 under the CM5. Move it to In2/In3 so In1 and In4 stay solid GND.
- **Vias:** 22 vias are 0.3 mm / 0.2 mm drill (0.05 mm annulus) — below the project's own rule (0.45 mm dia, 0.1 mm annulus) and below JLC's 6-layer minimum. Let DRC flag them and replace with 0.45/0.2 (or 0.4/0.2 if you confirm with JLC).
- **Stackup mismatch:** `compactPi.kicad_pcb` carries the MacroFab 6-layer (1 oz inner, 0.23/0.38/0.15/0.38/0.23 mm) while the netclass widths (0.1509 / 0.1537 / 0.1191 / 0.1029 mm) are from **JLC06161H-3313A** (0.5 oz inner, 3313 0.0994 mm prepreg, 0.55 mm cores). Two consequences: (1) KiCad's length/impedance tooling is wrong, (2) inner-layer current capacity is half what the file implies — the V_BATT/V_AUX/VBUS pours on In2 carry up to ~5 A. (3) JLC's own page labels **3313A as "duplicate stackup / do not select / to be cancelled"** — move to 3313 (or B/C) and recompute widths before routing more impedance-controlled traces.
- Copper-to-edge clearance is set to 0.01 mm; JLC wants ≥ 0.2 mm (0.3 mm safer for routed edges).
- Board outline is 47 × 105 mm; README says 45 × 105.

---

## 🟡 Minor findings

- **m-01** C46/C47: value 6.8 µF / 1812 footprint, but SPN `CGA5L1X7R2A475K160AC` is 4.7 µF 1206 100 V. One of them is wrong.
- **m-02** C11/C14/C20/C22 (`GRT32ER70J476`) are 6.3 V on a 5 V rail — weak DC-bias derating, and over-voltage until C-01 is fixed. Use 10 V.
- **m-03** C66/C67 are 10 µF in **0201** — not realistically available above 4 V. Use 0603.
- **m-04** TLVM13630 RT = 25.5 kΩ ≈ 520 kHz (schematic note says ~400 kHz; TI's table suggests 1 MHz / 13.0 kΩ for 5 V). Pick deliberately — ripple current in the fixed internal inductor scales with 1/f. PG tied to GND is allowed but NC is cleaner; consider a UVLO divider on U13 EN (e.g. start at ~5.8 V) instead of the plain 10 k to VIN so a sagging battery doesn't chatter.
- **m-05** Input bulk: datasheet suggests 47–100 µF electrolytic/polymer per converter input when fed by long cables (vehicle harness qualifies). 4× 4.7 µF ceramic across two converters is the bare minimum.
- **m-06** Ethernet: with pairs 2/3 open, the BCM54210 will advertise 1000BASE-T and rely on downshift. Set `max-speed = <100>` (or `ethtool -s eth0 advertise 0x0f`) in the overlay. Center taps C_RD/C_TD share one 100 nF — give each its own. The J5 pinout (1 TX+, 2 RX+, 3 TX−, 4 RX−) needs documenting for the mating cable. TRD1_RX pair has 2.9 mm intra-pair mismatch; USB2a pair 1.6 mm — fine electrically, but tidy to < 0.5 mm.
- **m-07** USB_OTG_ID floating → the USB-C port is device-only. Fine for rpiboot/gadget; add a solder jumper to GND if host mode on that port is ever wanted (`dtoverlay=dwc2,dr_mode=host`).
- **m-08** AT24CS64 A0–A2 and WP floating. Tie to GND explicitly (address 0x50, write-enabled) rather than relying on internal pull-downs.
- **m-09** J1 (Molex Nano-Fit 105310-1208): one pin each for V_BATT and V_AUX, pins 2/3/6/7 unused. Parallel them (6.5 A/pin rating, but you're at ~5 A worst case at 6 V in). Also check the right-angle plug's exit direction against J15/J2 in 3D — J1 sits 8 mm behind the USB-C.
- **m-10** SMAJ26A symbol pins are named A1/A2 (KiCad quirk for this unidirectional part). Pad 1 = cathode on D_SMA, which is on the positive rail here — correct, but worth a note for assembly checks.
- **m-11** No ESD on HDMI TMDS/DDC/HPD, none on the UART/I2C/CAN-connector 5 V pins, none on USB-C VBUS/CC. For a marine box with external cables, budget a TPD4E05U06-class array per connector group.
- **m-12** Housekeeping: title block still "Compact Pi AutoPilot", sheet "Autopilot Interfaces", nets `I2C_PWM_*`, `SPI_INT_*`; empty `sensors`/`ap_io` sheets and `CORE_EN`/`USB_DP`/`USB_DM` pins can go. No silkscreen board name/rev yet. Several top-level labels (`SPI_INT_*`, `I2C_INT_*`, `I2C_PWM_*`, `SPI_CAN_CE1`) are single-pin — fine as placeholders but ERC will list them; run ERC and clear it before more layout.
- **m-13** U17 footprint is a 3-pin TSOT-23; confirm RT9742**SN** is the fixed-enable 3-pin variant you intend (U1 is the 5-pin enable version).
- **m-14** The CM5 wireless variant's antenna sits at a module corner — keep connectors/copper clear of it on whichever corner lands near J1/J15.
- **m-15** Core rail budget: TLVM13630 is 3 A continuous / 4.9 A OCP. With USB and HDMI power on the peripheral rail that's adequate for a CM5 (~2–2.5 A worst case) but not generous; don't move anything else onto it.
- **m-16** SD_PWR_ON has a 10 k pull-up (R4) so the card is powered before the CM5 asserts it. Harmless, but the CM5 datasheet notes the power switch is the only way to reset the card — a pull-down would let firmware do a clean cycle. CM5IO uses a pull-down here.

---

## 🔵 Suggestions for the DAQ scope

- **S-01 RTC backup.** VBAT (pin 76) is unconnected. A logger without a battery-backed RTC time-stamps every boot wrong until NTP/GPS. Add an ML1220 (CM5 trickle-charges it via `rtc_bbat_vchg`) or CR1220 holder.
- **S-02 Use the freed GPIO.** Beyond C-10's two UARTs, UART4 (GPIO12/13), SPI0 (GPIO7–11), I2C0 (SCL0/SDA0) and GPIO16/22/23/24/26/27 are all free. A 2×8 or 2×10 header (or a third/fourth JST-GH) plus a dedicated **PPS input** (GPS time sync) costs nothing now and is painful to add later.
- **S-03 Analog / serial instruments.** SPI0 is a natural home for a precision ADC (ADS131M04/ADS1256 class). An isolated RS-485 transceiver on one UART covers NMEA 0183 and most marine sensors; the CAN port already covers NMEA 2000 electrically (document that it's unisolated and uses JST-GH, not Micro-C).
- **S-04 Indicators.** LED_nACT (pin 21) is free — add an activity LED next to the PWR LED; a CAN activity LED on ~RX0BF is a one-resistor addition.
- **S-05 Decide the USB-C role.** Sink-only (recommended for a DAQ box) simplifies M-02: PP5V terminated per TI, no VBUS sourcing, OTG_ID default device for rpiboot.
- **S-06 Always-on 3.3 V LDO** from V_COMMON (≈ 50 mA): powers TPS25751 VIN_3V3, could hold the MCP25625/RTC/wake logic, and decouples the PD controller from CM5 state.
- **S-07 Leak sensor input** was in the original README and is still useful in a wet enclosure — one GPIO with pull-up and ESD on a 2-pin GH.

---

## ✅ Verified correct

- **G-01** Ideal-diode ORing: LM74700 VCAP caps are between VCAP and ANODE (C9/C13/C23), EN tied to ANODE (allowed), FET source on the input/drain on V_COMMON — correct orientation; reverse-input blocking to −65 V.
- **G-02** TLVM13630 VLDOIN → VOUT, VCC 1 µF, AGND/PGND, DNC pins open, SW pin open — all per datasheet.
- **G-03** CM5 GPIO_VREF tied to +3V3 via net-tie; +3V3 load (SD switch, CAN VIO, LED) is well inside the 600 mA budget; PMIC_ENABLE/PWR_BUT/EEPROM_nWP/WL_nDisable/BT_nDisable/SD_VDD_Override left floating is correct per datasheet.
- **G-04** HDMI: DDC pull-ups and HPD level handling are on-module (1.8 kΩ, 5 V-tolerant HPD) — the direct connection is right; micro-HDMI pinout and pair mapping check out.
- **G-05** I2C1 (GPIO2/3) has 1.8 kΩ on-module pull-ups to GPIO_VREF — no external pull-ups needed on J6/J9.
- **G-06** CAN physical layer: common-mode choke, SZNUP2105L TVS, DNP 120 Ω termination, two daisy-chain connectors with the Dronecode 4-pin order (5 V, CAN_H, CAN_L, GND). STBY pulled low (normal mode).
- **G-07** Ethernet Bob-Smith termination (2× 75 Ω + 1 nF) and PHY-side CT decoupling present; pair mapping is self-consistent through the transformer.
- **G-08** USB-C: D+/D− on A6/A7 and B6/B7 tied (required), VBUS to TPS25751 VBUS_IN/VBUS with ≤ 10 µF connector-side capacitance, DRAIN pins commoned.
- **G-09** Decoupling density on the CM5 side, SD power switching from SD_PWR_ON, PWR LED via LED_nPWR open-drain — all fine.
- **G-10** Netclass widths match JLC's calculator for the chosen stackup: 50 Ω SE 0.1509, 90 Ω diff 0.1537/0.2032, 100 Ω diff 0.1191/0.2032 (L1) and 0.1029/0.2032 (inner).

---

## Suggested order of work

1. Fix the ten 🔴 items in the schematic (most are label renames and three resistor/pin changes), run ERC, update PCB.
2. Resolve M-01/M-02 with TI's Application Customization Tool open (the ADCIN straps and EEPROM image come out of the same session); decide sink-only vs DRP.
3. Close the input-protection decision (M-05) — it determines whether the ideal-diode stage stays as-is.
4. Switch the stackup to the JLC variant you'll actually order, re-check impedance widths and the inner-layer copper, then finish placement (thermal vias, parked parts, pours off In1/In4).
5. BOM pass: every Value/MPN populated (R1, R2, U3, FB1/2, Y1), cap voltage ratings, C46/C47.
6. Update README (scope, 47 mm, interface table, J5 pinout) and retire the autopilot naming.

## Re-review — commit 20c4914 "some stuff fixed" (2026-10-19 PM)

Re-extracted both netlists; PCB and schematic are in sync. `CM5_GPIO.kicad_sch` and `CM5_HighSpeed.kicad_sch` are unchanged since the first review, so every finding located on those sheets is still open.

| ID | Status | Evidence in current files |
|---|---|---|
| C-01 | ❌ Open | R10 / R13 still 53.6 kΩ over 10 kΩ → **6.36 V** on both rails |
| C-02 | ⚠️ Partial | `/5V0_PERIPHERAL` now sourced by U14→FB2 (CAN, J4, J6–J10 powered). `/5V_Peripheral` (J11 USB-host VBUS, U17 HDMI 5 V) still has **no source**. Power sheet still has an orphan `5V_PERIPHERAL` pin. |
| C-03 | ⚠️ Partial | FB1/FB2 now have 0805 footprints and are in the PCB, so the core path is closed. Still no value/MPN; an 0805 bead must be rated ≥ 5 A with low DCR for a 3 A CM5 rail — 0 Ω or a direct connection remains the safer choice. |
| C-04 | ✅ Fixed | J15 CC1/CC2 → U3 CC1/CC2 |
| C-05 | ❌ Open | U21 VDD (pin 3) still on 5 V; only VIO is on 3V3 |
| C-06 | ❌ Open (changed) | R27 10 k to 3V3 added, but R23 10 k to GND is still fitted → ~RESET sits at **1.65 V**, below VIH, so the part stays in reset. Remove R23. R27 is also still parked off-board. |
| C-07 | ❌ Open | GPIO19 → U21 SI, GPIO20 → U21 SO (still swapped) |
| C-08 | ❌ Open | U21 ~INT is a single-pin net |
| C-09 | ❌ Open | Module1 pin 93 (nRPIBOOT) still on GND |
| C-10 | ❌ Open | J4 / J10 pins 2–3 still single-pin nets |

Changes outside the critical list:
- **M-05 (partial):** F1/F2 added ahead of the TVS diodes. Reversed polarity now opens a fuse instead of burning the TVS. Two caveats: the value "10A" has no MPN, and 10 A is above the Nano-Fit per-contact rating (~6.5 A). Pick a specific 1206 part and size it to the connector/copper (roughly 5–7 A, slow-blow for the input-cap inrush). The TVS clamp above 40 V is still open.
- **PP5V now tied to the peripheral rail** (via the label merge). This makes the TPS25751 capable of *sourcing* 5 V out of the USB-C port, if its configuration allows the source role. Decide deliberately (S-05); if sink-only, configure it that way in the EEPROM image. PP5V also needs ≥ 10 µF at the pin.
- Still open: M-01 (ADCIN floating, R1/R2 unvalued), M-03 (U22/U23 GND pins NC), M-04 (R12), M-06, M-07, M-08 (J14/J15 shield pads still no-net), M-09, and M-10 (C19/C21/C26, R5/R6/R12/R19–R21, C54/C56, D4, NT1, R27 still parked; V_COMMON pour still on In4; 0.3/0.2 vias still present; MacroFab stackup still in the board file). U23 is now placed.

## Sources

- [TLVM13630 datasheet](https://www.ti.com/lit/ds/symlink/tlvm13630.pdf) — VFB 1.0 V, RT table, 5 V divider, abs-max
- [TPS25751 datasheet](https://www.ti.com/lit/ds/symlink/tps25751.pdf) — ADCIN straps, VIN_3V3, PP5V/PPHV, bypass minimums
- [LM74700-Q1 datasheet](https://www.ti.com/lit/ds/symlink/lm74700-q1.pdf) — VCAP, EN, reverse voltage
- [CM5 datasheet RP-008180-DS-7](https://pip-assets.raspberrypi.com/categories/944-raspberry-pi-compute-module-5/documents/RP-008180-DS-7-cm5-datasheet.pdf) — pin 93/111, 5 V limits, internal pulls
- [RP1 peripherals](https://pip-assets.raspberrypi.com/categories/892-raspberry-pi-5/documents/RP-008370-DS-1-rp1-peripherals.pdf) and [pinout.xyz SPI](https://pinout.xyz/pinout/spi) — SPI1 SIO0/SIO1 mapping
- [MCP25625 datasheet](https://static.chipdip.ru/lib/661/DOC028661833.pdf) — pinout, ~RESET, VDD/VDDA/VIO, oscillator
- [WE 82400102 datasheet](https://www.we-online.com/components/products/datasheet/82400102.pdf) — SOT-23-6 pin functions
- [RT9742GGJ5 (DigiKey)](https://azcus.digikey.com/en/products/detail/richtek-usa-inc/RT9742GGJ5/5880519) — 1 A power switch
