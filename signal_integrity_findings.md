# Signal Integrity Review - compactPi

This document outlines the signal integrity findings for the compactPi KiCad layout, focusing on high-speed nets including HDMI, USB, and Ethernet.

## Summary of Findings

Overall, the layout has several significant signal integrity issues, primarily related to length matching in differential pairs and missing routing for several high-speed interfaces.

## 1. HDMI Interface

| Signal Pair | Net IDs | Length 1 (mm) | Length 2 (mm) | Mismatch (mm) | Vias |
|---|---|---|---|---|---|
| HDMI0_CK | 136, 137 | 60.00 | 58.70 | **1.30** | 2 |
| HDMI0_D0 | 138, 139 | 53.19 | 59.12 | **5.93** | 0 |
| HDMI0_D1 | 140, 141 | 59.22 | 58.66 | 0.56 | 2 |
| HDMI0_D2 | 142, 143 | 57.44 | 57.47 | 0.03 | 0 |

### Observations:
- **Major Skew:** HDMI0_D0 has a mismatch of nearly 6mm, which is excessive for HDMI high-speed data.
- **Inconsistent Vias:** Lanes D1 and CK use 2 vias each, while D0 and D2 are routed without vias. This results in different impedance profiles and propagation delays across lanes.
- **Impedance Discontinuity:** HDMI0_CK (Net 136) exhibits trace width changes on the B.Cu layer (switching between 0.13mm and 0.3mm), which causes impedance mismatches.

### Recommendations:
- Re-route HDMI lanes to achieve sub-millimeter length matching within each pair.
- Harmonize via counts across all data and clock lanes.
- Maintain consistent trace widths throughout the high-speed sections.

## 2. USB Interfaces

| Signal Pair | Net IDs | Length 1 (mm) | Length 2 (mm) | Mismatch (mm) | Status |
|---|---|---|---|---|---|
| USB2 | 111, 112 | 5.09 | 4.89 | 0.20 | Routed |
| USB2a | 338, 339 | 61.82 | 62.76 | **0.94** | Routed |
| USB3-1_D | 120, 121 | 77.11 | 75.86 | **1.25** | Routed |
| USB3-0 RX/TX| 116-119 | - | - | - | **NOT ROUTED** |
| USB3-1 RX/TX| 122-125 | - | - | - | **NOT ROUTED** |

### Observations:
- **Unrouted Nets:** USB3-0 high-speed signals and USB3-1 RX/TX pairs are defined in the schematic but have no routing in the PCB.
- **Significant Skew:** USB3-1_D pair has a mismatch of 1.25mm.

### Recommendations:
- Route the missing USB 3.0/3.1 high-speed pairs if they are intended to be functional.
- Improve length matching on the USB2a and USB3-1_D pairs.

## 3. Ethernet Interface (10/100)

| Signal Pair | Net IDs | Length 1 (mm) | Length 2 (mm) | Mismatch (mm) |
|---|---|---|---|---|
| TRD0 | 146, 147 | 31.66 | 29.36 | **2.30** |
| TRD1 | 144, 145 | 27.49 | 26.68 | 0.81 |
| TRD0_TX | 284, 336 | 23.21 | 23.21 | 0.00 |
| TRD1_RX | 286, 287 | 24.35 | 24.35 | 0.00 |

### Observations:
- **Length Mismatch:** TRD0 pair has a 2.3mm mismatch. While 10/100 Ethernet is relatively robust, this is still a deviation from best practices.
- **Unrouted Pairs:** Extra Ethernet pairs (2 and 3) are not routed, which is consistent with the 10/100 Base-Tx specification mentioned in the schematic.

### Recommendations:
- Tune TRD0 and TRD1 for better length matching.

## 4. Other High-Speed Interfaces

- **PCIE:** All PCIE lanes (RX, TX, CLK) are defined in the schematic but **NOT ROUTED** in the PCB.
- **MIPI/DPHY:** MIPI/DPHY interfaces are defined in the schematic but **NOT ROUTED** in the PCB.

## 5. Board Stackup and Planes

- The board uses a 6-layer stackup with GND zones on In1.Cu and In4.Cu, providing good reference planes for signals on F.Cu and B.Cu.
- However, the large dielectric thickness (0.38mm) between In1-In2 and In3-In4 should be considered if routing high-speed signals on internal layers.

## Conclusion

The current layout is in a preliminary state. Most high-speed interfaces beyond basic USB 2.0 and 10/100 Ethernet are unrouted. Of the routed interfaces, HDMI exhibits severe length matching issues that will likely lead to functional failures at higher resolutions.
