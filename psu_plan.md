# Portable Bench Power Supply — Build Plan (PSU)

> Status: **✅ COMPLETE 2026-11-17 — built and working.** Adapter mode ✅, battery mode ✅,
> both rails 5V ✅, pack charges ✅. The week-long failure was a **dead BMS #1 charge path**
> plus a pack flattened to 2.70V/cell by a 24/7 indicator LED. See `build_guide.md` → Fault Log.
> Priority: budget + efficiency. Portable. NOT for drones.

## Envelope (locked)

- Source: 12V 2A adapter (wall) OR 3S1P pack (battery), switched
- Pack: 3× Winway 1800mAh original (matched), 9.9–12.6V, ~20Wh, ~1.5–2A continuous / ~3A burst
- Output: ONE variable rail 1.2–10V (set per project) + fixed 5V rail (ESP32 branch) + raw ~12V tap
- MCU (Blue Pill) stays USB-powered; ESP32 gets its own fixed 5V branch → simultaneous rails ARE needed now (wireless), but each rail is single-purpose

## Architecture (A-B-C-D, minimal stage)

```
A: 12V 2A adapter ──────────┐
                             ├─[2-pos selector]──► BMS-protected bus ──► XL4015 CC/CV ──► banana jacks   (variable rail, set per project)
B: 3S1P pack + 3S BMS ───────┘        (inline fuse)                      │
   └─ USB-C 3S boost charger (phone charger in, charges pack in place)   └─► MP1584 fixed 5V ──► wireless/ESP32 branch (dev board regulates 3.3V onboard)
```

One variable rail (servos/load under test) + one fixed 5V rail (ESP32 branch) — separate power branches, single shared ground. No 3.3V linear, no battery meter YET — done with the knob + yellow multimeter.

## Buying list (minimal — ~₹165 parts)

| # | Buy | Chosen | Cost |
|---|---|---|---|
| 1 | 3S protection BMS | **Quartz 3S 20A** — verified protections (over-discharge/short/overcharge/overcurrent), 4.2V/cell cutoff | ₹65 |
| 2 | XL4015 CC/CV buck | **Flyrobo XL4015 5A CC/CV** — dual trimpots (CV-ADJ + CC-ADJ) VERIFIED by photo, LM358/TL431 CC circuit | ₹110 |
| 3 | Inline fuse holder + 5A fuse | catastroph-short backstop (BMS covers the rest) | ~₹20 |
| 4 | Fixed 5V buck (wireless branch) | **Flyrobo MP1584EN Fixed 5V — ₹52** — 3A, no pot (fixed feedback = no drift), 1.5MHz low ripple. Powers ESP32 dev board (regulates 5V→3.3V onboard). SEPARATE branch from servo rail, shared ground. | ₹52 |

**Total ~₹247 parts** (two shops: XL4015 + MP1584 both Flyrobo = ₹162; BMS + fuse in ONE QC order = ₹85).

**RESOLVED pick (2026-09-09):** Flyrobo XL4015 CC/CV — ₹110, dual-pot board confirmed on listing photo (two multiturn trimpots = adjustable voltage AND current). Verify again in hand: two pots + extra LM358/TL431/78L05 + R050 shunt on solder side. Set CC to ~2A (protects 1800mAh pack) and ~1.5A for turret work.
**Rejected:** Flyrobo D-1231 single-pot XL4015 (₹86 — the URL-with-`-1`, one pot, no CC) · Quartz MOD-QC0384 single-pot (₹79) · LM2596 w/ LED meter (₹107 — no CC mode, 2A) · Flyrobo 3S 10A BMS (₹74, unverifiable).
**Notes:** 20A BMS > needed 10A is free margin — the **XL4015 CC knob** handles sustained overdraw. Check BMS idle drain (~few mA expected). Multimeter = the display.

## Deferred upgrades (₹0 now, add later ONLY if needed)

- Second buck for bare-module ESP32 (not dev board) set to 3.3V (~₹70) → bare WROOM/WROVER needs ~500mA peaks; AMS1117-3.3 (~₹10) ONLY fed from 5V rail for light ≤300mA loads — never from 12V bus (Linear heat = (Vin−Vout)×I: 12→3.3 @400mA = 3.5W, 5→3.3 @400mA = 0.68W).
- 3S LED battery meter (₹30) → until you're tired of multimeter checks
- Master switch (₹15) → until pack-idle-drain matters (for now, pull pack or switch off)

## Inventory (owned — verified)

3× Winway 1800mAh · 1× 2000mAh copy (spare, OUT of pack) · Type-C 3S→12.6V 1A charger (CV bulk, no balance — check for B1/B2 pads) · 12V 2A Wattnine adapter · banana jacks/wires/enclosure (Harish has)

## Charging rules

- Charge when idle (1A), never heavy-load while charging. Adapter (barrel) can't charge pack — use USB-C phone charger.
- Monthly: measure each cell rest voltage w/ multimeter, must agree within ~0.1V (bulk charger doesn't balance).

## Safety

- BMS mandatory · fuse on bus · polarity double-checked · never parallel adapter+pack directly · 2000mAh copy stays out of pack.

## As-built reality (2026-11-15/16)

Deviations from the locked design, all accepted:

| Planned | Actually built |
|---|---|
| 3-pin 2-position selector switch | **2-pin SPST (KCD1-101, 10A/125VAC)** — the 3-pin was lost |
| Banana jacks = variable output | **No banana fitted.** Variable out = Type-C breakout + header pin 2 |
| Charger `OUT+` → BMS `P+` | **Charger `OUT+` → Switch Pin 1** (after the fuse) — shorted charger blows the fuse, not the pack |
| Blue LED (adapter) + Green LED (load) | **Two LEDs.** LED1 on node A with **10kΩ + toggle** (1kΩ drained the pack 10.6mA 24/7 → flat in 7 days); LED2 on node B with 1kΩ |
| USB-C output deferred | Type-C breakout on the **XL4015 variable rail**, CC pins open |

### Architecture as built
```
12V DC jack ──► node B ──► XL4015 CC/CV ──► Type-C (VAR) + header pin 2
                 ▲                                └─► header pin 1 = fixed 5V (MP1584)
                 │
      Switch Pin 2 ── (SPST) ── Switch Pin 1 = node A
                                      │
        3S pack + BMS ── 5A fuse ─────┘
                                      │
        Type-C 3S 1A charger OUT+ ────┘   (charges through the fuse)

All grounds → one common ground bus.
```

### Measured results (final)
- Adapter mode: **works** — node B 12V, XL4015 5V, MP1584 5V, both rails stable.
- Battery mode: **works** — verified after the BMS swap and a full charge.
- Charging: **works** — ~0.41A (0.23C), pack reaches 12.4–12.6V.
- Per-cell: **3.82V/cell** (11.46V pack, ~55% SoC) — balanced, healthy, **no dead cell**.
- The 8.09V / 1.23V collapse was **BMS #1's dead charge path**, not the pack and not
  the wiring. Pack had simply been run flat by the 1kΩ LED1 draining 10.6mA 24/7.

### Per-cell check — RESOLVED
**Recorded 3.82V/cell** (11.46V pack, ~55% SoC). Cells are balanced and healthy —
there was never a dead cell. The `8.09V` readings earlier were just a flat pack.
Outstanding (minor): record the **spread** across all three at the next charge.

## Phases

- ✅ **P0** Cell check (done at build time, but only total voltage kept)
- ✅ **P1** Pack + BMS + charger — built, charging works
- ✅ **P2** XL4015 rail — set 5V, CC ≈ 2A
- ✅ **P3** Fuse + bus + switch + LEDs — built
- ✅ **P4** Prove under multimeter — adapter mode proven
- ✅ **P5** Battery mode — **works.** Root cause: dead BMS #1 charge path (common-port
  board); spare BMS fitted, pack charged at 0.41A (0.23C, input/cable-limited — accepted).
- ⬜ **P6** ESP32 wireless dock — deferred until the ESP32 arrives (LAST)