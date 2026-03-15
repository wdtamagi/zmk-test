# ZMK Test - Analog Stick Keyboard Configuration

## What This Repository Is

This is a ZMK user config repository for the **tmr** shield, a test/development keyboard running on a **nice!nano** (nRF52840). It builds firmware via GitHub Actions using `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`.

The shield uses **PS5 joystick hall-effect sensors** read via the `zmk-driver-analog-stick` module, which provides multi-mode analog stick input (switch keys, mouse pointer, scroll) switchable per keymap layer.

## Repository Structure

```
config/west.yml          # West manifest — declares ZMK and module dependencies
build.yaml               # GitHub Actions build matrix (nice_nano + tmr shield)
zephyr/module.yml         # Registers this repo as a Zephyr module (board_root)
boards/shields/tmr/
  tmr.dtsi               # Base shield devicetree (ADC, analog sticks, listeners)
  tmr.overlay            # Board-specific overrides (includes tmr.dtsi)
  tmr.keymap             # Keymap (minimal — input comes from analog stick listeners)
  tmr.conf               # Kconfig (analog stick, stack sizes, debug logging)
  Kconfig.shield         # Shield detection
  Kconfig.defconfig       # Default Kconfig values (keyboard name)
  tmr.zmk.yml            # Hardware metadata
```

## External Dependencies

### ZMK Firmware
- **Remote:** `https://github.com/zmkfirmware`
- **Revision:** `v0.3`
- Provides the core firmware, keymap system, BLE, USB HID, etc.

### zmk-driver-analog-stick (wdtamagi)
- **URL:** `https://github.com/wdtamagi/zmk-driver-analog-stick`
- **Revision:** `main`
- **Local path (cached):** `modules/zmk-driver-analog-stick/`
- Provides:
  - `zmk,analog-stick` — ADC-based analog stick driver with Q16.16 fixed-point processing, IIR biquad filter, per-axis linear deadzone, ratiometric deadzone-percent
  - `zmk,analog-stick-listener` — Layer-aware input listener routing stick input to switch (key), mouse pointer, or scroll mode
  - `zmk,analog-stick-split` — Split keyboard transport (per-event BLE forwarding)
  - Scan coordinator — Batched multi-stick scanning with GPIO grouping and CI-aware polling
  - HID accumulator — Cross-stick mouse/scroll report coordination

## Hardware Setup

- **MCU board:** nice!nano v2 (nRF52840, pro_micro interconnect)
- **Stick 0:** PS5 joystick hall-effect sensor (dual-axis, bidirectional)
  - X axis on AIN0, Y axis on AIN5
- **Stick 1:** Single-axis trigger sensor on AIN7 (P0.31)
- **Power:** Sensors powered from VCC (3.3V) — direct connection, no voltage divider needed (3.3V within nRF52840 SAADC 3.6V limit)
- **ADC config:** Gain 1/6, internal reference (0.6V × 6 = 3.6V effective range), 12-bit resolution, 10µs acquisition time

## Analog Stick Configuration

### Stick 0 (dual-axis PS5 joystick)
- **Channels:** AIN0 (X), AIN5 (Y)
- **Calibration:** min=0, center=1877, max=3750
- **Deadzone:** 5% of half-range (~92 counts)
- **Filter:** 2nd-order IIR biquad (coefficients carried over from previous HE config)
- **Layer 0:** Switch mode — UP/DOWN/LEFT/RIGHT arrow keys
- **Layer 1:** Mouse pointer mode — min speed 1, max speed 15

### Stick 1 (single-axis trigger)
- **Channel:** AIN7 (P0.31)
- **Calibration:** min=0, center=1877, max=3750
- **Deadzone:** 5% of half-range
- **All layers:** Switch mode — left direction = X key, right direction = Z key

## Layer Architecture

| Layer | Stick 0 | Stick 1 |
|-------|---------|---------|
| 0 (default) | Arrow keys (switch mode) | X / Z keys (switch mode) |
| 1 (mouse) | Mouse pointer movement | X / Z keys (switch mode) |

## Known Limitations

- **No layer switching from sticks:** The analog stick switch mode emits raw HID keycodes via `zmk_hid_keyboard_press()`, not ZMK behaviors. This means `&mo`, `&lt`, and other ZMK behaviors cannot be triggered by stick deflection. Layer switching requires either a physical button or a future behavior-aware mode in the driver.
- **Stick 1 placeholder keys:** Both directions of stick 1 are bound to regular keys (X/Z). The original intent was one direction for `&mo 1` (layer toggle) — blocked by the HID keycode limitation above.
- **BLE bandwidth with 5+ sticks:** The per-event split transport sends one BLE notification per axis event. Not a concern with 2 sticks.

## Calibration Reference Values

Estimated with 3.3V direct (no voltage divider), 12-bit ADC, 0-4095 range:
- **Center (neutral):** ~1877 raw ADC
- **Left/Up extreme:** ~0 raw ADC
- **Right/Down extreme:** ~3750 raw ADC

To recalibrate: enable `CONFIG_ZMK_ANALOG_STICK_LOG_LEVEL_DBG=y` in tmr.conf, read raw ADC values from logs, and update x-min/x-center/x-max (and y equivalents) in tmr.dtsi.

## Signal Processing Pipeline

```
ADC read → [IIR biquad filter] → rescale_axis (deadzone + normalize) → output [-127, +127]
                                                                              ↓
                                                              input_report(INPUT_EV_ABS)
                                                                              ↓
                                                              analog_stick_listener
                                                                              ↓
                                                          switch mode: zmk_hid_keyboard_press()
                                                          mouse mode: zmk_analog_stick_hid_move_add()
```

The scan coordinator reads both sticks in a single batched pass, then processes all sticks, then flushes one combined HID report.
