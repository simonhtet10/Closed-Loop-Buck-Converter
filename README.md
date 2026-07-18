# Closed-Loop-Buck-Converter
12 V → 5 V buck converter with an analog Type III compensator, designed from first principles, characterized on the bench and prototyped fully on a breadboard. Measured loop gain/phase via Middlebrook injection: 2.9 kHz crossover, ~100° phase margin, against design targets of 10 kHz and 65°. Regulates 5.0 V across 10–14 V input and 0.5–1.0 A load at ~87% efficiency. The dominant deviations from the design are traced to measured physical causes.


<img width="1594" height="786" alt="50mVBode" src="https://github.com/user-attachments/assets/44ff8684-95ec-4869-881a-b7d15cb99e46" />

### Results

| Parameter| Designed Specification | Measured Value |
| --- | --- | --- |
| Switching Frequency | 100kHz | 71.6kHz |
| Cutover Frequency| 10kHz | 3.1kHz |
| Phase at crossover | ~115° | 78° |
| Phase Margin | ~65° | ~102°|
| Line Regulation | N/A | 9.25 mV/V (0.74% over a 40% Vin swing) |
| Load Regulation | N/A | 0.74% over a 2× load step (Zout ≈ 74 mΩ)|
| Efficiency at 0.5A | N/A | ~87% |
| SW node overshoot | N/A | 3% (~12.4 V peak) — no snubber required |

Measurement was validated two ways: repeating the sweep at 2× injection amplitude shifted phase at crossover by 0.5°, confirming small-signal operation; and a rebuild of the injection setup reproduced a crossover frequency within 7% and phase within 2.4°.

### Design Approach

The plant was a standard LC buck filter -a 100µH inductor with 29mΩ ESR and 100µF capacitor- giving an LC double pole at 1592Hz and a ESR zero at 66.3kHz. The LC double pole contributes up to 180° of phase lag through resonance, more than a single-zero Type II compensator can recover.The Type III's dual-zero structure provides the phase boost needed to cross over with adequate phase margin. 

Rather than digital, an analog implementation was chosen because the compensator's poles and zeroes and physicla components rather than firmware coefficients- the transfer funciton can be probed directly and compared against its design, which is the deliverable of the project. 

The compensator places its first zero at the LC double pole to cancel it, a second zero at 800 Hz (half the crossover frequency) for additional phase boost below crossover, one pole at 50 kHz to roll off ahead of the switching frequency, and a second pole at the ESR zero to cancel it.

The components are not independently solvable.R1·C1 had to be back-solved from the loop gain requirement than taken from the standard equation in TI's app notes, which is circular. Solved instead from the constraint that the compensator must supply at least 1/[plant(fc)] at the crossover frequency gives us R1·C1 directly. Full derivations are stored in docs/design-notes.md.

### Measurement Methodology

Loop gain was measured by a Middlebrook voltage injection - a 100Ω resistor inserted in series between the output node and the compesnator's input network, with the AD2's waveform generator injecting a small AC disturbance through a DC blocking capacitor. The Network Analyzer sweeps thsi injection and computes the ratio of the two probe channels, yielding gain and phase versus frequency directly.

Middlebrook injection requires the impedance looking backward from the injection point to be much smaller than the impedance looking forward. The output impedance can be derived from the load regulation measurement, is approximately 74mΩ, while the forward impedance into the compensator is approximately 10kΩ. The 100Ω injection sits between them, being 1350 times larger and 100 times smaller respectively. 100Ω in series with 10kΩ introduces a 1% shift in the feedback divider's upper leg, which has a calculated effect of 25mV on the DC operating point. 

The measurement's linearity was verified by repeating the Bode sweep at half amplitude. Phase at crossover frequency matched to within 0.5° between the two truns, confirming that the loop was not being driven into duty-cycle clipping.

One thing worth noting for reproducing this measurement: both compensator input branches must be moved to the injection node. Leaving either branch fed directly from the output never breaks the loop and the measurement returns unity gain with zero phase shift.

Raw sweeps at both 50mV and 100mV are in measurements/.

### Measured Deviations from Design Specifications
The crossover landed at roughly a third of the target frequency. About half of that deficit is accounted for by modulator gain: the design assumes a 2.0V peak to peak triangle wave, and the oscillator as built produces 3.6V, which costs about 5.1dB for loop gain. The reduced switching frequency accounts for a further portion. The remainding gap can be attributed to component tolerances and the loop gain being lower than the idealized model predicted - this portion is not fully accounted for and is just reported as is.

The large phase margin is caused by th elow crossover. At 3.1kHz the loop crosses below the region where the LC double poles contributes significant phase lag, while both compensator zeros have already delivered most of their phase boost. Crossing in the compensator's maximum-boost region naturally produces a large margin. This means the loop is unconditionally stable but heavily overdamped with a slower transient recovery than the 65° target, which was chosen to balance the two.

### Oscillator's Frequency Limit
The oscillator was designed for 100kHz bu settles at ~71.6kHz with a hysteresis band nearly three times wider than intended. This was caused by the op-amp's slew rate: swinging the full supply rail takes ~3.3µs agianst a target period of 10µs, so the comparator spends a large fraction of every half-cycle mid transition. The integrator continues ramping past the ideal threshold before the transition fully registers, widening the hysteresis band.

Tuning the integrator's resistor produced diminishing returns
| R_int| Hysteresis Span | Frequency | Waveform |
| --- | --- | --- | --- |
| 33kΩ | 2.2V | 40kHz | Clean triangle|
| 10kΩ | 3.5V | 71kHz | Straight edges,rounded peaks- chose this  |
| 4.7kΩ | 5.7V | 87kHZ | Fully sinusoidal |

Each halving of R_int bought progressively less frequency while the hysteris span kept growing faster and past a point, the triangle degraded into a sinusoid. With a linear ramp the mapping of the error output to the duty cycle is proportional- a the error voltage is halfway up the ramp gives 50% duty cycle, moving the error voltage by 10% moves the duty cycle by 10%. A rounded ramp breaks that so the aggressive tuning points were rejected despite yielding a frequency closer to target. The selected value keeps rounding confined to the peaks so the converter's nominal operating point sits on the linear section of the ramp.

With this circuit topology, reaching 100kHz would require a dedicated fast comparator rather than a general op-amp, which is a compoentn-level limit.

### Notable Bugs
**Inverting Schmitt trigger**

The oscillator failed to start, with one stage latched at the rail and the other pinned to ground. The resistor arrangement produced and inverting Schmitt trigger which creates a  one-way latch- a falling triangle drives the square wave in the direction that pushes the triangle further down so the circuit never triggers its own rising transition. Rewiring the circuit to a non-inverting topology resolved this issue.

**Missing compensator zero** 

After closing the loop, the output was limit cyclced at 415mV p-p while the error amplifier swung across most of its range. One branch of the input network was removed when doing the Middlebrook injection , which collapsed the Type III to a Type II compensator, which reduced the phase margin by 45 to 90°. Installing the branch reduced the error amplifier's activity by 22x and removed the osicllation.

**Two extended debugging sessions turned out to be instrumentation and not the circuit**

A bench supply current limit set too low starved the rail and produced several misleading symptoms across the gate driver and power stage - including an apparently latched MOSFET that tested perfectly healthy independently. Separately, saturated network analyzer inputs returned an identical meaningless trace across four different injection circuit configurations until the DC offset was nulled. Both are documented in blank as distinguishing measurement issues from circuit behavior is a substantial amount of the work.

### Repo Contents

docs/design-notes
docs/build-guide
docs/bringup-log
measurement/
simulation/
hardware/





