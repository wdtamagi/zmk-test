# ZMK Test - Hall Effect Keyboard Configuration

## What This Repository Is

This is a ZMK user config repository for the **tmr** shield, a test/development keyboard running on a **nice!nano** (nRF52840). It builds firmware via GitHub Actions using `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`.

The shield currently uses a **PS5 joystick hall-effect sensor** (not traditional Gateron HE switches) to produce **analog gamepad axis output** via the gamepad forwarder input processor.

## Repository Structure

```
config/west.yml          # West manifest — declares ZMK and module dependencies
build.yaml               # GitHub Actions build matrix (nice_nano + tmr shield)
zephyr/module.yml         # Registers this repo as a Zephyr module (board_root)
boards/shields/tmr/
  tmr.dtsi               # Base shield devicetree (ADC, kscan, transforms, HE driver, signal processing)
  tmr.overlay            # Board-specific overrides (pin assignments, calibration values, gp_fw tuning)
  tmr.keymap             # Keymap and HE keymap input processor bindings
  tmr.conf               # Kconfig (stack sizes, debug logging, gamepad HID settings)
  Kconfig.shield         # Shield detection
  Kconfig.defconfig       # Default Kconfig values (keyboard name)
  tmr.zmk.yml            # Hardware metadata
```

## External Dependencies

### ZMK Firmware
- **Remote:** `https://github.com/zmkfirmware/zmk`
- **Revision:** `v0.3`
- Provides the core firmware, kscan drivers, keymap system, matrix transforms, BLE, USB HID, etc.
- The build workflow is `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`

### zmk-feature-hall-effect (cr3eperall)
- **URL:** `https://github.com/cr3eperall/zmk-feature-hall-effect`
- **Revision:** `main`
- **Local path (cached):** `.zmk/modules/zmk-feature-hall-effect/`
- Provides hall-effect specific drivers and input processors:
  - `he,kscan-direct-pulsed` — ADC-based kscan driver with groups (top/bot), calibration, switch-height thresholds
  - `he,input-processor-keymap` — maps HE sensor positions to input processor chains via a transform
  - `he,input-processor-socd` — simultaneous opposing cardinal directions resolution (requires 2 physical sensors)
  - `he,behavior-pulse-set` — pulse set behavior for HE sensors
  - `he,pulse-set-forwarder` — forwards pulse set data
  - `he,input-listener` — central listener that routes HE sensor data through input processor chains
  - Signal processing: `raw_sp` (raw signal processor with IIR filter), `he_trans` (HE transform), `he_pass`/`rt_pass` (passthrough processors)
  - Gamepad forwarder: `gp_fw` — maps HE sensor height values to HID gamepad axes using `rescale(RIGHT) - rescale(LEFT)` differential formula
- **Key headers:**
  - `<dt-bindings/he/mouse_forwarder.h>` — mouse axis constants
  - `<dt-bindings/he/gamepad_forwarder.h>` — gamepad axis constants (GP_L_LEFT, GP_L_RIGHT, etc.)
  - `<dt-bindings/he/pulse_set.h>` — pulse set constants
  - `<input/he_processors.dtsi>` — shared input processor node definitions (raw_sp, gp_fw, he_trans, he_pass, rt_pass)

## Hardware Setup

- **MCU board:** nice!nano v2 (nRF52840, pro_micro interconnect)
- **Sensor:** PS5 joystick hall-effect sensor (single axis, bidirectional)
- **Power:** Sensor powered from VCC (3.3V) — direct connection, no voltage divider needed (3.3V is within nRF52840 SAADC 3.6V limit)
- **ADC config:** AIN0, ADC_GAIN_1_6, ADC_REF_INTERNAL (0.6V reference, max measurable = 3.6V)
- **GPIO keys:** 2 columns on P0.17 and P0.20, 1 row on P0.6
- **HE enable pins:** top group on P1.0, bot group on P0.11 (required even without pulse-read)

## How the Joystick Mapping Works

One physical joystick axis is split into two virtual HE sensor groups reading the **same ADC channel** (AIN0):

1. **Top group** — covers the "left" direction: ADC values from 0 (extreme left) to 1130 (center)
2. **Bot group** — covers the "right" direction: ADC values from 1130 (center) to ~2400 (extreme right)

Each group has its own calibration range in `tmr.overlay`. The `he_transform` maps them to positions 0 and 1. The `he_keymap` input processor then routes them through:
- Position 0 (top/left) -> `&gp_fw GP_L_LEFT`
- Position 1 (bot/right) -> `&gp_fw GP_L_RIGHT`

The gamepad forwarder computes the axis output as `rescale(RIGHT) - rescale(LEFT)`, producing a single bidirectional HID gamepad left-stick X-axis.

## Calibration Reference Values

Estimated with 3.3V direct (no voltage divider), 12-bit ADC, 0-4095 range:
- **Center (neutral):** ~1877 raw ADC
- **Left extreme:** ~0 raw ADC
- **Right extreme:** ~3750 raw ADC

## Key Pitfalls

- The `raw_sp` filter-coefficients are critical and painful to debug — the comment in tmr.dtsi is a genuine warning
- `enable-gpios` must always be defined on HE groups even without `pulse-read` (build fails with `GPIO_DT_SPEC_GET_OR` at global scope otherwise)
- The nice!nano P0.13 controls the 3.3V VCC regulator — ensure it stays high for reliable sensor power
- ADC working-area values in `raw_sp` and `gp_fw` must be consistent and wide enough to cover actual sensor range, or height values wrap causing axis to snap to zero at extremes
- The kscan composite row/column offsets must align with the default_transform matrix mapping
