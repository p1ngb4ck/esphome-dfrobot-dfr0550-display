# esphome-dfrobot-dfr0550-display

## Purpose
The DFRobot DFR0550 5" DSI display (800 x 480) on an ESP32-P4. Its STM32 sits on the I2C bus of the DSI connector and
answers as the touch controller (0x38): the bus recovery at setup stalls the whole bus, and touch points are read in
single 4-byte transactions. The backlight cannot be controlled. Both options now live in esphome-mipi-display.

## How it works
This repository holds the page for the display; the two changes it needs are options of components owned by
esphome-mipi-display: `i2c` `bus_recovery: false` leaves out the bus recovery at setup that stalls the STM32, and
`ft5x06` `firmware_reads: true` reads the touch count (register 0x02) and then 4 bytes per point in separate
transactions, the subset the STM32 emulates. The display itself runs on upstream's `mipi_dsi` with `model: CUSTOM`
and the timings below; `mipi_dsi` needs the DSI PHY LDO (`esp_ldo`).

## Quick start
Chips: ESP32-P4

Needs: [esphome-mipi-display](https://github.com/p1ngb4ck/esphome-mipi-display),
[esphome-storage-drivers](https://github.com/p1ngb4ck/esphome-storage-drivers) (`esp_ldo`)

```yaml
external_components:
  - source: github://p1ngb4ck/esphome-mipi-display@main
    components: [display, ft5x06, i2c, mipi, mipi_dsi, touchscreen]
  - source: github://p1ngb4ck/esphome-storage-drivers@main
    components: [esp_ldo]
esp_ldo:
  - {channel: 3, voltage: 2.5V}
i2c:
  sda: GPIO7
  scl: GPIO8
  bus_recovery: false
display:
  - platform: mipi_dsi
    model: CUSTOM
    dimensions: {width: 800, height: 480}
    hsync_pulse_width: 2
    hsync_back_porch: 44
    hsync_front_porch: 134
    vsync_pulse_width: 2
    vsync_back_porch: 21
    vsync_front_porch: 7
    pclk_frequency: 30MHz
    lane_bit_rate: 720Mbps
    lanes: 1
    init_sequence:
      - delay 10ms
touchscreen:
  - platform: ft5x06
    firmware_reads: true
    transform: {mirror_x: true, mirror_y: true}
```

Use your board's pins (here the Guition JC-ESP32P4-M3-Dev, display on J2). Init works after a cold start, a restart
and an OTA alike; touch is complete.

## Main options
This repository has no CONFIGURATION.md; the options are documented in esphome-mipi-display.

| Option | What it does | Default |
|---|---|---|
| `i2c` `bus_recovery` | `false` skips the bus recovery at setup (needed for the DFR0550) | `true` |
| `ft5x06` `firmware_reads` | count, then 4 bytes per point, no setup access (needed for the DFR0550) | `false` |
| `mipi_dsi` `model: CUSTOM` + timings | 800 x 480, 1 lane, 720 Mbps, 30 MHz pixel clock (block above) | - |

## Catalog
Components: none of its own. The options are in esphome-mipi-display's
[`i2c`](https://github.com/p1ngb4ck/esphome-mipi-display/tree/main/esphome/components/i2c) and
[`ft5x06`](https://github.com/p1ngb4ck/esphome-mipi-display/tree/main/esphome/components/ft5x06); the display is
[`mipi_dsi`](https://github.com/p1ngb4ck/esphome-mipi-display/tree/main/esphome/components/mipi_dsi).

Branches:
- [main](https://github.com/p1ngb4ck/esphome-dfrobot-dfr0550-display/tree/main) (this page)
- The changes, as branches of esphome-mipi-display:
  [pr/i2c-bus-recovery-option](https://github.com/p1ngb4ck/esphome-mipi-display/tree/pr/i2c-bus-recovery-option),
  [pr/ft5x06-firmware-reads](https://github.com/p1ngb4ck/esphome-mipi-display/tree/pr/ft5x06-firmware-reads)
- The other branches of this repository are `dependabot/*` CI dependency updates, no display content
  ([all branches](https://github.com/p1ngb4ck/esphome-dfrobot-dfr0550-display/branches/all)).

Needed topic repos:
- [esphome-mipi-display](https://github.com/p1ngb4ck/esphome-mipi-display): `display`, `ft5x06`, `i2c`, `mipi`,
  `mipi_dsi`, `touchscreen`.
- [esphome-storage-drivers](https://github.com/p1ngb4ck/esphome-storage-drivers): `esp_ldo` (with `mipi_dsi`).

All topics together: `github://p1ngb4ck/esphome@dev`.

---

# Detailed version

## DFRobot DFR0550 5" DSI display on ESP32-P4

ESPHome `dev` plus all changes this display needs. Base: ESPHome `dev` at `a5a75738ad`. Each change as its own pull
request branch, both in [esphome-mipi-display](https://github.com/p1ngb4ck/esphome-mipi-display):

- [pr/i2c-bus-recovery-option](https://github.com/p1ngb4ck/esphome-mipi-display/tree/pr/i2c-bus-recovery-option):
  `i2c` option `bus_recovery` (default `true`); `false` skips the bus recovery at setup.
- [pr/ft5x06-firmware-reads](https://github.com/p1ngb4ck/esphome-mipi-display/tree/pr/ft5x06-firmware-reads):
  `ft5x06` option `firmware_reads` for the display's emulated touch controller.

Both options are chip-neutral (every chip `i2c` / `ft5x06` support); the display needs the ESP32-P4 (MIPI DSI) and
was tested there. Needs: [esphome-mipi-display](https://github.com/p1ngb4ck/esphome-mipi-display) and `esp_ldo` of
[esphome-storage-drivers](https://github.com/p1ngb4ck/esphome-storage-drivers); no other topic repo.

The display's STM32 sits on the I2C bus of the DSI connector and answers as the touch controller (0x38). With the bus
recovery active (ESPHome's default, at power-up) the STM32 seems to lock / stall the whole bus: SCL stays low from its
first addressing on until the display loses power. `bus_recovery: false` avoids this.

The STM32 emulates only a subset of the touch controller: the touch count (register 0x02) and 4 bytes per point read
in separate transactions, no setup access. `firmware_reads: true` reads it that way.

## Status

- Init: complete, after a cold start, a restart and a flash (OTA) alike
- Touch: complete
- Backlight control: not available. According to our tests the STM32 does not act on any backlight or power
  command of the Raspberry Pi drivers (registers 0x82, 0x83, 0x85, 0x86 at 0x45) nor on registers 0x01-0x07;
  the backlight stays on.
- Checks: upstream's CI jobs run locally (not on GitHub) and an own review; no open findings

## Example

Tested with the Guition JC-ESP32P4-M3-Dev, display on J2. The whole device config; the display part alone is the
Quick start above.

```yaml
esphome:
  name: esp32-p4-dfr0550-min
  friendly_name: ESP32-P4-DFR0550-Min

esp32:
  variant: ESP32P4
  engineering_sample: true
  flash_size: 16MB
  framework:
    type: esp-idf

external_components:
  - source: github://p1ngb4ck/esphome-mipi-display@main
    components: [display, ft5x06, i2c, mipi, mipi_dsi, touchscreen]
    refresh: always
  - source: github://p1ngb4ck/esphome-storage-drivers@main
    components: [esp_ldo]

psram:
  speed: 200MHz

esp_ldo:
  - channel: 3
    voltage: passthrough

esp32_hosted:
  active_high: true
  variant: ESP32C6
  reset_pin: GPIO54
  cmd_pin: GPIO19
  clk_pin: GPIO18
  d0_pin: GPIO14
  d1_pin: GPIO15
  d2_pin: GPIO16
  d3_pin: GPIO17

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret esphome_encryption_key

ota:
  - platform: esphome

logger:
  hardware_uart: UART0

i2c:
  sda: GPIO7
  scl: GPIO8
  bus_recovery: false

display:
  - platform: mipi_dsi
    model: CUSTOM
    dimensions:
      width: 800
      height: 480
    hsync_pulse_width: 2
    hsync_back_porch: 44
    hsync_front_porch: 134
    vsync_pulse_width: 2
    vsync_back_porch: 21
    vsync_front_porch: 7
    pclk_frequency: 30MHz
    lane_bit_rate: 720Mbps
    lanes: 1
    init_sequence:
      - delay 10ms

touchscreen:
  - platform: ft5x06
    address: 0x38
    firmware_reads: true
    transform:
      mirror_x: true
      mirror_y: true
    on_touch:
      - logger.log:
          format: "touch x=%d y=%d"
          args: [touch.x, touch.y]

lvgl:
```

## License

ESPHome's files: [LICENSE](LICENSE). The additions of this branch (the changes to `i2c` and `ft5x06`, this
README and its example): [LICENSE-MIT](LICENSE-MIT).

## Branch structure

This repository has no `pr/` branches; `main` is this page. The two changes are `pr/` branches of
[esphome-mipi-display](https://github.com/p1ngb4ck/esphome-mipi-display); their bases are in their `BRANCH.md`
there:

| Branch | Repository |
|---|---|
| `pr/i2c-bus-recovery-option` | p1ngb4ck/esphome-mipi-display |
| `pr/ft5x06-firmware-reads` | p1ngb4ck/esphome-mipi-display |
