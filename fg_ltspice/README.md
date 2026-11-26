# BEE 425 Function Generator: LTspice files

Six LTspice schematics, one per block of the board, plus a shared model file and custom symbols. Every net uses the **same name as on the KiCad board** (DAC_OUT, FILT_OUT, SUM_SIG, OFF_BUF, OUT_DRV, BNC, …), so a value in a table below can be compared directly with a probe on that net.

## How to use

1. Unzip **everything into one folder** (the `.asc`, `.asy` and `fg_models.lib` files must sit together; LTspice finds the custom symbols and models there).
2. Open an `.asc` in LTspice and press **Run**.
3. Results of the `.meas` lines appear in **View → SPICE Error Log**. Click a net to plot it.
4. To change a test condition, edit the `.param` line on the sheet (frequency, waveform, amplitude, offset, load).

| File | Block | Analysis | What to change |
|---|---|---|---|
| `01_power_rails.asc` | Jack → SS14 → R-78E (+5V), R1 (+VA), LT1054 + Q1 + R2 (−VA) | `.op` (DC voltages) | `VJACK`, load currents `I5V`, `IVA` |
| `02_dac_and_reference.asc` | REF3025 → U6B inverter → VREF_N; DAC7821 → U6A I-to-V (C16, D4) | `.tran` | `f`, `Vpk`, waveform `Ksin/Ksq/Ktri` |
| `03_filter_response.asc` | Sallen-Key filter U7A (R8, R9, C12, C13) | `.ac` (Bode plot) | — |
| `04_offset_pwm.asc` | PWM pin 29 → R10/C22 → U7B buffer → OFF_BUF | `.tran` (150 ms settle) | `Duty` |
| `05_summer_dc.asc` | U9A summer (R11–R14) → R15 → BNC → load | `.dc` sweep of SUM_SIG | `Voffbuf`, `RL`, `C23` |
| `06_full_chain.asc` | Code → DAC → I-to-V → filter → ADG719 → summer → BNC | `.tran` | `f`, `Vpk`, `Ksin/Ksq/Ktri`, `SWSEL`, `Voff`, `RL` |

**Waveform selection (sheets 02 and 06):** set exactly one of `Ksin`, `Ksq`, `Ktri` to 1 and the others to 0. For square, also set `SWSEL=1` (the ADG719 bypasses the filter, as the firmware does).

**Net-name mapping:** `VA_P` = +VA, `VA_N` = −VA, `P5V` = +5V (LTspice labels avoid leading + and − signs). `B_CODE` stands in for the Teensy writing the 12 DAC data lines at 2 MS/s.

## Expected values

All numbers below come from running these exact files (the netlists were rebuilt from the `.asc` geometry and run in ngspice). They are what a correct board should show, within the tolerances noted. `expected_waveforms.png` shows the waveforms.

### T1. Power rails (`01_power_rails.asc`, loads: 100 mA on +5V, 38 mA on each of ±VA)

| Net (board name) | Expected | Bench note |
|---|---|---|
| JACK (D1 anode) | 9.00 V | Whatever the adapter gives (regulated 9 V) |
| VIN_9V | 8.65 V | One Schottky drop below the jack |
| +5V (P5V) | 5.00 V | |
| D3 LED anode | 1.95 V | About 3 mA through R3 |
| +VA (VA_P) | 8.27 V | ±0.2 V depending on load |
| Q1 emitter (LT1054 VOUT) | −8.23 V | |
| NEG_RAW (Q1 collector) | −8.16 V | Q1 saturated: about 0.07 V across it |
| −VA (VA_N) | −7.78 V | Expect −7.3 to −7.9 V; depends on the LT1054's real output resistance (10 Ω assumed) |
| Jack current | 154 mA | |

### T2. Reference and DAC (`02_dac_and_reference.asc`, Vpk = 5 V)

| Net | Expected |
|---|---|
| VREF_P | +2.500 V |
| VREF_N | −2.500 V |
| DAC_OUT range (sine, triangle or square) | 0.090 V to 2.410 V (codes 147 to 3948); formula DAC_OUT = 2.5 V × code / 4096 |
| IOUT1 (virtual ground) | Within ±1 mV for sine/triangle; brief ±41 mV spikes at square steps. D4 never conducts. |

### T3. Filter (`03_filter_response.asc`)

| Quantity | Expected |
|---|---|
| Gain at 1 kHz | 1.000 (0 dB) |
| Gain at 100 kHz | 0.993 (−0.06 dB) |
| Peak gain | 1.0005 near 45 kHz (Q = 0.72 gives a tiny bump) |
| −3 dB frequency | 256 kHz (design 253 kHz) |
| Gain at 1 MHz | 0.064 (−24 dB) |

### T4. Offset path (`04_offset_pwm.asc`)

OFF_BUF = 3.3 V × duty. Output offset at the summer = 5.386 V − 3.233 × OFF_BUF, so **duty 0.5048 gives 0 V offset**.

| Duty | OFF_BUF average | OFF_BUF ripple (p-p) |
|---|---|---|
| 0.20 | 0.660 V | 0.96 mV |
| 0.505 | 1.667 V | 1.54 mV |
| 0.90 | 2.971 V | 0.62 mV |

The RC time constant is 15 ms, so after an offset change, wait about 75 ms before measuring.

### T5. Summer (`05_summer_dc.asc`, OFF_BUF = 1.6658 V)

OUT_DRV = −4.309 × SUM_SIG − 3.233 × OFF_BUF + 10.772 V.

| SUM_SIG | OUT_DRV | BNC into 1 kΩ | BNC open | BNC into 50 Ω |
|---|---|---|---|---|
| 0 V | +5.386 V | +5.130 V | +5.386 V | +2.696 V* |
| 1.25 V | 0.000 V | 0.000 V | 0.000 V | 0.000 V |
| 2.5 V | −5.386 V | −5.130 V | −5.386 V | −2.696 V* |

\*Into 50 Ω the OPA1612 would be supplying about 54 mA, right at its ~55 mA current limit. The model has no current limit, so on the bench, expect clipping at or below these values.

### T6. Full chain (`06_full_chain.asc`, RL = 1 kΩ unless noted)

| Case | Settings | OUT_DRV max / min | BNC max / min | Other |
|---|---|---|---|---|
| A | Sine 10 kHz, Vpk 5 | +5.00 / −4.99 V | +4.76 / −4.75 V | |
| B | Triangle 10 kHz | +4.93 / −4.92 V | +4.70 / −4.69 V | Corners slightly rounded by the filter |
| C | Square 10 kHz, SWSEL=1 | +5.00 / −4.99 V | +4.76 / −4.75 V | Edge −3.7 → +3.7 V in **0.30 µs** (op-amp slew limit) |
| D | Square 10 kHz, SWSEL=0 (filter **not** bypassed) | +5.47 / −5.46 V | +5.21 / −5.20 V | ~9% overshoot, 1.3 µs edge: this is why square uses the bypass |
| E | Sine 100 kHz | +4.94 / −4.93 V | +4.70 / −4.69 V | 1.2% droop (filter + sample-and-hold) |
| F | Triangle 100 kHz | +4.35 / −4.34 V | +4.14 / −4.13 V | Filter removes the sharp corners (known limitation, issue 31) |
| G | Square 100 kHz, SWSEL=1 | +5.00 / −4.99 V | +4.76 / −4.75 V | Same 0.30 µs edges |
| H | Sine 10 kHz, Vpk 2, Voff +2 V | +4.00 / +0.01 V | +3.81 / +0.01 V | Average 1.91 V at the BNC |
| I | Sine 10 kHz, open load | +5.00 / −4.99 V | +5.00 / −4.99 V | |

**Why the BNC reads lower than OUT_DRV:** R15 (49.9 Ω) and the load form a divider: 1000 / 1049.9 = 0.952.

**Why there's about 5 mV of offset:** the ADG719's ~4 Ω on-resistance adds to R11 (4.99 kΩ), so the mid-scale term is 0.08% weaker. Firmware calibration can remove it.

**The summer inverts.** The BNC waveform is upside-down relative to DAC_OUT. The firmware table accounts for this.

## Limitations

- The op-amps, DAC, switch, reference, regulator and LT1054 are **behavioural models** built from datasheet headline numbers, not vendor models. They're good for DC levels, gain, filter response and waveform shape. They don't model op-amp current limit, DAC glitches, switch charge injection, or the LT1054's 25 kHz ripple.
