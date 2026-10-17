# compactPi — Schematic Design Review

**Project:** compactPi — Compact RPi CM5 Autopilot Carrier
**Review Date:** 2026-03-27
**Schematic Rev:** 0 (dated 2026-02-08)
**Reviewer:** Claude (Cowork)
**License:** CERN-OHL-S v2

---

## Overview

This review covers all seven schematic sheets of the compactPi design:

| Sheet | Title | Status |
|---|---|---|
| `Power.kicad_sch` | Carrier Board Power Supplies | In Work |
| `CM5_GPIO.kicad_sch` | CM5 GPIO / Ethernet | In Work |
| `CM5_HighSpeed.kicad_sch` | CM5 High-Speed Interfaces | In Work |
| `sensors.kicad_sch` | Sensor Suite | In Work |
| `autopilot_conn.kicad_sch` | Autopilot Communication Connectors | In Work |
| `ap_io.kicad_sch` | Autopilot PWM I/O | In Work |
| `compactPi.kicad_sch` | Top-Level Integration | In Work |

The design targets a 45 × 105 mm carrier board for the Raspberry Pi Compute Module 5, aimed primarily at ArduPilot marine applications (ArduSub / ArduRover).

---

## Finding Summary

| ID | Sheet | Finding | Severity |
|---|---|---|---|
| F-01 | Power | TLVM13630 feedback resistors have no values assigned | 🔴 Critical |
| F-02 | Power | LM74700 load-switch configuration resistors unvalued | 🔴 Critical |
| F-03 | autopilot_conn | CAN bus 120 Ω termination resistor absent from schematic | 🔴 Critical |
| F-04 | compactPi | USB-C CC pulldown resistors not visible / unconfirmed | 🔴 Critical |
| F-05 | ap_io | PWM resistor packs RN1–RN4 have no values assigned | 🟠 Major |
| F-06 | sensors | BMI270 CS/SDI pin mode-select pullup absent | 🟠 Major |
| F-07 | ap_io | PCA9685 ~OE pin connection not confirmed | 🟠 Major |
| F-08 | sensors | 74LVC2T45 DIR pin connections not confirmed | 🟠 Major |
| F-09 | Power | Reverse-polarity protection at J1 is a documented TODO | 🟠 Major |
| F-10 | README | Part numbers in README do not match schematic | 🟡 Minor |
| F-11 | sensors | AK09918C and MMC5983MA share the same I2C bus — confirm address isolation | 🟡 Minor |
| F-12 | Power | SMAJ26A TVS clamp voltage (26 V) may be oversized for 5 V USB protection | 🟡 Minor |
| F-13 | CM5_GPIO | Purpose of R20/R21 (75 Ω) is not annotated | 🟡 Minor |
| F-14 | compactPi | Top-level sheet title block is empty (no title string) | 🟡 Minor |
| F-15 | All | Decoupling capacitors well-placed throughout — no gaps found | ✅ Good |
| F-16 | CM5_HighSpeed | HDMI DDC pull-ups R5/R6 are 2.2 kΩ — correct | ✅ Good |
| F-17 | autopilot_conn | Crystal load caps C52/C53 are 22 pF — standard value | ✅ Good |
| F-18 | sensors | Dual-voltage level shifting via 74LVC2T45 is the right device choice | ✅ Good |
| F-19 | ap_io | PCA9685 is a well-proven ArduPilot PWM expander choice | ✅ Good |

---

## Detailed Findings

### 🔴 F-01 — TLVM13630 Feedback Resistors Unvalued
**Sheet:** `Power.kicad_sch`
**Components:** R1, R2, R7, R8, R9, R10, R11, R13, R14
**Nets:** `5V0_C_FB`, `5V0_P_FB`, `5V0_C_RT`, `5V0_P_RT`

All resistors in the feedback networks of U13 and U14 (TLVM13630RDHR 3 A synchronous buck converters) are placed as generic `R_US` symbols with no value. The output voltage of each rail is entirely determined by the feedback divider ratio:

> V_OUT = 0.6 V × (1 + R_top / R_bot)

For a 5.0 V output this requires (for example) R_top ≈ 73.2 kΩ and R_bot ≈ 10 kΩ with a 1% tolerance. Without values populated, the output voltages for `5V0_CORE` and `5V0_PERIPHERAL` are undefined and the power supply cannot be validated.

The design already has a TODO note: *"RT Values for Power Supplies"*. This must be resolved before layout freeze.

**Recommendation:** Calculate feedback divider values per the TLVM13630 datasheet (SLVSEL1), populate R1/R2 (core rail) and R7–R14 (peripheral rail), and confirm with the 1% standard resistor series. Add the target output voltages as schematic annotations.

---

### 🔴 F-02 — LM74700 Load-Switch Configuration Resistors Unvalued
**Sheet:** `Power.kicad_sch`
**Components:** R_xx on U4, U7, U9 (LM74700QDBVRQ1) and U5, U6, U8 (CSD18511Q5A)

The LM74700 ideal diode controller drives an external N-channel MOSFET (CSD18511Q5A). The MOSFET gate resistor and the ILIM sense resistor must both be specified before this circuit can be validated. In the current schematic these are placeholder values. The forward voltage drop and current limiting threshold both depend on these components.

**Recommendation:** Determine the maximum expected input current per switch, calculate the sense resistor value for the desired ILIM threshold, and add a gate resistor of 10–100 Ω to limit gate drive current. Annotate the expected power dissipation on the MOSFETs.

---

### 🔴 F-03 — CAN Bus 120 Ω Termination Not Present
**Sheet:** `autopilot_conn.kicad_sch`
**Component:** U21 (MCP25625)
**Nets:** `CAN_H`, `CAN_L`

No 120 Ω resistor (or split 60 Ω / capacitor combination) was found between `CAN_H` and `CAN_L`. CAN bus requires both ends of the cable to be terminated to 120 Ω to prevent reflections. If this board will be one end of a CAN network, a 120 Ω termination resistor is required. A common approach for embedded autopilot boards is to make termination optional via a 0 Ω jumper or DNP solder bridge.

**Recommendation:** Add a 120 Ω resistor (or a 2 × 60 Ω split termination with a centre-tap capacitor to GND for improved common-mode rejection) between `CAN_H` and `CAN_L`, population-optional via a DNP designator. Alternatively, if termination is always provided by the cable harness, add a schematic note documenting this assumption.

---

### 🔴 F-04 — USB-C CC Pin Termination Unconfirmed
**Sheet:** `compactPi.kicad_sch` / `Power.kicad_sch`
**Component:** J15 (USB_C_Receptacle), U3 (TPS25751D)
**Nets:** `CC1`, `CC2`

The TPS25751D is a USB PD controller that manages the CC lines internally. Its `CC1` and `CC2` pins connect to the physical CC lines on the receptacle and apply the required 5.1 kΩ pull-down resistors internally when operating as a UFP (device). However, no explicit pull-downs were found in the schematic, and it is not confirmed whether these nets route directly from the connector to the TPS25751D or whether they pass through external resistors.

**Recommendation:** Verify that `CC1` and `CC2` from J15 connect directly to the TPS25751D `CC1`/`CC2` pins without additional series resistors that could affect impedance. Add a schematic note: *"CC termination provided internally by TPS25751D."* If the TPS25751D is removed in a future board revision, external 5.1 kΩ pull-downs must be added.

---

### 🟠 F-05 — PWM Resistor Packs RN1–RN4 Have No Values
**Sheet:** `ap_io.kicad_sch`
**Components:** RN1, RN2, RN3, RN4 (R_Pack04)

All four 4-resistor SIP networks sitting between the PCA9685 outputs and the JST-GH PWM connectors have no value assigned. Depending on the intended function these could be current-limiting resistors, series termination for the PWM signal, or pull-downs to hold outputs low while the PCA9685 initialises. Without knowing the connected servo/ESC input impedance the value cannot be inferred.

**Recommendation:** Decide on function (current-limit vs. series-termination vs. pull-down) and specify values. For RC servo/ESC inputs a series resistor of 33–100 Ω is typical for ESD protection without impeding the PWM signal at 50–400 Hz. If a pull-down is needed, 100 kΩ is a safe choice for outputs that might otherwise float during boot.

---

### 🟠 F-06 — BMI270 CS/SDI Mode-Select Pin Not Confirmed
**Sheet:** `sensors.kicad_sch`
**Component:** U10 (BMI270)

The BMI270 selects between SPI and I2C based on the logic level on the `CSB` (chip-select) pin at power-up:
- **SPI mode:** CSB goes low during a SPI transaction → mode auto-detected.
- **I2C mode:** CSB must be pulled high through a resistor.

The sensor sheet does show SPI signals (`SPI_COPI_1V8`, `SPI_CIPO_1V8`, `SPI_SCK_1V8`) routed to the BMI270, which suggests SPI mode is intended. However, no explicit CSB pull-up or SPI chip-select connection was confirmed in the schematic extraction.

**Recommendation:** Confirm that CSB is driven by the CM5 as a SPI chip-select and that no pull-up exists that would prevent SPI mode entry. Add a brief annotation: *"SPI mode — CSB driven by CM5 GPIO"*. If I2C mode is intended instead, add a 10 kΩ pull-up to `1V8` on CSB and remove the SPI net labels.

---

### 🟠 F-07 — PCA9685 ~OE Pin Connection Not Confirmed
**Sheet:** `ap_io.kicad_sch`
**Component:** U18 (PCA9685PW)
**Pin:** ~OE (pin 13)

The PCA9685 has an active-low output-enable pin. When ~OE is high, all PWM outputs are tri-stated. During CM5 boot there is a window where the I2C bus is uninitialised and the PCA9685 outputs may be in an unknown state. Tying ~OE low permanently enables the outputs from power-on; tying it to a GPIO allows the CM5 firmware to arm/disarm outputs independently.

**Recommendation:** Confirm the connection for ~OE. For a marine autopilot where inadvertent servo/ESC activation on boot is a safety concern, connecting ~OE to a CM5 GPIO with a pull-up resistor (so it defaults to tri-state/safe on cold boot) is strongly recommended.

---

### 🟠 F-08 — 74LVC2T45 DIR Pin Connections Not Confirmed
**Sheet:** `sensors.kicad_sch`
**Components:** U11, U12, U16, U19

The 74LVC2T45DC has a direction pin (`DIR`) that selects which side drives the bus (A→B or B→A). For SPI signals this pin should be tied to a fixed logic level matching the fixed direction of each signal line. For bidirectional signals (e.g., I2C SDA) the 74LVC2T45 is the wrong device — a device without a DIR pin such as the PCA9306 should be used instead, as auto-direction detection is needed for open-drain I2C.

**Recommendation:** Verify whether the signals passing through U11/U12/U16/U19 are unidirectional (SPI, interrupt) or bidirectional (I2C SDA). For each instance:
- Confirm DIR is tied to a fixed level and document the direction.
- If any instance handles I2C SDA, replace with PCA9306 or similar.

---

### 🟠 F-09 — Reverse Polarity Protection at J1 is an Open TODO
**Sheet:** `Power.kicad_sch`
**Component:** J1 (Molex 1053101208 — power input connector)

The design TODO list contains an unchecked item: *"Power input protection."* No reverse-polarity protection diode or P-channel MOSFET guard was found on the `V_BATT` input rail. The LM74700 load-switch controller does include a diode emulation function on its output, but it does not protect against reverse connection at the input.

**Recommendation:** Add a P-channel MOSFET or Schottky diode on the `V_BATT` input line, before any load-switch, to prevent damage from reversed battery connections. For a 25 W board operating up to 25 V, a P-channel MOSFET such as the Si7471DP (Vds = −30 V, Id = −10 A) with a gate resistor and a gate-to-source clamp zener is a clean solution with minimal voltage drop.

---

### 🟡 F-10 — README Part Numbers Do Not Match Schematic
**File:** `README.md`

The README lists several sensor part numbers that differ from what appears in the schematic:

| Item | README says | Schematic has |
|---|---|---|
| 6-axis IMU | ICM-20602 | BMI270 |
| Magnetometer | AK09915 | AK09918C |
| Barometer | BMP280 | BMP390 |

These appear to be earlier component selections that were revised. Outdated documentation can cause confusion for external contributors, assembly vendors, and future BOM generation.

**Recommendation:** Update the README sensor table to reflect the current schematic (BMI270, AK09918C, BMP390). Add a brief changelog note explaining the part changes.

---

### 🟡 F-11 — AK09918C and MMC5983MA I2C Address Isolation
**Sheet:** `sensors.kicad_sch`
**Components:** U15 (AK09918C), U20 (MMC5983MA)

Both devices are magnetometers sitting on the same I2C bus segment. Their fixed I2C addresses are:
- **AK09918C:** 0x0C (fixed, no address pins)
- **MMC5983MA:** 0x30 (fixed, no address pins)

These do not conflict with each other, which is good. However, the BMP390 barometer (U25) has a configurable address via the `SDO` pin (0x76 or 0x77). The PCA9685 also uses address pins A0–A5.

**Recommendation:** Create a schematic annotation table listing every I2C device address on each bus segment. Verify that no two devices on the same I2C bus segment share an address. Pay particular attention to the PCA9685 address-pin configuration in `ap_io.kicad_sch` — confirm A0–A5 are all connected to defined logic levels (GND or Vcc), not left floating.

---

### 🟡 F-12 — SMAJ26A TVS Rating for USB Input
**Sheet:** `Power.kicad_sch` / `compactPi.kicad_sch`
**Components:** D1, D2 (SMAJ26A), U22 (WE-TVS-82400102)

The SMAJ26A is a 26 V standoff TVS diode — appropriate for protecting the battery input rail (6–25 V). However, a 26 V TVS on a 5 V VBUS line provides very limited protection: the clamping voltage of the SMAJ26A is ~42 V, which would allow substantial overvoltage before clamping. The Würth WE-TVS (U22) used on the USB port appears to be a more appropriate low-voltage array.

**Recommendation:** Confirm that the SMAJ26A devices are placed on the battery input path only, not directly across the 5 V VBUS line. Verify the Würth WE-TVS-82400102 part number is correct and that its clamping voltage is suitable for USB VBUS (5 V nominal). The PRTR5V0U2X or similar 5 V USB ESD array would be a well-characterised alternative if the Würth part is not confirmed.

---

### 🟡 F-13 — Purpose of R20/R21 (75 Ω) Not Annotated
**Sheet:** `CM5_GPIO.kicad_sch`
**Components:** R20, R21

Two 75 Ω resistors appear in the CM5 GPIO sheet. Their function is not documented by a schematic annotation. In the context of an Ethernet or differential signal path, 75 Ω could be characteristic-impedance series resistors; in an SD card context they could be a series termination to reduce overshoot. Without annotation their intent is ambiguous.

**Recommendation:** Add a brief text note to the schematic near R20/R21 explaining their function (e.g., "SD clock series termination" or "differential series termination — 75 Ω matched to trace impedance").

---

### 🟡 F-14 — Top-Level Sheet Has Empty Title Block
**Sheet:** `compactPi.kicad_sch`

The top-level sheet title block does not include a `(title "...")` string. All sub-sheets include proper title text. This is a minor documentation issue but could cause confusion in generated outputs (PDFs, BOMs).

**Recommendation:** Add a title such as `"compactPi — Autopilot Carrier Rev 0"` to the top-level sheet title block.

---

## What's Working Well

### ✅ F-15 — Decoupling Strategy
100 nF bypass capacitors are present at virtually every IC power pin across all sheets, supplemented by bulk capacitors on every power rail. The use of multiple capacitor values (100 nF + 4.7 µF + 47 µF) on the main rails is good practice.

### ✅ F-16 — HDMI DDC Pull-ups
R5 and R6 are specified as 2.2 kΩ 1% for the HDMI DDC bus (I2C at 100 kHz). This value is within the HDMI specification and is appropriate for the expected bus capacitance on a compact PCB. The DDC I2C bus is correctly isolated from the CM5's general-purpose I2C buses.

### ✅ F-17 — Crystal Load Capacitors
C52 and C53 are 22 pF, a standard value for the MCP25625's external crystal. This matches the CAN controller's recommended load capacitance range and is appropriate for the 16 MHz oscillator that clocks the MCP25625.

### ✅ F-18 — Dual-Voltage Level Translation Choice
Using the 74LVC2T45 for the 1.8 V ↔ 3.3 V translation between CM5 and sensors is the right approach. The device supports the required voltage rails (VCCA down to 1.2 V, VCCB up to 3.6 V) and has sufficient drive strength for SPI at typical autopilot frequencies. Four instances provide per-signal direction control, which is cleaner than a single wide bus translator.

### ✅ F-19 — PCA9685 for PWM Expansion
The PCA9685 is a standard, well-supported device in the ArduPilot ecosystem (RCOutput_PCA9685 driver exists). Running 16 channels over I2C frees CM5 hardware PWM for other uses and simplifies the connector layout. Providing 4 × 6-pin connectors (4 channels each, with power and ground) matches common drone/marine ESC wiring conventions.

---

## Design Checklist — Status

| Category | Item | Status |
|---|---|---|
| Power | Switching regulator feedback resistor values defined | ❌ Open |
| Power | Load switch configuration values defined | ❌ Open |
| Power | Reverse-polarity protection at main input | ❌ Open TODO |
| Power | Input TVS clamping voltage appropriate for rail | ⚠️ Verify |
| Power | All power rails have bulk + bypass decoupling | ✅ |
| Power | Power sequencing documented | ⚠️ Not annotated |
| CAN | 120 Ω termination provided or documented as external | ❌ Open |
| CAN | CAN transceiver (MCP25625) SPI lines fully connected | ⚠️ Verify |
| USB | CC pin termination confirmed | ⚠️ Verify |
| USB | VBUS enable/disable logic present | ✅ (VBUS_EN) |
| USB | ESD protection on USB port | ✅ (U22) |
| HDMI | DDC pull-ups present and correctly valued | ✅ |
| HDMI | HPD signal routed | ✅ (HDMI0_HOTPLUG) |
| HDMI | CEC signal routed | ✅ (HDMI0_CEC) |
| Sensors | BMI270 SPI/I2C mode selection confirmed | ⚠️ Verify |
| Sensors | All I2C addresses unique and conflict-free | ⚠️ Verify |
| Sensors | Level shifter DIR pins tied to correct fixed levels | ⚠️ Verify |
| Sensors | Interrupt signals routed to CM5 GPIO | ✅ |
| PWM | PCA9685 ~OE pin controlled or tied | ⚠️ Verify |
| PWM | PWM output resistor values specified | ❌ Open |
| PWM | PCA9685 address pins (A0–A5) connected | ⚠️ Verify |
| General | All component values populated | ❌ Open (multiple) |
| General | README part numbers match schematic | ❌ Update needed |
| General | All sheet title blocks populated | ❌ (top sheet) |
| General | No-connect flags on intentionally unused pins | ✅ |

---

## Recommended Pre-Layout Actions

These items should be resolved before the layout is considered ready for tape-out:

1. **Populate TLVM13630 feedback resistors** (R1, R2, R7–R14) — determine target output voltages for `5V0_CORE` and `5V0_PERIPHERAL`, calculate divider values per the datasheet, and annotate target voltages on the schematic.

2. **Populate LM74700 sense and gate resistors** — calculate ILIM threshold per expected current, add gate resistor to each MOSFET.

3. **Add CAN termination** — place a 120 Ω DNP resistor between `CAN_H` and `CAN_L` (DNP allows flexibility for harness-terminated installations).

4. **Confirm USB-C CC routing** — trace CC1/CC2 from J15 to TPS25751D and add a schematic note confirming internal termination.

5. **Specify PWM resistor packs RN1–RN4** — decide on function and populate values.

6. **Confirm and annotate 74LVC2T45 DIR pin levels** — for each of U11/U12/U16/U19 document whether the connected signals are unidirectional or bidirectional, and confirm the device is appropriate.

7. **Connect and annotate PCA9685 ~OE** — tie to a CM5 GPIO with pull-up for safety, or tie to GND and document the decision.

8. **Confirm BMI270 CS/SDI mode configuration** — annotate SPI mode on the schematic.

9. **Add reverse-polarity protection** at J1 input (existing TODO).

10. **Update README** sensor part numbers to match schematic (BMI270, AK09918C, BMP390).

---

*Review based on schematic files dated 2026-02-08, Rev 0. Re-review recommended after all open items are resolved.*
