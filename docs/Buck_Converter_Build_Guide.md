# Buck Converter — Breadboard Build Guide (As-Built)

12 V → 5 V voltage-mode buck with an analog Type III compensator, built on an 830-tie breadboard and characterized with an Analog Discovery 2.

This is the *as-built* guide: it reflects what the finished converter actually is, including the values that changed during bring-up (the oscillator retuned to ~71.6 kHz, the Schmitt trigger rewired to non-inverting, Cint built from paralleled caps). Where a value differs from the original paper design, the reason is noted. The build order is sequenced so every subsystem's dependencies exist before it is tested.

> **Read the "Lessons / Gotchas" box at the end of each phase before powering up.** Several of them cost hours during the original build.

---

## Bench Supply Current Limit — set before every stage

| Stage | Current limit | Why |
|---|---|---|
| 1–4 (signal chain) | 100 mA | No power stage; nothing should draw more than a few mA |
| 5 first power-up | 150 mA | Cautious first switching |
| 5 open-loop check (loaded) | 600 mA | Load draws ~250 mA; leaves headroom |
| 6–7 closed loop, 0.5 A | 600 mA | — |
| 7 loaded, 1.0 A | 1.5 A | — |

**A supply in CC mode is reporting its *actual* limit.** If it reads CC at 50 mA, the limit is 50 mA regardless of what you meant to set. Verify the setpoint with a brief short-circuit test (leads together, output on, read the clamped current) before trusting it. A mis-set 50 mA limit produced an entire evening of false "gate driver / MOSFET" symptoms during the original build — every one of which was just a starved rail.

---

## Phase 0 — Layout & Rail Setup

1. **Zone the board.** One end = power components (MOSFET, Schottky, inductor, input/output caps). Other end = signal components (TLC2274, LM393, TL431, compensator passives). IR2104 sits at the boundary, close to the MOSFET gate. The zone split is an organizational choice along the length of the board.

2. **Verify rail continuity.** Many full-size boards have a break in the middle of each rail. Using the AD2 (Supplies = 3 V into one end + Voltmeter at the far end), confirm each rail reads continuous. Bridge any break with a jumper wire.

3. **Connect the bench supply** on the side nearest the power zone (minimizes the supply-lead loop). Red → red rail, black → blue rail, adjacent pair. Output OFF, 12.0 V, 100 mA limit set.

4. **Place the 470 µF input bulk cap** — long leg to red rail, stripe to blue rail — physically adjacent to where the MOSFET drain will sit, not near the supply entry. This is part of the high-dI/dt switching loop and placement matters. It's your only one.

5. **Bypass every IC** with a 100 nF ceramic across its supply pins as you place it. You have ~10; use them liberally.

---

## Phase 1 — Voltage Reference (TL431)

Build first. Both the oscillator and the PWM comparator need Vref.

1. Spread the TL431's TO-92 legs gently (grip at the base, bend gradually, test-fit often). **Confirm pinout against your specific datasheet** — Cathode/Reference/Anode order varies by manufacturer.

2. Place across three separate rows (each pin its own node).

3. **Wire it — shunt-regulator configuration:**
   - Anode → GND rail
   - Cathode → jumper → Reference pin (tie directly together)
   - **4.7 kΩ** from 12 V rail → Cathode row

   Bias current = (12 − 2.5)/4.7 k ≈ **2.0 mA**, safely above the ~1 mA minimum. (A 10 kΩ here gives only 0.95 mA — too close to the dropout floor.)

4. **Test:** AD2 Voltmeter on the Cathode/Reference node. Expect **2.49–2.50 V**. Don't proceed until solid.

> **Note:** the TL431 uses a single bias resistor, *not* a divider. Cathode and Reference tie together directly.

---

## Phase 2 — Oscillator (TLC2274 Sections A + D)

Section A = Schmitt trigger, Section D = integrator. Section C is reserved for the compensator (Phase 6), so the oscillator uses A and D and leaves C free.

Confirm the TLC2274 pinout from the datasheet. Bypass cap across V+ (pin 4) and V− (pin 11).

### Section D — Integrator (pins 12/13/14)

- **IN+ (pin 12):** Rmid1 (100 kΩ) from 12 V, Rmid2 (100 kΩ) to GND, 100 nF bypass to GND → sets ~6 V mid-rail bias
- **IN− (pin 13):** Rint from Section A's output; Cint from Section D's own output
- **OUT (pin 14):** the triangle wave; feeds Cint back and drives Section A's input resistor

**Rint = 10 kΩ** (retuned — see box). **Cint = 940 pF**, built as **two 470 pF C0G in parallel** across the same two rows (the design's 1 nF was not in stock; 2×470 pF is −6 %, absorbed by the tuning).

### Section A — Schmitt trigger, NON-inverting (pins 1/2/3)

Three resistors converge on IN+; IN− goes to the reference:

- **IN− (pin 2):** jumper → TL431 Vref node (2.5 V)
- **IN+ (pin 3):** three resistors —
  - **4.7 kΩ** from Section D's triangle output (pin 14)
  - **47 kΩ** from Section A's own output (pin 1) — positive feedback
  - **33 kΩ** to GND
- **OUT (pin 1):** feeds the 47 kΩ back to IN+, and feeds Rint into Section D

Measured thresholds: **1.91 V / 3.11 V**.

**Test:** Scope on pin 14 (triangle) and pin 1 (square). Expect a triangle centered ~2.46 V and a rail-to-rail square wave, both at **~71.6 kHz**. Set the scope range wide enough (5 V/div) to see the full 0–12 V square — at 500 mV/div it clips and looks wrong.

> **Gotchas / lessons:**
> - **The Schmitt must be NON-inverting.** An inverting arrangement latches: a falling triangle drives the square HIGH, pushing the triangle further down, and it never recovers (one stage pinned at 12 V, the other at ~16 mV). The 3-resistor network above is the correct non-inverting topology.
> - **~71.6 kHz, not 100 kHz — and that's the accepted final value.** The TLC2274's ~3.6 V/µs slew rate can't swing the rail fast enough for a clean 100 kHz triangle. Lowering Rint raises frequency but widens the hysteresis band and rounds the waveform; below ~10 kΩ the triangle degrades to a sinusoid, which breaks the linear Ve→duty mapping the compensator needs. 10 kΩ keeps the rounding confined to the peaks, with the ~42 % duty operating point on the clean linear section. Reaching 100 kHz would require a dedicated fast comparator, not tuning.
> - **Rb from the original parts list is unused** in the non-inverting topology.
> - Congested IN+ node (4 items in a 5-hole row): jumper to an adjacent empty row to extend it.

---

## Phase 3 — PWM Comparator (LM393)

Bypass cap across V+ (pin 8) and GND (pin 4).

- **IN− (pin 2):** Rin− (1 kΩ) from the triangle (Section D output)
- **IN+ (pin 3):** Rin+ (1 kΩ) from the Vref node *(temporary — moves to Ve in Phase 6)*; Rhys (680kΩ) from OUT back to IN+
- **OUT (pin 1):** Rpull (2.2 kΩ) to 12 V (open-collector pull-up); this is the probe point

**Test:** Scope on OUT. Expect a clean ~71.6 kHz square wave.

> **Gotchas / lessons:**
> - **Rhys sets DC hysteresis width, not switching speed.** Widening it does *not* cure edge chatter — that was proven by a 15× change with zero effect. The residual ~445 ns dip at each edge is the LM393 dwelling in its linear region as the slow triangle crosses threshold. It is benign: the dips bottom out at ~5 V, well above the IR2104's 3 V logic-high threshold, and the driver's propagation delay filters them. **Left unfixed on purpose.**
> - Keep Rhys near its original 820 kΩ. Dropping it to tens of kΩ loads the open-collector output and pulls the high level down.

---

## Phase 4 — Gate Driver (IR2104)

**Verify the IR2104PBF pinout from the datasheet — it differs from the IR2109.** Bypass cap across VCC and COM.

- VCC → 12 V, COM → GND
- **SD → 12 V** (tied high = enabled; if low, both outputs stay off)
- IN → LM393 output
- Bootstrap diode (1N4148): **anode → 12 V, cathode (stripe) → VB**
- Bootstrap cap (1 µF): between VB and VS
- **VS → temporarily jumper to GND** (no switching node yet; gives the bootstrap a defined reference). Moves to the SW node in Phase 5.
- HO → open (probe point). LO → unused.

**Test:** Scope on HO. Expect a clean 0–12 V square at ~71.6 kHz tracking the LM393. The edge chatter from Phase 3 should be absent here — the driver filters it.

> **Gotchas:** a floating IN (loose jumper) drifts HO permanently high. If HO is stuck high with IN good, suspect the bootstrap. But note: HO will *also* appear stuck if the rail is starved — check the supply current first (see the current-limit box).

---

## Phase 5 — Power Stage

Shortest wires here — this is the loop-area-critical section. **Power down and disconnect leads before wiring.**

1. **Gate resistor (10–22 Ω)** between HO and the MOSFET gate. Don't skip it — damps gate-node ringing.
2. **IRLZ44N:** confirm Gate/Drain/Source from the datasheet. Drain → 12 V rail (short, near the input cap). Source → **SW node** (new row).
3. **Move the VS jumper** from GND to the SW node.
4. **1N5819 Schottky:** cathode (stripe) → SW, anode → GND. Physically adjacent to the MOSFET. *(Substituted for the 1N5821 — DO-201's packaging leads are too thick for a breadboard. 1N5819 is DO-41 and gives 40 V vs 30 V, better overshoot margin although has less current margin.)*
5. **100 µH inductor:** SW → VOUT (new row). Not polarized.
6. **100 µF output cap:** + to VOUT, stripe to GND.

The **SW node** should carry four things: MOSFET source, Schottky cathode, inductor, IR2104 VS.

**Open-loop test (bootstrap needs a load — see box):**
Connect the 10 Ω load across VOUT→GND. Power at 600 mA. Scope the SW node.
- Square wave, ~0 → 12 V, ~71.6 kHz
- Overshoot ~3 % (≈12.4 V peak) — no snubber needed on this build
- Undershoot ~−0.3 V (Schottky Vf) — confirms clean freewheel
- VOUT ≈ 4.8 V open loop (D × Vin, no feedback yet)

> **Gotchas / lessons:**
> - **The bootstrap will not refresh at no load.** VS only gets pulled low by inductor freewheel current; with no load, current decays to zero, VS floats, the bootstrap cap depletes, and HO latches high (converter appears dead). **Connect the load before the first power-up.** This is not a fault — it's inherent to an asynchronous buck with a bootstrap driver.
> - If the SW node rings badly (>25–30 V overshoot): add a 10 Ω + 1 nF RC snubber across the Schottky. This build didn't need it.
> - A MOSFET pulled for testing holds gate charge and reads shorted until discharged (short gate-to-source for a few seconds). Don't condemn a FET without discharging first.

---

## Phase 6 — Compensator & Closing the Loop (TLC2274 Section C)

Section C = pins 8/9/10.

- **IN+ (pin 10):** jumper → Vref (2.5 V)
- **IN− (pin 9), summing node:** from VOUT — R1 (10 kΩ), **and** R3 (330 Ω) + C2 (10 nF) in series. *Both* branches land here.
- **Feedback (pin 8 = Ve → pin 9):** R2 (5.1 k, built as 4.7 k + 470 Ω) + C1 (39 nF) in series, **in parallel with** C3 (470 pF)
- **R4 (10 kΩ):** summing node → GND (sets Vout = 2.5 × (1 + R1/R4) = 5.0 V)

**Close the loop:** move the LM393 IN+ input from the Vref stub to **Ve** (relocate Rin+'s far leg from the Vref node to Ve).

**Test (0.5 A load, 600 mA limit):**
- VOUT tightens to **5.0 V** regulated
- Ve sits mid-rail (3–9 V) and *moves slightly* — a quiet, moving Ve is a working loop
- VOUT on the scope (short ground lead, probe at the output cap): flat DC + tens of mV ripple, **no slow large wobble**

> **Gotchas / lessons:**
> - **The summing node has 6 items in a 5-hole row** — extend to an adjacent row with a jumper.
> - **Do not omit the R3 + C2 branch.** Without it, Zi collapses to R1, the Type III degenerates to Type II, and you lose the zero at the LC pole (45–90° of phase boost at crossover). Symptom: a limit cycle — VOUT swinging ~400 mV and Ve swinging ~2.4 V p-p at ~12–15 kHz. Installing the branch cut Ve activity 22× and killed the oscillation.
> - An unloaded loop reads a misleading operating point (Ve near a rail, tiny input current). Always evaluate with the load connected.

---

## Phase 7 — Load & Line Characterization

**Line regulation:** VOUT via Voltmeter at 10 / 12 / 14 V in, 0.5 A load. Keep Vin > ~7 V or duty saturates.

**Load regulation:** VOUT at 0.5 A and 1.0 A (parallel a second 10 Ω), 12 V in. Raise the current limit to 1.5 A first as current draw increases.

**Ripple:** scope VOUT with a *short* ground lead right at the output cap. Long ground loops read switching spikes that aren't really on the rail.

**Thermals @ 1 A:** MOSFET barely warm (I²·Rds_on ≈ 28 mW), Schottky also barely warm (~230 mW), load resistor hot (2.5–5 W — by design).

Measured on this build: line reg ≈ 9 mV/V (~0.7 %), load reg ≈ 0.7 % (Zout ≈ 74 mΩ), efficiency ≈ 87 % @ 0.5 A.

> **Gotcha:** the load resistor runs at 80–120 °C surface temp by design — DO NOT grab it (learnt by experience), and keep it off the breadboard plastic and wire insulation.

---

## Phase 8 — Loop Gain (Middlebrook Injection)

Measures the actual crossover frequency and phase margin.

**Wiring:**
- **Rinj (100 Ω)** in series between VOUT and the compensator input node ("node B")
- **Both** Zi branches (R1 and R3+C2) fed from node B, not VOUT
- **W1 → 1 µF DC-blocking cap → node B** (blocks the ~5 V DC; passes the AC)
- **Ch1 → node B**, **Ch2 → VOUT**, grounds short and common

**Network Analyzer:**
- 500 Hz – 100 kHz, 100 steps, 50 mV amplitude
- **Channel 1 and Channel 2 offset = −5 V** ← required
- "Relative to Ref" ON (plots Ch2/Ch1 = loop gain)

**Read:** cursor to the 0 dB crossing → that's fc; phase there gives PM = 180° − φ(fc).

Measured on this build: **fc ≈ 3.1 kHz, PM ≈ 102°.** Validated by (a) re-running at 100 mV — phase agreed within 0.5°, confirming small-signal operation; (b) a full teardown/rebuild reproducing fc within 7 % and phase within 2.4 %.

> **Gotchas / lessons — this measurement produced the most false starts of the whole build:**
> - **Set both channel offsets to −5 V.** The channels sit at 5 V DC; the analyzer auto-ranges to resolve the small injection and the DC saturates the inputs. Symptom: a flat ~0 dB / ~0° trace, unchanged no matter what you adjust. This was mistaken for four different wiring problems before the offset was found.
> - **Both Zi branches must move to node B.** Leaving either fed from VOUT shorts across Rinj — the loop is never broken and you get the *same* flat 0 dB / 0° trace (a second, independent cause of the identical symptom).
> - **Diagnostic that disambiguates:** disconnect W1 and re-run. If the trace drops to the noise floor, injection works and the fault is on the measurement side (offsets). If it's unchanged, W1 isn't reaching the node.
> - Ground-referenced injection through a blocking cap works fine here — no isolation transformer needed — but it *is* a compromise vs. true floating series injection. The reproducibility and linearity checks are what justify trusting it.
> - Middlebrook validity holds: Zout ≈ 74 mΩ ≪ 100 Ω ≪ 10 kΩ (forward impedance). DC operating-point shift from Rinj ≈ 25 mV.

---

## Quick Reference — As-Built Substitutions

| Design value | As built | Reason |
|---|---|---|
| Schottky 1N5821 (DO-201) | 1N5819 (DO-41) | Thick leads didn't fit the breadboard; 40 V > 30 V is a bonus |
| Cint 1 nF | 2 × 470 pF ∥ (940 pF) | 1 nF not stocked; −6 % absorbed by tuning |
| Rint 22 kΩ | 10 kΩ | Retuned empirically for ~71.6 kHz given the slew-limited swing |
| R2 5.1 kΩ | 4.7 kΩ + 470 Ω (5.17 kΩ) | Series combination from the 1 % kit |
| Ra 82 kΩ | 47 kΩ + 33 kΩ (80 kΩ) | Series combination |
| Rhys 820 kΩ | 680 kΩ  | Adjusted during Phase 3 debugging |
| TL431 bias 10 kΩ | 4.7 kΩ | 0.95 mA → 2.0 mA, safe margin above dropout |
| Schmitt (inverting) | 3-resistor non-inverting | Original arrangement latched and wouldn't oscillate |
