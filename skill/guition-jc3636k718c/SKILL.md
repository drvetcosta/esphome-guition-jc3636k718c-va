---
name: guition-jc3636k718c
description: >
  Reference for building/editing ESPHome configs on the Guition JC3636K718C round
  knob display (ESP32-S3, 1.8" 360x360 ST77916, CST816 touch, rotary knob, PCM5100A
  DAC, PDM mic, WS2812 LED ring). Use whenever working on this board (or the
  base/core.yaml in this repo): correct pinout, ESPHome component choices, and the
  hard-won gotchas (knob isn't quadrature, GPIO0 ring strapping, 16 MB partitions need
  USB flash, LVGL performance limits, lambda/string pitfalls, battery is heuristic).
---

# Guition JC3636K718C - ESPHome working notes

This is the **K / Knob** series board. Its pinout is **different from the JC3636W518**
- do NOT reuse W518 pins (display, backlight, touch all differ). The values below come
from the official manufacturer demo `JC3636K718_knob_EN` (`pincfg.h` + `scr_st77916.h`)
and from making this device actually work.

## Board

- ESP32-S3 (240 MHz), **8 MB octal PSRAM @ 80 MHz**, **16 MB flash**.
- 1.8" **round** 360×360, driver **ST77916** (QSPI), RGB565, colors **inverted**.
- Touch **CST816** (I²C). Rotary **knob** (rotation only - **not pressable**).
- Audio out **PCM5100A** I²S DAC + speaker. Mic = I²S **PDM**.
- **WS2812 LED ring**, 13 LEDs, GRB.
- microSD (SD_MMC). USB-C; some units have Li-ion + wireless charging.

## Pinout (authoritative)

```
Display QSPI (ST77916):  SCK=11  CS=12  D0=13 D1=14 D2=15 D3=16  RST=17  BLK=21(PWM)
Touch CST816 (I2C):      SDA=9   SCL=10  INT=7  RST=8   (internal SDA/SCL pull-ups)
Knob:                    A=GPIO2  B=GPIO1   (see gotcha - NOT quadrature)
DAC PCM5100A (I2S):      BCK=3  WS/LRCK=45  DO=42   MUTE=46 (LOW=mute, hold HIGH)  no MCK
Microphone (I2S PDM):    SD/data=4  SCK/clk=5
LED ring WS2812:         DATA=0   (13 LEDs, GRB)  -- GPIO0 = BOOT/strapping
Battery ADC:             GPIO6  (~10k/10k divider -> x2)
microSD (SD_MMC):        CLK=39 CMD=38 D0=40 D1=41 D2=48 D3=47
```

## ESPHome component choices that work

- **Display:** `display: platform: qspi_dbi`, `model: CUSTOM`, `data_rate: 80MHz`,
  `invert_colors: true`, `color_order: rgb`, with the full `init_sequence` from
  `base/core.yaml` (ends with `3A=55` for RGB565, `21` invert, `11` sleep-out,
  120 ms delay, `29` display-on). The native ST77916 model was dev-only; CUSTOM +
  manual init runs on stable ESPHome.
- **SPI:** `type: quad`, `clk_pin: 11`, `data_pins: [13,14,15,16]`. **CS/RST go on the
  `display:` block** (`cs_pin: 12`, `reset_pin: GPIO17`), not on `spi:`.
- **Touch:** `touchscreen: platform: cst816`, `interrupt_pin: 7`, `reset_pin: 8`,
  on the `i2c` bus `sda: 9 / scl: 10`.
- **Backlight:** `output: ledc` on `GPIO21` → `light: monochromatic`.
- **Audio:** `i2s_audio` (out: bclk=3, lrclk=45; in: lrclk=5), `speaker: i2s_audio`
  (`dout=42`, `dac_type: external`), `microphone: i2s_audio` (`din=4`, `pdm: true`,
  16 kHz). DAC **MUTE on GPIO46** must be held HIGH → a `switch: gpio` with
  `restore_mode: ALWAYS_ON`.
- **LED ring:** `light: esp32_rmt_led_strip`, `pin: GPIO0`, `num_leds: 13`,
  `rgb_order: GRB`, `chipset: WS2812`, `rmt_symbols: 64`.
- **Knob:** two `binary_sensor: gpio` (GPIO2 and GPIO1), `inverted: true` + `pullup`,
  act on `on_press`. See gotcha.
- **esp32:** `flash_size: 16MB`, `partitions: partitions.csv`, framework `esp-idf`.
  `psram: mode: octal, speed: 80MHz`.

## Gotchas / lessons learned (the important part)

1. **The knob is NOT a quadrature encoder.** A PCNT `rotary_encoder` reads **zero**.
   Direction is encoded by *which pin pulses*: **left → GPIO2, right → GPIO1**, one
   clean LOW pulse per detent. Read two independent `binary_sensor` `on_press` instead.
   The knob can't be pressed (GPIO0 is BOOT) - trigger actions with a screen tap.

2. **LED ring on GPIO0 (strapping pin).** Works fine as WS2812 output after boot;
   ESPHome may print a strapping-pin warning - harmless (the vendor demo does the same).

3. **16 MB partitions → first flash MUST be over USB.** `partitions.csv` uses big app
   slots (~7.9 MB). A partition-table change can't be applied via OTA; flash once over
   USB, then OTA works.

4. **LVGL performance is tight on this S3 + QSPI panel.**
   - Keep `lvgl: buffer_size: 24%` (puts the draw buffer in fast internal RAM instead
     of slow PSRAM at the default 100%).
   - Keep `display: data_rate: 80MHz`.
   - A rotating full-screen HUD image and an animated radial equalizer were both tried
     and **removed** as too laggy. Add heavy full-screen redraws only very carefully.
   - `logger: level: INFO` - DEBUG is very noisy during voice and adds overhead.

5. **Lambda string/symbol pitfalls (LVGL labels):**
   - For MDI/Unicode glyphs in C++ lambdas use the actual UTF-8 bytes or `chr(0xF....)`
     style - declare each glyph in the `font:` `glyphs:` list or it renders as tofu.
   - `\n` inside a lambda works **inline** (`'...\n...'`); inside a block scalar (`|-`)
     a literal newline breaks YAML. Keep multi-line label text on one quoted line.
   - Montserrat has no `-`; guard "no data yet" cases with an empty string, not a dash.

6. **Album art (`online_image`)** decodes fine but the LVGL widget **won't redraw by
   itself** - you must call `lvgl.image.update` in `on_download_finished`. Use
   `http_request: verify_ssl: false` for self-signed HA. Skip reloading the same URL to
   cut flicker. Cover URL comes from the player entity's `entity_picture`.

7. **Fonts, icons and game sprites are fetched at compile time** (Google Fonts +
   MaterialDesign TTF, plus the PNGs under `assets/sprites/cool-cars/` and
   `space-wars/` from the GitHub repo), so the build host needs internet. They are
   baked into the firmware, so at runtime nothing depends on any server.

8. **Battery is a heuristic.** ADC on GPIO6 (×2), 64-sample oversampling + moving
   average to fight ETA6003 ripple (no cap on BAT_ADC). Voltage→% is a calibrated
   lookup (`vt[]`/`st[]`) - start ~4.10 V, brownout ~3.20 V - **not finished**, tune per
   unit. **No STAT pin**, so charging detection is "voltage rising or ≥4.13 V".

9. **Voice timers:** use `voice_assistant:` `on_timer_started/updated/finished/
   cancelled`. There is **no `on_timer_tick`** - do smooth local countdown in an
   `interval` and let HA fire `finished`.

10. **Wake word** (`micro_wake_word`) only runs while connected to HA. Start/stop it in
    `on_client_connected`/`on_client_disconnected` and around the VA lifecycle.

11. **Debugging:** API has no encryption here → stream logs with
    `python scripts/esplog.py <seconds>` (set the device host in the script). Don't reflash
    blind - check logs / HA state first.

## Architecture of this repo's `base/core.yaml` (for edits)

- Single big file. UI is **LVGL pages** (not a top_layer overlay for modes):
  `page_home`, `page_player`, `page_mode` (menu/timer/alarm/settings sub-screens),
  `page_settings`, `page_timer`, `page_controls`, `page_game`, `page_game2`.
- A **mode state machine** in `g_mode` drives `page_mode`; gestures are classified in
  `touchscreen.on_release` (swipe/tap/hold) and routed by a snapshot `g_mode0`.
- A **base carousel** `g_base` (clock / player / timer / car game / space game) on
  horizontal swipes.
- The **LED ring** is one HA light + reaction effects (`Listen`/`Think`/`Speak`/
  `Volume`/`Alarm`/`Timer`) chosen by `ring_update` with priority
  alarm > assistant > volume > timer > HA, snapshotting/restoring HA state.
- Per-user things to change live in `substitutions:` (HA URL, entity IDs) and a few
  **hard-coded** control-tile entities - see the [Configuration](https://github.com/MichalZaniewicz/esphome-guition-jc3636k718c-va/wiki/Configuration) wiki page.

When editing, change UI strings/labels only (not IDs or pin numbers), keep the
performance settings above, and validate the YAML before flashing.
