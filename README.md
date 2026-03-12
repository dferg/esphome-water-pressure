# Water Pressure Monitor

An ESPHome project that uses an ESP32 to read water pressure from four
FUSCH 200 PSI pressure transducers and report the values to Home Assistant.

## Bill of Materials

| Qty | Part | Notes |
|-----|------|-------|
| 1 | ESP32 dev board (e.g. ESP32-DevKitC, NodeMCU-32S) | Any board with accessible ADC1 pins |
| 4 | FUSCH 200 PSI Pressure Transducer (1/8" NPT) | 0.5–4.5 V output, 5 V supply |
| 4 | 4.7 k&Omega; resistor (1/4 W, 1% preferred) | Voltage divider R1 |
| 4 | 10 k&Omega; resistor (1/4 W, 1% preferred) | Voltage divider R2 |
| 1 | 5 V power supply (1 A+) | Powers ESP32 and transducers |
| — | Hookup wire, solder, connectors | — |

## Wiring

Each transducer has three wires:

| Wire | Color | Connection |
|------|-------|------------|
| VCC | Red | 5 V supply |
| GND | Black | Common ground |
| Signal | Green / Yellow | Voltage divider input |

### Voltage Divider Circuit (per sensor)

The transducer outputs 0.5–4.5 V.  The ESP32 ADC is only safe up to ~3.1 V,
so a resistor divider scales the signal down.

```
Transducer Signal ──┬── R1 (4.7 kΩ) ──┬── ESP32 ADC pin
                    │                 │
                    │                 R2 (10 kΩ)
                    │                 │
                   GND               GND
```

With R1 = 4.7 k&Omega; and R2 = 10 k&Omega; the divider ratio is
R2 / (R1 + R2) &asymp; 0.68, so:

| Pressure | Transducer Voltage | ADC Pin Voltage |
|----------|--------------------|-----------------|
| 0 PSI | 0.50 V | 0.34 V |
| 50 PSI | 1.50 V | 1.02 V |
| 100 PSI | 2.50 V | 1.70 V |
| 150 PSI | 3.50 V | 2.38 V |
| 200 PSI | 4.50 V | 3.06 V |

### Default Pin Assignments

| Sensor | GPIO | ADC Channel |
|--------|------|-------------|
| Pressure 1 | GPIO32 | ADC1_CH4 |
| Pressure 2 | GPIO33 | ADC1_CH5 |
| Pressure 3 | GPIO34 | ADC1_CH6 |
| Pressure 4 | GPIO35 | ADC1_CH7 |

All four pins are on ADC1 so they work while WiFi is active (ADC2 cannot be
used with WiFi on the ESP32).

## Software Setup

### Prerequisites

- Home Assistant with the ESPHome add-on **or** the ESPHome CLI
- A USB cable to flash the ESP32 for the first time

### Quick Start (ESPHome Dashboard)

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

## Configuration Reference

All behaviour is controlled through `substitutions`.  Override any of these
in your consumer YAML to customise the device without editing the package.

| Substitution | Default | Description |
|--------------|---------|-------------|
| `name` | `water-pressure` | Device name (used in mDNS, entity IDs) |
| `friendly_name` | `Water Pressure Monitor` | Human-readable name in Home Assistant |
| `board` | `esp32dev` | ESP32 board variant for platformio |
| `update_interval` | `5s` | How often each sensor is sampled |
| `max_psi` | `200` | Full-scale PSI rating of the transducer |
| `voltage_divider_factor` | `1.47` | (R1+R2)/R2 — adjust after calibration |
| `pressure_1_name` | `Pressure 1` | Friendly name for sensor 1 |
| `pressure_2_name` | `Pressure 2` | Friendly name for sensor 2 |
| `pressure_3_name` | `Pressure 3` | Friendly name for sensor 3 |
| `pressure_4_name` | `Pressure 4` | Friendly name for sensor 4 |
| `pressure_1_pin` | `GPIO32` | ADC pin for sensor 1 |
| `pressure_2_pin` | `GPIO33` | ADC pin for sensor 2 |
| `pressure_3_pin` | `GPIO34` | ADC pin for sensor 3 |
| `pressure_4_pin` | `GPIO35` | ADC pin for sensor 4 |

## Calibration

Each sensor exposes a **diagnostic voltage entity** (e.g. "Pressure 1 Voltage")
that reports the recovered transducer signal voltage.  This entity is the
foundation for all calibration methods below.

### Tier 1 — Voltage Divider Verification (recommended minimum)

Resistors have manufacturing tolerances (typically &plusmn;5 % for standard
parts).  Verify the divider factor once after assembly:

1. Power the ESP32 with one transducer connected and **no pressure applied**
   (sensor open to atmosphere).
2. In Home Assistant, note the **Voltage** diagnostic entity value for that
   sensor (e.g. 0.48 V).  This entity already applies the current
   `voltage_divider_factor` to recover the transducer signal voltage.
3. Using a multimeter, measure the actual voltage on the transducer **signal
   wire** (before the divider).  It should read roughly 0.5 V at 0 PSI.
4. If the two values do not match, compute a corrected factor:

   ```
   new_factor = current_factor * (multimeter_voltage / diagnostic_voltage)
   ```

   For example, if the current factor is 1.47, the diagnostic entity reads
   0.48 V, and the multimeter reads 0.50 V:

   ```
   new_factor = 1.47 * (0.50 / 0.48) = 1.531
   ```

5. Update the `voltage_divider_factor` substitution in your consumer YAML:

   ```yaml
   substitutions:
     voltage_divider_factor: "1.531"
   ```

6. Reinstall the firmware.  Repeat for each sensor if you used different
   resistors.

### Tier 2 — Zero-Point Offset Correction

After Tier 1, if the PSI reading at 0 pressure is slightly off:

1. Disconnect the transducer from the plumbing (open to atmosphere = 0 PSI
   gauge).
2. Note the PSI reading in Home Assistant (e.g. 2.3 PSI).
3. Add an offset filter to the sensor in your consumer YAML using `!extend`:

   ```yaml
   sensor:
     - id: !extend pressure_1
       filters:
         - offset: -2.3
   ```

4. Reinstall.

### Tier 3 — Multi-Point Calibration

For the highest accuracy, calibrate against a trusted reference gauge at
several pressure points.  ESPHome's `calibrate_linear` filter with
`method: exact` provides piecewise-linear interpolation between your
measurements.

Because `!extend` appends filters after the existing chain, this calibration
maps **uncalibrated PSI readings to reference PSI readings** (not raw
voltage).  This corrects the entire signal path in one step.

**Equipment needed:** A reference pressure gauge (analog or digital) installed
on the same line as the transducer.

**Procedure:**

1. At several pressure setpoints, record both the **PSI shown in Home
   Assistant** (the uncalibrated reading) and the **reference gauge PSI**.
   Use the table below to organise your measurements.

2. Add a `calibrate_linear` filter to the sensor in your consumer YAML:

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

   Replace the example values with your actual measurements.  The left side
   is the **uncalibrated PSI reading from Home Assistant**, and the right
   side is the **reference gauge PSI**.

3. Reinstall.  The `calibrate_linear` filter runs after the default
   multiply + lambda chain and corrects for divider tolerance, transducer
   non-linearity, and ADC imprecision.

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

Tips:
- Minimum two points are required; five or more give the best results.
- Include 0 PSI and at least one point near your normal operating pressure.
- Flush the line before recording to avoid trapped air affecting readings.
- Allow readings to stabilise for 10–15 seconds at each setpoint.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Reading stuck at 0 PSI | No 5 V to transducer, broken signal wire | Check continuity and power |
| Reading pegged at max | ADC saturated — voltage > 3.1 V at pin | Check voltage divider resistors |
| Noisy / jumping values | Poor grounding, long unshielded signal wires | Add shielding, twist signal + GND, shorten wires |
| All sensors read the same wrong value | `voltage_divider_factor` incorrect | Perform Tier 1 calibration |
| WiFi drops when reading sensors | Pins on ADC2 (GPIO 0, 2, 4, 12–15, 25–27) | Move to ADC1 pins (GPIO 32–39) |
| Negative PSI readings | Transducer offset or divider factor off | Perform Tier 2 calibration |

## Project Structure

```
water-pressure/
├── README.md                        # This file
├── AGENTS.md                        # AI agent instructions
├── water-pressure-esp32.yaml        # Main ESPHome package (entry point)
├── common/
│   └── pressure-sensor.yaml         # Sensor template (included once per sensor)
└── example.yaml                     # Example consumer configuration
```

## License

This project is provided as-is for personal and educational use.
