# Setup Guide

## Prerequisites

- Home Assistant OS (or Supervised)
- [BambuLab Integration](https://github.com/greghesp/ha-bambulab) installed and connected via LAN mode
- Shelly Plus 1PM (or similar smart relay with power + temperature monitoring)
- PTC enclosure heater wired through the Shelly relay
- SSH access to your HA instance (optional, for git-based updates)

## 1. Clone the repo

SSH into your HA instance and clone into `/config`:

```bash
cd /config
git clone https://github.com/mygrexit/x1c-preheat-home-assistant.git bambu-heater
```

## 2. Include the package

Add to your `/config/configuration.yaml`:

```yaml
homeassistant:
  packages:
    x1c_enclosure: !include bambu-heater/packages/x1c_enclosure.yaml
```

If you already have a `homeassistant:` block, just add the `packages:` section inside it — don't duplicate the block.

## 3. Replace entity IDs

The package is hardcoded to my hardware. You **must** replace these entity IDs with your own:

| Entity in package | What it is | Replace with |
|---|---|---|
| `switch.x1c_heater` | Shelly relay switch | Your relay switch |
| `sensor.x1c_heater_temperatur` | Shelly temperature sensor | Your relay's temp sensor |
| `sensor.x1c_heater_leistung` | Shelly power sensor | Your relay's power sensor |
| `sensor.x1c_00m00a2c0601660_druckstatus` | BambuLab print status | Your printer's status entity |
| `sensor.x1c_00m00a2c0601660_aktueller_arbeitsschritt` | BambuLab current stage | Your printer's stage entity |
| `sensor.x1c_00m00a2c0601660_temperatur_im_druckraum` | BambuLab chamber temp | Your printer's chamber temp |
| `sensor.x1c_00m00a2c0601660_verbleibende_zeit` | BambuLab remaining time | Your printer's remaining time |
| `sensor.x1c_00m00a2c0601660_zieltemperatur_vom_druckbett` | BambuLab bed target temp | Your printer's bed target |
| `fan.x1c_00m00a2c0601660_druckraumlufter` | BambuLab aux fan | Your printer's aux fan |
| `button.x1c_00m00a2c0601660_druckvorgang_anhalten` | BambuLab pause button | Your printer's pause button |
| `button.x1c_00m00a2c0601660_druckvorgang_fortsetzen` | BambuLab resume button | Your printer's resume button |
| `camera.x1c_00m00a2c0601660_kamera` | BambuLab camera | Your printer's camera entity |

Tip: In HA go to **Settings > Devices** and find your BambuLab printer and Shelly device to get the correct entity IDs.

## 4. Restart HA

**Settings > System > Restart**

## 5. Verify defaults

After first startup, check that these values are set correctly:

| Entity | Expected value |
|---|---|
| `input_number.x1c_enclosure_overtemp_limit` | 75°C |
| `input_number.x1c_enclosure_preheat_target` | 45°C |
| `input_number.x1c_enclosure_preheat_bed_threshold` | 85°C |
| `input_number.x1c_enclosure_comm_loss_minutes` | 5 min |
| `input_select.x1c_enclosure_mode` | aus |

If HA uses the `min` value as default instead, set them manually via **Developer Tools > States**.

## 6. Dashboard (optional)

The file `dashboard.yaml` contains a ready-made Lovelace dashboard. To use it:

**Option A — YAML mode (recommended if you update via git):**

Add to `configuration.yaml`:
```yaml
lovelace:
  dashboards:
    dashboard-bambulab:
      mode: yaml
      title: BambuLab
      filename: bambu-heater/dashboard.yaml
      icon: mdi:printer-3d
```

**Option B — Manual:**

Dashboard > Edit > Add Card > YAML Editor > paste contents of `dashboard.yaml`.

## Updating

### Option A — Auto-sync via GitHub Webhook (recommended)

The package includes a webhook automation that triggers `git pull + reload` on every push.

1. Deploy the package to HA and restart (so the webhook is registered)
2. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
3. Set:
   - **Payload URL:** `https://<your-ha-domain>/api/webhook/x1c-git-sync-06f81528bbb0314024bdda0a2a430f34`
   - **Content type:** `application/json`
   - **Events:** Just the push event
4. Save

Now every `git push` automatically syncs to HA.

**Important:** Change the webhook ID in the package (`x1c-git-sync-06f81528...`) to your own random string. Anyone who knows the ID can trigger a reload.

### Option B — Manual

```bash
cd /config/bambu-heater && git pull
```

Then reload in HA: **Developer Tools > YAML > Reload All YAML Configuration**

Or use the built-in script `script.x1c_sync_from_git` which does both (git pull + reload).
