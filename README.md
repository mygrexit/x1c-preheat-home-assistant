# X1C Preheat — Home Assistant

> **Personal use only.** This is my own setup for my own X1C, in my own
> Home Assistant. Entity IDs, friendly-names (mixed German/English), input
> defaults, the chamber sensor mappings — everything is hard-wired to my
> specific hardware. It is not designed to be cloned, installed, or
> reused. **No support, no warranty, do not file issues, do not open PRs.**
> The repo is public so I can `git pull` it to my HA, nothing more.
>
> **230V mains.** If you're reading this and getting ideas: don't.

## What it does (for my own reference)

- **Auto-Preheat:** Detects ABS/ASA/PC prints (bed target ≥85°C) and preheats the enclosure via PTC heater
- **Pause/Resume:** Pauses the print at the first layer if the chamber is still cold, auto-resumes when target is reached. Marker prevents re-pausing or auto-resuming user pauses.
- **Heater Off at Print End:** When the print finishes, heater shuts off immediately. Fans are left to the printer.
- **Safety:** 7+ independent layers (dual-sensor overtemp, sensor divergence, sensor failure, power watchdog, comm-loss, timer hardcap, HA-restart cleanup, heater interlock)

## Hardware

- BambuLab X1C (LAN mode, local MQTT via [ha-bambulab](https://github.com/greghesp/ha-bambulab))
- [Cirrus 40/2](https://www.schaltschrankheizer.de/Cirrus-40-2-230-Watt-230-V-24-V-Luefter/370090-01-FGC1515.2) PTC heater (230W, 230V, integrated fan)
- Shelly Plus 1PM (heater relay + temperature + power monitoring)

## Architecture

Everything in one package: [`packages/x1c_enclosure.yaml`](packages/x1c_enclosure.yaml). Loaded via:

```yaml
homeassistant:
  packages:
    x1c_enclosure: !include bambu-heater/packages/x1c_enclosure.yaml
```

The repo lives at `/config/bambu-heater/` on HA. The package file is symlinked into `/config/packages/`. Hourly auto-pull from GitHub keeps it in sync — see `script.x1c_sync_from_git` and the `x1c_auto_sync_hourly` automation.

## Vibecoded

Built with Claude. Multiple safety-audit passes. Still: I am not a safety engineer.

## License

MIT — but read the disclaimer at the top first.
