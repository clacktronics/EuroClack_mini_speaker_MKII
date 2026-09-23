# EuroClack Mini Speaker MKII: Design Review

**Project:** EuroClack Mini Speaker MKII. Three KiCad 8 projects:
- `main_board` (single sheet, 4-layer 49.5 × 20 mm PCB)
- `Rail_adaptor` (single sheet, 2-layer 10 × 22 mm PCB)
- `EuroClack_Mini_speaker_MKII_panel` (10HP, 3U PCB front panel)

**Date:** 2026-09-23
**Method:** [kicad-happy](https://github.com/aklofas/kicad-happy) skills (`kicad`, `emc`, `lcsc`), followed by manual checks against the raw `.kicad_sch`/`.kicad_pcb`/Gerber files and the manufacturer datasheets.
**Analyzers run:** `analyze_schematic.py`, `analyze_pcb.py --full`, `analyze_gerbers.py`, `cross_analysis.py`, `analyze_thermal.py`, `analyze_emc.py` on all three boards. The `lcsc` skill was used to resolve every JLCPCB/LCSC part number in the BOMs.

## Overview

The module is a mono Eurorack speaker. A 3.5 mm jack (J1) feeds a ferrite bead (FB3), a 10k series resistor (R3) and a 10k volume pot (RV1). The signal is AC-coupled into a **TI TPA3110D2** class-D amplifier. The amplifier runs in **PBTL (mono) mode** at 20 dB gain from the Eurorack **+12 V** rail. A P-MOSFET (Q1, AO3401A) provides reverse-polarity protection. The output filter uses 10 µH inductors with 1 nF shunt capacitors and drives a panel-mounted Tectonic TEBM28C10-4B speaker (4 Ω, 10 W, Ø55 mm class) through a 2-pin header (LS1). A side-actuated slide switch (SW1, "Limit") pulls the PLIMIT pin down to give a low-power mode. The rail adaptor takes +12 V and GND from a 16-pin Eurorack power header and presents them on a 2-pin socket.

The main-board circuit closely follows TI's PBTL reference design (datasheet Figure 46, SLOS528F). The pin mapping, gain strapping, PBTL strap, bootstrap caps, AVCC RC filter, GVDD/PLIMIT divider and SD/FAULT auto-recovery all match the datasheet. The problems I found are mostly around that core: the protection MOSFET's gate rating, the input level, stale production files and some mechanical/DFM details.

## Critical Findings

| # | Severity | Issue | Board | Section |
|---|----------|-------|-------|---------|
| 1 | **CRITICAL** | Q1 (AO3401A) gate-source voltage is −12 V in normal operation. That equals the datasheet absolute maximum (±12 V), with zero margin for rail tolerance or hot-plug transients. | main | [Q1](#q1-reverse-polarity-pmos) |
| 2 | **CRITICAL** | The Rail_adaptor production files are stale. The Gerbers show a **14 × 22 mm** board but the PCB is **10 × 22 mm**. The BOM/CPL call for an **SMD 2×8 female socket** at a different position, but the PCB has a **THT 2×8 male header**. | rail | [Rail adaptor](#rail-adaptor) |
| 3 | WARNING | The 2-pin power connectors number their pins in opposite order: Rail_adaptor J2 pin 1 = +12 V, main_board J2 pin 1 = GND. If they mate pin-1-to-pin-1 the module sees reversed polarity. Q1 blocks it, so nothing is damaged, but the module stays silent. | rail + main | [Power connectors](#power-interconnect-polarity) |
| 4 | WARNING | The JLC export turns `Cmts.User` into a `*-VScore.gbr` layer. On the main board that layer holds crosshair lines through the jack and pot centres, and on the panel it holds 54 dimension/annotation items. These could be read as V-cut lines or at least trigger an engineering query. | main, panel | [Fabrication outputs](#fabrication-outputs) |
| 5 | **CRITICAL** | **The amplifier can destroy the speaker.** With the TEBM28C10-4B (4 Ω, 10 W rated, 2.8 mm p-p linear / 7.2 mm p-p mechanical excursion), full-rail drive delivers about 15 W sine and about 30 W clipped square wave. At low frequencies that is about 11.7 mm p-p cone excursion, well past the mechanical limit. The default PLIMIT (≈3.45 V) does not limit at all, and the input has no high-pass filter near the speaker's 150 Hz rating condition. | main | [Speaker](#speaker-tectonic-tebm28c10-4b) |
| 6 | WARNING | Input level is far too hot for the gain. With 20 dB gain and 12 V PVCC, the amp clips at about 1.1 V peak input. A standard ±5 V Eurorack signal reaches that at about 45 % pot rotation, which turns #5 into square-wave drive. A ±10–12 V signal can push RINP past its 6.3 V absolute maximum. | main | [Input stage](#input-stage) |
| 7 | WARNING | With the 4 Ω speaker, output peak current reaches about 2.9 A at full rail. That is at the L1/L2 saturation rating (2.95 A) and above their 1.84 A rated current. The output traces are only 0.2–0.25 mm wide. | main | [Speaker](#speaker-tectonic-tebm28c10-4b) |
| 8 | WARNING | The panel has no speaker mounting holes. The speaker needs 4 × Ø3.3 mm holes on a 33.9 mm square. The Ø40 mm cutout is also smaller than the Ø46 mm front ring, so the surround (2.2 mm frontal movement, 1.5 mm frame lip) may strike the back of the panel. | panel | [Speaker](#speaker-tectonic-tebm28c10-4b) |
| 9 | WARNING | Courtyard overlaps: RV1 overlaps C13 (3.9 mm²), Q1 (1.5 mm²) and C3 (0.6 mm²). The pot body may sit on these parts. | main | [PCB layout](#pcb-layout-analysis) |
| 10 | WARNING | Copper-to-edge clearance on the electrolytics is too small: C6 is 0.25 mm from the edge and C1 is 0.35 mm. C6 is below JLC's usual 0.3 mm minimum. | main | [PCB layout](#pcb-layout-analysis) |
| 11 | WARNING | Current draw from the Eurorack +12 V rail is not bounded: about 1.4 A average at full sine into 4 Ω, and more when clipping. Fixing #5 with PLIMIT brings it to about 0.5 A. | main | [Power analysis](#power-analysis) |

## Speaker: Tectonic TEBM28C10-4B

The designer confirmed the speaker is from the [TEBM28C family](https://www.tectonicaudiolabs.com/product-family/tebm28c-family). The figures below are **datasheet-verified** from the TEBM28C10-4B datasheet (rev E, 2026-04-07). The lower-cost 4BE variant (rev D) is similar and is shown where it differs.

| Parameter | 4B | 4BE |
|-----------|----|-----|
| Nominal impedance / Re | 4 Ω / 3.65 Ω | 3.8–4 Ω / 3.7 Ω |
| Rated noise power | 10 W (pink noise, **2nd-order HPF at 150 Hz**, 6 dB crest factor) | 10 W (same conditions) |
| Fs / Qts | 167 Hz / 0.48 | 161 Hz / 0.82 |
| Bl / Cms | 3.04 T·m / 0.67 mm/N | 2.12 T·m / 0.88 mm/N |
| Linear / mechanical excursion | 2.8 / **7.2 mm p-p** | 2.8 / 8 mm p-p |
| Sensitivity (1 W / 1 m) | 80 dB | 78 dB |
| Size | frame fits a Ø54.6 mm circle, Ø46 mm front ring, Ø42 mm rear cup, depth 24.3 mm (+1.5 mm lip) | 55 × 24 mm |
| Mounting | 4 × Ø3.3 mm holes on a 33.9 × 33.9 mm square | |

### Load and amplifier compatibility
- 4 Ω nominal and 3.65 Ω Re are above the TPA3110D2 PBTL minimum of 3.2 Ω (SLOS528F §7.1). ✓
- The impedance curve (datasheet Fig. 3.3.1) stays at 3.5 Ω or more across the band, and rises to about 12 Ω at 20 kHz. That helps damp the output LC filter.
- Series resistance: PBTL output stage ≈ 0.24 Ω, plus 2 × 49 mΩ in the inductors, gives R_S ≈ 0.34 Ω. Full-rail peak current is therefore about 11.5 V / 4.0 Ω ≈ **2.9 A**.
  - That is at the SMDRI74-100NT saturation rating (2.95 A) and above its 1.84 A rated current.
  - Once PLIMIT is set as recommended below, the peak falls to about 1.7 A. That is within the inductor rating and acceptable for 0.5 mm traces. Widen OUTNL/OUTNR from 0.2–0.25 mm to at least 0.5 mm anyway.

### Excursion and power
Below Fs, cone excursion is roughly x ≈ Bl·I·Cms ≈ **2.04 mm per amp** (4B).

| Drive condition | Peak current | Excursion | Power into speaker | Verdict |
|-----------------|-------------|-----------|---------------------|---------|
| Linear-excursion limit (2.8 mm p-p) | 0.69 A | 2.8 mm p-p | ≈0.9 W sine at LF | the distortion onset for bass |
| Mechanical limit (7.2 mm p-p) | 1.77 A | 7.2 mm p-p | ≈5.7 W sine (6.45 V pk) | the damage threshold |
| **Current design, full rail** | 2.9 A | **≈11.7 mm p-p** | ≈15 W sine, ≈30 W square | **bottoms out and is 1.5–3× the power rating** |

The speaker's 10 W rating assumes a 150 Hz 2nd-order high-pass filter. The design's only high-pass is C13 with the amp's 60 kΩ input impedance, at about 2.7 Hz. So bass notes, kick drums and LFO-rate signals from a Eurorack patch hit the speaker at full amplitude below Fs. The hot input stage (#6) makes square-wave clipping easy to reach.

**Recommended fix (two parts, both cheap):**
1. **Set the power limit with PLIMIT.** Change **R6 from 10k to 3.3k**, which gives V_PLIMIT = 6.9 × 3.3 / 13.3 ≈ **1.71 V**.
   - The virtual rail becomes 4 × V_PLIMIT ≈ 6.8 V peak, which is about 6.3 V peak across the speaker.
   - That keeps excursion at the mechanical limit even at DC-to-Fs, and sits close to TI's own 1.76 V example (Table 3).
   - Sine power is then about 5 W and clipped square about 10 W, which matches the speaker's 10 W rating.
   - Limit mode (SW1 closed) is covered in [Power modes](#power-modes-eurorack-rail-vs-external-12-v) below. Its R8 needs re-scaling as well.
   - For the 4BE variant, 3.9k (≈1.94 V) is also safe.
   - GVDD load stays around 0.5–0.7 mA. Confirm on the bench that GVDD holds about 6.9 V.
2. **Add a high-pass filter near 120–150 Hz.** Change **C13 and C9 from 1 µF to 22 nF**. With Zi = 60 kΩ ±20 %, the corner lands at about 100–150 Hz.
   - Keep C9 equal to C13, as TI §9.3.3 requires, to avoid pop and nuisance DC-detect faults.
   - The speaker's own roll-off below Fs adds a second order acoustically. For a true 2nd-order electrical filter, add a series RC ahead of the pot.
   - The response graph (Fig. 3.3.1) shows output already 10 dB down at 70 Hz, so almost no audible bass is lost.

With both changes, the worst-case LF excursion stays within 7.2 mm p-p. Mid-band SPL at about 5 W is roughly 87 dB at 1 m (80 dB + 7 dB). +12 V draw falls to about 0.5 A average at full sine.

### Power modes: Eurorack rail vs external 12 V

**Designer intent:**
- SW1 = "Limit" is for powering from the Eurorack +12 V rail, with a draw of about **100 mA max**.
- The other position is for a planned **external 12 V DC jack**.

**Both positions need a limit.** The Eurorack position needs one to hold the current budget. The external position needs one to protect the speaker (finding #5).

Current estimate: I ≈ Iq + P_out / (η × 12 V), with Iq = 20 mA typical (35 mA max, SLOS528F §7.6) and η ≈ 0.7 at low power (an estimate). The worst case is a clipped or square-wave input, which Eurorack patches produce easily, where P = Vp² / R_L rather than Vp² / 2R_L.

| Setting | Parts | V_PLIMIT | Output Vpk | Worst-case (square) power | Est. +12 V draw |
|---------|-------|----------|------------|--------------------------|-----------------|
| Current design, limit | R6 = 10k, R8 = 1k | 0.58 V | ≈2.1 V | ≈1.1 W | **≈155 mA, over budget** |
| Current design, full | R6 = 10k | 3.45 V | rail (≈11.5 V) | ≈30 W | ≈1.5–3 A, and speaker damage |
| **Proposed, Eurorack** | R6 = 3.3k, **R8 = 750 Ω** | 0.40 V | ≈1.5 V | ≈0.54 W (≈0.27 W sine) | **≈85 mA** (sine ≈50 mA) |
| **Proposed, external 12 V** | R6 = 3.3k | 1.71 V | ≈6.3 V | ≈10 W (≈5 W sine) | ≈0.5 A sine, about 1.1 A square |

- R6 ∥ R8 = 3.3k ∥ 750 Ω ≈ 611 Ω, so V_PLIMIT = 6.9 V × 611 / 10.6k ≈ 0.40 V. Use 680 Ω for more margin (≈75 mA), or 820 Ω for more volume (≈93 mA).
- These figures are estimates. The 4 × V_PLIMIT rule has about ±13 % spread (§7.5: 6.75–8.75 V at V_PLIMIT = 2 V), and low-power efficiency is not specified. **Measure the +12 V current with a full-scale square wave in limit mode** and trim R8.
- Size the external adapter for at least 1.5 A at 12 V.

**Recommended: select the limit automatically instead of trusting the switch.** If the switch is left in "full" while on Eurorack power, the module will pull about 1 A from the bus. Instead, let the power source choose the mode:
- **DC jack with a switch contact.** Use a barrel jack whose switch pin changes state when a plug is inserted, and wire that contact in place of SW1.
  - With no plug (Eurorack power), R8 is connected and the module draws about 100 mA.
  - With a plug in, R8 is disconnected and the module runs at the ≈5 W speaker-safe setting.
  - SW1 can then be removed, or kept as a user "quiet" override (switch contact and SW1 in parallel).
- **Power-path isolation.** If both supplies can be connected at once, the external 12 V must not back-feed the Eurorack bus.
  - Use the jack's switch contact to disconnect the rail input.
  - Or diode-OR the two inputs, with a Schottky or a second PMOS ideal diode per input.
  - Q1's reverse-polarity protection should cover the DC jack too, because centre-negative 12 V adapters exist.
- The 440 µF of bulk capacitance charges at hot-plug on either input. This is normal for Eurorack, but it is where the 100 mA figure is briefly exceeded.

### Mechanical fit on the panel
- **Width:** the frame's widest point is set by the Ø46 mm ring (the mounting ears fall inside that, at about ±19.3 mm), so the speaker is about 46 mm wide. The 10HP panel is 50.5 mm, leaving about 2.2 mm each side. ✓ Vertically it spans y = 41.25–87.25, which is inside the 34.25–144.25 keep-in and clear of the main board at 124.25. ✓ Depth of about 26 mm is fine for Eurorack.
- **No mounting holes (#8):** the panel Edge.Cuts contain only the Ø40 mm cutout, jack, pot and rack slots. Add 4 × Ø3.3 mm holes (M3, or M2.5 with a clearance margin) at **(122.30, 47.30), (156.20, 47.30), (122.30, 81.20), (156.20, 81.20)**, which is ±16.95 mm around the cutout centre (139.25, 64.25). Check that the countersink or screw heads clear the panel graphics. If the speaker is glued on purpose, ignore this.
- **Surround clearance (#8):** the moving surround extends to roughly the Ø46 mm front ring and moves 2.2 mm forward, but the frame lip is only 1.5 ± 0.5 mm. With a Ø40 mm hole, the panel overlaps the surround, so at high excursion the surround may slap the back of the panel (buzzing).
  - Enlarge the cutout to about Ø46–47 mm, and add a grille if you want protection. Or use a gasket or spacer of at least 1 mm.
  - Measure the actual surround OD on a sample. The datasheet drawing does not dimension it separately.

## Component Summary

**main_board:** 32 components: 15 capacitors, 8 resistors, 3 ferrite/inductors, 1 IC, 1 MOSFET, 1 switch, 2 connectors, 1 speaker header. 22 nets. The only power rails are +12 V and GND.

| Ref | Value | LCSC | Resolved MPN (via `lcsc` skill) | Notes |
|-----|-------|------|------------------------------|-------|
| U2 | TPA3110 | C30132 | TI TPA3110D2PWPR | 8–26 V, HTSSOP-28 PowerPAD |
| Q1 | AO3401A | C15127 | AOS AO3401A | −30 V, **VGS ±12 V**, 4 A |
| L1, L2 | "L_Ferrite" | C58313 | SMDRI74-100NT | **10 µH** shielded power inductor, 1.84 A rated, 49 mΩ |
| C1, C6 | 220 µF | C2977551 | ROQANG RVT1C221M0605 | **16 V**, 105 °C, 95 mA ripple |
| C4, C5 | 470 nF | C13967 | CL21B474KBFNNNE | 50 V X7R |
| C2, C8–C13 | 1 µF | C28323 | CL21B105KBFNNNE | 50 V X7R |
| C14, C15 | 1 nF | C46653 | CL21B102KBCNNNC | 50 V X7R |
| FB3 | ferrite | C1002 | GZ1608D601TF | 600 Ω @ 100 MHz, 200 mA |
| SW1 | Limit | C466006 | XKB SK-3293S | SPDT right-angle slide, 50 mA / 12 V |
| R4 | 10 Ω | C17415 | 0805 | |

J1 (PJ398SM jack), J2/LS1 (headers) and RV1 (Alps RK09K) are hand-fit THT parts and are not in the JLC BOM.

**Sourcing:** The schematic has no MPN fields. LCSC numbers exist only in the JLC plugin database and BOM, so the analyzer reports 0 % MPN coverage (SS-001/DS-001). All LCSC parts above resolved and were in stock on 2026-09-23. Suggestion: add `LCSC` or `MPN` fields to the symbols so the schematic is the source of truth.

**Rail_adaptor:** J1 2×8 Eurorack power header and J2 1×2 socket. **Panel:** 4 plated rack-hole footprints, no electrical content.

## Power Tree

```
Eurorack bus 16-pin ──► Rail_adaptor J1 (pins 9,10 = +12V; 3–8 = GND; 1,2 / 11–16 unconnected)
                           │
                           └─► Rail_adaptor J2 (1 = +12V, 2 = GND)
                                   │  (2-pin mate — see pin-order finding #3)
                                   ▼
main_board J2 (1 = GND, 2 = VIN) ─► Q1 AO3401A (D=VIN, S=+12V, G=GND)  reverse-polarity PMOS
                                   ▼
                                 +12V ── C1, C6 220µF/16V (B.Cu) · C2, C8 1µF · C3, C7 100nF
                                   ├─► U2 PVCCL (27,28) / PVCCR (15,16)
                                   ├─► R4 10Ω ─► AVCC (7) ── C12 1µF          (TI Fig. 46: 10Ω + 1µF ✓)
                                   ├─► R1 100k ─► SD (1) + FAULT (2)          (auto-recovery ✓)
                                   └─► R2 100k ─► PBTL (14)                    (PBTL mode, slew-limit R ✓)
U2 GVDD (9, ≈6.9V) ── C11 1µF
   └─► R5 10k ─► PLIMIT (10) ── R6 10k ∥ C10 1µF ── GND   ⇒ V_PLIMIT ≈ 3.45 V (≈13.8 V virtual rail → no limit at 12 V)
                     └─ SW1 ─ R8 1k ─ GND                 ⇒ V_PLIMIT ≈ 0.58 V (≈ ±2.3 V out → ≈0.65 W into the 4 Ω speaker)
```

There are no regulators on the board. The analyzer's RS-001 ("+12V has no declared source") is expected, because the rail comes in through a connector.

## Analyzer Verification

### Component and net counts
- main_board: schematic has 32 components, PCB has 32 footprints, CPL has 28 SMD rows (plus 4 hand-fit THT parts). Counts match. **Raw-file verified.**
- The PCB reports routing complete with 0 unrouted nets.

### U2 TPA3110D2: pin mapping (datasheet-verified, SLOS528F Table 1, p. 5–6)

| Pin | Name | Net in design | Datasheet requirement | Status |
|-----|------|---------------|-----------------------|--------|
| 1, 2 | SD, FAULT | tied, 100k to +12V | FAULT→SD = auto-recovery (§9.3.9) | ✓ (see note) |
| 3, 4 | LINP, LINN | GND | PBTL: signal goes to the RIGHT input (§9.3.6) | ✓ |
| 5, 6 | GAIN0, GAIN1 | GND | 20 dB, Zi = 60 kΩ (Table 2) | ✓ |
| 7 | AVCC | +12V via 10 Ω, 1 µF | Fig. 46 uses 10 Ω + 1 µF | ✓ |
| 8 | AGND | GND | "Connect to the thermal pad" | ✓ |
| 9 | GVDD | 1 µF | "Add a 1 µF capacitor" (§9.3.5) | ✓ |
| 10 | PLIMIT | 10k/10k divider from GVDD, 1 µF | divider from GVDD + 1 µF (§9.3.4) | ✓ |
| 11, 12 | RINN, RINP | 1 µF to GND / 1 µF from pot wiper | single-ended: AC-ground the unused input with an equal cap (§9.3.3) | ✓ |
| 13 | NC | open | NC | ✓ |
| 14 | PBTL | 100k to +12V | high = PBTL; 100k series limits slew (§9.3.6) | ✓ |
| 15, 16, 27, 28 | PVCC | +12V | | ✓ |
| 17+21 / 22+26 | BS pins | shared 470 nF per side | Fig. 46 PBTL uses 0.47 µF per side | ✓ |
| 18+20 / 23+25 | OUT pins | tied per side | "Connect the positive and negative output together" | ✓ |
| 19, 24, 29 | PGND, PowerPAD | GND, 16 thermal vias to inner GND planes | | ✓ |

The symbol, the footprint (HTSSOP-28 EP with thermal vias) and the PCB pad-to-net data all agree with the datasheet pin table.

Note: SD and FAULT are "compliant to AVCC" (Table 1), but R1 pulls them to +12 V (PVCC) rather than to AVCC. The two differ by only about 0.2 V (20 mA × 10 Ω), so SD sits within VCC + 0.3 V and this is fine in practice. Tying R1 to AVCC would be cleaner.

### Q1: reverse-polarity PMOS
- Symbol `Q_PMOS_GSD` means pin 1 = G, 2 = S, 3 = D. That matches the AOS SOT-23 convention for AO3401A. The datasheet pinout is image-only, so this is **inference, high confidence**.
- The topology is correct: drain to input, source to +12 V, gate to GND. With correct polarity the body diode conducts first, then the channel enhances.
- **Problem (datasheet-verified, AO3401A abs-max table: VGS ±12 V):** once the channel is on, VGS ≈ −V(+12V). Eurorack supplies commonly run at 12.0–12.3 V and ring during hot-plug, so the gate is at or above its absolute maximum all the time.
- **Fix, either of:**
  - Add a 10–100 kΩ resistor in series with the gate (to GND) plus a 10 V Zener (or 9.1 V) from gate to source.
  - Swap to a SOT-23 PMOS rated **VGS ±20 V** (−30 V VDS, RDS(on) < 100 mΩ at −10 V). AO3407A is one candidate; check its current rating and LCSC stock before choosing.

### Connector pin tables

| Board | Ref | Pin | Net |
|-------|-----|-----|-----|
| main | J2 | 1 | GND |
| main | J2 | 2 | VIN (Q1 drain) |
| main | LS1 | 1 | Speaker + (after L1/C14, OUT_L) |
| main | LS1 | 2 | Speaker − (after L2/C15, OUT_R) |
| main | J1 | T / TN / S | signal / GND / GND (unplugged input is grounded ✓) |
| rail | J1 | 1–2 | (−12V) not connected ✓ |
| rail | J1 | 3–8 | GND |
| rail | J1 | 9–10 | +12V |
| rail | J1 | 11–16 | not connected ✓ |
| rail | J2 | 1 / 2 | +12V / GND |

### PCB and Gerber verification
- main_board: the Gerbers (2024-12-22) match the PCB on outline (49.5 × 20 mm), via count (63) and drill set. The CPL positions match the PCB footprints.
- Rail_adaptor: the Gerbers **do not match** the PCB (width 14 mm vs 10 mm). See finding #2.
- Panel: outline 50.5 × 128.5 mm matches 10HP / 3U. Rack slots are 3.2 × 5 mm at 7.5 mm from the left edge and 3.0 mm from the top/bottom. Hole pitch is 35.5 mm (7 HP). All match the Doepfer spec.

## Signal Analysis Review

### Input stage
- RV1/C13 at "15.9 Hz" and R4/C12 at "15.9 kHz" were reported as RC filters. R4/C12 is the AVCC supply filter and matches TI. RV1/C13 is a **false positive**. The real input high-pass is C13 (1 µF) with Zi = 60 kΩ, giving fc ≈ 2.7 Hz, which is fine.
- **Level (finding #6):**
  - Gain is 20 dB (×10 differential). The PBTL output clips at about ±11–12 V, so full scale is reached at about 1.1–1.2 V peak on RINP.
  - Top of pot = V_in × 10k/(10k + 10k) = V_in / 2. A ±5 V Eurorack audio signal therefore gives ±2.5 V on the pot and clips hard over the top half of the pot travel.
  - Hot signals or CVs at ±10–12 V put ±5–6 V on the wiper. AC-coupled onto the 3 V input bias, that is transiently beyond RINP's 6.3 V absolute maximum (SLOS528F §7.1). Current is limited by R3 to under 1 mA, so this is probably survivable, but it is out of spec.
  - **Suggestion:** raise R3 to about 33k. Full scale then lands near 1.2 V peak for a ±5 V input, which also keeps ±12 V inputs inside the pin rating. Alternatively, add a small clamp (for example 2 × 1N4148 to a divided rail) at the wiper.
- The impedances seen by RINP and RINN are only roughly matched: 0–5 kΩ source vs 0 Ω. TI recommends matching them to avoid pop and nuisance DC-detect faults (§9.3.8). The mismatch is small against Zi = 60 kΩ, so this is low risk.
- There is no power-up mute. SD rises with the rail through R1, but TI recommends holding SD low until the inputs settle (§9.3.8). An RC on SD (for example 100k/1 µF, about 100 ms) would reduce the turn-on pop and the risk of a DC-detect latch. A DC-detect fault only clears on a PVCC power cycle.

### Output filter
- The design uses **10 µH inductors** with 1 nF caps. The TI reference (§9.3.1.1) uses **ferrite beads** with about 1 nF caps, with the bead/cap resonance below 10 MHz. The LC here resonates at about 1.6 MHz, which is below 10 MHz and above the ≈310 kHz switching frequency, so it acts as an EMI filter rather than a carrier reconstruction filter. That is fine for a filter-less BD-modulation amp.
- With a load connected, the filter is well damped (Q ≈ 0.04). If the speaker is **unplugged** (LS1 is a header), each leg is an undamped LC near the 5th PWM harmonic. Suggestion: never run the amp without the speaker connected, or add TI's optional 10 Ω + 330 pF snubbers from each output to GND (§9.3.1.1).
- The inductor rating (1.84 A rated, 2.95 A saturation) is **exceeded with the 4 Ω TEBM28C10-4B at full rail** (about 2.9 A peak). It is fine once PLIMIT is set to about 1.7 V (about 1.7 A peak). See [Speaker](#speaker-tectonic-tebm28c10-4b).
- The symbol value "L_Ferrite" is misleading for a 10 µH power inductor. Set the value to `10uH`.

### PLIMIT / Limit switch
- Normal mode: V_PLIMIT = GVDD/2 ≈ 3.45 V. The virtual rail is 4 × V_PLIMIT ≈ 13.8 V, which is above PVCC, so **no limiting**. TI Table 3 shows 6.97 V giving 10.55 W into 8 Ω at 12 V.
- Limit mode (SW1 closed): R6 ∥ R8 = 909 Ω, so V_PLIMIT ≈ 6.9 × 909 / 10.9k ≈ 0.58 V. That gives about ±2.3 V output, roughly 0.65 W into the 4 Ω speaker. This is a working "quiet" mode. **With the TEBM28C10-4B, normal mode must be limited. Change R6 to 3.3k (≈1.71 V), as covered in the Speaker section.**
- GVDD load in limit mode is about 0.63 mA. GVDD is only specified at 100 µA (§7.5), so check that GVDD stays near 6.9 V in limit mode. If it drops, raise R5/R6/R8 proportionally (for example 47k/47k/4.7k).

### Decoupling

| Rail | Caps | Datasheet guidance (§11.1) | Status |
|------|------|----------------------------|--------|
| PVCC | 2 × 220 µF/16 V electrolytic, 2 × 1 µF, 2 × 100 nF | ≥220 µF bulk, 0.1–1 µF mid-frequency, **220 pF–1 nF HF cap at each PVCC end** | Bulk and mid-frequency ✓. **No 220 pF–1 nF HF cap**, so add one at each PVCC pin pair (suggestion). |
| AVCC | 1 µF after 10 Ω | 1 µF in Fig. 46 (text says 10 µF is adequate) | ✓ |
| GVDD | 1 µF | 1 µF | ✓ |
| PLIMIT | 1 µF | 1 µF | ✓ |

The 100 n and 1 µ caps sit 1.8–3.3 mm from U2 on the same side, which is good. The bulk electrolytics are on B.Cu about 9–13 mm away, which is acceptable. The electrolytic ripple rating is 95 mA at 120 Hz, which is low for class-D bulk duty at full power. A higher-ripple part is worth considering if the module will be run hard.

### Simulation Verification
SPICE was not run because ngspice, LTspice and Xyce are not installed in this environment. The passive values above were checked by hand calculation.

## Power Analysis

- **Power budget (finding #11):**
  - Quiescent current is 20 mA typical, 35 mA max (datasheet §7.6).
  - With the 4 Ω speaker at full rail (≈15 W sine), +12 V draw is about 1.4 A average, and more when clipping. With R6 = 3.3k (≈5 W sine), it is about 0.5 A.
  - That is a large share of many Eurorack PSUs, and class-D ripple goes back onto the shared rail.
  - **Suggestion:** state a realistic maximum current on the panel or docs. Also consider making the default PLIMIT a moderate limit (for example V_PLIMIT ≈ 1.8 V, about 5 W per TI Table 3) with the switch selecting full power.
- **Inrush:** there is 440 µF on +12 V behind Q1 and nothing limits inrush. This is typical for Eurorack modules but worth knowing on hot-plug.
- **Thermal:** `analyze_thermal.py` found no regulator or shunt heat sources and skipped. Manual estimate: at about 90 % efficiency, 10 W out means about 1.1 W dissipated. With RθJA ≈ 30 °C/W (JEDEC; this small board is probably worse), ΔT ≈ 35–50 °C. There are 16 thermal vias into two inner GND planes, so this is comfortably under the 150 °C thermal shutdown.

## PCB Layout Analysis

- **Stackup:** 4 layers, F.Cu signal / In1 GND / In2 GND / B.Cu signal with a B.Cu GND pour. EMC rule SU-001 ("adjacent signal layers") is a **false positive**: both inner layers are solid GND zones.
- **Thermal pad:** 16 thermal vias against a minimum of 9 (TV-001). ✓
- **+12V "2 islands" (PS-002 / connectivity graph):** this is a **false positive**. The via at (177.64, 150.56) for C1 is not placed on the endpoint of the F.Cu 0.5 mm track, but its copper overlaps that track (0.24 mm centre offset < 0.55 mm), so KiCad treats it as connected. Suggestion: snap the via onto the track end.
- **Courtyard overlaps (finding #9):** RV1 overlaps C13, Q1 and C3. The RK09K body sits flat on the board, so check in the 3D viewer that these parts clear the pot body and the mounting-tab area.
- **Board-edge clearance (finding #10):** C6 is at 0.25 mm and C1 at 0.35 mm (both electrolytics on B.Cu), and C14 at 0.7 mm. The design rule `min_copper_edge_clearance` is 0.2 mm, so KiCad DRC passes, but 0.25 mm is below common fab minimums (JLC: 0.3 mm routed edge). The J2 and LS1 header courtyards overhang the edge by 0.05 and 0.3 mm, which is fine for headers.
- **Trace widths (finding #7):** +12V uses 0.5 mm on its main runs, but OUTNL uses 0.2–0.25 mm and OUTNR uses 0.25–0.5 mm. Widen the class-D output and PVCC feeds to at least 0.5 mm where space allows. A net class for power/output nets would enforce this.
- **Via-in-pad:** C6 pad 1 has an untented via (VP-001) on an electrolytic pad. Expect some solder wicking; tent or move the via.
- **Assembly:** there are no fiducials (FD-001). JLC does not require them for simple boards; this is informational only. SMD parts are on both sides: 24 on top, and C1, C6, L1, L2 on the bottom. Two-sided assembly costs more at JLC, and moving those four to the top would help if space allows.
- **EMC:**
  - GP-001 and RP-001 flag the speaker output nets (C14/C15 nodes, OUTNR) and the input (FB3, RINP) as having partial reference-plane coverage and layer changes without an adjacent GND via.
  - The board has two inner GND planes, so the return path is always one dielectric away. The residual risk is low, but adding GND stitching vias next to the signal vias on OUTNR, C15-Pad1, RINP and AVCC is cheap.
  - The 23 mm speaker-output run (C15-Pad1) matters most, because the speaker leads are the main radiator. Keep them short and twisted.

## Mechanical: panel ↔ main board

- The panel jack hole (Ø6.2 mm) and pot hole (Ø7.4 mm) are 28.5 mm apart on y = 134.25. The main board's J1 and RV1 centres are also 28.5 mm apart. ✓
- The board is centred on the panel (0.5 mm margin each side). The jack and pot sit on the board's horizontal centreline, so mounting the board component-side toward the panel (flipped about the horizontal axis) keeps the jack on the left, matching the panel. If it were flipped about the vertical axis instead, the pot would be in the jack hole. Make sure assembly notes state the orientation.
- The board's vertical extent (124.25–144.25) ends exactly on the panel's 110 mm keep-in rectangle (`Cmts.User` rect at y = 34.25–144.25), which leaves no clearance to the bottom rail. Check with the actual rail profile.
- SW1 is actuated through the board-edge notch at x = 158–159.4. The panel has no opening for it, so it can only be reached from the side of the module. Confirm that is intended.

## Rail Adaptor

- **Finding #2, stale production files:**
  - `jlcpcb/gerber` is dated 2024-11-21 and shows a 14 × 22 mm outline. The current PCB is 10 × 22 mm, and the newest backup is 2025-02-28.
  - The BOM/CPL list `PinSocket_2x08_P2.54mm_Vertical_SMD` (LCSC C30734, which is actually a THT female header) at (155.65, 50.9). The PCB footprint is `PinHeader_2x08_P2.54mm_Vertical` THT at (152.8, 41.8).
  - Regenerate all fab outputs before ordering, and decide whether the intent is a male header (ribbon cable) or a female socket (plug directly onto the bus board).
- **Reverse-insertion hazard:** if the 16-pin connection is made 180° reversed (unshrouded header, or a socket straight onto the bus):
  - The adaptor's GND pins 3–8 land on bus pins 9–14, **shorting +12 V and +5 V to GND** on the bus.
  - Use a **shrouded/keyed** header, and put a clear "−12V / red stripe" marking on silk.
- **Finding #3, 2-pin polarity:** Rail_adaptor J2 is 1 = +12V / 2 = GND, but main_board J2 is 1 = GND / 2 = VIN. If the socket mates straight onto the header, the polarity is reversed and the module will not power up (Q1 blocks it).
  - Check the physical mating geometry.
  - Better: use the same pin order on both boards and add +/− silk labels.

## Power interconnect polarity

See Rail Adaptor above. Q1 protects the amplifier against a reversed 2-pin connection, but it cannot protect the *bus* against a reversed 16-pin connection on the adaptor.

## Fabrication outputs

- **Finding #4:** each `*-VScore.gbr` is exported from `Cmts.User` (`%TF.FileFunction,Other,Comment`). It contains:
  - main_board: 913 draws, including crosshair lines **through the board** at x = 168.5, x = 197 and y = 143.25, plus outline rectangles.
  - Panel: 2416 draws of dimension and annotation graphics.

  If the fab treats this file as V-cut data, the board is ruined. At minimum it will cause an engineering query. Move the annotations to `User.Drawings`, or remove `*-VScore.gbr` from the zips for boards with no V-cuts.
- main_board Gerbers are dated 2024-12-22, but the latest PCB backup is 2025-06-19. The via count, outline and CPL all still match, but regenerate before ordering to be sure.
- GR-004 (B.Paste has 8 flashes vs 92 copper pads) is expected: the bottom side has only 4 SMD parts.
- GR-002 (outline/copper extent mismatch) comes from the edge notch and from copper-free keep-outs. It is benign.
- Check the CPL rotations for Q1 (SOT-23), U2 (HTSSOP) and C1/C6 (electrolytics) in JLC's placement preview. These are the parts most often mis-rotated by the export.

## Component Lifecycle

A distributor lifecycle audit (`--lifecycle`) was not run because there are no MPN fields in the schematics. The `lcsc` lookup confirmed that every JLC BOM part is active and in stock (2026-09-23). TPA3110D2PWPR had 12.9k in stock.

## False Positives / Reviewer Overrides

| Finding | Why dismissed or downgraded |
|---------|-----------------------------|
| PP-001 "U2.7 AVCC has no DC path to a power rail" | AVCC is fed from +12 V through R4 (10 Ω). The analyzer does not pass through resistors. |
| RC-DET RV1/C13 15.9 Hz | Coupling cap into Zi = 60 kΩ; the pot is not the filter R. |
| RS-001 "+12V has no declared source" | The rail comes from the J2 connector. Expected. |
| PS-002 / connectivity "+12V 2 islands" | Via overlaps the track and KiCad connects it. Only cosmetic. |
| SU-001 "adjacent signal layers" (main) | In1 and In2 are full GND planes. |
| GP-001/GP-002/VS-001 (rail adaptor, panel) | The rail adaptor carries DC only and the panel has no circuitry. |
| XV-001 panel "component in PCB not schematic" | Rack-hole footprints (REF**). Expected. |
| EP-AUD / IO-001 no ESD on J1/J2 | A Eurorack audio jack with a 10k series resistor and ferrite is conventional; the 1 µF coupling cap and internal clamps are adequate. Low risk. |
| DS-001/SS-001 sourcing blockers | LCSC numbers exist in the JLC BOMs. Downgraded to a documentation suggestion. |

## Not Performed / Review Limits

- **SPICE:** not performed because no simulator is installed. Passive values were checked by hand.
- **Thermal analyzer:** ran but skipped (no regulator or shunt sources). A manual estimate is given above.
- **Lifecycle audit:** not performed (no MPNs in the schematics). LCSC stock was checked manually.
- **Datasheets:**
  - TPA3110D2 (TI SLOS528F), AO3401A and the Tectonic TEBM28C10-4B / -4BE datasheets were read in full. SK-3293S, SMDRI74-100NT and RVT1C221M0605 were checked through LCSC parameters only.
  - The AO3401A pinout diagram is image-only, so the Q1 pin mapping is inference based on the standard AOS SOT-23 G/S/D order.
  - The PJ398SM, RK09K and header footprints were not checked against vendor drawings.
- **Prior review delta:** no previous review or analysis runs exist.
- **KiCad ERC/DRC:** not re-run because `kicad-cli` is not available. The project's rule severities were read from `.kicad_pro`.
- **Panel ↔ main-board fit:** checked from 2D coordinates only. The STEP models were not checked for collisions.

## Verdict

**main_board: nearly ready, fix before the next fab run.** The amplifier core is a faithful implementation of TI's PBTL reference and is datasheet-verified pin for pin. Must fix: Q1's gate over-voltage (#1), and speaker protection (#5: R6 → 3.3k, C9/C13 → 22 nF). Should fix: the input level (#6), output current and trace width (#7), the pot courtyard collisions (#9), the electrolytic edge clearance (#10) and the V-score layer contents (#4).

**Rail_adaptor: not ready to order.** Its fab outputs do not match the PCB (#2), and its 2-pin pin order is opposite to the main board's (#3).

**Panel: needs a small revision.** Add the 4 speaker mounting holes and enlarge the speaker cutout to about Ø46–47 mm (#8). Also fix the V-score layer contents (#4). The outline and the jack/pot cutouts match the main board and the Eurorack spec, and the speaker fits the 10HP width.
