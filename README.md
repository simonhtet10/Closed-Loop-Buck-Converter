# Closed-Loop-Buck-Converter
12 V → 5 V buck converter with an analog Type III compensator, designed from first principles, characterized on the bench and prototyped fully on a breadboard. Measured loop gain/phase via Middlebrook injection: 2.9 kHz crossover, ~100° phase margin, against design targets of 10 kHz and 65°. Regulates 5.0 V across 10–14 V input and 0.5–1.0 A load at ~87% efficiency. The dominant deviations from the design are traced to measured physical causes.


<img width="1594" height="786" alt="50mVBode" src="https://github.com/user-attachments/assets/44ff8684-95ec-4869-881a-b7d15cb99e46" />

**Results**

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

**Design Approach**

The plant was a standard LC buck filter -a 100µH inductor with 29mΩ ESR and 100µF capacitor- giving an LC double pole at 1592Hz and a ESR zero at 66.3kHz. The LC double pole contributes up to 180° of phase lag through resonance, more than a single-zero Type II compensator can recover.The Type III's dual-zero structure provides the phase boost needed to cross over with adequate phase margin. 

Rather than digital, an analog implementation was chosen because the compensator's poles and zeroes and physicla components rather than firmware coefficients- the transfer funciton can be probed directly and compared against its design, which is the deliverable of the project. 

The compensator places its first zero at the LC double pole to cancel it, a second zero at 800 Hz (half the crossover frequency) for additional phase boost below crossover, one pole at 50 kHz to roll off ahead of the switching frequency, and a second pole at the ESR zero to cancel it.

The components are not independently solvable.R1·C1 had to be back-solved from the loop gain requirement than taken from the standard equation in TI's app notes, which is circular. Solved instead from the constraint that the compensator must supply at least 1/[plant(fc)] at the crossover frequency gives us R1·C1 directly. Full derivations are stored in docs/design-notes.md.

**Measurement Methodology**

Loop gain was measured by a Middlebrook voltage injection - a 100Ω resistor inserted in series between the output node and the compesnator's input network, with the AD2's waveform generator injecting a small AC disturbance through a DC blocking capacitor. The Network Analyzer sweeps thsi injection and computes the ratio of the two probe channels, yielding gain and phase versus frequency directly.

Middlebrook injection requires the impedance looking backward from the injection point to be much smaller than the impedance looking forward. The output impedance can be derived from the load regulation measurement, is approximately 74mΩ, while the forward impedance into the compensator is approximately 10kΩ. The 100Ω injection sits between them, being 1350 times larger and 100 times smaller respectively. 100Ω in series with 10kΩ introduces a 1% shift in the feedback divider's upper leg, which has a calculated effect of 25mV on the DC operating point. 

The measurement's linearity was verified by repeating the Bode sweep at half amplitude. Phase at crossover frequency matched to within 0.5° between the two truns, confirming that the loop was not being driven into duty-cycle clipping.

One thing worth noting for reproducing this measurement: both compensator input branches must be moved to the injection node. Leaving either branch fed directly from the output never breaks the loop and the measurement returns unity gain with zero phase shift.

Raw sweeps at both 50mV and 100mV are in measurements/.




