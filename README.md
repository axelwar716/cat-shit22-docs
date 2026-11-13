# CAT S22 Flip — device documentation

What is inside the CAT S22 Flip, how it is wired, how CAT's stock software behaves on it, and what can and cannot be done with it.

This is a map of the phone, not a guide to building anything. It covers the hardware, its electrical behaviour, the stock firmware (Android 11 Go, v30) with its bugs and limits, and a viable / not viable verdict for each area.

## Contents

| Page | What it covers |
|---|---|
| [Hardware and electrical](docs/hardware.md) | SoC, PMIC and rails, chip inventory with buses, addresses and GPIOs, pin map, battery and gauge hardware, memory, thermal, ESD, hard limits |
| [Power, battery and sleep](docs/power.md) | Fuel gauge behaviour, how the platform sleeps, what keeps it awake, modem and Wi‑Fi idle, CPU/GPU/DDR, lid-closed drain by build and reference battery results with the method |
| [Displays](docs/displays.md) | Main and outer screens: panels, video paths, backlights, stock outer-screen logic, rendering |
| [Keypad, keys and lid](docs/keypad-lid.md) | Matrix keypad and its stock sampling problem, keypad backlight, Hall sensor, every physical key on stock vs a GSI |
| [Audio, Bluetooth, NFC and pogo pins](docs/audio-bt-nfc.md) | Audio chips and path, Bluetooth stack, why music keeps the SoC awake, NFC, pogo contacts |
| [Modem, telephony, IMS/VoLTE](docs/telephony.md) | Modem firmware, bands, carrier profiles, stock behaviour with Spanish carriers, caller ID, VoLTE status |
| [Stock firmware, boot, kernel, vendor and ROMs](docs/platform.md) | Partitions and 56-entry GPT layout, boot chain and X.509 signatures, secure boot (`0x50`), EDL 9008 (`05c6:9008`) recovery and SoC jailbreak limits, kernel and vendor facts, GSIs, 2 GB RAM, known traps |

Each device page ends with a **Viable / not viable** table.

**Battery reference (own build, current):** ~0.6 %/h with the lid closed on LTE + IMS (about 7 days of standby); 8.5–9.7 %/h in active use with the screen on, depending on network (LTE only, Wi‑Fi, or both) (100 → 20 % in 8 h 26 min). Results, method and limits, so other builds can be compared: [power.md §11](docs/power.md#11-reference-results-for-comparison).

## How to read it

- **Stock** means the phone as CAT ships it: firmware v30, CAT's kernel and vendor.
- **Works** means it has been done on this phone. **Works (own build)** means it was done on the author's own build: a custom ROM, a vendor rebuilt from stock v30 keeping CAT's blobs, and a custom kernel; how it was done is not published. **Works (public kernel)** means it is done in the public [cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel) `stock-stable-5`, whose README documents how, so those rows may describe the mechanism. **Fixed** means a stock bug was fixed, with the same suffixes. **Reported working** means third-party or user reports, not verified here. **Viable** means it is technically possible; the row says what it costs (effort, risk, a hardware mod) or that it brings no gain. **Not viable** is kept for hard blocks only: missing hardware, signed or fused firmware, or behaviour set by the network. "Untested" or "partial" is added when a verdict isn't fully proven.
- Battery results are given as **%/h with the lid closed**, counted over the time the lid was shut, on the rated 2000 mAh (1 mA = 0.05 %/h). Each figure states its method (charge counter, rest voltage or awake-time counters); the charge counter under-reads idle drain. That is how drain fixes are compared across builds (see [power.md §10](docs/power.md#10-lid-closed-drain-by-build)); active-use reference results and the measurement method are in [§11](docs/power.md#11-reference-results-for-comparison).
- Evidence levels: **Measured** (read on the phone), **Inferred (strong)** (several independent sources agree, or the numbers match a known standard exactly), **Estimate** (a figure computed from datasheets or measured duty cycles, not resolvable by the phone's gauge), **Hypothesis** (a plausible mechanism with no supporting data yet). Facts not confirmed on a running stock phone say so.
- This repo contains no CAT, Qualcomm or third-party files, firmware or decompiled code.
- Fixes in the own build are named by what they solve ("lid-sleep fix"). This repo documents the stock phone and what is possible; it is not a build guide for those fixes. The exception is the **caller-ID fix** (hidden number on incoming VoLTE calls, tested with Lowi (Vodafone)): it is a modem configuration change, not part of any ROM, and [telephony.md](docs/telephony.md#caller-id-hidden-caller-number) documents the cause and the risk.

## Related

- Custom stability-focused kernel for the S22 Flip: [cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel).
- Mainline Linux on the S22 Flip (not Android): reported working by third parties, with calls, texts, data, both screens and lid-closed suspend.

## License

Documentation licensed under [CC BY-NC-SA 4.0](LICENSE): you may share and adapt it with attribution, for non-commercial purposes, under the same license. For commercial use, ask first.

© 2026 cat-s22. Licensed under CC BY-NC-SA 4.0.

CAT is a trademark of Caterpillar Inc. This project is not affiliated with Caterpillar or Bullitt Group.
