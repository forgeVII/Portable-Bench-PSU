# PSU Build Guide — v7 AS-BUILT + FAULT LOG

> **STATUS (2026-11-17): ✅ COMPLETE — BUILT AND WORKING (v9).**
> Adapter mode ✅ · Battery mode ✅ · Both rails 5V ✅ · Pack charges ✅
> Root cause of the week-long failure: **BMS #1 charge path died** (common-port board),
> plus a pack that had been run flat to 2.70V/cell by a 24/7 indicator LED.
> Spare BMS fitted → charging worked → rebuild verified. See **Fault Log** at the bottom.
>
> This file records what is **actually soldered**, not the plan.
> As-built changes vs the original plan:
> 1. Switch is a 2-pin SPST (KCD1-101, 10A/125VAC) — not a 3-pin selector.
> 2. Charger `OUT+` connects **after the fuse** (Switch Pin 1), not to BMS `P+`.
> 3. Type-C breakout sits on the **XL4015 variable rail** (no CC resistors).
> 4. **Two LEDs, but LED1 now has a 10kΩ** (was 1kΩ) **+ toggle** on node A, and LED2
>    has 1kΩ on node B. The 1kΩ LED1 drained 10.6mA 24/7 → flat pack in ~7 days; 10kΩ
>    drops it to 1.06mA → ~71 days, and the toggle makes it zero when stored.
> 5. Adapter input is the **round DC jack**, center = +. **No banana jacks** — header
>    pins + Type-C only (deferred, no spend).
> 6. BMS is the **spare** (BMS #1 died — common-port board, charge path failed).

## Vocabulary
- **Node A** = battery side of the switch. Live whenever the fuse is in.
- **Node B** = load side. Live when switch ON, or when the adapter is plugged.
- **Ground bus** = single common GND point. Every negative returns here.

---

# CONFIRMED AS-BUILT WIRING

### Node A — battery side
| From | To |
|---|---|
| BMS `P+` | 5A fuse IN |
| 5A fuse OUT | **node A** |
| Charger `OUT+` | **node A** |
| Switch Pin 1 | **node A** |
| LED1 anode → **10kΩ** → toggle → | GND |

### Node B — load side
| From | To |
|---|---|
| Switch Pin 2 | **node B** |
| DC jack **center (+)** | **node B** ← 12V adapter input |
| LED2 anode → 1kΩ → | GND |
| MP1584 `IN+` | **node B** |
| XL4015 `IN+` | **node B** |

### Grounds and outputs
| From | To |
|---|---|
| DC jack sleeve (−) | ground bus |
| BMS `P−` | ground bus |
| Charger `OUT−` | ground bus |
| XL4015 `IN−`, `OUT−` | ground bus |
| MP1584 `IN−`, `OUT−` | ground bus |
| MP1584 `OUT+` | ESP32 5V + header pin 1 |
| XL4015 `OUT+` | Type-C `VBUS` + header pin 2 |
| ESP32 TX / RX | header 3 / header 4 |
| Header 5 | ground bus |

**No banana jacks fitted.** Variable output exists only on the Type-C breakout and
header pin 2. For the turret, land on header pin 2 (+) and header pin 5 (−) — a
2.54mm header is fine at servo currents, but solder directly rather than using a
dupont lead for anything above ~1A.

### LED truth table (v9 — two LEDs)
| Switch | Adapter | Node A | Node B | LED1 (10kΩ + toggle) | LED2 (1kΩ) |
|---|---|---|---|---|---|
| OFF | unplugged | battery | dead | lit if toggle ON | dark |
| OFF | plugged | battery | 12V | lit if toggle ON | **lit** |
| ON | unplugged | battery | battery | lit if toggle ON | **lit** |
| ⚠️ ON | plugged | battery | 12V + battery | lit | **lit** |

⚠️ **Battery mode and CONFLICT are indistinguishable** — LED2 is lit either way.
The conflict is prevented by discipline only: **adapter plugged in ⇒ switch OFF.**
Making it physically impossible needs a 3-pin SPDT selector
(pin 1 = node A, pin 2 = node B, pin 3 = adapter) — deferred, no spend this round.

**LED1 drain after the 10kΩ fix:** `(12.6 − 2) ÷ 10kΩ = 1.06mA` → 25.4mAh/day →
**~71 days** instead of ~7. The toggle makes it zero. **Never go back to 1kΩ there.**

---

# GATE 0 — Cell check
1. DC 20V range, each cell at rest: **3.6–4.0V**, all within ~0.1V.
2. Below **3.0V** → don't use. Below **2.5V** → damaged.
3. **Negative reading = cell inserted backwards.** Stop immediately.

# GATE 1 — Pack + BMS
| # | From | To | Why |
|---|---|---|---|
| A1 | Cell1 − | BMS `B−` | pack negative reference |
| A2 | Cell1+/Cell2− junction | BMS `B1` | balance sense 1 |
| A3 | Cell2+/Cell3− junction | BMS `B2` | balance sense 2 |
| A4 | Cell3 + | BMS `B+` | pack positive |
| A5 | BMS `P+` | 5A fuse IN | fused at the pack |

Check: `B+`→`B−` ≈ 11.1V, `B1` ≈ 3.7V, `B2` ≈ 7.4V. **Swapped B1/B2 kills the BMS.**

# GATE 2 — Bench-test modules (no pack)
**XL4015:** 12V in, no load. CV pot → **5.0V**. Then CC: CV down to ~1–2V, meter in
10A mode across `OUT+`/`OUT−` for **2–3 seconds**, CC pot → **2.0A**. Restore CV 5V.
*Heating here is normal: ~1.5W of loss at 2A even with a shorted output, because the
freewheel diode carries full current. Warm = fine, 2–3s only.*

**MP1584:** 12V in → `OUT+` = **5.0V ± 0.2V**.

# GATE 3 — Switch + LEDs + adapter input
Adapter: DC jack center → **node B**, sleeve → ground bus.
Switch: Pin 1 → node A, Pin 2 → node B.
LED1: anode → node A, cathode → **10kΩ** → toggle → ground bus.
LED2: anode → node B, cathode → 1kΩ → ground bus.

**LED1 drain maths:** at 1kΩ, `(12.6−2)/1kΩ = 10.6mA` → 254mAh/day → pack flat in
**~7 days**. That is what pushed the pack to 2.70V/cell. At **10kΩ**: `1.06mA` →
25.4mAh/day → **~71 days**, and the toggle makes it zero.
**10kΩ is the as-built value — never revert to 1kΩ.**

**LED2 is on node B** so it is dark whenever the box is truly stored (switch OFF +
adapter unplugged). It only lights when the bus is actually live.

# GATE 4 — Modules + outputs
| # | From | To |
|---|---|---|
| D1 | XL4015 `IN+` | node B |
| D2 | XL4015 `IN−` | ground bus |
| D3 | XL4015 `OUT+` | Type-C `VBUS` + header pin 2 |
| D4 | XL4015 `OUT−` | ground bus |
| E1 | MP1584 `IN+` | node B |
| E2 | MP1584 `IN−` | ground bus |
| E3 | MP1584 `OUT+` | ESP32 5V + header pin 1 |
| E4 | MP1584 `OUT−` | ground bus |

- **Type-C = VARIABLE rail.** Label **"VAR V — CHECK KNOB"**. At 9V it kills a phone.
- CC1/CC2/D+/D− left open → plain power cables work; C-to-C negotiation does not
  (needs 2× 22kΩ Rp, deferred).

# GATE 5 — Charging
| # | From | To |
|---|---|---|
| G1 | Charger `OUT+` | **Switch Pin 1** (after fuse) |
| G2 | Charger `OUT−` | ground bus |

⚠️ **Pin 1, not Pin 2.** On Pin 2 the charger powers the modules instead of the pack.
Charge with switch OFF and 12V adapter unplugged.

# GATE 6 — Power-up
1. Switch OFF, adapter unplugged → node B dead, 0V rails.
2. Adapter plugged (switch still OFF) → node B 12V, XL4015 5V, MP1584 5V.
3. Adapter unplugged → node B dead.
4. Switch ON (adapter unplugged) → node B ≈ pack voltage, both rails 5V.

# GATE 7 — ESP32 dock (LAST)
| # | From | To |
|---|---|---|
| H1 | ESP32 `5V`/`VIN` | MP1584 `OUT+` (header pin 1) |
| H2 | ESP32 GND | ground bus |
| H3 | ESP32 TX | Blue Pill `PA10` |
| H4 | ESP32 RX | Blue Pill `PA9` |
| H5 | Blue Pill 5V | MP1584 `OUT+` |
| H6 | Blue Pill GND | ground bus |

Never 5V into `3V3`. Never double-feed (dock 5V + PC USB). Never power the Blue Pill
from the ESP32 `3V3` pin.

---

# FAULT LOG — 2026-11-16 (battery mode dead)

## Reported symptoms
1. LED1 went dim, then a LED died (replaced).
2. Switch ON → nothing runs, LED1 goes out.
3. Fuse area felt warm at one point; later no heat at all.

## Measurements taken
| Measurement | Reading | Meaning |
|---|---|---|
| Adapter mode, both rails | 5.0V, works | Modules + outputs + wiring are **good** |
| Pack `B+`→`B−` | **8.09V** | 2.70V/cell — **flat / possibly one dead cell** |
| Node A (after fuse) | 8.09V | fuse + holder fine at rest |
| Node B, switch ON | **1.23V** | BMS cutoff residual |
| Switch continuity | beeps | conducts at meter current |
| Switch resistance ON | < 0.5Ω | switch is good |
| Switch Pin2 → node B | beeps | wire intact |
| Continuity: node A → ground | no beep | **no short** |
| Fuse, end to end | beeps | fuse intact |

## Diagnosis
- **Not a short.** All continuity checks clean.
- **Not the switch, not the fuse, not the wiring.** All measured good.
- Switch and wire are < 0.5Ω, so node A and node B are electrically the same node.
  They cannot read 8V and 1.23V simultaneously → **node A also collapses when the
  switch closes.** The whole pack is collapsing.
- **1.23V = BMS residual leakage after a protection cutoff.**
- Arithmetic: `8.09V ÷ 3 = 2.67V/cell` vs BMS cutoff `≈2.5V/cell` → only **0.17V
  margin**. A worn cell at 0.2Ω sags `1A × 0.2Ω = 0.20V` → crosses the cutoff
  instantly. That is the 1.23V, measured milliseconds after the trip.
- **Why no heat now:** BMS cutoff is microseconds — no sustained current, nothing
  warms. The earlier heat happened when the pack still had charge to push current.
  Same fault, earlier stage.

## Prime suspect
```
2.70 + 2.70 + 2.69 = 8.09V   ← uniformly flat (recoverable)
3.75 + 3.75 + 0.59 = 8.09V   ← ONE DEAD CELL (pack finished)
```
Total voltage cannot distinguish them. **Per-cell measurement is mandatory.**

## Next steps (in order)
1. **Pack on charge now** (switch OFF, adapter unplugged, phone charger into the
   charger board). Target **12.4–12.6V**. Watch the first 30 min.
2. **Measure each cell** with polarity: `B1−B−`, `B2−B1`, `B+−B2`.
   - All ≈2.7V → flat only → charge and retest.
   - One <1.5V → **dead cell, do not charge at 1A.** Replace.
   - Any negative → reversed cell → fix orientation first.
3. **Under-load cell test** (after charge): switch ON, then measure each cell fast.
   Whichever drops below 2.5V is the weak cell.
4. **Retest battery mode.** If node B now holds ~11–12V → pack was just empty.
5. If it still collapses after a full charge → one cell has high internal
   resistance → replace it (or replace all three for a matched set).

## RESOLVED 2026-11-16 — BMS #1 dead, spare fitted, charging works

### BMS #1 — verdict
- Silkscreen has **no `C+`/`C−` pads** → **common-port board**. So the charger on
  `P+`/`P−` was the correct wiring all along; the separate-port theory is dead.
- Charge path was dead **inside** the board (charge FET / trace). Nothing downstream.
- Most likely cause: a **sense wire shorted to a neighbour** while untangling
  (`B1`↔`B2`↔`B+` is the classic killer of these boards), or a short on the output
  during the heating-fuse episode. Not age.
- **BMS #2 (spare) fitted → charge current confirmed. Battery mode path is now live.**

### Pad naming (learned the hard way)
| Pads | Meaning | Carries |
|---|---|---|
| `B−` `B1` `B2` `B+` | **B**attery — cells, inside the fence | `B−`/`B+` full pack current; `B1`/`B2` milliamps only |
| `P+` `P−` | **P**rotected output — outside the fence, through the FETs | full current |
| `C+` `C−` (if present) | separate charge port | — (not on this board) |

Identification test — black probe on `B−`, expect a staircase:
`B− 0.0V → B1 2.7V → B2 5.4V → B+ 8.1V` (flat pack).
`0 → 5.4 → 2.7 → 8.1` = **B1/B2 swapped** → do not connect `P+`.
Wire order: `B−` → `B1` → `B2` → `B+`, verify staircase, **then** `P+`/`P−`.

### Charge current 0.41A on a "1A" charger — normal, not a defect
Boost charger = power converter. `P_out ≈ P_in × 0.85`.
`1A × 12.6V = 12.6W` out needs `12.6 ÷ 0.85 ÷ 5V = 3.0A` in at 5V.
Measured 0.41A on a **5V 2A brick**, which *could* supply ~1.9A for a 1A charge at
8.1V — so the brick is **not** the cap. Remaining suspects, in order:
1. **USB cable drop** — measure at the module's USB-C pads; below 4.8V = bad cable.
2. **Module's own input-current limit (~0.8A)** — typical of these boards, which are
   designed around a 5V/1A source. The "1A" rating is then unreachable. Harmless.
3. **Factory trimpot set low** — some boards have an adjustable CC.
Current falls as the pack fills (constant power): 0.41A @8.1V → ~0.31A @10.5V → ~0.27A @12.4V.
`0.41A ÷ 1.8Ah = 0.23C` — gentle, ~5h to full. Accepted as-is: 0.23C is the
kindest rate for cells that were already run down to 2.7V.

## Open items
## Open items — final state
- [x] BMS swapped to spare — charging confirmed
- [x] Rebuild completed — **adapter mode and battery mode both verified working**
- [x] Pack charged to 12.4–12.6V
- [x] **LED1 back in service with 10kΩ + toggle** (was 1kΩ). Drain 10.6mA → 1.06mA,
      7 days → ~71 days, toggle = zero. Never revert to 1kΩ.
- [x] **Per-cell voltage recorded: 3.82V/cell** (= 11.46V pack, ~55% SoC)
      → cells are **balanced and healthy; there was never a dead cell.**
      Note: 3.82V/cell is also the **ideal long-term storage voltage** (40–60% SoC).
- [ ] Confirm the **spread** across the three cells (want all within 0.1V)
- [ ] Label the box (see below)
- ⬜ Banana jacks / 3-pin SPDT selector — deferred (no spend this round)
- ⬜ ESP32 wireless dock — pending the ESP32 arriving (do this LAST)

## Labels
1. DC input jack → `12V IN`
2. Type-C → `VAR V — CHECK KNOB`
3. Lid → `ADAPTER PLUGGED = SWITCH OFF` ← the **only** conflict guard now
4. **DC plug body** → `SWITCH OFF FIRST` (you can't plug it in without reading it)
5. Battery side → `PACK 12.6V MAX · FUSE AT PACK`

## Rules that must stay
1. One ground, one point.
2. Fuse at the pack.
3. Verify B1/B2 before connecting the BMS.
4. Charger only — the 12V adapter never touches the pack.
5. Adapter plugged in ⇒ switch OFF.
6. **Pack below 10.5V (3.5V/cell) = stop and charge.** The 2.5V BMS cutoff is a
   last-resort protection, not a fuel gauge.
7. XL4015 is a buck: max out ≈ input − 1.5V. From 9.9V you get ~8.4V max.
8. Adapter is 12V 2A ≈ 24W total. Keep combined output under ~15W.
