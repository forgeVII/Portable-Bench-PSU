# Portable Bench Power Supply

A two-rail bench supply that runs from either a **12 V wall adapter** or a **3S1P
18650 pack**, built on the bench from modules and verified end to end.

**Status: ✅ complete and working (2026-11-17).**
Adapter mode ✅ · Battery mode ✅ · Variable rail ✅ · Fixed 5 V rail ✅ · In-place charging ✅

---

## What it is

| | |
|---|---|
| **Input 1** | 12 V 2 A wall adapter → round DC barrel jack (center = +) |
| **Input 2** | 3S1P 18650 pack (3 × 1800 mAh), 11.1 V nominal / 12.6 V full / ~20 Wh |
| **Output 1** | XL4015 CC/CV buck — variable 1.2 V … (Vin − 1.5 V), set per project |
| **Output 2** | MP1584 fixed 5 V — dedicated branch for an ESP32 / MCU |
| **Charging** | 5 V USB-C → 3S 1 A boost charger module, charges the pack in place |
| **Protection** | 3S BMS (common port) + 5 A fuse at the pack |

## Architecture

```
12V DC jack ──► node B ──► XL4015 CC/CV ──► Type-C (VAR V) + header pin 2
                 ▲                              └── MP1584 ──► header pin 1 (5V fixed)
                 │
      SPST ──────┘ (pin 2 → node B, pin 1 → node A)
        │
3S pack + BMS ── 5A fuse ── node A ── charger OUT+ (charge current flows through the fuse)

All grounds → one common ground bus.
```

- **Node A** = battery side of the switch. Live whenever the fuse is in.
- **Node B** = load side. Live when the switch is ON, or when the adapter is plugged.

## Design decisions worth knowing

| Decision | Why |
|---|---|
| Charger `OUT+` lands **after the fuse** (node A) | A shorted charger blows the fuse instead of dumping amps into the pack |
| XL4015 is **buck only** | Max output ≈ Vin − 1.5 V; from 9.9 V you get ~8.4 V |
| Type-C breakout on the **variable** rail, CC pins open | 2-wire power-only port. Marked "VAR V" — it is **not** 5 V |
| One LED on node B (1 kΩ), one on node A (10 kΩ + toggle) | Node-A LED at 1 kΩ drained 10.6 mA 24/7 → flattened the pack in ~7 days |
| SPST switch instead of a source selector | The 3-pin selector was lost; conflict is prevented by discipline (below) |

## Safety rules

1. **Adapter plugged in ⇒ switch OFF.** With an SPST switch both sources can reach
   node B; there is no LED that can warn you. A 3-pin SPDT selector would make it
   impossible — deferred.
2. **Pack floor 10.5 V (3.5 V/cell).** The 2.5 V BMS cutoff is last-resort protection,
   not a fuel gauge.
3. **Charge with the switch OFF and the 12 V adapter unplugged.** Phone charger only —
   the 12 V adapter never touches the pack.
4. **Charger goes to `P+`/`P−` only.** Never `B+`/`B−` — that bypasses the protection FETs.
5. **Verify the `B−`/`B1`/`B2`/`B+` staircase before connecting `P+`.** Swapped `B1`/`B2`
   kills the BMS.
6. One ground, one point.

## Documents

| File | Contents |
|---|---|
| [`build_guide.md`](build_guide.md) | As-built wiring, node map, gate-by-gate checks, full fault log |
| [`psu_plan.md`](psu_plan.md) | Architecture, component list, design rationale, phases |
| [`notebook.md`](notebook.md) | The electronics learned here — V/I/P, BMS cutoffs, boost-charger maths, fault-finding |

## What went wrong (and what it taught)

The first battery-mode test collapsed to 1.23 V with no short, good switch, good fuse
and good wiring. Two separate faults:

1. **BMS #1's charge path was dead** (common-port board) — the pack had never actually
   been charging. A spare BMS fixed it.
2. **The 1 kΩ indicator LED on the always-live node A** drained 10.6 mA around the
   clock, which is what ran the pack down to 2.70 V/cell in the first place.

Full timeline in [`build_guide.md`](build_guide.md) → *Fault Log*.

## Not done yet

- 3-pin SPDT source selector (makes adapter/battery conflict physically impossible)
- Banana jacks on the variable output (currently header pins + Type-C)
- ESP32 branch — mounted on the MP1584 5 V rail; serial link to the Blue Pill
  (`TX → PA10`, `RX → PA9`) to be confirmed under load
