# Water Pressure Monitor

An ESPHome project that uses a LILYGO T-Display-S3 and ADS1115 16-bit ADC to
read water pressure from four FUSCH 200 PSI pressure transducers, display
live readings on the built-in LCD, and report values to Home Assistant.

**Disclaimer:** This project was vibe coded. It suits my purposes. Use at your own risk.

## Bill of Materials

| Qty | Part | Notes |
|-----|------|-------|
| 1 | LILYGO T-Display-S3 | ESP32-S3, 1.9" ST7789 LCD, Qwiic connector |
| 1 | QCRobot ADS1115 16-bit ADC | 4-channel, I2C, 3.3V I2C logic, 5V analog |
| 4 | FUSCH 200 PSI Pressure Transducer (1/8" NPT) | 0.5-4.5 V output, 5 V supply |
| 1 | 5 V power supply (1 A+) | Powers transducers and ADS1115 VDD |
| — | Hookup wire, Qwiic cable, connectors | — |

## Wiring

Each transducer has three wires:

| Wire | Color | Connection |
|------|-------|------------|
| VCC | Red | 5 V supply |
| GND | Black | Common ground |
| Signal | Green / Yellow | ADS1115 AIN0-AIN3 |

### System Connections

```
    T-Display-S3              QCRobot ADS1115
   ┌────────────┐            ┌───────────┐
   │ USB-C (5V) │            │ VDD       ├──── 5V supply
   │            │            │           │
   │ Qwiic port │──Qwiic─────│ I2C port  │  (3.3V logic)
   │ (3.3V I2C) │  cable     │           │
   │            │            │ AIN0      ├──── Transducer 1 Signal
   │   Display  │            │ AIN1      ├──── Transducer 2 Signal
   │   (built   │            │ AIN2      ├──── Transducer 3 Signal
   │    in)     │            │ AIN3      ├──── Transducer 4 Signal
   └────────────┘            │ GND       ├──── Common GND
                             └───────────┘
```

### I2C / Qwiic Connection

The T-Display-S3 has a Qwiic/Stemma connector (JST-SH 1.0mm 4-pin) that
provides 3.3V I2C. The QCRobot ADS1115 also uses 3.3V I2C logic regardless
of its supply voltage. Connect them with a standard Qwiic cable.

| T-Display-S3 (Qwiic) | QCRobot ADS1115 |
|-----------------------|-----------------|
| 3.3V | 3.3V (I2C power) |
| GND | GND |
| GPIO43 (SDA) | SDA |
| GPIO44 (SCL) | SCL |

The ADS1115 VDD must be powered from **5V** separately so the analog inputs
can accept the transducer's 0.5-4.5V signal range.

### Power Notes

- The T-Display-S3 is powered via USB-C (5V). Its onboard regulator provides
  3.3V to the ESP32-S3 chip and the Qwiic port.
- The QCRobot ADS1115 VDD connects to the **5V supply** (not the Qwiic 3.3V).
  This sets the analog input range to 0-5V. The I2C logic stays at 3.3V
  regardless of VDD.
- Transducers are powered from the same 5V supply.

## Display

The T-Display-S3 has a 1.9" ST7789 LCD (320x170 in landscape). The firmware
provides four display pages that you cycle through with the physical buttons.

### Page 1: Dashboard

All four sensors in a 2x2 grid. Each shows sensor name and current PSI.
Color-coded: green (normal), yellow (warning), red (critical).

### Page 2: Detail View

Single sensor shown large with min/max values and a horizontal bar gauge.
Press Button 2 to cycle which sensor is displayed.

### Page 3: Min/Max List

All four sensors in a compact list with current PSI and min/max values.
Press Button 2 to reset min/max tracking.

### Page 4: Bar Graph

Horizontal bar chart for all four sensors with color-coded fill and PSI
labels for quick visual comparison.

### Buttons

| Button | GPIO | Action |
|--------|------|--------|
| Button 1 | GPIO0 (left) | Cycle to next display page |
| Button 2 | GPIO14 (right) | Context-sensitive (see page descriptions) |

## Software Setup

### Prerequisites

- Home Assistant with the ESPHome add-on **or** the ESPHome CLI
- A USB-C cable to flash the T-Display-S3 for the first time

### Quick Start (ESPHome Dashboard)

Display fonts are loaded from **Google Fonts** at compile time (`gfonts://Roboto`).
The machine running ESPHome needs internet access when you click **Install**
(or run `esphome compile`).  The font is cached afterward.

1. Copy `example.yaml` into your ESPHome configuration directory.
2. Rename it to match your device (e.g. `water-pressure.yaml`).
3. Edit the `substitutions` section -- at minimum set sensor names.
4. Update the `packages:` URL to point to your fork/copy of this repository.
5. Add `wifi_ssid` and `wifi_password` to your ESPHome `secrets.yaml`.
6. Click **Install** in the ESPHome dashboard.

### Quick Start (CLI)

```bash
esphome run example.yaml
```

## Recovery: lost IP, wrong `.local` name, or blank display

### Why `my-water-pressure.local` does not resolve

The firmware defaults to **`name_add_mac_suffix: true`**.  The mDNS hostname is
**`<name>-<last 6 hex chars of MAC>.local`**, not plain `<name>.local`.

Example: `my-water-pressure-a1b2c3.local` (exact suffix comes from your chip).

**Fix for next flash:** In your consumer YAML, set:

```yaml
substitutions:
  name_add_mac_suffix: "false"
```

Then reinstall.  Use this only when you have a single device with that `name`.

### Find the device anyway

1. **USB serial logs** (best): connect USB, then run (adjust port and config path):

   ```bash
   esphome logs my-water-pressure.yaml --device /dev/ttyACM0
   ```

   On macOS the port is often `/dev/cu.usbmodem*` or `/dev/cu.usbserial*`.
   Boot logs show WiFi IP, connection status, and the full device name.

2. **Router DHCP / ARP:** Check your router's client list for a hostname
   containing `my-water-pressure` or `esphome`.

3. **Fallback AP:** If WiFi credentials are wrong, after a timeout the device
   opens an access point (merged from the package).  Connect to the
   **Fallback Hotspot** SSID (name includes your device name), password
   **`12345678`**, then use the captive portal to set WiFi again.

### Blank display

The LCD backlight is a Home Assistant **light** entity (`LCD Backlight`).
After a clean boot it should restore **on**.  If the screen stays black:

- Watch **serial logs** for boot loops, I2C errors, or exceptions.
- Try a **USB power-only** boot (some hubs share power poorly with the display).
- Reflash; temporarily set `name_add_mac_suffix: "false"` and fix WiFi so you
  can reach the API and toggle the backlight entity.

## Configuration Reference

All behaviour is controlled through `substitutions`. Override any of these
in your consumer YAML to customise the device without editing the package.

| Substitution | Default | Description |
|--------------|---------|-------------|
| `name` | `water-pressure` | Device name (mDNS, entity IDs) |
| `name_add_mac_suffix` | `true` | If true, hostname is `<name>-<mac>.local` |
| `friendly_name` | `Water Pressure Monitor` | Human-readable name in HA |
| `update_interval` | `5s` | Sensor sampling interval |
| `max_psi` | `200` | Full-scale PSI of transducer |
| `psi_warn` | `80` | Yellow threshold on display |
| `psi_crit` | `120` | Red threshold on display |
| `pressure_1_name` | `Pressure 1` | Friendly name for sensor 1 |
| `pressure_2_name` | `Pressure 2` | Friendly name for sensor 2 |
| `pressure_3_name` | `Pressure 3` | Friendly name for sensor 3 |
| `pressure_4_name` | `Pressure 4` | Friendly name for sensor 4 |
| `i2c_sda` | `GPIO43` | I2C data pin |
| `i2c_scl` | `GPIO44` | I2C clock pin |
| `ads1115_address` | `0x48` | ADS1115 I2C address |
| `pressure_1_cal_mult` … `pressure_4_cal_mult` | `1.0` | Span scale (see Calibration) |
| `pressure_1_cal_offset` … `pressure_4_cal_offset` | `0.0` | PSI added after scale (see Calibration) |

## Calibration (step-by-step for new users)

Each channel has a **diagnostic entity** in Home Assistant named like
**"Main Supply Voltage"** (your sensor name + ` Voltage`). It shows the
recovered transducer signal in volts (~0.5 V at 0 PSI, ~4.5 V at full scale
for a 200 PSI sensor). Use it to sanity-check wiring before trusting PSI.

The reported **PSI** is computed in firmware, then your **calibration
substitutions** are applied:

```text
PSI_final = (PSI_raw × cal_mult) + cal_offset
```

Defaults are `cal_mult: 1.0` and `cal_offset: 0.0` (no change). Set these in
**your** consumer YAML under `substitutions:` (same file as `packages:` and
`wifi:`). Flash again after each change.

### Step 1 -- Zero PSI (atmospheric) offset

Do this **first** for each sensor that reads wrong at no pressure.

1. **Relieve pressure:** Open the transducer to atmosphere (disconnected from
   pressurized plumbing, or a vented port). True gauge pressure ≈ 0 PSI.
2. **Read Home Assistant:** Note the **PSI** entity (not Voltage), e.g. it
   shows `2.3` PSI but should be `0`.
3. **Compute offset:** You want to subtract the error.

   `cal_offset = -(what HA shows at 0 PSI)`

   Example: HA shows `2.3` → set `pressure_1_cal_offset: "-2.3"`.
4. **Add to your YAML** (example for sensor 1):

   ```yaml
   substitutions:
     pressure_1_cal_offset: "-2.3"
   ```

5. **Install** the firmware again. Recheck at 0 PSI; tweak if needed.

Leave `pressure_N_cal_mult` at `1.0` until Step 2.

### Step 2 -- Span correction (optional)

Use when **zero looks good** but **high pressure** does not match a trusted
reference gauge.

1. Apply a known stable pressure (e.g. ~100 PSI) and read **HA PSI** and the
   **reference gauge**.
2. **Multiplier** (simple scaling, keeps your offset from Step 1 in mind):

   `cal_mult ≈ (reference_PSI) / (HA_PSI_before_mult)`

   Only use the HA reading **before** you would apply mult/offset, or
   temporarily set `cal_offset` back to `0.0` to measure raw error, then set
   both together. Easiest: after Step 1 is done, at a high point:

   - Let `R` = HA PSI at the reference point (with offset already applied).
   - Let `T` = true reference PSI.

   Then `cal_mult_new = cal_mult_old * (T / R)`.

   Example: after offset, HA reads `95` at true `100`, mult was `1.0` →
   `cal_mult = 1.0 * (100 / 95) = 1.053` →

   ```yaml
   pressure_1_cal_mult: "1.053"
   ```

3. **Install** and verify at low and high pressure.

### Quick reference -- all calibration keys

Set any combination in your consumer `substitutions:` (strings with numbers
are fine):

| Substitution | Typical use |
|--------------|-------------|
| `pressure_1_cal_offset` | Zero correction for sensor 1 (negative of error at 0 PSI) |
| `pressure_1_cal_mult` | Span scale for sensor 1 (usually 1.0, then adjust after zero) |
| `pressure_2_cal_offset` … `pressure_4_cal_offset` | Same for sensors 2–4 |
| `pressure_2_cal_mult` … `pressure_4_cal_mult` | Same for sensors 2–4 |

Example -- two sensors adjusted, two left default:

```yaml
substitutions:
  pressure_1_cal_offset: "-1.8"
  pressure_2_cal_mult: "1.02"
  pressure_2_cal_offset: "-0.5"
```

### Step 3 -- Multi-point calibration (advanced)

If the sensor is non-linear or you want many matched points, you can still
append ESPHome's `calibrate_linear` filter in your consumer YAML. It runs
**after** mult and offset, and maps **uncalibrated PSI → true PSI** at each
point.

1. Temporarily set `pressure_N_cal_mult` to `1.0` and `pressure_N_cal_offset`
   to `0.0` for that sensor (or note readings with your current cal applied
   and map **displayed** PSI → reference).
2. Collect pairs `(HA_reading, reference_gauge)` at several pressures.
3. In your consumer YAML (below `packages:`), add:

   ```yaml
   sensor:
     - id: !extend pressure_1
       filters:
         - calibrate_linear:
             method: exact
             datapoints:
               - 0.0 -> 0.0
               - 38.5 -> 40.0
               - 79.2 -> 80.0
               - 118.8 -> 120.0
               - 159.0 -> 160.0
               - 198.5 -> 200.0
   ```

   Replace numbers with **your** measurements (left = HA PSI, right =
   reference PSI).

4. **Install.** For that sensor, you can leave `cal_mult` / `cal_offset` at
   defaults so `calibrate_linear` does all the work, or combine carefully
   (linear runs last).

**Recording sheet (print or copy):**

```
Sensor: ___________________________  Date: ____________

Step 1 (0 PSI):  HA read ______ PSI   ->  pressure_N_cal_offset: ______

Step 2 (high):   Reference ______ PSI   HA read ______ PSI   ->  cal_mult: ______

| Point | Reference (PSI) | HA reading (PSI) | Notes |
|-------|-----------------|-------------------|-------|
| 1     |                 |                   |       |
| 2     |                 |                   |       |
| 3     |                 |                   |       |
| 4     |                 |                   |       |
```

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Reading stuck at 0 PSI | No 5V to transducer | Check power and wiring |
| Reading pegged at max | ADS1115 VDD not 5V | Wire VDD to 5V supply |
| Noisy/jumping values | Poor grounding, long wires | Shorten wires, add shielding |
| I2C errors in log | Bad Qwiic cable, wrong address | Check cable, verify ADDR pin |
| Display blank | Backlight off | Check LCD Backlight entity in HA |
| Negative PSI readings | Transducer offset | Perform Tier 1 calibration |
| Min/Max stuck at initial | No readings yet | Wait for first sensor update |
| Font / compile error | Offline build host, firewall | Allow ESPHome to reach Google Fonts at compile time |

## Project Structure

```
water-pressure/
├── README.md                        # This file
├── AGENTS.md                        # AI agent instructions
├── water-pressure-esp32.yaml        # Main ESPHome package (entry point)
├── common/
│   ├── pressure-sensor.yaml         # Sensor template (included 4x)
│   ├── display.yaml                 # Display config, pages, fonts (Google Fonts), colors
│   └── buttons.yaml                 # Button definitions and actions
└── example.yaml                     # Example consumer configuration
```

## License

This project is provided as-is for personal and educational use.
