# Power Chat Notebook

Living knowledge base for this chat. Updated whenever there's something worth keeping.

**Active build:** Portable Bench Power Supply — see `psu_plan.md` (design locked, A-B-C-D architecture).

## Foundations (locked in)

The big three, and they're one equation:

- **Voltage (V)** — the push/pressure. Fixed by source (220V wall, 5V USB, 3.3V MCU, 3.7V cell).
- **Current (I)** — the flow. DRAWN by the device, not pushed by the supply.
- **Power (P)** — the work. `P = V × I`

Formulas (know any 2, get the 3rd):
- `P = V × I` — watts
- `I = P / V` — amps
- `V = P / I` — volts

## Words that come up

- **Stall** — motor asked to move but blocked. Pulls MAX current, heats fast, burns out. Worst-case number for wiring/supply sizing.
- **Peak vs idle** — peak/stall is the ceiling (brief), idle is normal running.
- **Safety factor / headroom** — supply rated at **1.5–2×** max draw. Covers surges, aging, sag, inrush spikes.

## Current scale (food analogy — the intuition)

Current needs "gram intuition". This is the personal scale:

| Scale | Food | Current | Real examples |
|---|---|---|---|
| µA (microamp) | grain of salt | 0.000001 A | MCU sleep, RTC, capacitor leak |
| mA (milliamp) | pinch of spice | 0.001 A | LEDs, sensors, earbuds running, HC-SR04 (15mA) |
| few hundred mA | spoon of sugar | 0.1–0.5 A | phone on call, SG90 idle, laser, Blue Pill |
| 1 A | bowl of rice | 1 A | phone charging, SG90 peak, LED strip |
| 2–10 A | full meal | 2–10 A | laptop, kettle (~9A), iron, mixer, AC |
| tens of A | feast / whole house | 20–60 A | full house on 220V India mains |

**Mental rule:** under ~200mA → can run off MCU 3.3V pin. Above that → needs a real rail. Why servos (750mA each) get their own supply.

## Real device quantities (known values)

House 220V 50Hz — the current ISN'T fixed, it changes with what's switched on. Supply is rated (India single-phase ~15–30A into house); current is drawn.

| Device | Voltage | Current |
|---|---|---|
| LED bulb | 220V | ~0.05–0.1 A (few W) |
| Ceiling fan | 220V | ~0.25 A (~60W) |
| TWS earbuds | 3.7V / 5V charge | run ~5–10mA/bud, charge ~100mA |
| Mobile | 5V charge | 1–2 A; fast charge more |
| Laptop | 19–20V | 2–3 A (45–65W) |
| Desktop PC | 220V | 1.5–2.5 A (300–500W) |
| Kettle / iron | 220V | ~9 A (2000W) |

Pattern: bigger physical size → bigger power → bigger current.

## Multimeter usage

- **Voltage = parallel** (across a live point). Black in COM, red probe on point. Reversed probes just show minus.
- **Current = series** (break the circuit, current flows THROUGH meter). Mixing these up = blown fuse / damaged meter.
- **Resistance = unpowered only.** OL = open/broken.

## Electricity bill basics (India)

- **1 unit = 1 kWh** = 1000W running 1 hour. `units = watts × hours / 1000`
- Slab-based tariff: first N units at lower rate, then higher. Rates vary by state.
- Bill = energy charge (units × slab rate) + fixed charge + duty/tax.

## Why the MCU reboots when servos move (brownout)

1. Servos over-draw the shared supply
2. Supply can't deliver → **voltage sags** (5V rail dips well below 5V)
3. MCU sees voltage below minimum → **brownout reset** → reboot

The servo doesn't steal the MCU's budget directly — over-drawing the shared supply makes voltage sag, and the MCU dies from low voltage. Fix: **separate servo rail + shared ground** so the MCU rail never sags.

## MCU pin basics (Blue Pill STM32F103)

- Supply VDD = 3.3V. Reads logic HIGH above ~2V (~65% VDD), LOW below ~1V (~30% VDD).
- **Input pins sense VOLTAGE, not current** — near-infinite impedance, µA. Never feed power into a pin.
- **Output pins are weak:** ~25mA absolute max, keep ≤10mA. Do NOT drive servos/motors.
- LED on a pin needs a resistor: `R = (3.3 − LED drop) / wanted current`.
- Under-voltage → brownout RESET (protection). Over-voltage/current/polarity → permanent death.
- Rail sag does NOT kill the MCU — it resets it. Fear over-voltage, not under-voltage.

## Why each rail type (module fits the current)

- **Buck (switching) = high current.** From 12V bus, linear would burn `(12−Vout)×I` as heat — 14W @5V/2A. Buck at ~90% loses ~1W. Use for 5V/3A (servos) and the variable XL4015.
- **Linear = low current only.** AMS1117-3.3 feeds the MCU (~100mA): `(5−3.3)×0.1 = 0.17W` heat, fine. Cascade it off the **5V rail**, not the 12V bus (5× less heat).
- **CV/CC on the variable rail:** voltage knob = output, current knob = ceiling. Set CC below what your prototype *should* draw → a wiring fault clamps current instead of burning.
- **BMS sizing:** rated ≥ 2× continuous draw. 3S pack @ ~1.8A → 10A board (not the common 25A). BMS watches each cell; one weak cell = entire pack cuts off.
- **Fuse sizing:** `~1.5–2× daily max`, sits above burst. 2A continuous → 5A fuse.

## C-rate & pack current limits

- `continuous current = capacity × C-rate`. 1800mAh × 1C = **1.8A**; bursts ~2C (3.6A) for seconds.
- In a series pack EVERY cell carries the full load current → pack limit = one cell's limit.
- BMS rating: `≥ 2 × continuous` → 3.6A → **10A board**. It must be the right cell count (3S). Verify the real sustained limit with the Phase-0 sag test.

## Linear vs buck energy math

- Linear = series resistor. `P_heat = (V_in − V_out) × I` → 12→5V @2A = **14W heat**, and input current = output current (24W in for 10W out).
- Buck = fast switch + inductor. Input power ≈ output power; current steps down too (2A@5V pulls only ~0.95A @12V). Heat ~1W, runtime ~40% longer on battery.
- Why bucks cost more: controller IC (feedback loop) + MOSFET + inductor + diode/caps = a control *system*. Linear = 1 transistor (₹5). Paying for control, not voltage.

## Fuse basics

- Fuse = sacrificial thin wire; `heat = I²×R×t`. Normal = cool, over-current = melts, opens the circuit.
- Not instant — brief bursts pass (3A servo peak OK on a 5A fuse); sustained over-current pops at ~1.5× rating.
- Holder = contacts + slot; makes fuse replaceable and encloses the live ends.
- Placement: series on the **+ wire, right at the pack** → BMS P+ → fuse → XL4015 +IN. P− runs straight. Fuse protects the whole wire run downstream. Rated ~1.5–2× daily max, above burst.

## CC vs CV (the two-pot lesson)

- **CV = constant voltage** — most buck modules (incl. ₹79 single-pot XL4015, LM2596): set the voltage, it holds it no matter the load. NO current limit of its own → "5A" only means what the chip can *survive*, not a cap you set.
- **CC = constant current cap** — the current-limit knob; if the load tries to draw more than the set point, the module **drops voltage** to hold current at the cap. Protects both the prototype and the 1800mAh pack.
- "Built-in current limit function" ≠ adjustable: on the plain board that's the *chip's* internal ~7A trip + short hiccup. Doorknob, not a knob — a 3–6A sustained draw slips past it and the fuse (5A) won't pop.
- Buy rule: if you need a current cap, the module must have **two pots (CV-ADJ + CC-ADJ)** and a sense resistor (0.05Ω). Verify the board, not the listing slogan.

## Continuity is not resistance (the beep trap)

- Continuity mode beeps for anything under ~**50Ω**. A burnt switch contact at 20Ω "beeps fine," then drops the entire supply under load.
- Real test: **ohms range**, subtract probe-to-probe offset. A switch or wire path should read **< 0.5Ω**.
- `V_drop = I × R`. 20Ω at 1A = 20V of drop; at 10mA it's 0.2V. That's exactly why it passes at meter current and fails at load.

## BMS cutoff — how to recognise it

- Resting 8V, then **~1.2V at the output** the instant a load connects = the BMS has tripped. That residual is FET leakage, not a supply voltage.
- Cutoff is **microseconds** → no sustained current → **no heat**. If nothing warms up, it's a protection trip, not a short.
- Reset: remove the load, apply charge voltage briefly.
- A BMS that keeps tripping at rest voltage = a cell is already below the cutoff line.

## Flat pack vs dead cell — the 8.09V trap

- Total voltage cannot tell them apart:
  `2.70 + 2.70 + 2.69 = 8.09V` (recoverable) and `3.75 + 3.75 + 0.59 = 8.09V` (dead cell).
- Always measure per cell: `B1−B−`, `B2−B1`, `B+−B2`. **Negative reading = cell inserted backwards.**
- Margin maths: 2.67V/cell resting vs 2.5V cutoff = **0.17V margin**. A 0.2Ω cell at 1A sags 0.2V → instant trip.
- **High internal resistance is invisible on a voltmeter.** "Cells are okay" measured at rest means voltage only — not capacity, not IR.

## Standing drain of an indicator LED

- `I = (V − Vf) / R`. On 12.6V with 1kΩ: `(12.6 − 2)/1000 = 10.6mA`.
- `10.6mA × 24h = 254mAh/day` → an 1800mAh pack is flat in **~7 days**.
- With 10kΩ: 1.06mA → ~72 days.
- Any indicator that stays lit while the box is "off" is a drain. Put it after the switch, or raise the resistor.

## Double-feeding (one source per board)

- Two regulators on one node: the higher setpoint pushes current **into** the other. `I = ΔV / R_total`. 5.05V vs 5.00V through 0.1Ω = **0.5A backwards**.
- Back-feeding a PC USB port is worst when the host is asleep/off — current leaks through the port's protection diodes into the host rails.
- Wall adapters are worse: 12.6V pack vs 12.0V adapter through 0.25Ω = **2.4A into the adapter's output**.
- Rule: **one 5V source per board.** Adapter plugged ⇒ battery switch OFF.

## Why a buck heats during a CC short test

- Output ≈ 0V so output power ≈ 0, yet the board burns ~1.5W. Loss scales with **current**, not output voltage.
- `P_loss = I²(R_shunt + R_inductor) + Vf × I × (1 − D)`. With the output shorted D≈0, so the freewheel Schottky carries full current continuously: `2A × 0.5V = 1.0W`.
- Normal 12→5V @2A loses ~1.1–1.4W — the CC test is no worse. Warm is fine; keep the short to 2–3 seconds.

## Voltage drop = the fault locator

- Measure both ends of a suspect path **under load**: `ΔV = I × R`, `P = ΔV × I`.
- 0.1V across a joint at 1A = 0.1W (fine). 1V at 1A = 1W inside a 1cm part (hot).
- Two nodes joined by <0.5Ω **cannot** read different voltages. If they seem to, you measured at different times — the source is collapsing, not the wire.

## Why a "1A" boost charger only gives 0.4A

- A boost converter **converts power, it does not create it**: `P_out ≈ P_in × efficiency` (~0.85).
- So input current is always **higher** than output current when boosting:
  `1A × 12.6V = 12.6W` out needs `12.6 ÷ 0.85 ÷ 5V = 3.0A` in at 5V.
- A 5V **1A** brick (5W) caps output at `5 × 0.85 ÷ 8.1 ≈ 0.5A`.
- Measured case: 0.41A on a **5V 2A** brick, which should manage ~1A at 8.1V — so the
  cap was **not** the brick. Suspects left: **USB cable voltage drop** (check the
  module's USB-C pads, want >4.8V), the module's own **~0.8A input-current limit**
  (these boards assume a 5V/1A source, so the "1A" spec is often unreachable), or a
  factory trimpot set low. Always measure before blaming the supply.
- Consequence: **charge current falls as pack voltage rises** (constant input power):
  0.41A @8.1V → 0.31A @10.5V → 0.27A @12.4V. Normal, not a fault.
- To actually get 1A you need a **5V 2–3A** supply. Never a PD C-to-C cable — an
  un-negotiated module sees 0V, not more current.
- C-rate: `I ÷ capacity`. `0.41A ÷ 1.8Ah = 0.23C` — gentle, best for cell life, just slower.

## Prove each path alone before assembling

- The whole week was spent inside a **fully assembled box**, where every fault looks
  like every other fault. The fix took minutes once the charge path was tested on its
  own: pack + BMS + fuse + charger, nothing else.
- Rule: **prove the charge path → prove the discharge path → only then add switches,
  modules, LEDs, outputs.** A half-wired box is a box you can debug.
- Corollary: **"adapter mode works, battery mode doesn't" already localises the fault.**
  The adapter bypasses the switch, so a working adapter path proves modules + outputs +
  grounds are fine, and points straight at the battery branch.

## Balance and charge level are two different facts

- **3.82V/cell on all three = perfectly balanced.** That is what "cells are okay"
  actually means — the *spread*, not the absolute number.
- **3.82V/cell is also only ~55% charged.** Full is ~4.15–4.20V/cell.
- So one number answers two questions: `3.82 / 3.82 / 3.82` = matched + half full.
  `4.2 / 4.2 / 3.6` = full but unbalanced (and the low one is the weak one).
- **3.75–3.85V/cell is the ideal long-term storage voltage** (40–60% SoC). Lithium ages
  fastest sitting full or empty. A pack parked at 3.8V/cell will outlive one parked at 4.2V.
- Corollary: don't "top it up before storing" — charge it when you're about to use it.

## Rules that must stay

- MCU must NOT power servos (separate higher-power rail).
- Buffer: servos get their own supply; ground is SHARED (all grounds tied).
- Never measure current across a live supply in voltage mode.
- Supply sizes: worst-case draw × 1.5–2 headroom, shared ground.
- **A beep is not a low resistance.** Measure ohms before blaming a switch or joint.
- **Pack floor = 10.5V (3.5V/cell).** The 2.5V BMS cutoff is last-resort protection, not a fuel gauge.
- **One 5V source per board.** Adapter plugged ⇒ battery switch OFF.