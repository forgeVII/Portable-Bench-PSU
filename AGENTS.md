# Power Management - Chat Workspace

Dedicated opencode chat folder for power/electrical questions: voltage,
current, wattage, per-device load calculations, power supplies, and
separating the MCU from higher-power stuff (servos, motors, lasers, etc.).

## Why this chat exists

Harish regularly gets confused about voltage vs current vs power because
every device wants a different combination. This chat is the safe place to
work out loads and power design before buying or wiring anything.

## Real project

The active hardware target is the laser turret:

- **Real project folder:** `C:\learning\projects\embedded\laser_turret`
- **Git remote:** https://github.com/forgeVII/laser_turret (private)
- **Hardware plan:** `TRACKER.md` (component list, Power Design section,
  buy list: Blue Pill + 2x SG90 + laser 5V TTL + HC-SR04 + external 5V)
- **Loaded components to model:** 2x SG90 servo, HC-SR04 (5V), laser module
  (5V TTL), Blue Pill STM32F103 (3.3V logic, USB powered)

## Known user gaps (from Turret chat)

- What 1 servo consumes (peak vs stall vs idle), and what the total draw is
  for 2 servos + HC-SR04 + laser + MCU together.
- Why some things are 5V, some 3.3V, some need their own supply.
- How a supply must be sized (total current + headroom), shared ground.
- The MCU must NOT power the servos (separate higher-power rail).

## Instructions for the assistant

- Work through load math step by step. The user is a beginner at this.
- Use concrete numbers for the actual components in the buy list. No fluff,
  no extra components, no "you might also want" additions.
- Voltage / current / watt: state the formula used, then the numbers.
- If you cannot access a file, tell the user to grant the external-directory
  permission prompt.
- Never commit or push unless the user explicitly asks.