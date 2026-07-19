# Design Notes — Type III Compensator Derivation

Complete design derivation for the 12 V → 5 V voltage-mode buck converter. This documents the paper design; where the as-built converter deviates (switching frequency, oscillator tuning), see the build guide and README. All frequencies here are the *design* targets.

---

## 1. System Parameters

| Parameter | Value |
|---|---|
| Input voltage | 12 V |
| Output voltage | 5 V |
| Output current | 1 A |
| Switching frequency (design) | 100 kHz |
| Output ripple target | ≤ 50 mV |
| Control | Voltage-mode, analog Type III compensator |

The converter is a step-down switcher: it switches the input on and off at high frequency and filters the result to a lower average output. The open-loop gain is

$$T(s) = G_c(s)·G_{vd}(s)·H(s)$$

where G_c is the compensator, G_vd the plant (control-to-output), and H the feedback sensor gain.

---

## 2. Power Stage Component Selection

### Inductor

Duty cycle: D = Vout / Vin = 5/12 ≈ 0.417.

Inductor value from the allowed ripple current (standard target is 20–40% of I_max):

$$L = \frac{(V_{in}-V_{out})\,D}{\Delta I_L · f_{sw}} = \frac{7 \times 0.417}{0.3 \times 10^5} \approx 100\ \mu H$$

Selected: **RL-5480HC-4-100** (100 µH, 2.45 A, 33 mΩ DCR).

### Output capacitor

For a switching regulator the ripple is dominated by the capacitor's ESR:

$$V_{ripple} \approx \Delta I_L \times R_{ESR}, \qquad V_{ripple} = \frac{\Delta I_L}{8·f_{sw}·C}$$

Requiring V_ripple ≤ 50 mV sets both a capacitance floor and an ESR ceiling:

$$C \ge 7.5\ \mu F \;(\text{use } 100\ \mu F), \qquad R_{ESR} \le \frac{50\text{ mV}}{0.3\text{ A}} \approx 167\ m\Omega$$

Selected: **Panasonic 16SEPC100M** (100 µF, 24 mΩ ESR max — comfortably under the 167 mΩ ceiling).

---

## 3. Plant Frequencies

With L = 100 µH, C = 100 µF, R_ESR ≤ 66.7 mΩ:

**LC double pole:**

$$f_{LC} = \frac{1}{2\pi\sqrt{LC}} = \frac{1}{2\pi\sqrt{100\mu \cdot 100\mu}} \approx 1591\text{ Hz} \approx 1.59\text{ kHz}$$

**ESR zero:**

$$f_{ESR} = \frac{1}{2\pi R_{ESR} C} = \frac{1}{2\pi (24\text{m})(100\mu)} \approx 66.3\text{ kHz}$$

The plant transfer function, including the ESR zero:

$$G_{vd}(s) = V_{in} \cdot \frac{1 + s R_{ESR} C_{out}}{s^2 L C_{out} + s\left(\frac{L}{R} + R_{ESR} C_{out}\right) + 1}$$

---

## 4. Why Type III

The LC double pole introduces up to **180° of phase lag** through resonance. A Type II compensator (one zero) cannot recover enough of that to cross over with margin. A Type III has **two zeros** and can provide up to 180° of phase boost — enough to counter the LC pole and cross over with the target margin.

**Target phase margin: 60°.** Target crossover: f_c = 10 kHz (≈ f_sw/10, the standard choice — high enough for good transient response, low enough to stay well below the switching frequency).

At 10 kHz the loop is already past the LC pole's −180° region, so the compensator's zeros must be positioned to have delivered their boost by crossover.

---

## 5. Pole / Zero Placement

Standard Type III placement for this plant:

| Element | Frequency | Placement rule | Purpose |
|---|---|---|---|
| f_z1 | 1.59 kHz | at f_LC | cancel the LC double pole |
| f_z2 | 800 Hz | at 0.5·f_LC | extra phase boost below f_c |
| f_p1 | 50 kHz | between 0.5·f_sw and f_sw | HF roll-off before switching |
| f_p2 | 66.3 kHz | at f_ESR | cancel the ESR zero |

---

## 6. Component Values (Using Texas Instrumental's SLVA662 Type III Compensator procedure)

Op-amp Type III topology: input network Zi = R1 ∥ (R3 + C2), feedback network Zf = (R2 + C1) ∥ C3, with Vref on the non-inverting input.

**Convention used here:** the 800 Hz zero (f_z2) is assigned to the input branch and the 1.6 kHz zero (f_z1) to the feedback branch, per the SLVA662 mapping. The circuit itself doesn't care which physical branch produces which zero — this is just the bookkeeping the equations follow.

**Set R1 = 10 kΩ** as the design anchor. With Vref = 2.5 V and Vout = 5 V, the lower divider resistor follows from R1/R4 = 1:

$$R_4 = 10\ k\Omega \quad\Rightarrow\quad V_{out} = V_{ref}\left(1 + \frac{R_1}{R_4}\right) = 5.0\text{ V}$$

### From the input branch (Zi): f_z1 = 1590 Hz, f_p1 = 50 kHz

$$R_3 = \frac{R_1 f_{z1}}{f_{p1} - f_{z1}} = \frac{10k \cdot 1590}{50k - 800} \approx 328\ \Omega \;\rightarrow\; \mathbf{330\ \Omega}$$

$$C_2 = \frac{f_{p1} - f_{z1}}{2\pi R_1 f_{p1} f_{z1}} = \frac{50k - 1590}{2\pi \cdot 10k \cdot 50k \cdot 1590} \approx 9.6\text{ nF} \;\rightarrow\; \mathbf{10\ nF}$$

### The circular-dependency problem with R2

The TI closed-form expression for R2 contains **f_p0** as an input. But f_p0 is set by C1, and C1 depends on R2 — the equation references its own answer. **R2 cannot be solved directly.**

Instead, R1·C1 is **back-solved from the loop gain requirement**: at crossover the compensator must supply exactly the gain the plant lacks.

### Back-solving R1·C1

First, the required compensator gain at f_c. The plant's control-to-output magnitude at 10 kHz:

$$|G_{vd}(f_c)| = V_{in}\left(\frac{f_{LC}}{f_c}\right)^2 \sqrt{1 + \left(\frac{f_c}{f_{ESR}}\right)^2} = 12\left(\frac{1590}{10k}\right)^2\sqrt{1 + \left(\frac{10k}{66.3k}\right)^2} \approx 0.307$$

For unity loop gain the compensator must supply the inverse:

$$|H(f_c)| = \frac{1}{0.307} \approx 3.25 \;(+10.2\text{ dB})$$

The Type III magnitude, with the C1 ≫ C3 approximation, factors into a zero-boost numerator N, a pole-rolloff denominator D, and the integrator factor K = 2π f_c R1 C1:

$$|H(f_c)| = \frac{N}{K \cdot D}, \qquad
N = \sqrt{1+\left(\tfrac{f_c}{f_{z1}}\right)^2}\sqrt{1+\left(\tfrac{f_c}{f_{z2}}\right)^2} \approx 79.8, \qquad
D = \sqrt{1+\left(\tfrac{f_c}{f_{p1}}\right)^2}\sqrt{1+\left(\tfrac{f_c}{f_{p2}}\right)^2} \approx 1.03$$

Solving for K:

$$K = \frac{N}{|H(f_c)|\, D} = \frac{79.8}{3.25 \times 1.03} \approx 23.78$$

$$R_1 C_1 = \frac{K}{2\pi f_c} = \frac{23.78}{2\pi \cdot 10k} \approx 378.5\ \mu s$$

### C1, R2, C3 follow in sequence

$$C_1 = \frac{R_1 C_1}{R_1} = \frac{378.5\mu}{10k} = 37.85\text{ nF} \;\rightarrow\; \mathbf{39\ nF}$$

$$R_2 = \frac{1}{2\pi f_{z2} C_1} = \frac{1}{2\pi \cdot 800 \cdot 37.85n} \approx 5.26\ k\Omega \;\rightarrow\; \mathbf{5.1\ k\Omega}$$

$$C_3 = \frac{1}{2\pi f_{p2} R_2} = \frac{1}{2\pi \cdot 66.3k \cdot 5.26k} \approx 457\text{ pF} \;\rightarrow\; \mathbf{470\ pF}$$

### Final compensator values

| Part | Exact | E24 | Sets | Placed frequency |
|---|---|---|---|---|
| R1 | 10 kΩ | 10 kΩ | gain, f_p0 | f_p0 = 408 Hz |
| R2 | 5.26 kΩ | 5.1 kΩ | f_z2, f_p2 | f_z2 = 800 Hz |
| R3 | 329 Ω | 330 Ω | f_z1, f_p1 | f_p1 = 48.2 kHz |
| R4 | 10 kΩ | 10 kΩ | Vout | 5.00 V |
| C1 | 37.85 nF | 39 nF | f_p0, f_z2 | f_z2 = 800 Hz |
| C2 | 9.68 nF | 10 nF | f_z1, f_p1 | f_z1 = 1541 Hz |
| C3 | 457 pF | 470 pF | f_p2 | f_p2 = 66.4 kHz |

The C1/C3 ratio of ~83× validates the C1 ≫ C3 approximation used in the derivation. Estimated phase margin from this placement: ~65°.

---

## 7. Op-Amp Selection — Why TLC2274, Not TL074

The TL074 cannot run on a single 12 V supply here. Its input common-mode range has a ~4 V dropout from the negative rail — on a 12 V single supply, the lowest usable input is ~4 V. But every critical node in this circuit sits *below* that:

| Node | Voltage | TL074 usable? | TLC2274 |
|---|---|---|---|
| Oscillator thresholds | 2.0–3.0 V | ✗ | ✓ |
| Error amp IN+ (Vref) | 2.5 V | ✗ | ✓ |
| Error amp IN− (feedback) | ~2.5 V | ✗ | ✓ |
| Error amp output Ve | 2.0–3.0 V | ✗ | ✓ |

The **TLC2274** is rail-to-rail input (0–12 V on a single supply), GBW = 2.2 MHz (open-loop gain at f_c is ~68× above the required compensator gain), and its max supply of 16 V covers the 12 V rail. All within spec.

---

## 8. Modulator (PWM Comparator) Gain

The comparator compares Ve against the oscillator ramp. Modulator gain is set by the ramp amplitude:

$$K_{pwm} = \frac{V_{in}}{V_{ramp}} \left(\frac{f_{LC}}{f_c}\right)^{-2}\ \text{(embedded in the plant gain above)}$$

The design assumed a 2 V peak-to-peak ramp (V_M = 1 V). **The as-built oscillator produces a ~3.6 V ramp**, which lowers modulator gain by ~5.1 dB and is the dominant reason the measured crossover landed below the 10 kHz design target — see the README.

---

## 9. Design vs. As-Built

This document is the *design*. The physical converter deviates in ways traced in the README and build guide, most importantly:

- **Switching frequency 71.6 kHz, not 100 kHz** — limited by the op-amp oscillator's slew rate.
- **Ramp amplitude 3.6 V, not 2 V** — a consequence of the same slew limit, costing ~5.1 dB of loop gain.
- **Component substitutions** (R2 as 4.7 k + 470 Ω, Cint as 2×470 pF, etc.) — see the build guide's substitution table.

Measured result: crossover ≈ 3.1 kHz, phase margin ≈ 102°. The over-damped margin is a direct consequence of the reduced crossover, since at 3.1 kHz the loop crosses in the compensator's maximum-boost region.
