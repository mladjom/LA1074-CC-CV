# LT1074 Adjustable CV/CC Bench Power Supply

A buck-derived bench power supply built around the LT1074CT1 switching regulator, with adjustable output voltage and adjustable current limiting, plus a digital voltage/current readout.

## Overview

- **Topology:** LT1074CT1 buck converter
- **Voltage control:** Adjustable via potentiometer (RV1), feeding the LT1074's FB pin
- **Current control:** Adjustable fold-back current limit via an op-amp sense/compare loop, summed into the LT1074's VC (compensation) pin
- **Reference:** TL431-based VREF (2.5V), used by both the current-set pot and the current-sense op-amp
- **Readout:** DSN-VC288 digital voltmeter/ammeter module

## Main power stage

| Component | Function |
|---|---|
| U1 – LT1074CT1 | Buck regulator (VIN pin5, VSW pin4, FB pin1, VC pin2, GND pin3) |
| L1 | Buck inductor |
| D1 – MBR745 | Catch (freewheeling) diode |
| Cin1 | Input bulk capacitor (470µF) |
| COUT | Output bulk capacitor (470µF) |
| RV1 (5K) | Output voltage adjustment — sets FB divider ratio, adjusts VOUT |
| R2 (0.1Ω) | Current-sense shunt, in the ground return path |

**Output voltage** is set by RV1, which forms the feedback divider into the LT1074's FB pin (regulated against its internal ~2.21V reference). Turning RV1 sweeps VOUT from a few hundred mV up to close to VIN.

## Current-limit loop

| Component | Function |
|---|---|
| R2 (0.1Ω) | Current-sense shunt |
| U1A (LM358) | Difference amplifier — converts the voltage across R2 into a single-ended ISENSE signal (via R3/R4) |
| RV2 (10K) | Current setpoint (ISET), derived from VREF |
| U1B (LM358) | Compares ISENSE vs ISET, output = ICONTROL |
| R5 (1K) + clamp diode | Sums ICONTROL into the LT1074's VC pin |
| C3 (10nF) | Filtering on ICONTROL |

**How it works:** Under normal operation (load current below the ISET threshold), U1B's output (ICONTROL) sits high, well above VC's normal operating range. The clamp diode between R5 and VC is wired **anode toward VC, cathode toward R5/U1B**, so it stays reverse-biased and the current loop has no effect on regulation — the LT1074 is free to run pure CV mode.

If load current exceeds the ISET threshold, ISENSE rises above ISET, U1B's output swings low — below VC's level — forward-biasing the clamp diode. This lets U1B pull VC down through R5, reducing the LT1074's duty cycle and folding back the output voltage to hold current at the ISET limit. This is a standard current fold-back scheme, not a hard/foldback-free limiter — output voltage will sag as needed to keep current at the set point.

**Diode direction matters:** the opposite orientation (cathode toward VC) permanently forward-biases the diode any time ICONTROL idles high, jamming VC upward and preventing CV regulation entirely. If voltage ever "gets stuck near VIN and won't adjust," this is the first thing to check.

## Reference and setpoint

- **U2 (TL431LP) + R6 + C4** generate VREF (~2.5V), used as the reference rail for RV2 (current setpoint).
- **RV2 (10K)**, referenced to VREF and PGND, sets ISET — the current threshold at U1B's non-inverting input.

## Digital meter (DSN-VC288)

The module has 5 wires: thin red, thin black, yellow, thick red, thick black.

**Wiring (as built):**

| Wire | Connects to | Purpose |
|---|---|---|
| Thin red | VCC | Module power (+) |
| Thin black | **VOUT− node** (same point as thick black — not the general PGND bus) | Module power (−) / measurement reference |
| Yellow | VOUT+ | Voltage sense input |
| Thick red | Load negative terminal | Current path in |
| Thick black | VOUT− node (R2's load-side junction) | Current path out, through internal shunt (R018, 0.018Ω) |

**Current path order:** `VOUT+ → load → module thick red → internal shunt (R018) → module thick black → VOUT− → R2 (0.1Ω) → PGND`

R2 and the module's internal shunt are in series, so both should read the same current. This depends on thin black sharing the exact same node as thick black — if thin black is instead tied to the general PGND bus, current can bypass R2 through the module's internal ground connection and both R2 and the display will read incorrectly.

**Calibration:** The module has trim pots on the back (`V_ADJ`, `I_ADJ`) for fine calibration against a known reference. As built, current readout runs roughly 6–8% high across the tested range (0.85–1.29A vs. calculated actual) — within expected tolerance for this class of module, adjustable via `I_ADJ` if tighter accuracy is needed.

## Known limitations / notes

- Module power (30V max on all VC288 pins) — stay within that if this supply is ever run at a higher input voltage.
- The module's internal shunt (0.018Ω) adds a small amount of extra series resistance beyond R2 — negligible at low current, more noticeable near the 10A ceiling of the module.
- R2's current-sense node (used by the CC loop) is physically distinct from — but in series with — the meter's shunt. Both are valid current references; R2/U1A is what actually closes the CC loop, the VC288 is readout only.
