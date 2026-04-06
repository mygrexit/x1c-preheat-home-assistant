# X1C Slow Cooldown Mode — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a closed-loop, temperature-driven slow cooldown profile to the X1C enclosure heater package, activated via a persistent toggle, with hot-swap support that lets the user switch profiles mid-cooldown.

**Architecture:** Single-file change in `packages/x1c_enclosure.yaml`. New `input_boolean.x1c_slow_cooldown_mode` toggle gates a second branch inside the existing `script.x1c_enclosure_cooldown` (one script, two profiles, `mode: restart`). A new template sensor `sensor.x1c_effective_cooldown_min` becomes the single source of truth for safety-timer math, replacing four direct reads of `input_number.x1c_enclosure_cooldown_minutes`. The fixed 35 min `timer.x1c_enclosure_post_print` duration is templated to extend to 55 min in slow mode. A new hot-swap automation re-triggers the cooldown script when the toggle flips on while a cooldown is already running.

**Tech Stack:** Home Assistant YAML packages, Jinja templates, generic_thermostat, Bambu LAN MQTT integration, Shelly Plus 1PM (heater relay + temp sensor).

**Spec:** [`docs/superpowers/specs/2026-04-06-slow-cooldown-mode-design.md`](../specs/2026-04-06-slow-cooldown-mode-design.md)

## Notes for the Implementer

- **No unit-test framework.** This is a HA YAML package. "Validation" = (a) `ha core check` for syntax, (b) Developer Tools → Template for live template evaluation, (c) runtime observation via Logbook. Each task includes the relevant validation step in lieu of a unit test.
- **Deploy workflow is git-based.** Push to `main` on `github.com/mygrexit/x1c-preheat-home-assistant` triggers a webhook → HA pulls + reloads automatically. After each task: commit and push, then validate on the live instance. The repo is **a separate repo** dedicated to the heater package; commits in *this* repo (`bambulab-hassio`) are documentation-only and do not trigger deploy. **The implementer must understand:** edits to `packages/x1c_enclosure.yaml` here serve as the source of truth and need to be mirrored to the live `x1c-preheat-home-assistant` repo (or this repo IS the live one — verify with `git remote -v` before starting).
- **All comments in YAML must be English** per project convention (see `CLAUDE.md`).
- **`mode: restart` script branching:** the `choose` reads `is_state(...)` *once* at script invocation. If the user toggles the input_boolean mid-script, the in-flight phases continue with the original profile — this is intentional and the hot-swap automation in Task 8 is what makes mid-cooldown switching work.
- **Hot-swap honesty:** the design's "leave the spec as is" instruction means this plan also captures the bauteillüfter and Phase-C ambient honesty notes (originally proposed as spec edits) inside YAML comments at the relevant code sites — see Tasks 5 and 9.

## Pre-flight

- [ ] **Step 0.1: Confirm repo identity**

```bash
cd /home/niko/workspace/projects/bambulab-hassio
git remote -v
```

If the remote is `bambulab-hassio` (not `x1c-preheat-home-assistant`), you must mirror each `packages/x1c_enclosure.yaml` change into the live repo as well, or the live HA instance won't receive updates. If the remote IS `x1c-preheat-home-assistant`, push triggers the webhook directly.

- [ ] **Step 0.2: Validate baseline syntax**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.` Establishes a clean baseline before any edit.

---

## File Structure

**Modified:**
- `packages/x1c_enclosure.yaml` — all changes live here

**New sections inside that file:**
- `input_boolean:` (new top-level key, currently absent)
- New template sensor `x1c_effective_cooldown_min` inside the existing `template:` block
- New `input_number.x1c_enclosure_slow_cooldown_minutes` inside the existing `input_number:` block
- New automation `x1c_slow_cooldown_hot_swap`

**Modified sections inside that file:**
- `script.x1c_enclosure_cooldown` — wrap existing sequence in a `choose` with slow-branch as first option
- `script.x1c_enclosure_turn_on` — `cooldown_min` variable now reads from template sensor
- `script.x1c_start_heating_for_print` — same
- `automation.x1c_enclosure_print_started` — same
- `automation.x1c_enclosure_update_safety` — same
- `automation.x1c_enclosure_print_ended` — `timer.start` duration becomes templated
- Template sensor `X1C Enclosure Status` — show `(slow, ...)` label when toggle is on

---

### Task 1: Add `input_boolean.x1c_slow_cooldown_mode` toggle

**Files:**
- Modify: `packages/x1c_enclosure.yaml` — add new top-level `input_boolean:` block right after the `input_select:` block (~line 44)

- [ ] **Step 1.1: Insert the input_boolean section**

After the closing of `input_select:` (after the line `    icon: mdi:radiator` near line 44) and before `# --- Configuration Parameters ---`, insert:

```yaml
# --- Toggles ---
input_boolean:
  x1c_slow_cooldown_mode:
    name: 'X1C Enclosure: Slow Cooldown Mode'
    icon: mdi:snowflake-alert
```

- [ ] **Step 1.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 1.3: Reload and verify entity exists**

```bash
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" -X POST http://supervisor/core/api/services/homeassistant/reload_all'
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" http://supervisor/core/api/states/input_boolean.x1c_slow_cooldown_mode'
```

Expected: JSON response with `"state": "off"` (default state for new input_boolean).

- [ ] **Step 1.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): add slow cooldown mode toggle"
git push
```

---

### Task 2: Add `input_number.x1c_enclosure_slow_cooldown_minutes`

**Files:**
- Modify: `packages/x1c_enclosure.yaml` — add inside the existing `input_number:` block, immediately after `x1c_enclosure_cooldown_minutes` (after line 63)

- [ ] **Step 2.1: Insert the input_number**

After the `x1c_enclosure_cooldown_minutes` block (the one ending with `unit_of_measurement: min` around line 63), insert:

```yaml
  x1c_enclosure_slow_cooldown_minutes:
    name: 'X1C Enclosure: Slow Cooldown (min)'
    min: 30.0
    max: 90.0
    step: 1.0
    mode: slider
    unit_of_measurement: min
    icon: mdi:snowflake-alert
```

- [ ] **Step 2.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 2.3: Reload and set initial value**

HA uses `min` as initial state for input_numbers (per `CLAUDE.md`), so explicitly set the desired default:

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -X POST http://supervisor/core/api/services/homeassistant/reload_all && sleep 2 && curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_number.x1c_enclosure_slow_cooldown_minutes","value":50}'"'"' http://supervisor/core/api/services/input_number/set_value'
```

Expected: 200 OK. Then verify:

```bash
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" http://supervisor/core/api/states/input_number.x1c_enclosure_slow_cooldown_minutes'
```

Expected: `"state": "50.0"`.

- [ ] **Step 2.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): add slow cooldown safety-timer minutes config"
git push
```

---

### Task 3: Add `sensor.x1c_effective_cooldown_min` template sensor

**Files:**
- Modify: `packages/x1c_enclosure.yaml` — add to the existing `template:` block, inside the first `- sensor:` list (around line 184, after the `X1C Print Remaining (min)` sensor)

- [ ] **Step 3.1: Insert template sensor**

After the `X1C Print Remaining (min)` sensor block (the one ending around line 184), and before the `- binary_sensor:` block, add:

```yaml
    - name: "X1C Effective Cooldown Minutes"
      unique_id: x1c_effective_cooldown_min
      icon: mdi:timer-cog-outline
      unit_of_measurement: min
      # Single source of truth for safety-timer sizing. Returns slow-cooldown
      # minutes when slow mode is on, otherwise the regular cooldown minutes.
      # All four safety-timer math sites read this sensor instead of the raw
      # input_number, so flipping the toggle automatically resizes every timer.
      state: >
        {% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}
          {{ states('input_number.x1c_enclosure_slow_cooldown_minutes') | int(50) }}
        {% else %}
          {{ states('input_number.x1c_enclosure_cooldown_minutes') | int(20) }}
        {% endif %}
```

- [ ] **Step 3.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 3.3: Reload and verify both states**

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -X POST http://supervisor/core/api/services/homeassistant/reload_all'
sleep 3
# With toggle off (default) → expect 20
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" http://supervisor/core/api/states/sensor.x1c_effective_cooldown_min'
```

Expected: `"state": "20"`. Now turn the toggle on and re-check:

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_on'
sleep 1
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" http://supervisor/core/api/states/sensor.x1c_effective_cooldown_min'
```

Expected: `"state": "50"`. Then turn back off:

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_off'
```

- [ ] **Step 3.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): add effective_cooldown_min template sensor"
git push
```

---

### Task 4: Replace `cooldown_min` reads in 4 locations

**Files:**
- Modify: `packages/x1c_enclosure.yaml` lines 651, 738, 828, 996 (each is a `- variables:` block where `cooldown_min:` is bound)

**Important:** This task changes the source from `input_number.x1c_enclosure_cooldown_minutes` to `sensor.x1c_effective_cooldown_min`. Behaviour with the toggle off is **identical** to today (the sensor returns the same input_number value). With the toggle on, all four sites pick up the slow value automatically.

- [ ] **Step 4.1: Edit `script.x1c_enclosure_turn_on` (~line 651)**

Find:
```yaml
        cooldown_min: "{{ states('input_number.x1c_enclosure_cooldown_minutes') | int(20) }}"
```
in the variables block of `script.x1c_enclosure_turn_on`. Replace with:
```yaml
        cooldown_min: "{{ states('sensor.x1c_effective_cooldown_min') | int(20) }}"
```

- [ ] **Step 4.2: Edit `automation.x1c_enclosure_print_started` (~line 738)**

Same pattern — find the line in the variables block of `print_started` and replace with the sensor read.

- [ ] **Step 4.3: Edit `automation.x1c_enclosure_update_safety` (~line 828)**

Same pattern — find and replace in the `update_safety` variables block.

- [ ] **Step 4.4: Edit `script.x1c_start_heating_for_print` (~line 996)**

Same pattern — find and replace in the `start_heating_for_print` variables block.

- [ ] **Step 4.5: Sanity check — exactly four edits**

```bash
grep -n "states('input_number.x1c_enclosure_cooldown_minutes')" packages/x1c_enclosure.yaml
```

Expected: **no output** (all four replaced).

```bash
grep -n "states('sensor.x1c_effective_cooldown_min')" packages/x1c_enclosure.yaml
```

Expected: 4 lines.

- [ ] **Step 4.6: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 4.7: Smoke test — verify safety-timer math is unchanged with toggle off**

In Developer Tools → Template (or via REST API), evaluate:
```jinja
{{ states('sensor.x1c_effective_cooldown_min') | int(20) }}
```
Expected with toggle off: 20. With toggle on: 50.

- [ ] **Step 4.8: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "refactor(x1c): route safety-timer cooldown_min through effective sensor"
git push
```

---

### Task 5: Add slow-cooldown branch to `script.x1c_enclosure_cooldown`

**Files:**
- Modify: `packages/x1c_enclosure.yaml` lines 340–377 (the existing cooldown script)

**Design intent:** wrap the entire current sequence in a `choose` whose first condition is the slow toggle. The slow branch reads the printer chamber sensor and exits each phase on threshold OR max timer. The fast branch is the existing sequence verbatim.

- [ ] **Step 5.1: Replace the script body**

Find the existing block:
```yaml
  x1c_enclosure_cooldown:
    alias: 'X1C Enclosure: Controlled Cooldown'
    description: 30 min cooldown — heater ramps down, fan ramps up incrementally
    mode: restart
    sequence:
    # Phase 1: Heater stays on, fan low (0-10 min)
    - action: fan.set_percentage
      target:
        entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
      data:
        percentage: 30
    - delay: '00:10:00'
    # Phase 2: Heater off — lower target to 20°C so thermostat doesn't re-enable
    - action: climate.set_temperature
      target:
        entity_id: climate.x1c_gehause_thermostat
      data:
        temperature: 20
    - delay: '00:05:00'
    - action: fan.set_percentage
      target:
        entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
      data:
        percentage: 50
    - delay: '00:05:00'
    - action: fan.set_percentage
      target:
        entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
      data:
        percentage: 70
    - delay: '00:05:00'
    - action: fan.set_percentage
      target:
        entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
      data:
        percentage: 100
    - delay: '00:05:00'
    - action: script.x1c_enclosure_turn_off
```

Replace with:

```yaml
  x1c_enclosure_cooldown:
    alias: 'X1C Enclosure: Controlled Cooldown'
    description: >
      Two profiles: fast (30 min, ramped) or slow (closed-loop on chamber temp,
      ~35-47 min adaptive). Profile is chosen at script invocation by reading
      input_boolean.x1c_slow_cooldown_mode. The hot-swap automation re-triggers
      this script when the toggle flips on mid-cooldown.
    mode: restart
    sequence:
    - choose:
      # =====================================================================
      # SLOW PROFILE — closed-loop on chamber temperature, gentle for ABS/ASA/PC
      # =====================================================================
      - conditions: "{{ is_state('input_boolean.x1c_slow_cooldown_mode', 'on') }}"
        sequence:
        # Heater off immediately — slow profile starts from zero active input.
        - action: climate.set_temperature
          target:
            entity_id: climate.x1c_gehause_thermostat
          data:
            temperature: 20
        # Phase A — Soak/equalise. Chamber sealed, aux fan circulates internally
        # to break up vertical air stratification around the part.
        # NOTE (empirical): the bauteillüfter at 10% addresses a secondary
        # gradient (top-vs-side air stratification, ~5-10°C). The dominant
        # gradient is bed-to-air (~30°C) and is unaffected. Defensible, not
        # clearly critical. If parts warp more than without, set this to 0.
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
          data:
            percentage: 0
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_bauteillufter
          data:
            percentage: 10
        - wait_template: >
            {{ states('sensor.x1c_00m00a2c0601660_temperatur_im_druckraum') | float(999) < 50 }}
          timeout: '00:20:00'
          continue_on_timeout: true
        # Phase B — Gentle purge. Slight exhaust, circulation continues.
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
          data:
            percentage: 10
        - wait_template: >
            {{ states('sensor.x1c_00m00a2c0601660_temperatur_im_druckraum') | float(999) < 40 }}
          timeout: '00:15:00'
          continue_on_timeout: true
        # Phase C — Approach ambient. Aux fan off, chamber exhaust ramps up.
        # NOTE (empirical): in warm rooms (>25°C ambient) the 32°C threshold
        # may not be reachable with 25% exhaust — Phase C then degrades to a
        # pure 10-min timed phase. That is acceptable, the timer is the
        # documented fallback.
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_bauteillufter
          data:
            percentage: 0
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
          data:
            percentage: 25
        - wait_template: >
            {{ states('sensor.x1c_00m00a2c0601660_temperatur_im_druckraum') | float(999) < 32 }}
          timeout: '00:10:00'
          continue_on_timeout: true
        # Phase D — Final purge to clear residual warm air and VOCs.
        - action: fan.set_percentage
          target:
            entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
          data:
            percentage: 50
        - delay: '00:02:00'
        - action: script.x1c_enclosure_turn_off
      # =====================================================================
      # FAST PROFILE (default) — original 30 min ramped cooldown, unchanged
      # =====================================================================
      default:
      # Phase 1: Heater stays on, fan low (0-10 min)
      - action: fan.set_percentage
        target:
          entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
        data:
          percentage: 30
      - delay: '00:10:00'
      # Phase 2: Heater off — lower target to 20°C so thermostat doesn't re-enable
      - action: climate.set_temperature
        target:
          entity_id: climate.x1c_gehause_thermostat
        data:
          temperature: 20
      - delay: '00:05:00'
      - action: fan.set_percentage
        target:
          entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
        data:
          percentage: 50
      - delay: '00:05:00'
      - action: fan.set_percentage
        target:
          entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
        data:
          percentage: 70
      - delay: '00:05:00'
      - action: fan.set_percentage
        target:
          entity_id: fan.x1c_00m00a2c0601660_druckraumlufter
        data:
          percentage: 100
      - delay: '00:05:00'
      - action: script.x1c_enclosure_turn_off
```

- [ ] **Step 5.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.` If you see indentation errors, the most common cause is mixing the `- choose:` indent with the script's `sequence:` — every `choose`/`default`/`conditions` element must be indented one extra level inside the script.

- [ ] **Step 5.3: Reload and dry-test fast branch (toggle off)**

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -X POST http://supervisor/core/api/services/homeassistant/reload_all && sleep 2 && curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST http://supervisor/core/api/services/script/x1c_enclosure_cooldown'
```

Then immediately stop it (we don't want to actually run a 30-min cooldown):
```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"script.x1c_enclosure_cooldown"}'"'"' http://supervisor/core/api/services/script/turn_off'
```

Expected: in the Logbook the script entered the *default* (fast) branch and set the chamber fan to 30%. No template errors.

- [ ] **Step 5.4: Dry-test slow branch (toggle on)**

```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_on && sleep 1 && curl -s -H "Authorization: Bearer $TOKEN" -X POST http://supervisor/core/api/services/script/x1c_enclosure_cooldown && sleep 3 && curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"script.x1c_enclosure_cooldown"}'"'"' http://supervisor/core/api/services/script/turn_off && curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_off'
```

Expected: Logbook shows climate target → 20°C, druckraumlüfter → 0%, bauteillüfter → 10% (the start of the slow branch). No template errors.

- [ ] **Step 5.5: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): add slow cooldown branch to cooldown script"
git push
```

---

### Task 6: Template the `print_ended` post_print timer duration

**Files:**
- Modify: `packages/x1c_enclosure.yaml` ~line 794–798

**Why:** the fixed 35 min `timer.start` would expire at minute 35 of a slow cooldown that takes up to 47 min, causing `cooldown_backup` to call `script.x1c_enclosure_turn_off` mid-Phase-C and abort the run.

- [ ] **Step 6.1: Edit the timer.start step**

Find in `automation.x1c_enclosure_print_ended` (in the `default:` branch of its `choose`):
```yaml
      - action: timer.start
        target:
          entity_id: timer.x1c_enclosure_post_print
        data:
          duration: '00:35:00'
```

Replace with:
```yaml
      - action: timer.start
        target:
          entity_id: timer.x1c_enclosure_post_print
        data:
          # Slow cooldown can run up to ~47 min (Phase A 20 + B 15 + C 10 + D 2).
          # 55 min in slow mode = 47 min profile + 8 min safety margin.
          duration: >
            {% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}
              00:55:00
            {% else %}
              00:35:00
            {% endif %}
```

- [ ] **Step 6.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 6.3: Verify template renders**

In Developer Tools → Template, paste:
```jinja
{% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}
  00:55:00
{% else %}
  00:35:00
{% endif %}
```
Expected with toggle off: `00:35:00`. Toggle on: `00:55:00`.

- [ ] **Step 6.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "fix(x1c): extend post_print timer to 55 min in slow cooldown mode"
git push
```

---

### Task 7: Update status sensor label to show slow mode

**Files:**
- Modify: `packages/x1c_enclosure.yaml` line 159 (the `nachlauf` branch of the `X1C Enclosure Status` template sensor)

- [ ] **Step 7.1: Edit the cooldown label**

Find:
```yaml
        {% elif mode == 'nachlauf' %}Cooldown ({{ temp | round(1) }}°C)
```

Replace with:
```yaml
        {% elif mode == 'nachlauf' %}{% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}Cooldown (slow, {{ temp | round(1) }}°C){% else %}Cooldown ({{ temp | round(1) }}°C){% endif %}
```

- [ ] **Step 7.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 7.3: Reload and verify both labels render**

Toggle on, then evaluate `sensor.x1c_enclosure_status` in Developer Tools → States. The state attribute should reflect the slow label when mode is `nachlauf`. With nothing currently cooling down, just verify the template doesn't error: paste in Developer Tools → Template:

```jinja
{% set mode = 'nachlauf' %}
{% set temp = 45.5 %}
{% if mode == 'nachlauf' %}{% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}Cooldown (slow, {{ temp | round(1) }}°C){% else %}Cooldown ({{ temp | round(1) }}°C){% endif %}{% endif %}
```

Expected with toggle off: `Cooldown (45.5°C)`. Toggle on: `Cooldown (slow, 45.5°C)`.

- [ ] **Step 7.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): show slow cooldown label in status sensor"
git push
```

---

### Task 8: Add hot-swap automation for mid-cooldown profile switch

**Files:**
- Modify: `packages/x1c_enclosure.yaml` — append a new automation in the automations section, near the other cooldown-related automations (after `x1c_enclosure_cooldown_backup`, ~line 810)

**Why:** without this, toggling slow mode on while a fast cooldown is already running has no effect — the script branched at invocation. The user explicitly asked to be able to activate slow mode "shortly after a print," which implies the print-ended cooldown may already be running. This automation re-triggers the cooldown script when the toggle flips on, and since the script is `mode: restart`, it cleanly aborts the in-flight fast branch and starts the slow branch from scratch.

**Important:** the automation must NOT fire when the toggle flips on outside an active cooldown (e.g., during a print or when the heater is off), otherwise it would spuriously start a cooldown. Guard with a condition on `input_select.x1c_enclosure_mode == 'nachlauf'`.

- [ ] **Step 8.1: Insert the automation**

After the closing of `x1c_enclosure_cooldown_backup` automation (after the `action: script.x1c_enclosure_turn_off` line near line 810) and before the next section header, insert:

```yaml
  - id: x1c_slow_cooldown_hot_swap
    alias: 'X1C: Slow Cooldown Hot-Swap → Restart Cooldown Script'
    description: >
      When the user toggles slow cooldown ON while a cooldown is already running,
      restart the cooldown script so it picks up the slow profile. Without this,
      the in-flight fast cooldown would continue to completion because the script
      reads the toggle only at invocation time. mode: restart on the cooldown
      script ensures the abort/restart is clean.
    mode: single
    trigger:
    - platform: state
      entity_id: input_boolean.x1c_slow_cooldown_mode
      from: 'off'
      to: 'on'
    condition:
    # Only act if a cooldown is actually running. Outside cooldown the toggle
    # is just a passive setting for the next print.
    - condition: state
      entity_id: input_select.x1c_enclosure_mode
      state: nachlauf
    action:
    - action: script.x1c_enclosure_cooldown
    # Also extend the post_print backup timer to the slow duration. The
    # original print_ended automation started it at 35 min; we now need 55.
    - action: timer.start
      target:
        entity_id: timer.x1c_enclosure_post_print
      data:
        duration: '00:55:00'
    - action: persistent_notification.create
      data:
        title: 'X1C Enclosure: Slow Cooldown Activated'
        message: >
          Cooldown was already running — restarting with slow profile.
          ({{ now().strftime('%H:%M:%S') }})
        notification_id: x1c_slow_cooldown_hot_swap
```

- [ ] **Step 8.2: Validate config**

```bash
ssh niko@hassio.rlxd.xyz 'ha core check'
```

Expected: `Configuration is valid.`

- [ ] **Step 8.3: Verify the guard works (toggle outside cooldown)**

Make sure mode is currently `aus`:
```bash
ssh niko@hassio.rlxd.xyz 'curl -s -H "Authorization: Bearer $(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2)" http://supervisor/core/api/states/input_select.x1c_enclosure_mode'
```

If state is `aus`, toggle slow mode on:
```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_on'
```

Expected: in Logbook, the trigger fires but the condition fails, automation does not run. No cooldown started. Then toggle off again to clean up:
```bash
ssh niko@hassio.rlxd.xyz 'TOKEN=$(grep SUPERVISOR_TOKEN /etc/profile.d/homeassistant.sh | cut -d= -f2); curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X POST -d '"'"'{"entity_id":"input_boolean.x1c_slow_cooldown_mode"}'"'"' http://supervisor/core/api/services/input_boolean/turn_off'
```

- [ ] **Step 8.4: Commit**

```bash
git add packages/x1c_enclosure.yaml
git commit -m "feat(x1c): add hot-swap automation for mid-cooldown profile switch"
git push
```

---

### Task 9: End-to-end runtime validation runbook

This task does not modify code. It is a runbook the implementer must execute on the next real ABS/ASA print to confirm the slow cooldown works end-to-end.

- [ ] **Step 9.1: Pre-print — toggle slow mode on**

Via HA UI or REST: turn `input_boolean.x1c_slow_cooldown_mode` to **on** before the print starts. Verify `sensor.x1c_effective_cooldown_min` = 50 and `sensor.x1c_enclosure_status` shows the regular preheat/printing label (slow label only appears in `nachlauf`).

- [ ] **Step 9.2: During print — verify safety timer math**

Look at the safety timer auto-off time (`sensor.x1c_enclosure_auto_off`). It should be sized using slow cooldown minutes (50) not fast (20) — i.e., later than it would have been pre-feature.

- [ ] **Step 9.3: At print end — observe phase transitions**

When the print ends, the cooldown script starts. In the Logbook, verify in order:
1. Climate target → 20°C
2. Druckraumlüfter → 0%
3. Bauteillüfter → 10%
4. (wait until chamber < 50°C OR 20 min)
5. Druckraumlüfter → 10%
6. (wait until chamber < 40°C OR 15 min)
7. Bauteillüfter → 0%, druckraumlüfter → 25%
8. (wait until chamber < 32°C OR 10 min)
9. Druckraumlüfter → 50%
10. (2 min)
11. `script.x1c_enclosure_turn_off` runs

Note the wall-clock time of each transition. Expected total: 35–47 min. `sensor.x1c_enclosure_status` should display `Cooldown (slow, ...)` throughout.

- [ ] **Step 9.4: Verify timer is the longer one**

During the cooldown, check `timer.x1c_enclosure_post_print` `finishes_at` attribute. It should be 55 min from the print-ended event, not 35.

- [ ] **Step 9.5: Hot-swap test (next print, optional)**

Start with slow mode **off**. Wait until print ends and fast cooldown begins. Within the first 5 minutes, toggle slow mode **on**. Expected: persistent notification "Slow Cooldown Activated", cooldown script restarts, status sensor switches to `Cooldown (slow, ...)`, phases proceed as in Step 9.3.

- [ ] **Step 9.6: Document observations**

Append to the spec file (`docs/superpowers/specs/2026-04-06-slow-cooldown-mode-design.md`) a short "Validation Notes" section with the actual phase durations observed and any deviations. This closes the loop on the empirical questions raised in Task 5 comments (bauteillüfter benefit, Phase C in warm ambient).

- [ ] **Step 9.7: Final commit (if observations doc was added)**

```bash
git add docs/superpowers/specs/2026-04-06-slow-cooldown-mode-design.md
git commit -m "docs(x1c): add slow cooldown validation notes from first run"
git push
```

---

## Self-Review Checklist

(Already executed by the plan author. Findings:)

- **Spec coverage:** All sections of the spec map to tasks. New input_boolean → Task 1; input_number → Task 2; template sensor → Task 3; 4 cooldown_min replacements → Task 4; cooldown script branch → Task 5; post_print timer → Task 6; status label → Task 7. **Spec gap addressed:** the spec did not include the hot-swap automation; per user instruction it lives in this plan instead → Task 8. Empirical honesty notes (bauteillüfter, Phase C) live as YAML comments in Task 5 instead of spec edits.
- **Placeholder scan:** No TBDs, no "add error handling," no "similar to Task N." All YAML provided literally.
- **Type consistency:** `input_boolean.x1c_slow_cooldown_mode` (Task 1) is referenced consistently in Tasks 3, 5, 6, 7, 8. `sensor.x1c_effective_cooldown_min` (Task 3) is referenced consistently in Task 4. `input_number.x1c_enclosure_slow_cooldown_minutes` (Task 2) only flows through the template sensor in Task 3. `timer.x1c_enclosure_post_print` (existing) referenced consistently as `00:55:00` in slow mode in both Tasks 6 and 8.
- **Order dependency:** Tasks 1–3 must come before Task 4 (sensor must exist before reads). Task 4 can be before or after Task 5 — independent. Tasks 6, 7, 8 are independent of each other and of Task 5 (but should come after Task 1 for the toggle entity). Task 9 is the runtime validation runbook and must be last.
