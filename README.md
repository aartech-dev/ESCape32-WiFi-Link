# AART Remora™ Programmer

Web-based wireless programmer for the [Remora™ electronic commutator (eCom)](https://aart.dev) — a brushless motor speed controller for slot car racing built around the ESCape32 firmware. Runs on an ESP32-S2 Wi-Fi module; any browser on any device connects without installing an app.

**[⬇ Latest release](https://github.com/aartech-dev/ESCape32-WiFi-Link/releases/latest)** — download the prebuilt, ready-to-flash image. **[📖 Wiki](https://github.com/aartech-dev/ESCape32-WiFi-Link/wiki)** — screenshots of every tab, for both the AART and NSR brands.

---

## File Structure

```
aart_remora_programmer/
│
├── README.md                     ← This file
├── CMakeLists.txt                ← ESP-IDF top-level project file
├── sdkconfig.defaults            ← ESP32-S2 pin defaults (UART, LED)
├── mock_server.py                ← Python desktop simulator (no ESP32 needed)
├── test_mock.py                  ← Automated Python test suite
│
├── docs/                         ← Design/feasibility docs (not user-facing)
│   ├── midi-direct-connect-feasibility.md
│   └── standalone-programmer-feasibility.md
│
└── main/                         ← ESP-IDF component (application source)
    ├── CMakeLists.txt            ← Embeds & gzips root*.* at build time
    ├── main.c                    ← HTTP server, WebSocket bridge, DNS, NVS
    ├── build_defs.h.in           ← CMake template → build_defs.h
    ├── Kconfig.projbuild         ← menuconfig: UART pins, LED pin
    ├── idf_component.yml         ← Component dependencies (mdns)
    ├── root.html                 ← Single-page web UI (6 tabs, 17 languages)
    ├── root_en.json              ← UI strings – English
    ├── root_de.json              ← UI strings – German
    ├── root_fr.json              ← UI strings – French
    ├── root_it.json              ← UI strings – Italian
    ├── root_es.json              ← UI strings – Spanish
    ├── root_pt.json              ← UI strings – Portuguese (Brazilian)
    ├── root_nl.json              ← UI strings – Dutch
    ├── root_sv.json              ← UI strings – Swedish
    ├── root_da.json              ← UI strings – Danish
    ├── root_uk.json              ← UI strings – Ukrainian
    ├── root_lv.json              ← UI strings – Latvian
    ├── root_fi.json              ← UI strings – Finnish
    ├── root_et.json              ← UI strings – Estonian
    ├── root_cs.json              ← UI strings – Czech
    ├── root_pl.json              ← UI strings – Polish
    ├── root_lt.json              ← UI strings – Lithuanian
    └── root_zh.json              ← UI strings – Chinese (Simplified)
```

---

## Quick Start

### Desktop simulation — Python mock server (no ESP32 needed)

The mock server perfectly simulates the ESP32's HTTP and WebSocket behaviour,
including realistic eRPM / voltage telemetry driven by the throttle slider.

```bash
# From the project root:
python3 mock_server.py

# Open in your browser:
open http://localhost:8080        # macOS
xdg-open http://localhost:8080    # Linux

# Custom port:
python3 mock_server.py --port 9090

# Run the automated test suite:
python3 test_mock.py --start-server
```

### Flash to ESP32-S2

**Prebuilt image (no ESP-IDF needed):** download the merged binary from the
[latest release](https://github.com/aartech-dev/ESCape32-WiFi-Link/releases/latest)
and flash it to **offset 0x0** with
[Adafruit WebSerial ESPTool](https://adafruit.github.io/Adafruit_WebSerial_ESPTool)
or `esptool --chip esp32s2 write-flash 0x0 ESCape32-WiFi-Link-ESP32-S2.bin`.

**Build from source:**

```bash
# Install ESP-IDF 5.x, then from the project root:
idf.py set-target esp32s2
idf.py build flash monitor

# sdkconfig.defaults sets the correct pins automatically.
# To change pins:  idf.py menuconfig → ESCape32-WiFi-Link configuration

# On the device:
#   Connect to Wi-Fi AP:  ESCape32-WiFi-Link  (open, no password by default)
#   Open browser:         http://192.168.4.1
#              or:        http://escape32.local
```

Joining the AP normally pops up your phone's captive-portal sign-in browser
automatically, loading the app directly — no extra tap needed.

### Building a different brand

This same source tree also builds two single-purpose variants, each with its
own color theme, a system-standard font instead of Trebuchet MS, a
brand-only built-in motor catalog, and no Settings/Music tabs:

- **NSR Programmer** — NSR's own color theme (white/red/black).
- **Slot.it Programmer** — Slot.it's color theme (yellow/red-orange/black,
  sampled from their logo), with a built-in preset for their 2,000Kv 1106
  motor tuned from their own test data (`timing=12, freq_min=freq_max=48,
  duty_spup=duty_rate=20`).

Select a brand with the `BRAND` CMake variable (default: `aart`):

```bash
idf.py set-target esp32s2
idf.py build -D BRAND=nsr flash monitor
# or: idf.py build -D BRAND=slotit flash monitor

# Or build into a separate directory to keep every binary around:
idf.py -B build_nsr -D BRAND=nsr build
idf.py -B build_slotit -D BRAND=slotit build
```

`main/CMakeLists.txt` is the single source of truth for what differs
between brands (logo text, colors, default AP name, built-in motors) — see
the `BRAND STREQUAL "nsr"` / `"slotit"` blocks there. `mock_server.py
--brand nsr` (or `--brand slotit`) mirrors the same branding for local
testing without hardware.

---

## How It Works

```
Browser ──WebSocket──► ESP32-S2 HTTP/WS server (main/main.c)
                               │
                               ├─ Text commands (show / get / set / save / reset /
                               │   info / throt / play / _wifi_get / _wifi_set /
                               │   _preset_save)
                               │   passed as ASCII over RS-485 UART to ESC
                               │
                               └─ Binary protocol (_probe / _info / _update)
                                   val+~val byte pairs with CRC32
                                       │
                               RS-485 UART ──► Remora™ eCom board
                                              (ESCape32 firmware)
```

All static content (HTML, JSON language files) is gzipped and embedded in
the firmware at build time by the CMake build system — no file system or SD
card is required on the ESP32-S2.

---

## UI Tabs

### eCom — Essential motor parameters

The 13 parameters most relevant to slot car operation, with a motor preset
system backed by the Motor Database.

| Parameter | Purpose |
|---|---|
| `damp` | Complementary PWM — enable unless limiting output voltage |
| `revdir` | Reverse motor direction |
| `timing` | Advance timing — higher = faster no-load speed, more current |
| `freq_min` / `freq_max` | PWM frequency kHz — low = lower losses; high-Kv motors may need 48+ kHz |
| `duty_min` / `duty_max` | Output voltage range — normally 100% |
| `duty_spup` | Spin-up power limit — caps inrush current at startup |
| `duty_ramp` | Power ceiling at the kERPM ramp threshold |
| `duty_rate` | Duty cycle slew rate — lower = softer throttle response |
| `throt_ztc` | Zero-throttle coasting — freewheel instead of active braking at zero throttle |
| `analog_min` / `analog_max` | Analog input setpoints — normally 0 / 1440 |

The live telemetry bar at the bottom shows eRPM, voltage, and current. When a
motor with a known pole count is selected, mechanical RPM and Kv are also
displayed and updated in real time as the throttle slider moves.

**Start/Stop:** the live throttle sliders (eCom and Settings tabs) are
flanked by a **Stop** button at the zero end and a **Start** button at the
full-throttle end, so both ends of the range are one tap away without
having to land the slider precisely. Any decrease in throttle — whether
from the Stop button or from dragging the slider down — is ramped down in
small steps rather than applied as a single instant drop, to limit
regenerative-braking current spikes from a loaded motor. Increases (Start
included) are applied immediately. Enabling `throt_ztc` addresses
regenerative current at the source by having the ESC coast instead of
actively brake at zero throttle, which is strongly recommended whenever the
power supply cannot sink regenerative current.

### Motors — Motor database

A browser-local IndexedDB database of motor specifications. Fields include
vendor, model, Kv (rated and measured), pole count, stator geometry,
winding details, rotor dimensions, shaft, fixation, inductance, resistance,
mass, mounting dimensions, and a reference URL. Nine built-in motors are
pre-loaded (EMAX, Parma, Do-Slot, AMAX, AART). Any motor in the database
can be selected as the active motor for RPM / Kv display in the eCom tab.

### Firmware — OTA update

Upload a new ESCape32 binary directly from the browser to the connected ESC
over the Wi-Fi link. Supports bootloader and firmware image targets,
write-protection control, and a real-time progress bar.

### Wi-Fi — Access Point configuration

Change the AP SSID and password. Settings are saved to ESP32 NVS flash and
take effect after the next power cycle.

### Settings — Full parameter table

Complete ESCape32 parameter set with live throttle control and the same
eRPM / RPM / Kv telemetry display as the eCom tab.

### Music — RTTTL melody editor

Edit and play the ESC startup melody using RTTTL notation, with volume
and beacon controls.

---

## Language Support

The UI is fully internationalised. On first load it auto-detects the
browser's own language (`navigator.language`) and switches to it if it's
one of the supported languages, falling back to English otherwise; your
choice is then remembered (`localStorage`) for future visits regardless of
browser locale. Switch language manually anytime with the selector in the
top-right corner. All strings — including eCom parameter hints, motor
database field labels, and tab content — switch instantly. Supported
languages: **English, German, French, Italian, Spanish, Portuguese
(Brazilian), Dutch, Swedish, Danish, Ukrainian, Latvian, Finnish, Estonian,
Czech, Polish, Lithuanian, Chinese (Simplified)**.

Only English is embedded inline in the page at build time, for an instant
first paint with no network round-trip. Every other language is fetched on
demand from `GET /?<lang>` the first time it's selected, then cached in
memory for the rest of the session — see `setlang()` in `root.html`. This
is why each `root_XX.json` is still individually gzipped and embedded in
the firmware image even though only English ships inline in `root.html`
itself; at 17+ languages, inlining all of them added tens of KB to the
compressed image for translations most sessions never touch.

The on-demand fetch runs safely alongside an open WebSocket connection:
ESP-IDF's httpd multiplexes all sockets with `select()` rather than
blocking on one connection at a time, so a quick static-file GET
interleaves with WS traffic without stalling either one.

---

## NVS Storage Layout

| Namespace | Key | Value |
|---|---|---|
| `remora` | `ssid` | AP SSID (default: `ESCape32-WiFi-Link`) |
| `remora` | `pass` | AP password (default: empty — open network) |
| `remora` | `p_<slug>` | JSON blob of a saved ESC parameter preset |

---

## Hardware

| Signal | ESP32-S2 GPIO |
|---|---|
| UART TX (to ESC) | 33 |
| UART RX (from ESC) | 16 |
| Status LED | 15 |

Pins can be changed via `idf.py menuconfig` → **ESCape32-WiFi-Link
configuration**, or by editing `sdkconfig.defaults` before the first build.

The UART runs at 38,400 baud, RS-485 half-duplex mode.

---

## Requirements

### ESP32-S2 build
- [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/) 5.x
- ESP32-S2 module
- Remora™ eCom board (ESCape32 firmware) connected via RS-485 UART

### Desktop simulation
- Python 3.7+ (mock server and tests — stdlib only, no pip installs required)

---

## Project

**AART — Adrian & Richard's Technologies**  
[aart.dev](https://aart.dev)  
ESCape32 firmware: [escape32.org](https://escape32.org)

© Adrian & Richard's Technologies. Hardware designs open source.  
Firmware uses [ESCape32](https://github.com/neoxic/ESCape32) — GPL-3.0.
