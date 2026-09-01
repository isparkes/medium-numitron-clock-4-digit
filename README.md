# DA2000 4-Digit Numitron Clock Firmware

Firmware for an ESP8266-based clock that drives a 4-digit **Numitron** (incandescent 7-segment) display. It handles time keeping (DS1307 RTC + NTP), WiFi configuration, a web-based settings UI, ambient-light dimming, WS2812B backlighting, PIR-based auto-blanking, and OTA updates — all through a single push-button and a browser.

See [`SPECIFICATION.md`](SPECIFICATION.md) for a full technical reference (pin map, HTTP API, config fields, button/menu behaviour, module breakdown).

## Features

- WiFi setup via captive portal (WiFiManager) — no hardcoded credentials
- Real-time clock via DS1307+, kept in sync with NTP (configurable pool, timezone, update interval)
- Web UI for status and configuration, reachable at `http://<hostname>.local`
- Single-button menu system for on-device configuration (time/date set, display, dimming, backlight, motion sensor, etc.)
- Automatic brightness control via LDR (light-dependent resistor)
- PIR motion sensor support to blank the display when no one is around
- WS2812B RGB backlight/underlight with several colour strategies (fixed, cycling, time-of-day colour, day-of-week colour)
- "Slots" display effects (date flashed periodically with a wipe or bang transition)
- OTA firmware updates via the web UI
- Configuration persisted as JSON on SPIFFS flash storage
- Remote value display API for pushing arbitrary numbers to the display over HTTP

## Hardware

- Board: Wemos D1 R2 / Mini (or LOLIN equivalent)
- DS1307+ RTC over I2C
- 4-digit Numitron display driven via shift registers
- LDR for ambient light sensing
- PIR motion sensor (optional)
- WS2812B addressable LEDs for backlight/underlight and separator indicators
- Single push button for on-device menu navigation

Full pin assignments are listed in `SPECIFICATION.md`.

## Building

This is an Arduino sketch (`ESP8266Clock/ESP8266Clock.ino`). Build with the Arduino IDE:

1. Install the ESP8266 board package (board manager URL: `http://arduino.esp8266.com/stable/package_esp8266com_index.json`).
2. Install the required libraries (see [Dependencies](#dependencies) below).
3. Select board **Wemos D1 R2 & Mini / LOLIN (Wemos) D1 R2 & mini**, with:
   - Flash: 80MHz
   - CPU: 160MHz
   - Upload speed: 921600
   - Flash size: 4M (1M SPIFFS)
   - lwIP Variant: v2 Higher Bandwidth (no features)
   - Erase Flash: Only sketch
4. Open `ESP8266Clock/ESP8266Clock.ino` and upload.

### Dependencies

Install via **Sketch → Include Library → Manage Libraries** (or as a `.zip` where noted):

| Library | Version | Notes |
|---|---|---|
| TimeLib | 1.5.0 | Michael Margolis |
| WiFiManager | 0.16.0 | tzapu |
| ArduinoJson | 5.13.2 | bblanchon — note: v5 API, not v6/v7 |
| ESPAsyncUDP | master | me-no-dev |
| NeoPixelBus | 2.3.4 | Makuna |
| Adafruit GFX | 1.6.1 | only needed for the (currently unused) OLED module |
| Adafruit SSD1306 | 2.0.1 | only needed for the (currently unused) OLED module |

Plus the standard ESP8266 Arduino core libraries: `FS`, `Wire`, `ESP8266WiFi`, `ESP8266WebServer`, `ESP8266HTTPUpdateServer`, `ESP8266mDNS`, `ESP8266HTTPClient`, `DNSServer`.

## First boot

1. On first power-up (no saved config), the clock performs a factory reset and starts a WiFi access point named after its hostname, with password `SetMeUp!`. Connect to it and follow the captive portal to join your WiFi network.
2. Once connected, find the clock at `http://<hostname>.local` (or check its IP via the AP status page).
3. Default web UI login is `admin` / `setup` (change this under **Security** in the config page).
4. Holding the button down through power-up performs a factory reset.

## Web UI

| Path | Purpose |
|---|---|
| `/` | Status dashboard (connectivity, time source, uptime, diagnostics) |
| `/clockconfig` | Main settings form (display, LDR, blanking, backlight, security) |
| `/time` | NTP pool / timezone / update interval |
| `/utility` | Links to restart, WiFi reset, OTA update, force NTP sync, factory reset |
| `/setvalue?value=NNNNNN&time=SS&format=FFFFFF` | Push an arbitrary value onto the display |
| `/update` | OTA firmware upload |

Full endpoint reference in `SPECIFICATION.md`.

## On-device controls

A single push button drives everything on the device itself:

- **Short press** (in normal time mode): cycle temporary info displays (seconds, date, LDR reading, version, IP address, performance counter).
- **Press and hold (~1–2s)**: enter the settings menu; each further press advances through settings, with values incremented on release. The menu auto-saves and exits after 10 seconds of inactivity.
- **Hold at boot**: factory reset.

## License / Attribution

Slots transition effects credited to "Judge & Ty" in-source. No license file is currently present in this repository.
