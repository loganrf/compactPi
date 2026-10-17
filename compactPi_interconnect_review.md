# compactPi — Interconnect Review

**Project:** compactPi — Compact RPi CM5 Autopilot Carrier
**Review Date:** 2026-03-29
**Schematic Rev:** 0 (dated 2026-02-08)
**Reviewer:** Claude (Cowork)
**Method:** Automated parse of all seven `.kicad_sch` files — hierarchical labels, sheet pin connections, net label positions, and wire endpoint analysis of `compactPi.kicad_sch` (top level)

> This review focuses specifically on **interconnect correctness**: unconnected hierarchical pins, signal direction conflicts, and voltage domain mismatches. For component-level findings (missing resistor values, CAN termination, etc.) see `compactPi_schematic_review.md`.

---

## Finding Summary

| ID | Severity | Sheet(s) | Finding |
|---|---|---|---|
| I-01 | 🔴 Critical | `autopilot_conn` / top | UART1 and UART2 signals have no connections at top-level — CM5 UART is not reachable |
| I-02 | 🔴 Critical | `sensors` / top | Sensor interrupt outputs (INT_IMU1, INT_IMU2, INT_PRES) are unconnected at top-level |
| I-03 | 🔴 Critical | `sensors` / top | `I2C_SCL_1V8` and `I2C_SDA_1V8` sheet pins are unconnected — level-shifter architecture incomplete |
| I-04 | 🟠 Major | `Power.kicad_sch` | `5V_PERIPHERAL` hierarchical label exists inside Power sheet but has no matching pin on the top-level sheet block — orphaned label, causes ERC error |
| I-05 | 🟠 Major | `CM5_HighSpeed` | `5V0_Peripheral` hierarchical label appears **twice** in the same sub-sheet, producing duplicate sheet pins on the top-level |
| I-06 | 🟠 Major | `autopilot_conn` / `CM5_GPIO` | `I2C_SCL_3V3` is marked **output** from both `autopilot_conn` and `CM5_GPIO` (via SCL1) onto the same `I2C_EXT_SCL1` net — conflicting drivers |
| I-07 | 🟡 Minor | All | Power and I2C net names differ between sub-sheet hierarchical labels and top-level net labels — naming inconsistency without electrical impact |
| I-08 | 🟡 Minor | `sensors` | Only the barometer CS (`CS_PRES_3V3`) exits the sensors sheet; IMU/magnetometer CS signals are not visible in the hierarchical interface |

---

## Detailed Findings

### 🔴 I-01 — UART1 and UART2 Not Connected to CM5

**Affected pins (top-level `compactPi.kicad_sch`):**

| Pin name | Shape | Position on `autopilot_conn` block |
|---|---|---|
| `UART1_RX_In` | output | (382.27, 124.46) |
| `UART1_TX_Out` | input | (382.27, 127.00) |
| `UART2_RX_In` | output | (382.27, 132.08) |
| `UART2_TX_Out` | input | (382.27, 134.62) |

Wire endpoint analysis shows **no wires and no net labels** at any of these coordinates on the top-level schematic. Additionally, `CM5_GPIO.kicad_sch` contains **zero UART-related hierarchical labels** — the CM5's UART0/UART1 TX/RX pins are not exported through the hierarchical interface at all.

The UART connectors themselves are properly wired to the hierarchical labels inside `autopilot_conn.kicad_sch`, so the connector side is correct. The gap is between the connector sheet and the CM5 GPIO sheet.

**Impact:** Neither UART port is usable. ArduPilot relies on serial UART for GPS, telemetry radio, and companion link. This will prevent the board from functioning as an autopilot carrier.

**Recommendation:** Add hierarchical labels for UART1_TX, UART1_RX, UART2_TX, UART2_RX to `CM5_GPIO.kicad_sch` (choosing the correct CM5 GPIO pins — typically GPIO14/15 for UART0 and GPIO0/1 for UART2 in BCM numbering). Then draw wires on the top-level connecting those sheet pins to the corresponding `autopilot_conn` sheet pins, using matching net labels (e.g., `UART1_TX`, `UART1_RX`).

---

### 🔴 I-02 — Sensor Interrupt Outputs Unconnected

**Affected pins (top-level `compactPi.kicad_sch`):**

| Pin name | Shape | Position on `sensors` block |
|---|---|---|
| `INT_IMU1_3V3` | output | (364.49, 36.83) |
| `INT_IMU2_3V3` | output | (364.49, 39.37) |
| `INT_PRES_3V3` | output | (364.49, 43.18) |

All three are on the right edge of the `sensors` sheet block. Wire endpoint and net label analysis shows **nothing connected** at these positions on the top-level. The interrupt signals are properly defined inside `sensors.kicad_sch` but do not reach any CM5 GPIO pin.

**Impact:** Without interrupt routing, ArduPilot must poll the sensors over SPI/I2C rather than using IRQ-driven data-ready signals. This increases latency and CPU load for the IMU loop — particularly significant for an autopilot where IMU sample timing is critical. The BMP390 barometer data-ready interrupt is also unused.

**Recommendation:** Add hierarchical labels for three CM5 GPIO interrupt inputs to `CM5_GPIO.kicad_sch`. Connect them on the top-level to `INT_IMU1_3V3`, `INT_IMU2_3V3`, and `INT_PRES_3V3`. Suggested GPIO assignments: GPIO23, GPIO24, GPIO25 (check CM5 datasheet for interrupt-capable pins). Use dedicated net labels such as `IMU1_INT`, `IMU2_INT`, `PRES_INT`.

---

### 🔴 I-03 — 1.8 V I²C Bus Unconnected at Top Level

**Affected pins (top-level `compactPi.kicad_sch`):**

| Pin name | Shape | Position on `sensors` block |
|---|---|---|
| `I2C_SCL_1V8` | output | (323.85, 53.34) |
| `I2C_SDA_1V8` | bidirectional | (323.85, 55.88) |

These pins are on the left edge of the `sensors` sheet block. Wire endpoint and net label analysis shows **nothing connected** at these positions on the top-level.

Inside `sensors.kicad_sch`, these labels carry the 1.8 V side of the 74LVC2T45 level-shifted I²C bus. The 3.3 V side (`I2C_SCL_3V3`, `I2C_SDA_3V3`) is connected to the top-level `I2C_INT_SCL0` / `I2C_INT_SDA0` net and routes correctly to CM5_GPIO SCL0/SDA0. The 1.8 V side has nowhere to go.

This raises a structural question about the level-shifter topology:

- If the CM5 I²C pins operate at **3.3 V**, the 74LVC2T45 should translate 3.3 V (CM5 side) to the sensor's native voltage (1.8 V or 3.3 V depending on sensor VDD). In this case the 1.8 V side connects to the sensor ICs *within* the sensors sheet and should not exit to the top level.
- If the sensors sheet incorrectly exports I2C_SCL_1V8 to the top level as a hierarchical label, the two unconnected sheet pins are a sign that the level-shifter schematic layout needs to be reviewed.

Separately, using the 74LVC2T45 for I²C requires careful DIR pin management because I²C is open-drain bidirectional (see also I-06). For the sensor-side I²C bus, a purpose-built bidirectional level shifter such as the PCA9306 is preferable.

**Recommendation:**
1. Determine whether the sensor ICs on the `sensors` sheet operate at 1.8 V or 3.3 V.
2. If the level translation is entirely internal to the sensors sheet (CM5 3.3 V → 74LVC2T45 → 1.8 V sensors), remove the `I2C_SCL_1V8` and `I2C_SDA_1V8` hierarchical labels from `sensors.kicad_sch` and make them local net labels instead.
3. If a separate 1.8 V I²C bus is genuinely needed at the top level, add matching hierarchical labels to `CM5_GPIO.kicad_sch` and wire them on the top level.

---

### 🟠 I-04 — Orphaned `5V_PERIPHERAL` Label in Power Sheet

**Sheet:** `Power.kicad_sch`

The Power sub-sheet contains **three** hierarchical labels related to the 5 V peripheral rail:

| Label name | Shape | Status |
|---|---|---|
| `5V0_PERIPHERAL` | output | ✅ Has matching sheet pin on top-level |
| `5V0_CORE` | output | ✅ Has matching sheet pin on top-level |
| `5V_PERIPHERAL` | bidirectional | ❌ No matching sheet pin on top-level |

Because `5V_PERIPHERAL` ≠ `5V0_PERIPHERAL` (missing the `0`), KiCad cannot match it to any top-level sheet pin. This produces an ERC "Pin not connected to hierarchical label" error.

**Recommendation:** Either delete `5V_PERIPHERAL` from `Power.kicad_sch` if it is a leftover, or rename it to `5V0_PERIPHERAL` if it was intended to carry an additional connection to the peripheral rail (and add the corresponding sheet pin to the top-level).

---

### 🟠 I-05 — Duplicate `5V0_Peripheral` Hierarchical Label in CM5_HighSpeed

**Sheet:** `CM5_HighSpeed.kicad_sch`

`5V0_Peripheral` appears as a hierarchical label at two distinct locations within `CM5_HighSpeed.kicad_sch`:

| Position | Shape |
|---|---|
| (257.81, 207.01) | input |
| (165.10, 152.40) | input |

Both have `input` shape, so there is no direction conflict — they are two entry points for the same net inside the sheet. This is electrically valid (both connect to the same 5 V0 peripheral rail) but unusual and produces two identically named pins on the `CM5_HighSpeed` sheet block at the top level.

**Recommendation:** Verify this is intentional (e.g., 5 V0 is distributed to two physically separate areas of the CM5_HighSpeed sheet). If so, add a comment explaining why two instances are needed. If it is a copy/paste duplication, remove the redundant label.

---

### 🟠 I-06 — Conflicting `output` Directions on `I2C_EXT_SCL1`

**Sheets:** `CM5_GPIO.kicad_sch`, `autopilot_conn.kicad_sch`

The external I²C SCL bus (`I2C_EXT_SCL1` on the top-level net) has **two hierarchical labels both marked as `output`**:

| Sheet | Hierarchical label | Shape | Top-level net |
|---|---|---|---|
| `CM5_GPIO` | `SCL1` | output | `I2C_EXT_SCL1` |
| `autopilot_conn` | `I2C_SCL_3V3` | output | `I2C_EXT_SCL1` |

In KiCad ERC, two `output` shapes on the same net are flagged as a pin conflict (multiple drivers).

The likely design intent is:
- CM5 drives SCL1 (open-drain, pulled up inside `CM5_GPIO` or on the top-level)
- `autopilot_conn` carries the SCL to the external JST-GH I²C connector (a passive pass-through)

If the SCL line simply passes through `autopilot_conn` to the connector without any modification, the hierarchical label `I2C_SCL_3V3` in `autopilot_conn` should be shape `input` (receiving SCL from the top-level) rather than `output`. The `output` marking suggests the signal originates inside `autopilot_conn`, which is not the case for an I²C master on the CM5.

**Recommendation:** Change the `I2C_SCL_3V3` hierarchical label in `autopilot_conn.kicad_sch` from `output` to `input`. Verify the same for `I2C_SDA_3V3` in `autopilot_conn` — for I²C SDA (open-drain bidirectional) the correct shape is `bidirectional`, which it already is. Also confirm the `I2C_SDA_3V3` in `ap_io.kicad_sch` — it is currently `input`; for I²C SDA this should also be `bidirectional`.

---

### 🟡 I-07 — Power and I²C Net Naming Inconsistency (No Electrical Impact)

Several hierarchical labels in sub-sheets use different capitalisation or names than the net labels on the top-level schematic that they ultimately connect to. This does not cause electrical errors (the connections are established by net labels on the top-level wires, not by matching names across sheets), but it is confusing and risks mistakes during future edits.

| Sub-sheet | Internal hierarchical label | Top-level net label | Discrepancy |
|---|---|---|---|
| `CM5_GPIO` | `5V0_Core` | `5V0_CORE` | Case: `Core` vs `CORE` |
| `CM5_HighSpeed` | `5V0_Peripheral` | `5V0_PERIPHERAL` | Case: `Peripheral` vs `PERIPHERAL` |
| `CM5_GPIO` | `3V3_Peripheral` | `3V3` | Name: peripheral qualifier dropped at top level |
| `CM5_GPIO` | `1V8_Peripheral` | `1V8` | Name: peripheral qualifier dropped at top level |

**Recommendation:** Adopt a consistent naming convention. The simplest fix is to update the sub-sheet hierarchical labels to match the top-level net labels exactly (e.g., change `5V0_Core` → `5V0_CORE`, `3V3_Peripheral` → `3V3`). This makes the design easier to trace and avoids confusion when performing ERC or generating netlists.

---

### 🟡 I-08 — IMU and Magnetometer Chip-Select Signals Not Visible at Hierarchical Interface

**Sheet:** `sensors.kicad_sch`

The sensors sheet exports only one SPI chip-select signal through the hierarchical interface: `CS_PRES_3V3` (the barometer BMP390 select, driven by CM5 SPI0_CE0).

The two `CSB` component pin names that appear as local labels inside `sensors.kicad_sch` are not exported and are therefore connected to something entirely internal to that sheet. This is consistent with the BMI270 being operated in **I²C mode** (CSB tied high via a pull-up to disable SPI). If that is the design intent, it should be documented on the schematic.

**Impact:** Low, provided the intent is confirmed. However, if the BMI270 is intended for SPI (which provides better performance at high ODRs), a CE signal from a CM5 GPIO must be exported through the hierarchical interface and wired on the top level.

**Recommendation:** Add a schematic text note near the BMI270 in `sensors.kicad_sch` stating either "I²C mode — CSB tied to 1V8 via R_xx" or "SPI mode — CSB driven by CM5 GPIO_xx". If SPI mode is intended, add a `CS_IMU1_3V3` hierarchical label (input) and export it to the top level.

---

## Cross-Sheet Net Connectivity Map

The table below shows how the correctly-wired nets flow between sub-sheets through the top-level.

| Top-level net | Source sheet (output) | Destination sheet(s) (input) | Status |
|---|---|---|---|
| `5V0_CORE` | Power | CM5_GPIO | ✅ |
| `5V0_PERIPHERAL` | Power | CM5_HighSpeed, ap_io, autopilot_conn | ✅ |
| `3V3` | CM5_GPIO (as `3V3_Peripheral`) | sensors, ap_io, autopilot_conn | ✅ |
| `1V8` | CM5_GPIO (as `1V8_Peripheral`) | sensors | ✅ |
| `GPIO_VREF` | CM5_GPIO | CM5_HighSpeed | ✅ |
| `I2C_INT_SCL0` / `SDA0` | CM5_GPIO (SCL0/SDA0) | sensors | ✅ |
| `I2C_EXT_SCL1` / `SDA1` | CM5_GPIO (SCL1/SDA1) | autopilot_conn | ⚠️ Direction conflict on SCL (I-06) |
| `I2C_PWM_SCL` / `SDA` | CM5_GPIO (SCL4/SDA4) | ap_io | ✅ |
| `SPI_INT_*` (SPI0) | CM5_GPIO | sensors | ✅ |
| `SPI_CAN_*` (SPI1) | CM5_GPIO | autopilot_conn | ✅ |
| `UART1_TX/RX`, `UART2_TX/RX` | *(none)* | autopilot_conn | ❌ **I-01** |
| `INT_IMU1/2_3V3`, `INT_PRES_3V3` | sensors | *(none)* | ❌ **I-02** |
| `I2C_SCL_1V8`, `I2C_SDA_1V8` | sensors | *(none)* | ❌ **I-03** |
| `ID_SC`, `ID_SD` | CM5_GPIO | CM5_HighSpeed | ✅ |
| `VBUS_EN` | CM5_HighSpeed | Power | ✅ |
| PCIe (CLK, TX, RX, RST, etc.) | CM5_HighSpeed | *(top-level termination)* | ✅ |

---

## Recommended Actions (Priority Order)

1. **[I-01]** Wire `UART1_TX_Out`, `UART1_RX_In`, `UART2_TX_Out`, `UART2_RX_In` on the top-level schematic. Add corresponding hierarchical labels in `CM5_GPIO.kicad_sch`. This is required for basic autopilot function.

2. **[I-02]** Wire `INT_IMU1_3V3`, `INT_IMU2_3V3`, `INT_PRES_3V3` on the top-level to three CM5 GPIO inputs. Add corresponding hierarchical labels in `CM5_GPIO.kicad_sch`.

3. **[I-03]** Decide whether `I2C_SCL_1V8` / `I2C_SDA_1V8` should be internal to the sensors sheet or externally connected. Remove or properly route them.

4. **[I-04]** Delete or rename `5V_PERIPHERAL` in `Power.kicad_sch` to resolve the orphaned hierarchical label ERC error.

5. **[I-06]** Change `I2C_SCL_3V3` in `autopilot_conn.kicad_sch` from `output` to `input` shape to resolve the dual-driver ERC conflict on `I2C_EXT_SCL1`.

6. **[I-05]** Verify the duplicate `5V0_Peripheral` in `CM5_HighSpeed.kicad_sch` is intentional; add a comment if so.

7. **[I-07]** Harmonise sub-sheet hierarchical label names with top-level net names (capitalisation).

8. **[I-08]** Document BMI270 interface mode (I²C vs SPI) on the schematic.

---

*Reviewed from schematic files dated 2026-02-08, Rev 0. Re-review recommended after open items are resolved. Run KiCad ERC after addressing I-01 through I-06 to confirm no additional conflicts.*
