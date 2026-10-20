# compactPi — Design Review (consolidated, rev 3)

**Project:** compactPi — CM5 carrier, scoped as a general-purpose marine data-acquisition board
**Review date:** 2026-10-19 (evening refresh, supersedes the morning review and both re-review addenda)
**Files reviewed:** working tree on top of commit `f49593c` — all `*.kicad_sch` saved 19:54, `compactPi.kicad_pcb` saved 19:58, `compactPi.kicad_pro`
**Companion files:** `Firmware/devicetree/compactpi-overlay.dts`, `config-compactpi.txt`

## Method

- Rebuilt connectivity from the schematic S-expressions (wires, junctions, labels, sheet pins, symbol pin transforms) and extracted the pad→net map from the PCB. **They now agree pin-for-pin** — the PCB has been updated from the schematic since the last pass.
- Device behaviour checked against current datasheets: TLVM13630, TPS25751 (SLVSH93A incl. §8.4.1 pin-strap table), LM74700-Q1, CM5 datasheet (RP-008180-DS-7), RP1 peripherals, MCP25625, WE 82400102.
- Correction carried forward from the first pass: KiCad **does** merge a local label and a hierarchical label with the same name on one sheet. An earlier version of C-02 said otherwise.

## Bottom line

All ten original critical findings except the UART wiring are closed in the schematic, and the board now matches the schematic. **What's left before fab is mostly BOM integrity, a few ESD and shield connections, and layout completion.** One item is still capable of destroying hardware: the 20 V USB-PD rail capacitors (K-1) carry 6.3 V/100 nF part numbers even though their Value fields were updated.

| Category | Open | Closed this session |
|---|---|---|
| 🔴 Critical | 2 (K-1 BOM on 20 V rail, K-2 UARTs) | C-01…C-09, N-01 value |
| 🟠 Major | 7 | M-01, M-02 (mostly), M-04, M-05 (partial), M-06 (partial) |
| 🟡 Minor | 9 | several |

---

## Status of original critical findings

| ID | Finding | Status | Evidence |
|---|---|---|---|
| C-01 | 5 V bucks at 6.36 V | ✅ Fixed | R10/R13 = 39.2 kΩ → **4.92 V**. See m-01 for margin. |
| C-02 | Peripheral 5 V rail unsourced | ✅ Fixed | One `5V0_PERIPHERAL` net: U14→FB2 feeds U21 VDDA, J4/J6–J10, J11, U17, U3 PP5V |
| C-03 | FB1 missing from PCB | ⚠️ Partial | In PCB (0805), but FB1/FB2 still have no value/MPN → see M-05 |
| C-04 | USB-C CC unconnected | ✅ Fixed | J15 CC1/CC2 → U3 CC1/CC2 |
| C-05 | MCP25625 VDD on 5 V | ✅ Fixed | VDD/VIO on 3V3, VDDA on 5 V |
| C-06 | ~RESET held low | ✅ Fixed | R23 DNP, R27 10 k to 3V3 |
| C-07 | SPI MOSI/MISO swapped | ✅ Fixed | GPIO19 ← U21 SO, GPIO20 → U21 SI; SPI0 labels also corrected |
| C-08 | CAN ~INT unconnected | ✅ Fixed | U21 ~INT → GPIO25 |
| C-09 | nRPIBOOT tied to GND | ✅ Fixed | Floating on internal pull-up |
| C-10 | UARTs not wired | ❌ Open → **K-2** | Pin order fixed; still no CM5 connection |

---

## 🔴 Critical — open

### K-1 — Part numbers on the 20 V `V_USB` rail are still 6.3 V / 100 nF parts
`V_USB` (TPS25751 PPHV → U8 ideal diode) reaches **20 V** under a PD contract.

| Ref | Value field | SPN field | Problem |
|---|---|---|---|
| C35 | 22 µF | `GRT32ER70J476KE13` | 47 µF **6.3 V** |
| C37 | 22 µF | `GRT32ER70J476KE13` | 47 µF **6.3 V** |
| C6 | 10 µF | `AGC0603X7R101-104K` | 100 nF part; a 10 µF 0603 must also be ≥ 25 V |

If the BOM is built from the SPN column, a 6.3 V MLCC goes on a 20 V rail and fails short across the USB-C input. Replace with ≥ 25 V parts (35 V preferred for derating): e.g. 22 µF 1210 35 V for C35/C37, 10 µF 0603/0805 25 V for C6. The same SPN-copy pattern shows up on C25/C27 (fixed) and C46/C47 (m-02) — a full BOM pass is worthwhile.

### K-2 — UART connectors still not connected to the CM5
J4 and J10 pins 2/3 are single-pin nets (`UART1_TX_Out`, `UART1_RX_In`, `UART2_TX_Out`, `UART2_RX_In`). The pin order is now Pixhawk-correct (pin 2 = TX out, pin 3 = RX in).
**Fix:** J4 → UART0 (GPIO14 TX / GPIO15 RX), J10 → UART2 (GPIO4 TX / GPIO5 RX) or UART4 (GPIO12/13). The device-tree `config-compactpi.txt` already has the matching `uart0-pi5` / `uart2-pi5` lines commented out ready to enable. Consider 33–100 Ω series resistors.

---

## 🟠 Major — open

### M-03 — USB ESD arrays still have no ground
U22 (USB-C data) and U23 (J11 host data): pin 2 (GND) and pin 5 (VDD) are both unconnected, so neither clamps anything. Connect pin 2 to GND on both. Pin 5: leave open on U22 (VBUS can be 20 V, above the part's 6 V VDD rating); on U23 it can go to `5V0_PERIPHERAL` or stay open per WE guidance.

### M-05 — Input protection and fuses
- F1/F2 ("10A", 1206) have no MPN, and 10 A exceeds the Nano-Fit per-contact rating (~6.5 A). Size to roughly 5–7 A slow-blow (the input caps produce an inrush spike).
- TVS clamp (SMAJ26A, ~42 V at rated surge) is still above the TLVM13630 and CSD18511Q5A 40 V abs-max. Options unchanged: eFuse with OVP, or a lower-clamp TVS with a stand-off that still clears your highest battery voltage.
- FB1/FB2 have no value/MPN. In series with 3 A rails an 0805 bead must be ≥ 5 A with single-digit mΩ DCR; a 0 Ω jumper or a direct connection is the lower-risk choice.

### M-08 — USB-C and microSD shields are not connected in the PCB
The schematic puts J15 `S1` and J14 pin `11` on GND, but the footprints name those pads `SH`, so the PCB shows the shell pads with **no net**. Either rename the symbol pins to `SH` or the footprint pads to `S1`/`11`.

### M-09 — CAN crystal
Y1 still has no value or MPN (the device-tree overlay assumes 16 MHz). C52/C53 = 22 pF came from the ceramic-resonator table; a 2016 crystal typically wants 12–18 pF depending on its CL. C68 (47 pF on CLKOUT) does nothing useful.

### M-11 (new) — 2.2 kΩ pull-ups added on HDMI DDC
R5/R6 now pull `HDMI0_SDA` / `HDMI0_SCL` to 3V3. The CM5 datasheet says the DDC lines already have **1.8 kΩ pull-ups on the module** and can connect directly to the HDMI connector. With both fitted the effective pull-up is ~1 kΩ, which exceeds the 3 mA I2C sink spec, and the two pull-ups may sit on different rails. R5/R6 are also currently parked off-board. **Recommend deleting them (or marking DNP).**

### M-06 — Peripheral rail enable
R18 now pulls PERIPHERAL_EN low ✅. The enable is still CM5 `VBUS_EN` (RP1 GPIO42), so the peripheral rail depends on how the firmware/kernel drives that pin. The device-tree overlay describes it as a fixed regulator, but check on hardware (`pinctrl get 42`) that the stock CM5 tree doesn't already own it.

### M-10 — Layout
- **17 parts still placed off-board:** C19, C21, C26 (U13/U14 VIN bypass — must sit at the VIN pins), C54, C56, C69, D4, R5, R6, R18, R19, R20, R21, R22, R27, R28, R29.
- **No thermal vias** under U14 or U21's exposed pad; U13 has only three V_COMMON vias nearby. Each TLVM13630 dissipates ~2 W at 25 V in / 3 A.
- **In4 now carries three power pours** (`USB_LDO_3V3`, `V_COMMON`, `5V0_CORE`) on the layer that is the GND reference for B.Cu and In3. Keep In1 and In4 as solid GND; move power to In2/In3.
- **61 vias are 0.3 mm / 0.2 mm** (0.05 mm annulus), 27 of them on `5V0_CORE`. That's below the project's own 0.45 mm / 0.1 mm-annulus rule and below JLC's 6-layer minimum; replace with 0.45/0.2.
- **Stackup:** the board file still carries the MacroFab 6-layer stackup while the netclass widths come from JLC `3313A` — which JLC itself marks as a duplicate stackup to be cancelled. Pick the JLC stackup you'll order, re-enter it, and re-check impedance widths before routing more of the HDMI/USB/Ethernet pairs.
- Copper-to-edge clearance rule is 0.01 mm; JLC needs ≥ 0.2 mm.

---

## 🟡 Minor — open

- **m-01** 39.2 kΩ gives 4.92 V nominal: about 170 mV above the CM5's 4.75 V minimum before FB1 DCR, trace drop and the 1 % VFB tolerance. 40.2 kΩ (5.02 V) is a better centre.
- **m-02** C46/C47 value 6.8 µF / 1812, SPN is a 4.7 µF 1206 part. C11/C14/C20/C22 are 6.3 V parts on a 5 V rail (weak derating); 10 V preferred.
- **m-03** C66, C67, C69 are 10 µF in **0201** — not realistically available. Use 0603.
- **m-04** U3 (TPS25751) Value field is empty — blank BOM line.
- **m-05** TPS25751 USB_DP/USB_DM (pins 26/27) are labelled but unconnected and not NC-flagged; ERC will flag them. Mark NC or wire to J15 D± for BC1.2.
- **m-06** **J12 provisioning (Tag-Connect TC2030):** pin 1 is `USB_LDO_3V3`. If the programmer *drives* VCC on that pin it back-feeds the TPS25751's internal LDO. Use it only as a sense/VREF pin, or add ~1 kΩ in series. EEPROM WP default-protected via R3 (4.7 k) with J12.6 to ground for writes ✅. U2 is now an AT24C256C (32 KB), which covers the full-flash binary.
- **m-07** **Board-ID EEPROM U10 (DNP)** on ID_SC/ID_SD at 0x50 with 4.7 k pull-ups ✅. Its WP (R32 to 3V3) has no pad or jumper to ground, so a populated U10 can't be programmed in place. Add a test pad or solder jumper.
- **m-08** U2/U10 use the `AT24CS64-MAHM` (DFN, 9-pin) symbol on an 8-pin SOIC footprint; pin 9 (EP) has no pad. Harmless, but it will show as a symbol/footprint mismatch. Switch to a matching SOIC-8 symbol.
- **m-09** Housekeeping: title block still "Compact Pi AutoPilot", sheet "Autopilot Interfaces", labels `I2C_PMIC_*` / `SCL4` / `SDA4` (it's RP1 I2C3 on CM5), empty `sensors`/`ap_io` sheets, README still describes the autopilot scope and 45 mm width (board is 47 mm).

---

## ✅ Resolved since the first pass

- **USB-PD controller (TPS25751):** ADCIN1 → LDO_3V3 = **7**, ADCIN2 → LDO_1V5 = **5** → I2C address index #1 (**0x20**), default behaviour **AlwaysEnableSink** (verified against SLVSH93A §8.4.1). I2Cc pull-ups 4.7 k; nI2Cc_IRQ pulled up; LDO_3V3 and LDO_1V5 now 10 µF each. VIN_3V3 fed from CM5 +3V3. **I2Ct now wired to the CM5 on GPIO6/7 (RP1 I2C3, `/dev/i2c-3`) with 4.7 k pull-ups, and nI2Ct_IRQ → GPIO26 with a 4.7 k pull-up**, so the CM5 can clear the dead-battery flag and read contract status. PP5V tied to the peripheral rail (decide sink-only vs dual-role in the EEPROM configuration).
- **Config EEPROM:** U2 now AT24C256C (32 KB), A0–A2 to GND (0x50), WP pulled up with J12 access.
- **VBUS bleeder:** R12 now 10 kΩ (40 mW at 20 V).
- **Reverse polarity:** F1/F2 ahead of the TVS diodes — a reversed battery now opens a fuse instead of burning the TVS.
- **PCB synchronised** with the schematic.

## ✅ Verified correct (unchanged)

Ideal-diode ORing (VCAP to ANODE, EN to ANODE, FET orientation); TLVM13630 VLDOIN/VCC/SW/DNC handling; CM5 GPIO_VREF on 3V3 and +3V3 load well under 600 mA; I2C1 relies on on-module 1.8 k pull-ups; CAN physical layer (common-mode choke, SZNUP2105L, DNP 120 Ω, Dronecode pinout, STBY low); Ethernet Bob-Smith termination and pair mapping; USB-C D± both orientations tied; SD power switching; netclass widths match JLC's calculator for the stackup they were computed against.

---

## 🔵 Suggestions (DAQ scope, unchanged)

- **RTC backup:** VBAT (pin 76) is still unconnected — a logger without one timestamps every cold boot wrong until NTP/GPS.
- **Free I/O:** SPI0, I2C0 (SCL0/SDA0), UART4 and GPIO16/22/27 are unused — a header or extra JST-GH plus a GPS PPS input costs nothing now.
- **Indicators:** LED_nACT (pin 21) is free for an activity LED.
- **ESD on external connectors** (HDMI, JST VCC pins, CAN 5 V, USB-C VBUS/CC) for a marine enclosure.

## Suggested order of work

1. **K-1:** fix the V_USB capacitor part numbers, then do a full SPN-vs-Value BOM pass.
2. **K-2:** wire J4/J10 to UART0/UART2.
3. M-03, M-08, M-11: three quick schematic/library fixes.
4. Pick the fab stackup, re-enter it, fix the In4 pours and the 0.3 mm vias, place the 17 parked parts, add thermal vias.
5. Choose Y1, F1/F2 and FB1/FB2 part numbers.
6. Housekeeping and README.

## Sources

- [TLVM13630 datasheet](https://www.ti.com/lit/ds/symlink/tlvm13630.pdf)
- [TPS25751 datasheet](https://www.ti.com/lit/ds/symlink/tps25751.pdf) and [§8.4.1 Pin Strapping](https://www.ti.com/document-viewer/TPS25751/datasheet/GUID-54B833CE-F926-46CE-A70B-AE3187D04054)
- [LM74700-Q1 datasheet](https://www.ti.com/lit/ds/symlink/lm74700-q1.pdf)
- [CM5 datasheet RP-008180-DS-7](https://pip-assets.raspberrypi.com/categories/944-raspberry-pi-compute-module-5/documents/RP-008180-DS-7-cm5-datasheet.pdf)
- [RP1 peripherals](https://pip-assets.raspberrypi.com/categories/892-raspberry-pi-5/documents/RP-008370-DS-1-rp1-peripherals.pdf)
- [MCP25625 datasheet](https://static.chipdip.ru/lib/661/DOC028661833.pdf)
- [WE 82400102 datasheet](https://www.we-online.com/components/products/datasheet/82400102.pdf)
