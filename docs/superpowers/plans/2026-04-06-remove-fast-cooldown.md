# Plan: Remove Cooldown Logic — Heater Off at Print End

Date: 2026-04-06
Target file: `packages/x1c_enclosure.yaml` (+ `dashboard.yaml`)

## Goal

When a print finishes, the heater turns off immediately and the chamber/part
fans are left untouched — exactly as the printer manages them on its own
(typically off after a print). No fan ramps, no controlled cooldown phase, no
"nachlauf" mode.

Safety timer / hardcap / post_print backup timer / overtemp / divergence /
comm-loss watchdogs all stay intact.

## Behavior changes

- **Print ends** (`x1c_druck_aktiv` off, or status failed/finished/offline)
  → call `script.x1c_enclosure_turn_off`. Done. No `nachlauf` mode, no fan
  manipulation.
- **`script.x1c_enclosure_turn_off`** continues to: heater switch off,
  thermostat off, mode → `aus`. **Removes** the `fan.turn_off` calls — fans
  belong to the printer.
- **`script.x1c_enclosure_emergency_off`** keeps killing the heater + setting
  mode to `fehler`, but **stops** forcing chamber fan to 100%. Fans stay
  whatever the printer left them at. (Overtemp is a heater problem; forcing
  printer fans on isn't our job and conflicts with the "printer owns fans"
  invariant.)
- The `nachlauf` input_select option and the cooldown template-sensor branch
  become dead. Leave the option in `input_select` (state history continuity),
  but remove the rendering branch in `X1C Enclosure Status`.

## Removals

1. `script.x1c_enclosure_cooldown` — entire script (lines 340–377).
2. `input_number.x1c_enclosure_cooldown_minutes` — entire helper (lines 57–63).
3. `timer.x1c_enclosure_post_print` is **kept** (still acts as the
   belt-and-suspenders backup that fires `turn_off` if anything else fails),
   but its duration becomes a small fixed value (e.g. `00:02:00`) since there
   is no longer a cooldown window to cover.
4. All `script.turn_off: script.x1c_enclosure_cooldown` calls in
   `turn_off` / `emergency_off` (lines 223–225, 255–257).
5. Fan manipulation in `turn_off` (lines 226–230) and `emergency_off`
   (lines 258–262).
6. `nachlauf` rendering branch in `X1C Enclosure Status` template (line 159).
7. Dashboard row for `input_number.x1c_enclosure_cooldown_minutes`
   (`dashboard.yaml` lines 85–86).

## Edits to safety-timer math

`cooldown_min` is currently added to the safety-timer budget so the timer
extends past the print's expected end. With the cooldown gone, replace every
`cooldown_min` reference with `safety_buffer_minutes` (already exists, default
10 min) — the buffer alone is sufficient slack past `restzeit_min`.

Affected blocks:
- `automation: x1c_enclosure_print_started` action vars (lines ~651, 666)
- `automation: x1c_enclosure_*` second block (lines ~738, 751)
- `automation: x1c_enclosure_update_safety` (lines ~828, 847)
- `automation: x1c_enclosure_preheat_resume` recalc block (lines ~996, 1002)

In each: drop the `cooldown_min` variable and change
`(remaining_min + cooldown_min + buffer_min)` → `(remaining_min + buffer_min)`.

## Edits to `print_ended` automation (lines 758–799)

Replace the `choose` block entirely with a single sequence:

```yaml
action:
  - action: script.x1c_enclosure_turn_off
```

Drop:
- the `nachlauf` mode select
- the `timer.x1c_enclosure_post_print` start  *(see note below)*
- the `script.x1c_enclosure_cooldown` call
- the failed/offline branch (no longer special — turn_off applies to all
  end-states uniformly)

**Backup timer note:** Since `turn_off` is now the only path and runs
synchronously, the post_print backup timer is largely redundant. Either:

- (a) **Drop `timer.x1c_enclosure_post_print` and the
  `x1c_enclosure_cooldown_backup` automation entirely.** Simpler. The safety
  timer still provides a hard ceiling.
- (b) Keep them as a 2-min belt-and-suspenders catch-all.

**Recommendation: option (a).** The safety timer + overtemp watchdogs already
cover any failure mode where `turn_off` doesn't actually shut off the relay,
and the cooldown_backup automation only existed because `script.x1c_enclosure_cooldown`
itself had a 30-minute window where things could go wrong.

## Files to change

| File | Change |
|---|---|
| `packages/x1c_enclosure.yaml` | All edits above |
| `dashboard.yaml` | Remove cooldown_minutes row |

## Validation

1. `python -c "import yaml; yaml.safe_load(open('packages/x1c_enclosure.yaml'))"` → no errors.
2. Grep for stray references: `cooldown_min`, `x1c_enclosure_cooldown`,
   `x1c_enclosure_cooldown_minutes`, `nachlauf`, `post_print` should all
   either be gone or only appear in expected spots (state history,
   `input_select` option list).
3. Commit + push → webhook reloads HA → Developer Tools → check no startup
   errors, mode shows `aus` after a test print, heater relay confirmed off,
   chamber fan state matches what the printer set (not forced).
4. Trigger a manual `script.x1c_enclosure_emergency_off` and confirm fans are
   **not** touched.

## Risks

- **Heater shuts off the instant `druck_aktiv` flips off** — if the printer
  ever flaps that sensor mid-print we'd kill heat. Mitigation: the existing
  `x1c_enclosure_preheat_resume` automation re-enables on `paused_user`, and
  flapping during `printing` step is not a known failure mode. No change from
  current cooldown behavior in this regard (it also fires on the same trigger).
- **`nachlauf` state in history breaks dashboards** that filter on it.
  Mitigation: leaving the input_select option present even if unused.
- **Removing emergency-off fan blast** means in a true overtemp event the
  chamber stays sealed at whatever fan setting the printer has. Acceptable
  because overtemp triggers also cut the heater (root cause), and venting via
  printer fans is the printer's responsibility per the new invariant. If this
  feels wrong, we can leave the `emergency_off` fan-100% line in as a one-off
  exception — flag during review.
