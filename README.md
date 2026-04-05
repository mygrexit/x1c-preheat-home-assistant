# X1C Preheat — Home Assistant

Automatic enclosure heating for the BambuLab X1C with a PTC heater, controlled via Home Assistant.

> **Disclaimer:** This project is heavily vibecoded, actively used by me (one person) for my own printer, and is **experimental**. It controls a 230V/230W PTC heater — if you don't know what you're doing, don't. I'm sharing this in case someone wants to build something similar and feels confident enough to do so. **No support, no warranty, use at your own risk.**

## What it does

- **Auto-Preheat:** Detects ABS/ASA/PC prints based on bed target temperature (≥85°C) and automatically preheats the enclosure
- **Pause/Resume:** Pauses the print at the first layer if the chamber is still cold, auto-resumes once target temperature is reached
- **Controlled Cooldown:** 30-minute cooldown phase after printing (heater ramps down, fan ramps up incrementally)
- **Safety:** 7 independent safety layers (dual-sensor overtemp, sensor divergence, power watchdog, comm-loss, timer hardcap, HA restart protection, heater interlock)

## Hardware

- BambuLab X1C (LAN mode)
- [Cirrus 40/2](https://www.schaltschrankheizer.de/Cirrus-40-2-230-Watt-230-V-24-V-Luefter/370090-01-FGC1515.2) PTC heater (230W, 230V, with integrated fan)
- Shelly Plus 1PM (switches the heater, monitors temperature + power)

## Software

- Home Assistant OS
- [BambuLab Integration](https://github.com/greghesp/ha-bambulab) (LAN mode, local MQTT)
- Shelly Integration (native in HA)

## Architecture

Everything lives in a single package file: [`packages/x1c_enclosure.yaml`](packages/x1c_enclosure.yaml)

```
┌─────────────────────────────────────────────────┐
│  State Machine                                  │
│  off → preheating → printing → cooldown → off   │
│                                       error ↗   │
├─────────────────────────────────────────────────┤
│  Generic Thermostat (UI + Safety Cutoff)        │
│  └→ Shelly Plus 1PM → PTC Heater               │
├─────────────────────────────────────────────────┤
│  Safety Layers (7x independent)                 │
│  - Overtemp Shelly (unconditional)              │
│  - Overtemp printer sensor (independent HW)     │
│  - Sensor divergence (>15°C for 5 min)          │
│  - Sensor failure                               │
│  - Power watchdog                               │
│  - Comm loss (printer offline)                  │
│  - Timer hardcap                                │
└─────────────────────────────────────────────────┘
```

**17 automations, 7 scripts, 10 configurable parameters** — all in one YAML file.

## Setup

See [`SETUP.md`](SETUP.md) for the full guide.

Quick version:
1. Clone repo onto HA: `git clone https://github.com/mygrexit/x1c-preheat-home-assistant.git bambu-heater`
2. Add to `configuration.yaml`:
   ```yaml
   homeassistant:
     packages:
       x1c_enclosure: !include bambu-heater/packages/x1c_enclosure.yaml
   ```
3. Restart HA
4. Replace entity IDs to match your hardware (Shelly, BambuLab integration)

## Important

- **Entity IDs are hardcoded to my hardware.** You'll need to replace `switch.x1c_heater`, `sensor.x1c_heater_temperatur`, `sensor.x1c_00m00a2c0601660_*` etc. with your own entity IDs.
- **The generic thermostat is not an active controller** — the PTC heater is self-regulating. The thermostat serves as a UI widget and safety cutoff (max 75°C).
- **230V.** Seriously. If you're not comfortable working with mains voltage, don't build this.

## Vibecoded?

Yes. The entire development — architecture, automations, safety audits, debugging — was done in collaboration with Claude (Anthropic). The code went through multiple safety audit rounds, but at the end of the day I'm just a guy with a 3D printer and an AI, not a safety engineer. Use at your own risk.

## License

MIT — do whatever you want with it, but don't blame me if your printer catches fire.
