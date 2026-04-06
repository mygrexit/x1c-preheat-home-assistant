# Project: Home Assistant Automation Packages

## HA Instance

- **Host:** hassio.rlxd.xyz
- **SSH:** `niko@hassio.rlxd.xyz` (password in SSH addon config, port 22)
- **SSH Addon:** Advanced SSH & Web Terminal (`a0d7b954_ssh`)
- **Supervisor Token:** Rotates on every addon restart. Get fresh token via: `grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh`
- **API via SSH:** `curl -H "Authorization: Bearer $SUPERVISOR_TOKEN" http://supervisor/core/api/...`
- **Files are root-owned:** Use `sudo tee` to write, not direct redirect

## Architecture

- All automations live in **HA packages** under `/config/packages/` on HA
- `configuration.yaml` uses `packages: !include_dir_named packages` — any YAML in that dir is auto-loaded
- One package = one self-contained feature (automations, scripts, helpers, timers, all in one file)
- Entity IDs are hardcoded to this specific hardware setup

## Packages

| Package | File | What it does |
|---|---|---|
| X1C Enclosure Heater | `x1c_enclosure.yaml` | PTC heater control, auto-preheat, pause/resume, 7 safety layers |
| Door Siren | `door_siren.yaml` | Cat gate + door alarm, retry logic, hardcap, battery monitoring |

## Deploy Workflow

**Preferred (x1c_enclosure.yaml):** Commit + push to GitHub → webhook triggers HA to `git pull` + reload automatically. No SSH needed.

**Manual (door_siren.yaml, not in git):**
1. Write YAML content
2. Deploy via SSH: `cat content | sshpass ... ssh ... 'sudo tee /config/packages/file.yaml > /dev/null'`
3. Restart HA: `ha core restart` (from SSH with sourced env) or via Supervisor API
4. Set input_number defaults via API (HA uses `min` as initial, not a sensible default)

## Git Repo (x1c-preheat only)

- **Repo:** github.com/mygrexit/x1c-preheat-home-assistant
- **Auto-sync:** GitHub webhook → HA webhook automation → `git pull` in `/config/bambu-heater/` → reload
- **Webhook ID:** `x1c-git-sync-06f81528bbb0314024bdda0a2a430f34`
- **Git user:** `mygrexit`
- **Repo location on HA:** `/config/bambu-heater/` (NOT `/config/` itself)
- **Live file is a symlink:** `/config/packages/x1c_enclosure.yaml` → `/config/bambu-heater/packages/x1c_enclosure.yaml`. Required so that `git pull` is actually picked up by HA. Without the symlink, pulls are invisible to HA.
- **Git safe.directory:** Must be set globally for root (`/homeassistant/bambu-heater` AND `/config/bambu-heater`), otherwise `git pull` fails with „dubious ownership" and the shell_command swallows the error silently.
- Door siren package is NOT in git (deployed directly to HA)

## Conventions

- All comments, aliases, descriptions, notifications in **English**
- input_select state values stay in German (internal state, referenced in conditions everywhere)
- Defense-in-depth: multiple independent safety layers
- Scripts for reusable actions (turn_on, turn_off, emergency_off)
- `mode: restart` for automations with delays (cancels on re-trigger)
- Retry logic for Zigbee commands (unreliable)
- Safety hardcap timers so nothing runs forever
- HA restart automation to reset state
- Pushover for push notifications (`notify.pushover`)

## Hardware

- **BambuLab X1C** — LAN mode, local MQTT via HA integration
- **Shelly Plus 1PM** — Heater relay + temp/power monitoring
- **Zigbee (ZHA):** Door sensor, cat gate sensor, siren
- **PTC Heater:** Cirrus 40/2, 230W, 230V, self-regulating
