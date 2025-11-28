# Adaptive Motion + Lux Lighting Blueprint

This document explains how `yasl.yaml` drives lights based on motion, lux, night mode, and the global enable entity. Use it as a reference when troubleshooting or customizing the blueprint.

## Inputs

- **lights** – One or more `light` entities the automation controls.
- **motion_sensor** – Motion/presence `binary_sensor` (required even if you plan to disable night mode).
- **lux_sensor** – `sensor` providing illuminance readings.
- **disable_entity** – Master switch; when `off`, the automation shuts down immediately.
- **enable_night_mode** – Boolean that allows/disallows motion-driven night lighting.
- **Night/Day tuning** – Night start/end hours, brightness, color temperatures, and lux thresholds.
- **Global tuning** – Brightness/CT hysteresis and transition time.

## Derived Variables

At each run the automation computes:

- `is_night` using the configured start/end hours (handles wrap-around at midnight).
- Current brightness/CT of the first light to provide hysteresis (minimum step) logic.
- Daytime CT target based on sun elevation and clamped to 1500–6500 K.
- Adaptive brightness target derived from the lux thresholds with linear interpolation between `lux_low` and `lux_high` and hysteresis via `min_brightness_step`.

## Triggers

| Trigger ID     | Description                                      |
|----------------|--------------------------------------------------|
| `motion_on`    | Motion sensor turns `on`.                         |
| `motion_off`   | Motion sensor turns `off`.                        |
| `lux`          | Lux sensor state changes.                         |
| `time_tick`    | Template clock tick every 2 minutes.             |
| `disable_off`  | Global disable entity turns `off`.               |
| `disable_on`   | Global disable entity turns `on`.                |

All logic paths live inside one `choose` block, so exactly one branch executes per trigger.

## Execution Paths

### 1. Global Disable (`disable_off`)

- Fires when the disable entity goes `off`.
- Immediately turns off all lights using the configured transition.
- Stops the automation (`stop` action) so no other branch runs while disabled.

### 2. Night – Motion On (`motion_on` + night enabled)

- Conditions: trigger id `motion_on`, disable entity `on`, `is_night`, and `enable_night_mode` true.
- Lights turn on instantly at `night_brightness` and `ct_night` with optional transition.

### 3. Night – Motion Off (`motion_off` + night enabled)

- Conditions: trigger id `motion_off`, disable entity `on`, `is_night`, and night mode enabled.
- Waits `off_delay_seconds`, re-checks the motion sensor is still `off`, then turns off the lights.

### 4. Day – Lux-Based Control (`lux` trigger)

- Conditions: trigger id `lux`, disable entity `on`, and `not is_night`.
- Inner `choose` block:
  - **Lux low branch**: if lux `< lux_low`, compute adaptive brightness and daytime CT (including hysteresis) and call `light.turn_on`.
  - **Lux high branch**: if lux `> lux_high`, turn lights off with transition.

### 5. Periodic Re-evaluation (`time_tick` or `disable_on`)

- Runs every 2 minutes and whenever the automation is re-enabled, provided the disable entity is currently `on`.
- Inner `choose` block (first matching branch executes):
  1. **Night enforcement** – If `is_night` and any controlled light is on, turn them off. This ensures that when the schedule enters the night window the lights start from an off state, regardless of night-mode setting.
  2. **Daytime lux high** – If it is daytime and lux is above `lux_high`, turn lights off (useful when re-enabled or when lux didn’t trigger by itself).
  3. **Daytime backup turn-on** – If it is daytime, all lights are currently off, and lux is below `lux_low`, compute adaptive brightness/CT and turn the group on. This covers scenarios where the lux sensor hasn’t emitted a fresh state change but conditions are already dark.
  4. **Daytime adapt active lights** – If it is daytime and any controlled light is on, recompute adaptive brightness/CT and call `light.turn_on`. Lux-based turn-on behavior still comes from the dedicated `lux` trigger path; this branch just keeps existing daytime scenes synced.

### 6. Implicit Global Enable (`disable_on`)

- When the disable entity turns `on`, the periodic block executes (see above). This immediately reevaluates whether lights should be on/off based on current lux/night state.

## Notes and Best Practices

- Because motion triggers only do anything at night *and* when `enable_night_mode` is true, you can satisfy the required motion input by pointing to a dummy motion sensor if you want purely lux-based control.
- The `time_tick` trigger guarantees that both night enforcement and daytime adaptation run even if the lux sensor is stale, so the lights never get stuck in the wrong state for more than ~2 minutes.
- Whenever `disable_entity` is `off`, none of the other logic executes; when it turns back `on`, the lights immediately reevaluate via the periodic branch.
