# ESPHome configs

[ESPHome](https://esphome.io/) configurations for a few home devices: a round
knob remote for Music Assistant, a converted Logitech speaker dock, and a pair
of LED matrix clocks.

Each file in the repository root is one device. Shared or reusable parts live in
[`packages/`](packages/) and are pulled in with `!include`.

| Config | Board | What it is |
| --- | --- | --- |
| [`crowpanel-dhr55146d-sendspin-remote.yaml`](crowpanel-dhr55146d-sendspin-remote.yaml) | ESP32-S3 (Elecrow CrowPanel 1.46" round) | Knob + touch remote for Music Assistant, with cover art, station lists and light control |
| [`logitech-purefi-screen.yaml`](logitech-purefi-screen.yaml) | ESP32 | Logitech Pure-Fi dock turned into a network speaker with a TFT screen and IR remote |
| [`led-matrix-clock-01.yaml`](led-matrix-clock-01.yaml), [`led-matrix-clock-02.yaml`](led-matrix-clock-02.yaml) | ESP8266 | MAX7219 LED matrix clock with date, weather and a night mode |

---

## Getting started

ESPHome **2026.9.0 or newer** is required by the CrowPanel config (it relies on
the platform-ised `image:` component and on package-relative `.h` includes). The
others are less fussy.

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Copy the example secrets file and fill it in — `secrets.yaml` is git-ignored and
is never committed:

```sh
cp secrets.yaml.example secrets.yaml
```

Then build and flash:

```sh
# over USB
.venv/bin/esphome run --device /dev/ttyUSB0 crowpanel-dhr55146d-sendspin-remote.yaml

# over the air, once it is on the network
.venv/bin/esphome run --device OTA crowpanel-dhr55146d-sendspin-remote.yaml

# validate only
.venv/bin/esphome config crowpanel-dhr55146d-sendspin-remote.yaml
```

`esphome upload` does **not** recompile — use `esphome run` after editing a
config, or you will flash the previous build.

There are equivalent VS Code tasks in [`.vscode/tasks.json`](.vscode/tasks.json).

---

## Knob remote — `crowpanel-dhr55146d-sendspin-remote.yaml`

<img width="2000" height="1500" alt="image" src="https://github.com/user-attachments/assets/30c2e5bb-73b9-4c29-a05e-aa812af4eda3" />


A [Sendspin](https://github.com/music-assistant/sendspin) client and remote
control for [Music Assistant](https://www.music-assistant.io/), built on the
Elecrow CrowPanel 1.46" round HMI display (SKU **DHR55146D**) — an ESP32-S3R8
with 16 MB flash, 8 MB octal PSRAM, a 360×360 round LCD, a rotary encoder with a
push switch, and a capacitive touch panel.

It is a port of the square 240×240 Sendspin remote from
[anabolyc/esp8266-radio-box](https://github.com/anabolyc/esp8266-radio-box) to a
round screen and a knob.

### What it does

- Plays whatever the Music Assistant group is playing, with **full-screen 360 px
  cover art** and a cross-fade between tracks
- Shows a clock when nothing is playing
- Browsable **lists** — SomaFM channels, BBC shows, ambient scene automations —
  that start playback through Home Assistant
- A **lights screen** with a 16-colour grid, on/off and brightness
- Returns to the player after 60 s of inactivity

### Hardware

Pins are from Elecrow's own schematic, not from their example sketch — the
sketch's comment on the touch controller has the reset and interrupt lines the
wrong way round.

| Function | Pin | Notes |
| --- | --- | --- |
| LCD SCLK / MOSI | GPIO10 / GPIO11 | 4-wire SPI at 80 MHz |
| LCD CS / DC / RST | GPIO9 / GPIO3 / GPIO14 | GPIO3 is a strapping pin, hardwired to the panel |
| LCD backlight | GPIO46 | LEDC, its own pin — also a strapping pin |
| Touch SDA / SCL | GPIO6 / GPIO7 | CST816D at `0x15` |
| Touch INT / RST | GPIO5 / GPIO13 | |
| Encoder A / B | GPIO45 / GPIO42 | GPIO45 is a strapping pin |
| Encoder push switch | GPIO41 | active low |
| Power LED | GPIO40 | active low |

The display is a **JD9855** driven with the **ST77961** register set. The
`init_sequence` in [`display-round.yaml`](packages/crowpanel-dhr55146d/display-round.yaml)
is converted from LovyanGFX's `Panel_ST77961`, because ESPHome has no model for
this panel.

Present on the board but **not driven by this config**:

- 8× WS2812 ring on GPIO48, through a 3.3 V→5 V level shifter, with its 5 V rail
  gated by a P-MOS that GPIO17 switches
- A second, level-shifted I²C bus on GPIO38/GPIO39, broken out to a 4-pin JST
  connector for your own peripherals — note that this is *not* the touch bus

### Controls

The knob and the touch panel both drive the same actions.

| Input | On the player | On a list | On the lights screen |
| --- | --- | --- | --- |
| Turn knob | volume | move the selection | previous / next colour |
| Press knob | play / pause | play or trigger the selected item | lights on / off |
| Hold knob | next screen | next screen | next screen |
| Swipe left / right | previous / next screen | previous / next screen | previous / next screen |
| Swipe up / down | volume | volume | brightness |
| Tap | — | — | — |

Screens are a cycle: player → each list → each other screen → player. The knob
hold only goes forwards; a horizontal swipe goes either way and wraps at both
ends, so swiping back from the player opens the last screen directly.

The player counts as **one** stop, not two — it shows the cover art while the
group is playing and the clock otherwise.

| Browse list | Lights | Clock |
| --- | --- | --- |
| <img width="2000" height="1500" alt="image" src="https://github.com/user-attachments/assets/72885e96-7a7f-4aa4-a484-c704ffebe5ce" /> | <img width="2000" height="1500" alt="image" src="https://github.com/user-attachments/assets/f2af7805-8696-4de1-bfb2-b31099f319ea" /> | <img width="2000" height="1500" alt="image" src="https://github.com/user-attachments/assets/cf599770-f08d-47d6-978a-ca06b3391981" /> |

### Not implemented

- **Deep sleep.** On the ESP32-S3 only GPIO0–21 are RTC pins, and every knob pin
  (41, 42, 45) is outside that range, so none of them can wake the chip. Only
  the backlight is switched off, and that is off by default since the board is
  USB powered. The touch INT line on GPIO5 *is* an RTC pin, so this is possible
  if you wire wake-up to touch instead.
- **Previous / next track and seek.** On the button-based remote these were a
  double click and a long press, which a single knob cannot offer. A spare
  gesture is the obvious home for them.

### Packages

Later packages `!extend` earlier ones, so the include order in the device config
matters.

| Package | Role |
| --- | --- |
| [`input-knob.yaml`](packages/crowpanel-dhr55146d/input-knob.yaml) | Maps the encoder onto the four button sensors the other packages expect |
| [`display-round.yaml`](packages/crowpanel-dhr55146d/display-round.yaml) | SPI bus, the JD9855 panel and its init sequence, backlight, power LED |
| [`sendspin-addon-display.yaml`](packages/crowpanel-dhr55146d/sendspin-addon-display.yaml) | LVGL setup, fonts, cover art, the playback and clock pages |
| [`sendspin-addon-knob.yaml`](packages/crowpanel-dhr55146d/sendspin-addon-knob.yaml) | Volume and play/pause on the knob |
| [`browse.yaml`](packages/crowpanel-dhr55146d/browse.yaml) | The screen cycle, the virtual list, and the inactivity timeout |
| [`somafm-stations.yaml`](packages/crowpanel-dhr55146d/somafm-stations.yaml) | 46 SomaFM channels, logos fetched at compile time |
| [`bbc-stations.yaml`](packages/crowpanel-dhr55146d/bbc-stations.yaml) | BBC shows, as Music Assistant library URIs |
| [`ambient-scenes.yaml`](packages/crowpanel-dhr55146d/ambient-scenes.yaml) | Scene automations, with icon glyphs instead of images |
| [`lights.yaml`](packages/crowpanel-dhr55146d/lights.yaml) | The lights screen |
| [`touch.yaml`](packages/crowpanel-dhr55146d/touch.yaml) | Touch controller, and swipe/tap recognition |
| [`touch-actions.yaml`](packages/crowpanel-dhr55146d/touch-actions.yaml) | What each gesture does |
| [`display-sleep.yaml`](packages/crowpanel-dhr55146d/display-sleep.yaml) | Backlight timeout, off by default |

The browse list is **virtual**: five fixed LVGL rows are refilled as the
selection moves, so a long list costs no extra display memory.

Gesture recognition is done on the raw touchscreen triggers rather than LVGL's
`on_swipe_*`, so it works on every page without touching any page definition.
Thresholds are all substitutions at the top of `touch.yaml`.

---

## Logitech Pure-Fi dock — `logitech-purefi-screen.yaml`

<img width="2000" height="1500" alt="image" src="https://github.com/user-attachments/assets/b45e92d6-98bd-4caa-8ae7-8c6a776e5935" />

A Logitech Pure-Fi speaker dock with its electronics replaced by an ESP32,
turning it into a network speaker with a local display. It pulls the audio and
light packages from
[sonocotta/esparagus-media-center](https://github.com/sonocotta/esparagus-media-center)
over the network and adds three local ones:

| Package | Role |
| --- | --- |
| [`sendspin-screen-only.yaml`](packages/sendspin-screen-only.yaml) | Sendspin media player |
| [`logitech-purefi-screen--display.yaml`](packages/logitech-purefi-screen--display.yaml) | The TFT UI |
| [`logitech-purefi-screen--ir.yaml`](packages/logitech-purefi-screen--ir.yaml) | IR receiver for the original remote |

It has I²S audio out, an SPI TFT, a WS2812 indicator, and a weather forecast
pulled from Home Assistant.

The board's touch controller and SD card slot share the display's SPI bus, and
this config only parks their chip-select lines with a GPIO switch each rather
than using them — so neither is wired up as a component. `batt_sense_pin`
(GPIO34) is likewise declared but has no sensor on it yet.

---

## LED matrix clocks — `led-matrix-clock-01.yaml`, `led-matrix-clock-02.yaml`

<img width="1092" height="749" alt="image" src="https://github.com/user-attachments/assets/50a950b4-62fd-4512-8d99-dbbb208b8fe4" />

An ESP8266 driving four chained MAX7219 8×8 modules — a 32×8 display. The two
files are the same device twice, differing only in name, night-mode hours and
whether kids mode defaults on.

Within each minute the display cycles:

- **0–25 s and 30–55 s** — the time, with a blinking colon
- **25–30 s** — the date
- **55–60 s** — the outside temperature and a weather glyph

Before that it shows a Wi-Fi icon while disconnected and a clock icon until SNTP
has synced. **Kids mode** replaces everything with a sleeping-face glyph between
`night_start_hour` and `night_end_hour`, as a "stay in bed" signal; it is
exposed as a switch in Home Assistant.

Fonts live in [`fonts/`](fonts/): `pixelmix.ttf` for the digits and the Material
Design Icons webfont for the glyphs.

---

## Making these your own

Every config assumes Home Assistant entities from the author's own setup. At
minimum, change these:

| Config | Substitution | Currently |
| --- | --- | --- |
| Knob remote | `sendspin_resume_entity` | `media_player.living_room` |
| Knob remote | `lights_entities`, `lights_state_entity` | three named lights |
| Knob remote | the `automation.ambient_*` ids in `ambient-scenes.yaml` | scene automations |
| Dock, clocks | `weather.forecast_home` and `sensor.weather_forecast_*` | a weather integration |

`name:` in each config is the ESPHome node name and must be unique on your
network. The clock and dock configs still carry their generated
`esphome-web-xxxxxx` names.

The knob remote needs **"Allow the device to perform Home Assistant actions"**
enabled for it in Home Assistant. Without it, selecting a station appears to do
nothing at all: `homeassistant.action` is never dispatched, so neither the
success nor the error branch runs, and the loading spinner just times out.

---

## Credits

- Panel init sequence adapted from
  [LovyanGFX](https://github.com/lovyan03/LovyanGFX) (`Panel_ST77961`)
- Board pinouts from
  [Elecrow's CrowPanel 1.46" repository](https://github.com/Elecrow-RD)
- The square Sendspin remote this one was ported from:
  [anabolyc/esp8266-radio-box](https://github.com/anabolyc/esp8266-radio-box)
- Icons: [Material Design Icons](https://pictogrammers.com/library/mdi/)

---

## License

[GPL-3.0](LICENSE).

The bundled fonts in [fonts/](fonts/) and the icon and artwork URLs the configs
fetch at compile time are third-party assets and keep their own terms.
