# IQS7222C — ESPHome external component

ESPHome driver for the **Azoteq IQS7222C** capacitive touch & proximity controller (up to 10 channels / buttons, I²C).

Each touch channel is exposed as a `binary_sensor`, so you get all the usual ESPHome goodness — `filters`, `on_press`/`on_release`, automations, etc. The sensor itself is fully event-driven over the IQS `RDY` line, so there is no polling overhead.

## Why use it

- Turn an IQS7222C into clean ESPHome binary sensors (buttons, sliders, proximity).
- Tune every channel/cycle register from YAML, or just drop in the chip's sane defaults and go.
- Trigger touches in software (`emulate_touch`) for testing UIs without touching the panel.

## Installation

```yaml
external_components:
  - source: github://latonita/esphome-iqs7222c
    components: [iqs7222c]
```

## Minimal configuration

```yaml
i2c:
  - id: i2c_1
    sda: GPIO15
    scl: GPIO14

iqs7222c:
  id: iqs
  i2c_id: i2c_1
  rdy_pin:
    number: GPIO13
    mode: INPUT
  mclr_pin:
    number: GPIO22
    mode: OUTPUT

binary_sensor:
  - platform: iqs7222c
    id: button_ch_1
    channel: 1
    on_press:
      - logger.log: "Touched CH1"
```

## Component options (`iqs7222c:`)

| Option        | Type           | Default | Description                                                          |
|---------------|----------------|---------|----------------------------------------------------------------------|
| `rdy_pin`     | pin (input)    | —       | **Required.** IQS `RDY` line (active-low, signals data ready).       |
| `mclr_pin`    | pin (output)   | —       | **Required.** IQS `MCLR` master-clear / reset (active-low).          |
| `address`     | i2c address    | `0x44`  | I²C address of the device.                                           |
| `enable_ati`  | boolean        | `true`  | Run ATI (Automatic Tuning Implementation) calibration after init.    |
| `init_delay`  | time           | `0s`    | Delay before the init sequence starts (lets power/peripherals settle).|
| `no_init`     | boolean        | `false` | Skip writing the register set — use the chip's current/NVM config.   |
| `test_mode`   | boolean        | `false` | Verbose diagnostics / register dump for debugging.                   |

Standard `id` and `i2c_id` are also supported.

## Binary sensor options (`binary_sensor:` → `platform: iqs7222c`)

| Option        | Type      | Default | Description                                  |
|---------------|-----------|---------|----------------------------------------------|
| `channel`     | int 0–9   | —       | **Required.** IQS channel this sensor maps to.|
| `iqs7222c_id` | id        | auto    | Parent hub (only needed with multiple hubs). |

All standard `binary_sensor` options (`name`, `id`, `filters`, `on_press`, …) apply. A typical debounce:

```yaml
    filters:
      - delayed_on: 25ms
      - delayed_off: 5ms
```

## Advanced register tuning

If you omit them, built-in defaults are written to the chip. To match the output of the **Azoteq IQS7222C GUI / configuration tool**, you can override the raw register blocks directly. All values are hex.

- `button_0` … `button_9` — per-button thresholds: `prox_threshold`, `enter_exit`, `touch_threshold`, `touch_hysteresis`, `prox_event_timeout`, `touch_event_timeout`.
- `ch_0` … `ch_9` — per-channel setup: `setup`, `ati_settings`, `multipliers`, `ati_compensation`, `ref_ptr`, `refmask`.
- `cycle_0` … `cycle_4` — conversion cycles: `conv_freq_frac`, `conv_freq_period`, `settings`, `ctx_select`, `iref_0`, `iref_1`.
- `global_cycle` — `setup_0`, `setup_1`, `coarse_divider_preload`, `fine_divider_preload`, `compensation_preload_0`, `compensation_preload_1`.

```yaml
iqs7222c:
  id: iqs
  i2c_id: i2c_1
  rdy_pin: { number: GPIO13, mode: INPUT }
  mclr_pin: { number: GPIO22, mode: OUTPUT }
  button_1:
    prox_threshold: 0x0A
    touch_threshold: 0x32
  ch_1:
    setup: 0x0140
    ati_settings: 0x02C0
    multipliers: 0x0500
    ati_compensation: 0x055A
    ref_ptr: 0x0000
    refmask: 0x0000
```

> These map 1:1 to the IQS7222C register map (see the Azoteq datasheet). Leave them out unless you have a tuned configuration to apply.

## Software-emulated touch

The hub exposes `emulate_touch(channel)` for triggering a channel from a lambda — handy for testing UI navigation without the physical panel:

```yaml
button:
  - platform: template
    name: "Emulate CH3 touch"
    on_press:
      - lambda: 'id(iqs).emulate_touch(3);'
```

## Notes

- Requires the device's `RDY` and `MCLR` pins wired to the ESP — the driver is interrupt/handshake-driven, not polled.
- Default I²C address is `0x44`.
- Maintainer: [@latonita](https://github.com/latonita).
