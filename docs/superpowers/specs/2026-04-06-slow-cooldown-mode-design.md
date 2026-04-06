# X1C Enclosure: Slow Cooldown Mode — Design

**Date:** 2026-04-06
**Package:** `packages/x1c_enclosure.yaml`
**Status:** Approved design, ready for implementation plan

## Motivation

The existing cooldown profile (`script.x1c_enclosure_cooldown`, ~30 min) is optimised
for a balance of speed and part safety. For large ABS/ASA/PC parts, this is still
too aggressive: forced chamber-air convection decouples air from the heated bed,
creating thermal gradients across the part that cause warping, delamination and
layer cracks.

The user wants an **alternative, closed-loop, temperature-driven slow cooldown
profile** (~35–47 min adaptive) that can be activated via a persistent toggle.
When the toggle is on, every cooldown trigger (print end, manual heater off) runs
the slow profile instead of the fast one.

## Scope

In scope:
- New `input_boolean.x1c_slow_cooldown_mode` toggle.
- Slow-cooldown branch inside the existing `script.x1c_enclosure_cooldown`.
- New `input_number.x1c_enclosure_slow_cooldown_minutes` (default 50) for safety-timer sizing.
- New template sensor `sensor.x1c_effective_cooldown_min` as single source of truth for safety-timer math.
- Dynamic `timer.x1c_enclosure_post_print` duration (35 min normal / 55 min slow).
- UI state indication: cooldown mode label reflects active profile.
- Replace direct `cooldown_min` reads in 4 automations with the effective sensor.

Out of scope:
- Tuning thresholds via UI (hardcoded in script — simpler, less to break).
- Changing fast-cooldown behaviour.
- New safety layers (existing layers remain authoritative).
- Rate-based (dT/dt) control loop (waypoint control is pragmatic and sufficient).

## Physical Rationale

When a print ends, the printer auto-disables the bed heater and its own aux fans.
The bed (~8–10 kg Al+glass) remains the dominant thermal mass and continues
radiating heat into the chamber for 30–60 minutes. Our only remaining actuators
are:
- `switch.x1c_heater` (PTC, 230 W) — kept **off** throughout slow cooldown
- `fan.x1c_00m00a2c0601660_druckraumlufter` (chamber exhaust)
- `fan.x1c_00m00a2c0601660_bauteillufter` (part cooling / aux, repurposed for internal circulation)

Key insight: running the aux fan at a **low setting with the exhaust closed**
circulates air *inside* the sealed chamber without net heat loss. This destroys
thermal stratification (hot air at ceiling, cold at floor) and equalises the
temperature field around the part. The same entity is already used at 100 %
during preheat (`x1c_enclosure_turn_on`, line 642+) to accelerate warm-up — we
reuse the pattern in reverse.

## Cooldown Profile (Slow Branch)

Closed-loop on `sensor.x1c_00m00a2c0601660_temperatur_im_druckraum` (printer
chamber sensor, already used in safety divergence check). Each phase exits on
**temperature threshold OR max timer, whichever first** (max timer is a
fallback against sensor failure).

| Phase | Exit condition | Heater target | Chamber exhaust | Aux fan (circulation) |
|---|---|---|---|---|
| **A — Soak / equalise** | Chamber < 50 °C *or* 20 min | 20 °C (off) | 0 % | 10 % |
| **B — Gentle purge** | Chamber < 40 °C *or* 15 min | 20 °C (off) | 10 % | 10 % |
| **C — Approach ambient** | Chamber < 32 °C *or* 10 min | 20 °C (off) | 25 % | 0 % |
| **D — Final purge** | 2 min fixed | 20 °C (off) | 50 % | 0 % |
| **End** | — | Call `script.x1c_enclosure_turn_off` (kills heater + fans + state) |

Worst-case wall-clock: 20 + 15 + 10 + 2 = **47 min**. Typical: 35–40 min.

First action of the slow branch sets `climate.x1c_gehause_thermostat` target to
20 °C so the thermostat cannot re-enable the heater during Phase A. (Fast profile
does this only after 10 min — that's intentionally different.)

## Fast Branch (unchanged)

The existing 30-minute fast profile stays exactly as implemented today. Branching
is a single `choose` at the top of `script.x1c_enclosure_cooldown`:

```yaml
- choose:
  - conditions: "{{ is_state('input_boolean.x1c_slow_cooldown_mode', 'on') }}"
    sequence: [slow phases...]
  default: [current fast phases...]
```

The toggle is read **once** at script invocation. `mode: restart` ensures a
re-trigger cleanly restarts the whole script (existing behaviour, no change).

## Conflict Resolution with Heat Logic

### Conflict 1 — Fixed 35-min `post_print` timer

`automation.x1c_enclosure_print_ended` currently starts
`timer.x1c_enclosure_post_print` with fixed `duration: '00:35:00'`. When the
timer finishes, `cooldown_backup` calls `script.x1c_enclosure_turn_off` which
cancels the cooldown script mid-flight. At 47 min slow-cooldown this would abort
in Phase C.

**Fix:** Template the duration based on the toggle:

```yaml
- action: timer.start
  target:
    entity_id: timer.x1c_enclosure_post_print
  data:
    duration: >
      {% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}
        00:55:00
      {% else %}
        00:35:00
      {% endif %}
```

55 min = 47 min profile + 8 min safety margin.

### Conflict 2 — Safety-timer sizing reads `cooldown_min` directly

Four locations compute safety timer duration as
`(remaining_min + cooldown_min + buffer_min) * 60`:

- `script.x1c_enclosure_turn_on` (line 651, 666)
- `automation.x1c_enclosure_print_started` (line 738, 751)
- `automation.x1c_enclosure_update_safety` (line 828, 847)
- `script.x1c_start_heating_for_print` (line 996, 1002)

In slow mode, `cooldown_min` (=20) is too small, which can cause the safety
timer to fire during Phase B.

**Fix:** Create a template sensor as single source of truth:

```yaml
template:
- sensor:
  - name: 'X1C Effective Cooldown Minutes'
    unique_id: x1c_effective_cooldown_min
    unit_of_measurement: min
    state: >
      {% if is_state('input_boolean.x1c_slow_cooldown_mode', 'on') %}
        {{ states('input_number.x1c_enclosure_slow_cooldown_minutes') | int(50) }}
      {% else %}
        {{ states('input_number.x1c_enclosure_cooldown_minutes') | int(20) }}
      {% endif %}
```

All four locations replace their `cooldown_min` variable binding with:

```yaml
cooldown_min: "{{ states('sensor.x1c_effective_cooldown_min') | int(20) }}"
```

One-line change per site, semantics stay identical, nothing else in the math
changes.

### Conflict 3 — Heater must be off immediately in slow mode

Fast profile keeps heater target at its working setpoint for the first 10 min of
cooldown. Slow profile must start from zero active input.

**Fix:** First step of the slow branch sets
`climate.set_temperature` target to 20 °C, before any delay. Already covered in
the profile table above, documented here for traceability.

## Non-Conflicts (verified, no changes needed)

- `automation.x1c_enclosure_print_started` calls `script.turn_off` on the
  cooldown script (line 727). New print while slow cooldown runs → clean abort. ✅
- `automation.x1c_enclosure_heat_off` cancels cooldown script and turns both
  fans off (line 696–703). Manual `climate → off` during slow cooldown → clean
  shutdown. ✅
- Safety automations (overtemp, sensor divergence, sensor invalid) trigger
  `script.x1c_enclosure_emergency_off` which is independent of the cooldown
  script and overrides everything. ✅
- `script.x1c_enclosure_cooldown` is already `mode: restart`; the internal
  branch is stable across re-triggers. ✅
- `automation.x1c_enclosure_auto_preheat` only reacts to bed target temp for
  the next print; has no interaction with cooldown timing. ✅

## New / Modified Helpers and Entities

**New:**
- `input_boolean.x1c_slow_cooldown_mode` — persistent toggle, name
  "X1C Enclosure: Slow Cooldown Mode", icon `mdi:snowflake-alert`.
- `input_number.x1c_enclosure_slow_cooldown_minutes` — default 50, min 30, max 90, step 1.
- `sensor.x1c_effective_cooldown_min` — template sensor (see Conflict 2).

**Modified:**
- `script.x1c_enclosure_cooldown` — add `choose` with slow branch as first option.
- `automation.x1c_enclosure_print_ended` — template `timer.start` duration.
- `script.x1c_enclosure_turn_on` — `cooldown_min` variable sources from template sensor.
- `automation.x1c_enclosure_print_started` — same.
- `automation.x1c_enclosure_update_safety` — same.
- `script.x1c_start_heating_for_print` — same.
- `input_select.x1c_enclosure_mode` rendering (template in `sensor.x1c_enclosure_status` around line 159):
  current `"Cooldown (XX°C)"` becomes `"Cooldown (slow, XX°C)"` when the toggle
  is on, so the UI reflects the active profile.

## Rollout / Validation

1. Deploy via existing git-sync workflow (push → webhook → `git pull` → reload).
2. Verify `sensor.x1c_effective_cooldown_min` reads 20 (toggle off) and 50 (toggle on).
3. Dry-run fast cooldown with toggle off — must behave exactly as before.
4. Dry-run slow cooldown with toggle on during a small ABS print — watch phase
   transitions in the logbook, confirm exit-on-temperature works.
5. Verify `timer.x1c_enclosure_post_print` shows 55 min when slow toggle is on at
   print end.

## Risks

- **Chamber sensor stale/unavailable:** `wait_template` with a `timeout` equal
  to the phase max-timer makes each phase degrade gracefully to a pure timed
  phase. No hang risk.
- **User toggles slow mode mid-cooldown:** irrelevant — the script already
  branched at invocation time and runs its phases to completion. The new value
  only affects the *next* cooldown.
- **Bauteillüfter at 10 % blows directly on the part:** at 10 % this is
  circulation, not cooling. Physically preferable to the stratification gradient
  it replaces. If user empirically disagrees later, a one-line code change
  disables it.
