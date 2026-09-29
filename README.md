# XeWe LED - Ceiling — voice-controlled ceiling lights on an ESP8266

Personal project · 2023-09 → 2024-07 · Solo: Max Dokukin · Status: Completed · Related: [XeWe LED OS](https://github.com/xewe-labs/xewe-led-os)

![Ceiling strip lighting the room green](static/media/resources/IMG_1931.webp)

## Overview

Smarthome Lights ESP is the firmware behind a 276-LED addressable strip mounted along the ceiling of a room. An ESP8266
joins the home WiFi and exposes the lights to Apple HomeKit (and therefore Siri) and to Amazon Alexa at the same time,
so either ecosystem can switch them, dim them and change their color. Each strip has two modes: solid color and a
Perlin-noise "fade" that drifts the strip through slowly moving waves around the chosen hue. The same sketch also drives
a 61-LED desk strip from the same board (documented in [xewe-led-desk](https://github.com/xewe-labs/xewe-led-desk)); the
ceiling strip grew from a 64-LED ceiling-only prototype to 151 and then 276 LEDs.

## Highlights

- One ESP8266 serves **both** HomeKit (5 accessories: a bridge, 2 lightbulbs, 2 "Fade" switches — `my_accessory.c`) and Alexa (4 Espalexa devices — `Alexa.h`), and keeps them in sync
- Perlin-noise animation (`PerlinFade.h`, FastLED `inoise8`) with a hue window that narrows to 20% at the red end of the hue wheel (hue start < 10000 or > 55000 of 65535)
- Smooth 900 ms transitions for color, brightness and Perlin hue changes, rendered every 10 ms (`LedController.h`, `PerlinFade.h`)
- State (RGB, brightness, mode, on/off) persisted in 6 EEPROM bytes per strip, so the lights come back as they were after a reboot (`MemoryController.h`)
- Picking white (saturation < 35) automatically switches from the Perlin fade to solid color ("auto perlin off for the white color feature")

## How it works

```
HomeKit (iPhone / Siri) ─┐                                   ┌→ desk strip   (61 LEDs, D1)
                         ├→ setters → LedController ×2 ─────┤
Alexa (Espalexa)  ───────┘     ↑           │ frame() / 10 ms └→ ceiling strip (276 LEDs, D2)
                               └─ sync ←───┘  EEPROM (1500 / 1506)
```

- **`Smarthome-Lights-ESP-Desk.ino`** — pins and LED counts, WiFi connect, creates one `LedController` per strip, starts HomeKit and Alexa, and runs the main loop (HomeKit loop, Alexa loop, one frame per strip, re-sync both assistants when a value changed, `delay(10)`).
- **`LedController.h`** — per-strip state machine: modes `SOLID_COLOR`, `SOLID_COLOR_TRANSITION`, `PERLIN`; hue/saturation and RGB setters (HomeKit speaks HSV, Alexa speaks RGB); 900 ms linear brightness ramp; on/off keeps the last brightness.
- **`PerlinFade.h`** — per-LED color from 2-D Perlin noise (LED index × 10, time counter +5 per frame) mapped to hue ± half the hue window, saturation 245–255 and brightness 100–255; hue changes glide over 900 ms.
- **`Color.h`** — RGB↔HSV conversion and linear color mixing for transitions.
- **`MemoryController.h`** — reads/writes the 6-byte state block of a strip in emulated EEPROM (`EEPROM.begin(4096)`).
- **`HomeKit.h` + `my_accessory.c`** — HomeKit characteristics, setters that call the controllers, and `syncValuesHomekit()`; accessory database with a bridge ("XeWe Lights", firmware revision 3.0).
- **`Alexa.h`** — Espalexa devices "Desk Lights", "Ceiling Lights" (color) and "Desk Lights Fade", "Ceiling Lights Fade" (on/off), plus `syncValuesAlexa()`.
- **`Archived Versions/`** — earlier working copies: an Alexa-only sketch, HomeKit V1 (single strip, with an Alexa bridge), V2 (two strips, classes, Perlin fade, HomeKit only) and V3 (EEPROM memory) with ESP crash logs from debugging the HomeKit server.

## Results

| Metric | Value | Note |
|---|---|---|
| Ceiling strip | 276 LEDs on pin D2 | 64 → 151 → 276 over the project's versions |
| Desk strip (same board) | 61 LEDs on pin D1 | |
| Smart-home endpoints | 5 HomeKit accessories + 4 Alexa devices | one ESP8266, both ecosystems at once |
| Transition time | 900 ms | color, brightness and Perlin hue |
| Frame period | 10 ms | per strip |
| Persisted state | 6 bytes per strip | RGB, brightness, mode, on/off |

The numbers come from the constants in the sketch; no power or latency measurements were recorded. The crash logs in
`Archived Versions/DeskLights_Homekit_V3_Working/esp crash logs/` show about 31–33 KB of free heap while the HomeKit
server handles pair-verify sessions.

## Getting started

Arduino IDE with the ESP8266 board package, and these libraries:

- Arduino-HomeKit-ESP8266 (provides `arduino_homekit_server.h`)
- Espalexa
- Adafruit NeoPixel
- FastLED (only `inoise8` is used)

```text
1. Put the sketch folder in a directory named Smarthome-Lights-ESP-Desk (the Arduino IDE requires folder = .ino name).
2. Provide your own wifi_info.h that defines wifi_connect() for your network.
3. Adjust DESK_LED_PIN / DESK_LED_NUM / CEILING_LED_PIN / CEILING_LED_NUM in the .ino for your strips.
4. Select an ESP8266 board (e.g. NodeMCU), compile and upload; open Serial Monitor at 115200 baud.
5. In the Home app, add the "XeWe Lights" bridge with the setup code from my_accessory.c;
   ask Alexa to discover devices.
```

Note: the committed `LedController.h` is missing a semicolon after `setMode(SOLID_COLOR)` in the three `//NEW CODE`
blocks; add them before compiling. Change the HomeKit setup code in `my_accessory.c` from the library default.

## Documents

- Photos and videos of the installation: [`static/media/resources/`](static/media/resources/)
- Earlier versions and crash logs: [`Archived Versions/`](Archived%20Versions/)
- Related: [XeWe LED OS](https://github.com/xewe-labs/xewe-led-os) — modular LED firmware for ESP32 boards ([project page](https://maxdokukin.com/projects/xewe-led-os)); [xewe-led-desk](https://github.com/xewe-labs/xewe-led-desk) — the desk strip on the same firmware
