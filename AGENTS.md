# Agent Instructions -- Water Pressure Monitor

## Project Overview

ESPHome firmware for a LILYGO T-Display-S3 that reads four FUSCH 200 PSI
pressure transducers via a QCRobot ADS1115 16-bit ADC over I2C, displays
live readings on the built-in 1.9" LCD, and reports PSI values to Home
Assistant.

## Architecture

```
water-pressure-esp32.yaml          (entry point)
  ├── esp32-s3 + psram + i2c + ads1115
  ├── packages:
  │     ├── common/pressure-sensor.yaml   vars: A0_GND (x4)
  │     ├── common/display.yaml           SPI, LCD, pages, fonts, colors, globals
  │     └── common/buttons.yaml           GPIO0 + GPIO14 handlers
  └── wifi_signal, uptime, restart
```

## File Responsibilities

| File | Purpose |
|------|---------|
| `water-pressure-esp32.yaml` | Substitutions, ESP32-S3 platform, PSRAM, I2C, ADS1115 hub, package includes |
| `common/pressure-sensor.yaml` | Reusable single-sensor template: ADS1115 + copy for PSI |
| `common/display.yaml` | Octal SPI bus, mipi_spi display, backlight, fonts, colors, globals, 4 page lambdas, min/max interval |
| `common/buttons.yaml` | GPIO0 (page cycle) and GPIO14 (context action) binary sensors |
| `example.yaml` | Consumer config demonstrating remote package usage |
| `README.md` | Human-facing docs |
| `AGENTS.md` | This file |

## Hardware

- **Board:** LILYGO T-Display-S3 (ESP32-S3, 16MB flash, 8MB PSRAM)
- **Display:** 1.9" ST7789 LCD, 320x170 landscape, octal SPI via mipi_spi
- **ADC:** QCRobot ADS1115, VDD=5V (analog range), I2C=3.3V logic
- **I2C:** GPIO43 (SDA), GPIO44 (SCL) via Qwiic/Stemma connector
- **Buttons:** GPIO0 (Button 1, left), GPIO14 (Button 2, right)
- **Backlight:** GPIO38 via LEDC PWM

## Substitution Conventions

- All substitutions declared in `water-pressure-esp32.yaml` with defaults.
- `name_add_mac_suffix` (default `true`): when true, mDNS is `<name>-<mac>.local`.
  Consumers may set `"false"` for a fixed `<name>.local` (single device only).
- Sensor names: `pressure_N_name` (N = 1-4).
- Display thresholds: `psi_warn` (yellow), `psi_crit` (red).
- I2C: `i2c_sda`, `i2c_scl`, `ads1115_address`.
- Template vars in `common/pressure-sensor.yaml` prefixed with `sensor_`.
- Package never references `!secret`.

## Display Pages

| Page | ID | Content | Button 2 Action |
|------|----|---------|-----------------|
| 1 | `page_dashboard` | 2x2 grid, all sensors, color-coded PSI | None |
| 2 | `page_detail` | Single sensor large, min/max, bar gauge | Cycle sensor |
| 3 | `page_minmax` | All sensors list with min/max | Reset min/max |
| 4 | `page_bars` | Horizontal bar chart, all sensors | None |

Button 1 (GPIO0) cycles pages sequentially.

## Globals

| ID | Type | Purpose |
|----|------|---------|
| `detail_sensor` | `int` | Index (0-3) of sensor shown on detail page |
| `min_psi` | `float[4]` | Tracked minimum PSI per sensor |
| `max_psi` | `float[4]` | Tracked maximum PSI per sensor |

Min/max are updated every 2 seconds by an interval component.
Reset by Button 2 on the min/max page.

## Sensor Architecture

Each channel produces two HA entities from a single ADS1115 read using `copy`:

1. **Voltage sensor** (`${sensor_id}_voltage`, diagnostic) -- `platform: ads1115`.
   Filter: median (window 5). Gain 6.144 for 0-4.5V range.

2. **PSI sensor** (`${sensor_id}`, normal) -- `platform: copy`.
   Filters: lambda (`PSI = (V - 0.5) / 4.0 * max_psi`, clamped at 0), then
   `multiply` by `sensor_cal_mult`, then `offset` by `sensor_cal_offset`.
   Consumer substitutions: `pressure_N_cal_mult` (default `1.0`),
   `pressure_N_cal_offset` (default `0.0`).  Optional `!extend` +
   `calibrate_linear` runs after these filters.

## Font Conventions

Three sizes from **Google Fonts** (`gfonts://Roboto` in `common/display.yaml`).
This avoids local `file:` paths that break when the YAML is pulled as a remote
package (paths resolve from the consumer's config directory, not the repo).

| ID | Size | Usage |
|----|------|-------|
| `font_large` | 32px | Detail view PSI value |
| `font_medium` | 20px | Dashboard PSI, list values |
| `font_small` | 14px | Headers, labels, min/max, hints |

Glyph sets are kept minimal to reduce flash usage. When adding new text to
lambdas, ensure required characters are in the relevant font's `glyphs` list.

## Color Conventions

| ID | Hex | Usage |
|----|-----|-------|
| `color_green` | 00CC44 | PSI below warn threshold |
| `color_yellow` | DDAA00 | PSI at/above warn, below crit |
| `color_red` | DD2200 | PSI at/above crit threshold |
| `color_white` | FFFFFF | Primary text |
| `color_gray` | 888888 | Labels, secondary text |
| `color_dark_gray` | 333333 | Header bar background |
| `color_cyan` | 00BBCC | Min/max values |
| `color_bg` | 000000 | Screen background |

## Editing Guidelines

- Keep `common/pressure-sensor.yaml` generic (no display references).
- Keep display lambdas in `common/display.yaml`, button logic in
  `common/buttons.yaml`.
- Never add `!secret` references to package files.
- When adding new substitutions, always provide a default value.
- When adding new display text, verify glyphs are in the font definition.
- ADS1115 multiplexer channels: A0_GND through A3_GND.

## Validation

After any YAML changes, validate syntax by running:

```bash
/opt/homebrew/bin/esphome config water-pressure-esp32.yaml
```

The command must exit with `Configuration is valid!` and exit code 0. Fix
all errors before considering a change complete.
