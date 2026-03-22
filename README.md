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
    T-Display-S3                QCRobot ADS1115
   ┌────────────┐              ┌───────────┐
   │ USB-C (5V) │              │ VDD       ├──── 5V supply
   │            │              │           │
   │ Qwiic port │──Qwiic───── │ I2C port  │  (3.3V logic)
   │ (3.3V I2C) │  cable      │           │
   │            │              │ AIN0      ├──── Transducer 1 Signal
   │   Display  │              │ AIN1      ├──── Transducer 2 Signal
   │   (built   │              │ AIN2      ├──── Transducer 3 Signal
   │    in)     │              │ AIN3      ├──── Transducer 4 Signal
   └────────────┘              │ GND       ├──── Common GND
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

## Calibration

Each sensor exposes a **diagnostic voltage entity** (e.g. "Pressure 1 Voltage")
that reports the transducer signal voltage.

### Tier 1 -- Zero-Point Offset Correction

If the PSI reading at 0 pressure is slightly off:

1. Open the transducer to atmosphere (0 PSI gauge).
2. Note the PSI reading in Home Assistant (e.g. 2.3 PSI).
3. Add an offset filter via `!extend`:

   ```yaml
   sensor:
     - id: !extend pressure_1
       filters:
         - offset: -2.3
   ```

4. Reinstall.

### Tier 2 -- Multi-Point Calibration

For the highest accuracy, calibrate against a trusted reference gauge.
ESPHome's `calibrate_linear` filter with `method: exact` provides
piecewise-linear interpolation.

Because `!extend` appends filters after the existing chain, this maps
**uncalibrated PSI readings to reference PSI readings**.

1. At several setpoints, record HA PSI and reference gauge PSI.
2. Add a `calibrate_linear` filter:

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

3. Reinstall.

**Calibration Data Recording Sheet:**

```
Sensor: ___________________________  Date: ____________

| Setpoint | HA Reading (PSI) | Reference Gauge (PSI) | Notes |
|----------|------------------|-----------------------|-------|
| 1        |                  |                       |       |
| 2        |                  |                       |       |
| 3        |                  |                       |       |
| 4        |                  |                       |       |
| 5        |                  |                       |       |
| 6        |                  |                       |       |
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
