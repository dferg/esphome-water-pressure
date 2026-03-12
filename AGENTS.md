# Agent Instructions — Water Pressure Monitor

## Project Overview

ESPHome firmware for an ESP32 that reads four FUSCH 200 PSI pressure
transducers via ADC and reports PSI values to Home Assistant.

## Architecture

```
water-pressure-esp32.yaml          (entry point)
  └─ packages:
       ├─ common/pressure-sensor.yaml   vars: sensor 1
       ├─ common/pressure-sensor.yaml   vars: sensor 2
       ├─ common/pressure-sensor.yaml   vars: sensor 3
       └─ common/pressure-sensor.yaml   vars: sensor 4
```

The main YAML declares substitutions with defaults and includes the sensor
template four times using ESPHome's packages-as-templates feature.  Each
inclusion passes a unique `sensor_id`, `sensor_pin`, and `sensor_name` via
`vars`.

Consumers pull the package from a remote git repository and override
substitutions in their own YAML (see `example.yaml`).

## File Responsibilities

| File | Purpose |
|------|---------|
| `water-pressure-esp32.yaml` | Device definition: substitutions, ESP32 platform, connectivity, dashboard import, package includes |
| `common/pressure-sensor.yaml` | Reusable single-sensor template: ADC config, filter chain, diagnostic voltage entity |
| `example.yaml` | Reference consumer config demonstrating remote package usage and overrides |
| `README.md` | Human-facing docs: BOM, wiring, setup, calibration, troubleshooting |
| `AGENTS.md` | This file — AI agent guidance |

## Substitution Conventions

- All substitutions are declared in `water-pressure-esp32.yaml` with sensible
  defaults so the package works out of the box.
- Sensor-specific substitutions follow the pattern `pressure_N_name`,
  `pressure_N_pin` where N is 1–4.
- Template vars in `common/pressure-sensor.yaml` are prefixed with `sensor_`
  to avoid collisions with top-level substitutions.
- Consumers override substitutions in their own YAML; the package itself never
  references `!secret`.

## Sensor Architecture

Each sensor channel produces two Home Assistant entities from a single ADC
read using the `copy` platform:

1. **Voltage sensor** (`${sensor_id}_voltage`, diagnostic) — the ADC sensor.
   Filter chain: median (window 5) then multiply by `voltage_divider_factor`.
   Reports recovered transducer signal voltage in volts.

2. **PSI sensor** (`${sensor_id}`, normal) — a `copy` of the voltage sensor.
   Applies a lambda that converts voltage to PSI:

```
PSI = (V_sensor - 0.5) / 4.0 * max_psi
```

Where:
- 0.5 V is the transducer output at 0 PSI
- 4.0 V is the span (4.5 V - 0.5 V)
- `max_psi` is the transducer full-scale rating (default 200)

Negative results are clamped to 0.

Using `copy` instead of a second `adc` sensor avoids ESPHome's "pin used in
multiple places" validation error and eliminates redundant ADC reads.

## Calibration Math Reference

### Voltage Divider

```
V_adc = V_sensor * R2 / (R1 + R2)
V_sensor = V_adc * (R1 + R2) / R2
```

Default: R1 = 4.7 kΩ, R2 = 10 kΩ → factor = 1.47

### PSI Conversion

The FUSCH transducer outputs a linear 0.5–4.5 V signal across its rated
pressure range:

```
PSI = (V_sensor - 0.5) / 4.0 * max_psi
```

### calibrate_linear Override

For multi-point calibration, consumers append a `calibrate_linear` filter
using `!extend` on the sensor ID.  Because `!extend` concatenates filter
lists, the calibration runs after the default multiply + lambda chain.
Datapoints map uncalibrated PSI readings (from Home Assistant) to reference
gauge PSI.  Use `method: exact` for piecewise-linear interpolation.

## Editing Guidelines

- Keep the sensor template (`common/pressure-sensor.yaml`) generic.  It must
  work when included multiple times with different vars.
- Never add `!secret` references to package files; use substitutions instead.
- When adding new substitutions, always provide a default value.
- Pin assignments must stay on ADC1 (GPIO32–GPIO39) to avoid WiFi conflicts.

## Validation

After any YAML changes, validate syntax by running:

```bash
/opt/homebrew/bin/esphome config water-pressure-esp32.yaml
```

The command must exit with `Configuration is valid!` and exit code 0.  Fix
all errors before considering a change complete.
