# Standalone Programmer Device — Design Scoping

_Design feasibility scoping — 2026-09-24_

A pocket-sized, screen-and-encoder device that programs the eCom directly over UART — no phone, no laptop, no app, modeled on a real precedent in this exact market.

## Overview

A handheld, tethered programmer: a small MCU, a display, an encoder, and a UART probe or cable that talks to the ESC directly — no phone, no laptop, no app, no WiFi, no captive portal, none of the transport questions the other documents in this project wrestle with. It's the simplest possible protocol story of everything considered so far, because the "host" and the "ESC interface" are the same physical device: this MCU just becomes a third thing speaking the existing UART command protocol directly, the same way `wshandler()` does today over WebSocket.

This isn't a hypothetical category. A real competitor in this exact market already sells one — see Reference Product below — which is useful: it means the form factor, price point, and rough feature scope are already market-validated, not guesswork.

**Where it sits alongside the other work:** three tiers, each trading capability for convenience differently — the WiFi-Link product (full UI, any browser, no install, but fights captive-portal/WiFi quirks), the direct-connect bridge documented separately (MIDI-over-USB or plain UART, phone/laptop-based, no app on Android in some variants), and this standalone unit (zero host device needed at all, but the smallest screen and the narrowest feature scope of the three). Building this doesn't replace either of the others — it's a different use case: quick field adjustments without reaching for a phone.

## Reference Product: LatSlot's eCOM Programmer

[Latvian Slot Production](https://www.latslot.lv/) sells one for their own eCOM line — confirmed by reading the actual product page, not from memory:

| Spec | Value |
| --- | --- |
| Price | €100 (currently sold out) |
| Dimensions | 74×55×11mm |
| Display | OLED |
| Input | Buttons, stepped through settings sequentially |
| Power | USB-C, 5V |
| ESC connection | 3-pin pogo-needle probe pressed against programming pads — no cable, no solder |
| Safety | Wrong-hookup protection |
| Scope | Two settings only: motor timing, and an Auto-Brake disable toggle |

The narrow scope is the key takeaway, not a limitation to copy: LatSlot's device gets away with plain buttons and a tiny menu *because* it only exposes two settings. ESCape32 exposes far more (13 eCom-essential parameters, 30+ in the full Settings table), so the same button-only interface would be unpleasant to navigate — the input method needs to scale better than LatSlot's did (see Hardware, below).

## Hardware

**MCU:** no USB host/client complexity is needed here (unlike the direct-connect bridge) — just I2C/SPI for a display, a couple of GPIO/quadrature inputs for an encoder, and one UART for the ESC. That opens the field wide:

- **ESP32-C3**, for team familiarity — this project already has deep ESP-IDF tooling, testing infrastructure, and toolchain experience from the WiFi-Link firmware itself. Overkill on paper (no WiFi/BLE needed for v1), but it leaves a wireless option open later (e.g. syncing saved presets from the phone app) without a hardware respin.
- **STM32G0 / similar low-pin-count Cortex-M0+**, if BOM cost at volume matters more than reusing existing tooling — cheaper, lower power, plenty of headroom for a menu-driven UI.
- Either is a reasonable default; this is mainly a build-vs-buy-familiarity tradeoff, not a technical one.

**Display:** a monochrome OLED (SSD1306-class, 128×64 or 128×32), I2C, a couple of dollars, no backlight needed, good contrast outdoors — the standard choice for this form factor and almost certainly what LatSlot's own unit uses. Text-based scrollable menus fit it well.

**Input — the part worth getting right:** LatSlot's plain buttons work for a 2-setting menu; ESCape32's larger parameter set needs something that scales better. A **rotary encoder with an integrated push-button** (e.g. EC11) is the standard answer — rotate to navigate or adjust a value, click to select/confirm — one component doing what would otherwise take three or four buttons, and a well-worn UX pattern (this is exactly how most guitar-pedal and synth menu systems with a small OLED work). Optionally pair it with one dedicated back/cancel button for a simple two-control scheme.

**Power:** USB-C, 5V in, matching LatSlot exactly — no battery/charge-management complexity needed, since the natural use case (tethered to the ESC for a programming session) doesn't call for battery life. A trivial 5V→3.3V regulator if the chosen MCU is 3.3V logic.

**UART to the ESC:** identical physical layer and protocol to what `main.c` already implements — same baud rate, same framing, same command set. This device is a third "host" for a protocol that already exists; no new protocol design needed on the ESC side at all.

## Firmware & Menu Design

**Zero new protocol work.** The device issues the exact same ASCII commands `wshandler()` already passes straight through today — `show`, `get`, `set`, `save`, `reset`, `info`, `throt` (`main/main.c:312` onward) — directly over UART. There's no transport to translate, no framing to invent; this MCU just is the UART master, the same role the ESP32-S2 plays today via the WebSocket bridge.

**Menu structure:** a scrollable list navigated by the encoder, roughly mirroring `root.html`'s tab structure conceptually:

- **eCom essentials** (13 parameters) and the **full Settings table** (30+) — rotate to select a parameter, click to edit its value, rotate to adjust, click to commit (sends `set`).
- **Info / version** — a static screen showing `_info`'s bootloader/firmware revision, useful for quick field diagnostics without any other tool.
- **Throttle test** — encoder sets 0–2000 live, OLED shows eRPM/voltage/current, mirroring the Start/Stop behavior already in `root.html`'s `rampThrotTo()` (same ramped-decrease-only logic makes sense to replicate here too, for the same regenerative-braking-current reason).

**Deliberately out of scope for v1**, left to the phone/laptop tools where a real screen and file picker make sense:

- **Motor database** — needs far more input/screen than a 128×64 OLED + encoder can reasonably support.
- **OTA firmware update** — needs file selection and careful progress/error handling; better left where that UI already exists.
- **Wi-Fi AP config** — not applicable to a device with no radio.
- **Music/RTTTL editor** — typing melody strings via a rotary encoder is impractical.
- **Presets** — possibly worth *loading* a saved preset by slug/index if useful in the field, but creating/editing one stays on the richer tools.

This split isn't arbitrary — it's the same shape as LatSlot's own scope decision, just with a bigger "in scope" list because ESCape32 exposes more parameters than their two.

## Connector/Probe Design

Two real options, and this is the one design choice that reaches beyond the programmer itself into the eCom board's own layout:

**Cable + header connector** — the same approach the existing WiFi-Link board already uses to reach the ESC. Simplest to build, mechanically robust, no alignment fuss. Needs a cable and a mating connector or header pads on the eCom board, which likely already exist since the WiFi-Link board depends on them today.

**Pogo-pin needle probe** — LatSlot's approach: press-and-hold against exposed pads, no connector or solder needed on the ESC side, faster in-field use (touch and go rather than plugging in a cable). Needs careful pad spacing/alignment, and explicit **wrong-hookup protection** in the firmware or hardware — LatSlot calls this out specifically, presumably because a hand-held probe is easy to misalign or reverse compared to a keyed connector.

**Open dependency:** whether the eCom board has (or would need) dedicated probe-friendly pads is a question for the board's own design, not something this document can answer — worth a conversation with whoever owns that PCB before committing to the pogo-pin approach specifically.

## Effort, Risk & Open Questions

| Component | Effort | Risk |
| --- | --- | --- |
| MCU + OLED + encoder selection | Low — commodity parts, well-trodden combination | Low |
| Firmware: UART command layer | Low — reuses the existing protocol verbatim, no new design | Low |
| Firmware: menu/navigation UI | Medium — new code, but a standard embedded-UI pattern | Low–Medium |
| Power (USB-C 5V) | Low | Low |
| Connector: cable + header | Low — mirrors the existing WiFi-Link↔ESC link | Low |
| Connector: pogo-pin probe | Medium–High — needs eCom board pad coordination, alignment/wrong-hookup protection | Medium — depends on a PCB change outside this device's own scope |
| Enclosure/case | Medium — the first mechanical-design item in this whole line of exploration; everything before this was software or off-the-shelf modules | Medium — tooling cost, feel-in-hand matters for a €100-class product |

### Open questions

- Cable+header or pogo-pin probe — and does the eCom board have, or need, dedicated probe pads? This is the one item here that isn't purely this device's own decision.
- Which parameter set ships in v1 — eCom essentials only, or the full Settings table too? Affects menu depth and development time.
- MCU: ESP32-C3 (team familiarity, future wireless headroom) vs. a cheaper Cortex-M0 part purely for BOM cost at volume — does volume justify optimizing for unit cost yet?
- Encoder-only vs. encoder-plus-back-button — worth a quick physical mockup before committing either way.
- Is this a product sold alongside the ESC (LatSlot's model, €100), or a bundled/included accessory? Changes the cost-vs-polish tradeoff throughout.
