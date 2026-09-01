# DA2000 4-Digit Numitron Clock — Technical Specification

This document describes the internal design of the firmware in `ESP8266Clock/`. For a user-facing overview, see [`README.md`](README.md).

## 1. Platform

- Target: ESP8266 (Wemos D1 R2/Mini or LOLIN equivalent), Arduino core, single `.ino` sketch plus per-module `.cpp`/`.h` files (no PlatformIO project in this repo).
- Flash layout: 4M flash, 1M SPIFFS (config + stats stored as JSON via ArduinoJson 5.x).
- Software version constant: `SOFTWARE_VERSION` in `ClockDefs.h` (currently `7`), shown on the display at boot and in the settings menu.

## 2. Pin Assignments

| Wemos label | GPIO | Signal | Defined in |
|---|---|---|---|
| D0 | 16 | `BLANKPin` — shift register output-enable/blank | `OutputManagerNumi4.h` |
| D1 | 5 | I2C SCL (RTC) | `ESP8266Clock.ino` (`Wire.begin(4, 5)`) |
| D2 | 4 | I2C SDA (RTC) | `ESP8266Clock.ino` |
| D3 | 0 | `inputPin1` — menu button, active-low | `ClockButton.h` |
| D4 | 2 | `pirPin` — PIR motion sensor input | `ClockDefs.h` |
| D5 | 14 | `LATCHPin` — digit shift register latch | `OutputManagerNumi4.h` |
| D6 | 12 | `DATAPin` — digit shift register data | `OutputManagerNumi4.h` |
| D7 | 13 | `CLOCKPin` — digit shift register clock | `OutputManagerNumi4.h` |
| RX | 3 | `LED_DOUT` — WS2812B data (NeoPixelBus, DMA over UART) | `LEDManager.h` |
| A0 | ADC | `LDRPin` — ambient light sensor | `ClockDefs.h` |
| — | 1 | `BUILTIN_LED_PIN` (defined, not directly driven) | `ClockDefs.h` |

RTC I2C address: `0x68` (`RTC_I2C_ADDRESS` / `DS1307_I2C_ADDRESS`).

Digit segments are pushed out as bytes over the shift-register chain (`shiftOut32`/`digitsOut` in `OutputManagerNumi4.cpp`). Segment bit assignment is `SEG_A..SEG_G` + `SEG_IND` (indicator), with an alternate mapping selectable via `#define IV_9_V1_0_A` in `OutputManagerNumi4.h` to compensate for a wiring error on an early rev of the IV-9 4-digit board (currently commented out — the "correct" mapping is active by default).

`DIGIT_COUNT` is defined twice: `6` in `ClockDefs.h` (vestigial — only referenced defensively in `Transition::updateRegularDisplaySeconds`, which is a no-op on this build) and `4` in `OutputManagerNumi4.h`, which is the value that governs actual display logic.

## 3. Startup Sequence (`setup()`)

1. Create the `OutputManager` singleton and show a POST-style message on the tubes.
2. Instantiate `WiFiManager`; wire the NTP "got time" callback (`newTimeUpdateReceived`).
3. Set up debug manager (serial output, off by default) and LED manager.
4. `Wire.begin(4, 5)` — start I2C for the RTC.
5. Advance through a sequence of diagnostic steps, each setting a status LED colour and a display message: `DIAGS_START` → `DIAGS_SPIFFS` → `DIAGS_WIFI` → `DIAGS_NTP` → `DIAGS_RTC` → `DIAGS_DEBUG` (IDs in `DisplayDefs.h`). Colours: RED = failed/missing, YELLOW = in progress, GREEN = OK, BLUE = user action taken (e.g. factory reset).
6. Poll the button for ~250ms (25 × `checkButton`); if held down, perform a **factory reset** and restart.
7. Load config from SPIFFS (`spiffs.getConfigFromSpiffs`). If absent, run `factoryReset()`. On success, apply web-auth, NTP pool/TZ/interval from the loaded config and recalculate LED manager variables.
8. If `current_config.testMode` is set, run an interactive **tube/LED self-test** loop: cycles digits 0–9 and flashes the separator/status LEDs; exits (and clears `testMode`, saving config) when the button is pressed while digit "8" is showing.
9. Start `WiFiManager.autoConnect()` using the device hostname as AP SSID and `SetMeUp!` as AP password; 60s config-portal timeout (`CONFIG_PORTAL_TIMEOUT`). Falls back to AP mode (`WiFi.softAPIP()`) if it can't join a network.
10. Enable `WiFi.setAutoReconnect(true)`.
11. Attempt an initial NTP fetch, retrying for up to 10s.
12. Register HTTP handlers (`setServerPageHandlers()`), start the web server, enable OTA at `/update` (creds `admin`/`update`), start mDNS (`<hostname>.local`, `http`/`tcp`/80).
13. Configure the PIR pin pull-up per config.
14. Detect and read the RTC (`testRTCTimeProvider`, `getRTCTime`); remember whether an RTC was ever seen (`onceHadAnRTC`) so a later RTC loss can be indicated.
15. Show the current time, briefly overridden by the version display (`TEMP_MODE_VERSION`, 5s).
16. Load usage stats from SPIFFS, set initial day-of-week for the backlight manager.

## 4. Main Loop

Runs every ~10ms (`delay(10)` at the end of `loop()`):

1. `server.handleClient()`, `mdns.update()`.
2. Track loop iterations per second (`impressionsPerSec`) — surfaced as a diagnostic ("Impressions/Sec").
3. On a real-time second rollover: run per-second processing, and — once per minute/hour/day as appropriate — the corresponding scheduled tasks (see below). A `triggeredThisSec` latch prevents double-firing within the same second.
4. Poll the button (`button1.checkButton`).
5. Determine menu navigation: a ≥1s or ≥2s press previews the next mode (`nextMode`); releasing commits it (`currentMode = nextMode`).
6. Dispatch to `setNextMode()` (when transitioning into a new mode — initializes that mode's display) or `processCurrentMode()` (steady-state handling: reacts to short button presses to change a value, redraws the display).
7. Sample the PIR pin (accumulates `pirConsecutiveCounts`, debounced/evaluated in `checkPIR()`).
8. Read the LDR (`getDimmingFromLDR()`), feed it to `LEDManager` and `OutputManager`.
9. `OutputManager::Instance().outputDisplay()` — push the digit buffer to the shift registers.
10. `setLeds()` — compute separator LED / backlight pulse state and push to `LEDManager`.

### Scheduled tasks

- **Once per second** (`performOncePerSecondProcessing`): decrement the temp-display-mode timer (auto-return to time display, or auto-save-and-exit the settings menu after `TIME_SET_MODE_DUR_MS` = 10s of inactivity); decrement blanking-suppression timer; recompute `blanked` from day/hour blanking rules or PIR state (native blanking is skipped entirely once a PIR is detected as installed); reset PIR debounce counter; trigger an NTP refresh if the scheduled interval has elapsed; recompute L/R status-LED source values.
- **Once per minute** (`performOncePerMinuteProcessing`): re-sync the internal `TimeLib` clock from the RTC (RTC is treated as authoritative even when NTP time is "valid"); bump uptime/tube-on-time counters; refresh cached IP.
- **Once per hour** (`performOncePerHourProcessing`): attempt to reconnect a dropped WiFi connection.
- **Once per day** (`performOncePerDayProcessing`, at hour==0): persist usage stats to SPIFFS; refresh day-of-week for the backlight manager.

## 5. Button & Menu System

`ClockButton` (`ClockButton.cpp/h`) polls a GPIO with 5-consecutive-read debouncing (`debounceCounter`) and derives duration-based state: `isButtonPressed[Now|1S|2S|8S]`, `isButtonPressedAndReleased`, `isButtonPressedRelease[1S|2S|8S]`. Thresholds: >1000ms, >2000ms, >8000ms since debounce-confirmed press start.

### Normal time mode (`MODE_TIME`)

- **Short press while displaying the time normally**: cycles through temporary info screens (`TEMP_MODE_*`, each shown for `TEMP_DISPLAY_MODE_DUR_MS` = 5s):
  0. Minutes/seconds
  1. Date
  2. LDR reading (normalised 100 dark – 999 bright)
  3. Software version
  4–7. IP address, one octet per press
  8. Loop impressions/sec (performance counter)
- **Short press while blanked (or within the blank-suppress selection window)**: steps through a temporary "un-blank" duration instead: 1st press = 10s, 2nd = 1 hour, 3rd = 4 hours (`blankSuppressStep`, capped at 3). The selection window itself times out after 5s (`TEMP_DISPLAY_MODE_DUR_MS`).
- **Press ≥1s or ≥2s**: previews entry into the settings menu (`MODE_HOURS_SET`, i.e. `MODE_MIN + 1`); committed on release.

### Settings menu (`MODE_*`, `ClockDefs.h`, values 0–38)

Entered via a ≥1s/≥2s press from `MODE_TIME`; each subsequent qualifying press advances `currentMode` by one. A short press *within* a settings step increments/toggles that step's value (wrapping at its max back to its min); the change takes effect on button release. The whole menu auto-exits to `MODE_TIME` (saving config to SPIFFS) after `TIME_SET_MODE_DUR_MS` = 10s without a press. Order:

1. `MODE_HOURS_SET` / `MODE_MINS_SET` / `MODE_SECS_SET` — set the clock (writes through to the RTC via `setRTCTime()`); skipped straight to `MODE_12_24` if NTP time is currently valid.
2. `MODE_DAYS_SET` / `MODE_MONTHS_SET` / `MODE_YEARS_SET` — set the date (no leap-year handling for Feb — always caps at 28 days).
3. `MODE_12_24` — 12h/24h display.
4. `MODE_LEAD_BLANK` — blank vs. show leading zero.
5. `MODE_DATE_FORMAT` — MM-DD vs DD-MM.
6. `MODE_DAY_BLANKING` — see §7.
7. `MODE_HR_BLNK_START` / `MODE_HR_BLNK_END` — blanking window hours (skipped if the selected day-blanking mode doesn't use hours).
8. `MODE_BLANK_MODE` — blank tubes / LEDs / both.
9. `MODE_USE_LDR` — enable/disable auto-dimming.
10. `MODE_LDR_THRSH` / `MODE_LDR_SENS` — LDR threshold and sensitivity (step 10, wraps at `SENSOR_THRSH_MAX`/`SENSOR_SENSIT_MAX`).
11. `MODE_MIN_DIM_UP` / `MODE_MIN_DIM_DOWN` — minimum dim level, adjustable up or down (step 10).
12. `MODE_SLOTS_MODE` / `MODE_SLOTS_FREQ` — date "slots" transition style and frequency.
13. `MODE_LED_BLINK` — separator LED blink pattern.
14. `MODE_PIR_TIMEOUT_UP` / `MODE_PIR_TIMEOUT_DOWN` — motion timeout, adjustable up or down (step 10s).
15. `MODE_BACKLIGHT_MODE` — backlight colour strategy.
16. `MODE_RED_CNL` / `MODE_GRN_CNL` / `MODE_BLU_CNL` — fixed backlight RGB channels (0–15) — only meaningful in fixed mode, but always steppable.
17. `MODE_CYCLE_SPEED` — colour-cycle speed (skipped unless `BACKLIGHT_CYCLE` is selected).
18. `MODE_BL_DIM` / `MODE_UL_DIM` — backlight / underlight dim factor (step 5).
19. `MODE_SEP_DIM` — separator LED dim level (bright/dim).
20. `MODE_L_IND_MODE` / `MODE_R_IND_MODE` — left/right status-LED source.
21. `MODE_DO_NOT_DIM_INDS` — keep indicator LEDs at full brightness regardless of dimming (for tubes using decimal points as indicators).
22. `MODE_USE_PIR_PULLUP` — toggle the PIR pin's internal pull-up.
23. `MODE_PREHEAT` — digit filament pre-heat strength.
24. `MODE_VERSION` — read-only software version display.
25. `MODE_TUBE_TEST` — all-digits-lit test pattern.

Several steps are conditionally skipped via `setNewNextMode()` when not relevant to the currently selected mode of an earlier step (e.g. hour-blanking fields, cycle-speed, RGB channels).

### Hold-at-boot

Holding the button through the first ~250ms of `setup()` triggers a full `factoryReset()` (config only — WiFi credentials, held by `WiFiManager`, are untouched) followed by a restart.

## 6. Time Handling

- **Sources**: `TimeLib` provides the in-RAM "current time"; the DS1307 RTC is the authoritative backing store; NTP (`NtpAsync`, async UDP) is the network source of truth when reachable.
- On NTP update, `newTimeUpdateReceived()` parses the server's CSV time string (`grabInts`, ordering `SYNC_YEAR,SYNC_MONTH,SYNC_DAY,SYNC_HOURS,SYNC_MINS,SYNC_SECS` = 0,1,2,3,4,5 in `ClockDefs.h`) into `TimeLib` via `setTime()`, then writes it through to the RTC (`setRTCTime()`).
- Every minute, the internal clock is **re-synced from the RTC** regardless of NTP validity (`getRTCTime(true)` in `performOncePerMinuteProcessing`) — the RTC is treated as more reliable than free-running `millis()`-based time between NTP syncs.
- RTC presence is tested each read/write (`testRTCTimeProvider()`, I2C probe at `0x68`) — if the RTC disappears after having been seen once (`onceHadAnRTC`), this is tracked (though not currently surfaced beyond a debug log line).
- NTP config: pool (default `pool.ntp.org`), POSIX TZ string (default `CET-1CEST,M3.5.0,M10.5.0/3`), update interval (default 7261s, range 60–86400s configurable via `/time`).
- Manual time-set via the settings menu (hours/mins/secs/day/month/year) writes through to the RTC on every increment.
- Display formatting is entirely handled in `OutputManagerNumi4` — `loadNumberArrayTime()`, `loadNumberArrayDate()` (format depends on `dateFormat`), `loadNumberArrayYear()`, `loadNumberArrayMinsSecs()`.

## 7. Blanking

Two independent blanking mechanisms feed a single `blanked` flag, evaluated once per second while in `MODE_TIME`:

- **Native (schedule-based) blanking** — `checkBlanking()` / `current_config.dayBlanking` (`ClockDefs.h`): `NEVER` (default) / `WEEKEND` / `WEEKDAY` / `ALWAYS` / `HOURS` (start/end hour, wraps across midnight if start > end) / and four OR/AND combinations of weekend-or-weekday with hours.
- **PIR (motion) blanking** — once a PIR is detected as installed (first time the pin is *not* seen as "no motion" for more than half a second's worth of loop iterations), native day/hour blanking is disabled entirely and blanking is instead driven purely by `current_config.pirTimeout` (60–3600s, default 300s) seconds of no detected motion.
- **Blank suppression** — a short press while blanked (or in blanking-mode) temporarily un-blanks for a user-selected duration (10s / 1hr / 4hr), overriding both of the above until it expires.
- **What gets blanked** — `current_config.blankMode`: tubes only, LEDs only, or both (default).

## 8. Display Engine (`OutputManagerNumi4`)

- Singleton (`OutputManager::Instance()`), owns a 4-digit buffer (`digit_buffer_t`: value + per-digit display type `BLANKED`/`NORMAL`/`BLINK`) and a separate value buffer (`value_buffer_t`) used by the `/setvalue` HTTP API.
- 7-segment bitmaps defined for digits 0–9 and hex letters A–F, plus a custom glyph set (S, P, I, F, n, E, t, C, O, blank, r, A, U, L, b, d) used to spell out POST/diagnostic messages ("SPIFFS", "FAIL", etc.) on the tubes themselves during boot.
- **Leading-zero blanking**: optional, applied only to the first digit position, configurable.
- **Blink**: asymmetric on/off duty cycle (`BLINK_COUNT_ON=70` impressions on, `BLINK_COUNT_OFF=55` off) — deliberately asymmetric because Numitron filaments light up faster than they fade, used to highlight the digit pair currently being edited in the settings menu.
- **Preheat**: `preheatStrength` (`PREHEAT_V_STRONG`..`PREHEAT_OFF`) — a technique to keep Numitron filaments gently warmed to reduce visible light-up lag, strength configurable / disableable.
- Output path: digit buffer → `shiftDigitsOut`/`digitsOut` → `shiftOut32` → shift-register chain (`LATCHPin`/`CLOCKPin`/`DATAPin`), gated by `BLANKPin`.
- Also provides highlight helpers used by the settings menu to blink specific digit pairs (`highlight0and1`, `highlight2and3`, `highlight4and5`, `highlightDaysDateFormat`, `highlightMonthsDateFormat`, `highlightYearsDateFormat`) and generic config-value display helpers (`displayConfig`, `displayConfig2/3`, `displayConfigWide`, `confBlink`).

## 9. "Slots" Transition Effects (`DA2000-Transition`)

`Transition` objects periodically interrupt the normal time display to flash the date, in one of two styles, controlled by `current_config.slotsMode` / `slotsFreq`:

- **Wipe** (`transitionWipe`, in/out/hold = 800/700/2800ms) — digits blank left-to-right, then the date fills in left-to-right; after the hold, blanks left-to-right again and the time fills back in.
- **Bang** (`transitionBang`, in/out/hold = 400/400/3200ms) — all digits blank simultaneously, then the whole date appears at once; same in reverse to return to the time.

Triggered at `second() == 50` in `MODE_TIME`, gated by `slotsFreq`: off, every minute, every 5 minutes (`minute() % 5 == 0`), or every 10 minutes (`minute() % 10 == 0`). While a transition is "on screen" (`activeTransition->runEffect()` returns true), normal time/value display is suppressed.

## 10. LDR Auto-Dimming

`getDimmingFromLDR()` (in the `.ino`) reads the raw ADC value, subtracts a configurable brightness threshold (`thresholdBright`, 0–500, default 200 — readings below this are treated as full brightness), scales by a configurable sensitivity factor (`sensitivityLDR`, 100–400%, default 300%), and smooths the result over time (`sensorSmoothCountLDR`, default 100 — higher = slower adaptation). The result is clamped so it never goes below `minDim` (100–800, default 400) worth of headroom, and is fed into both the digit-blanking logic (`OutputManager`) and the LED backlight dimming (`LEDManager`). Disabling `useLDR` fixes brightness at maximum (LDR value = 0).

## 11. LED / Backlight Management (`LEDManager`)

Drives WS2812B addressable LEDs (NeoPixelBus, DMA-based output on the UART TX-adjacent RX pin) for three logical LED groups: **backlight**, **underlight**, and **separator/status indicators**.

- **Backlight colour strategies** (`backlightMode`):
  - `BACKLIGHT_FIXED` — static RGB from `redCnl`/`grnCnl`/`bluCnl` (0–15 each, scaled through `rgb_backlight_curve`).
  - `BACKLIGHT_CYCLE` — smoothly cycles through colours at a configurable `cycleSpeed` (4–64).
  - `BACKLIGHT_COLOUR_TIME` — colour keyed to the currently displayed digit value, via a 10-entry per-digit-value RGB lookup table (`colourTimeR/G/B[0..9]`).
  - `BACKLIGHT_DAY_OF_WEEK` — colour keyed to the day of week, via a 7-entry lookup table (`dayOfWeekR/G/B[0..6]`).
- Perceptual brightness correction via a 256-entry gamma table (`dim_curve`).
- Independent dim factors for backlight (`backlightDimFactor`, 10–100%) and underlight (`extDimFactor`, 10–100%), both further scaled by the LDR reading and optionally by a "pulse" effect (`useBLPulse`) synced to the seconds tick.
- Separator LEDs (`led1State`/`led2State`, driven from `ledMode` — railroad-crossing alternation, slow/fast/double blink, always on/off) and two status indicator LEDs (`ledLState`/`ledRState`), independently sourced from `statusModeL`/`statusModeR`: NTP-valid, WiFi-connected, AM/PM, PIR-active, forced off, or forced on. `doNotDimIndLEDs` optionally exempts these indicators from LDR/blank dimming (useful when a display's decimal points double as indicators).
- Diagnostic boot-sequence LED (`setDiagnosticLED`) — reuses the LED hardware to show RED/YELLOW/GREEN/BLUE status during the `setup()` diagnostic sequence.

## 12. Persistent Configuration (SPIFFS)

`SPIFFS_CLOCK` (`SPIFFS.cpp/h`) persists two JSON documents to flash via ArduinoJson 5.x:

- **`spiffs_config_t`** — all user-configurable settings (~40 fields): NTP pool/TZ/update-interval, hour mode, min-dim, day-blanking mode + hours, blank mode, LDR enable/threshold/sensitivity/smoothing, backlight mode + RGB channels + pulse/dim toggles + dim factors + cycle speed, slots mode/frequency, PIR timeout + pullup, leading-zero blank, date format, separator LED mode, L/R status LED source, test-mode flag, web-auth enable + username + password, preheat strength, separator dim factor, indicator-dim-exempt flag, and a few fields defined but currently unused by the active logic (`scrollback`, `fade`, `fadeSteps`, `scrollSteps`, `antiGhost`).
- **`spiffs_stats_t`** — cumulative uptime minutes and tube-on-time minutes, persisted once per day and read back at boot.

Writes are conditional (`saveToSpiffsIfChanged`) — the web config handlers and settings menu only trigger a SPIFFS write when a value actually changed, to limit flash wear. `testMountSpiffs()` / `getSpiffsMounted()` support the boot-time diagnostics and status page.

### Factory reset

`factoryReset()` (in the `.ino`) restores every config field to its documented default, resets NTP settings, re-applies default web credentials (`admin`/`setup`, auth on), sets `testMode = true` (so the next boot runs the tube self-test), and saves. It does **not** touch WiFi credentials — those live in `WiFiManager`'s own storage and require the separate `/resetwifi` action (or equivalent) to clear.

## 13. HTTP API

Server: `ESP8266WebServer` (port 80) + `ESP8266HTTPUpdateServer` (OTA) + `ESP8266mDNS` (`<hostname>.local`). All handlers except `/`, `/local.css` require HTTP Basic Auth when `current_config.webAuthentication` is true (default), using `current_config.webUsername`/`webPassword` (default `admin`/`setup`).

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | no | Status dashboard: connection info, MAC/SSID, NTP status, mDNS URL, uptime, I2C bus scan, LDR/brightness, motion status, time source, RTC status, tube-on hours, ESP8266 diagnostics (heap/flash/chip ID/sketch hash), compact status-flag string. |
| GET/POST | `/time` | yes | View/set NTP pool, update interval, TZ string. |
| GET | `/reset` | yes | Restart the device. |
| GET | `/resetwifi` | yes | Clear WiFiManager's saved WiFi credentials and restart into AP mode. |
| GET | `/ntpupdate` | yes | Force an immediate NTP sync; redirects to `/utility`. |
| GET | `/factoryreset` | yes | Reset all config (not WiFi) to defaults; redirects to `/utility`. |
| GET/POST | `/clockconfig` | yes | View/set the full settings form (display, LDR, blanking, backlight, separators, security). |
| GET | `/setvalue` | yes | `?value=NNNNNN&time=SS&format=FFFFFF` — display an arbitrary number for `time` seconds (0–255, 255 = show indefinitely) with per-digit format (`0`=blank, `1`=normal, `2`=blink); `value` is taken mod 1,000,000. |
| GET | `/utility` | yes | Links to restart / WiFi reset / OTA update / force NTP / factory reset. |
| GET | `/debug` | yes | Enable serial debug output; redirects to `/utility`. |
| POST | `/update` | separate creds (`admin`/`update`) | OTA firmware upload. |
| GET | `/local.css` | no | Minified Bootstrap-derived stylesheet served from PROGMEM. |
| GET | `/reset-tmp` | no | Only advertised when disconnected — restarts into WiFi AP config mode. |
| — | 404 | no | Dumps method/URI/args as plain text. |

`checkServerArgBoolean/Int/Byte/String()` (`HtmlServer.cpp`) parse form fields against the current config value, apply the change, and return whether anything actually changed — used to gate conditional SPIFFS writes.

## 14. Debugging

`DebugManager` (`DebugManager.cpp/h`) is a thin wrapper: when enabled, `debugMsg(String)` writes to `Serial`; disabled by default at boot, can be turned on via the `/debug` web endpoint (persists only for the current session — not saved to SPIFFS) or by wiring `debugVal = true` in `setup()`. Most modules (`NtpAsync`, `SPIFFS_CLOCK`) accept a `DebugCallback` function pointer wired to `debugMsgLocal()`, which forwards to the shared `debugManager` instance.

## 15. Known Gaps / Unused Code

- `spiffs_config_t` fields `scrollback`, `fade`, `fadeSteps`, `scrollSteps`, `antiGhost` are persisted but not read anywhere in the current display/menu logic — likely reserved for features described in the sketch header comment ("Digit fading", "Digit scrollback") that aren't implemented in this version.
- `DIGIT_COUNT` is defined as both `6` (`ClockDefs.h`, effectively dead) and `4` (`OutputManagerNumi4.h`, the value actually used) — the header-level definition doesn't match the hardware this build targets.
- `OLED.cpp/h` implements a complete SSD1306 status-line driver (WiFi/NTP/PIR/blank/AM-PM icons, scrolling messages) but is never included or instantiated by `ESP8266Clock.ino` — dead code in this build.
- `LED_RAILROAD_X` (`DisplayDefs.h`) is explicitly commented "Not yet implemented" and is unreachable via configuration (`ledMode` only ranges 0–`LED_MODE_MAX`=5).
- No unit tests, CI, or license file are present in the repository.
- Leap years are not handled in `incDays()` — February is always capped at 28 days when incrementing the date via the on-device menu.
