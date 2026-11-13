# Keypad, keys, keypad backlight and lid

Scope: the matrix keypad and its stock sampling problem, the keypad backlight, the lid (Hall) sensor, how every physical key and the lid behave on stock compared with a generic AOSP GSI, and what this means for keyboard apps (IMEs).

Related: [hardware.md](hardware.md) (chips, buses, GPIO map), [displays.md](displays.md) (outer screen and what happens on lid close), [platform.md](platform.md) (SoC, kernel, vendor), [power.md](power.md) (sleep and battery cost).

## Overview

| Part | Hardware | Linux side (stock) |
|---|---|---|
| Front keypad | 5×5 matrix of dome contacts on SoC (TLMM) GPIOs | `gpio-matrix-keypad` driver, built into the kernel; input device `soc:matrix_keypad` |
| Red key | PM8916 PMIC power-on block (`kpdpwr`) | `qpnp_pon`, `KEY_POWER` (116) |
| Volume down | PM8916 PMIC power-on block (`resin`) | `qpnp_pon`, `KEY_VOLUMEDOWN` (114) |
| Volume up | TLMM GPIO 91, pull-up | `gpio-keys` (`soc:gpio_keys`), `KEY_VOLUMEUP` (115) |
| Keypad backlight | TLMM GPIO 130, on/off only | OEM LED driver, `/sys/class/leds/keyboard_light` |
| Lid sensor | Hall sensor on TLMM GPIO 44, powered from PM8916 `l5` | OEM `hall_sensor` driver + `switch_hall` class |

---

## Matrix keypad

### Hardware

- **5×5 matrix.** 25 positions are declared to Linux. The keys are dome contacts that close a column onto a row.
- **Columns (outputs):** TLMM GPIO 96, 99, 90, 89, 129. 2 mA drive, active-high (the DT node has no `gpio-activelow`).
- **Rows (inputs and interrupts):** TLMM GPIO 126, 127, 45, 97, 62. Each row has its own interrupt, armed on both edges. **GPIO 45 is the `1 4 7 *` row.**
- **Stock row bias:** on stock the rows idle low through the SoC's internal pull-down.
- The keypad node lives in the DTBO (`gpio-matrix-keypad`, with `linux,wakeup`), so a key press can wake the phone from suspend.
- The stock driver is Qualcomm's public 4.9 `matrix_keypad.c` with vendor changes:
  - a double read per row: the key only counts if the row follows the column;
  - samples with all 5 columns active are thrown away (anti-ghosting);
  - a row "discharge" step (see Stock software defects below);
  - a kernel log line for every key change, which prints the row, column and keycode.

### Key codes on the matrix

| Physical key | Linux code | Notes |
|---|---|---|
| 0–9 | `KEY_0`–`KEY_9` | |
| `*` / `#` | `KEY_NUMERIC_STAR` (522) / `KEY_NUMERIC_POUND` (523) | |
| C (delete) | `KEY_BACKSPACE` (14) | |
| OK | `KEY_ENTER` (28) | |
| D-pad | `KEY_UP/LEFT/RIGHT/DOWN` | |
| Back | `KEY_BACK` (158) | |
| Green | `KEY_PHONE` (169) | |
| Home | `KEY_HOMEPAGE` (172) | |
| Recents | `KEY_APPSELECT` (580) | |
| Speaker key (top left, above green) | `KEY_MICMUTE` (248) | |
| Orange (programmable) key | `KEY_NUMERIC_B` (525) | |
| (none) | `KEY_NUMERIC_A` (524) | **No physical key emits it.** It is a leftover entry in the driver's key map. |

The device also reports `MSC_SCAN`.

### The stock problem: doubled letters and lost light taps

This happens on CAT's stock firmware. It is not a LineageOS/GSI issue: the DT, DTBO and keypad driver are byte-for-byte the same.

**Bounce:** the dome contacts bounce for **9–18 ms** when they close and open.

**Stock sampling:** the stock kernel reads the keys **once**, about 10 ms after the first edge (HZ=100 tick plus workqueue lag). The row interrupts stay off during that wait, and nothing re-scans while a key is held. The next scan only comes with the next interrupt.

That single, late, tick-quantised sample causes both symptoms:

- **Doubled letters (ghosts).** If the sample lands in a bounce gap, the driver reports a release. The contact then closes again and produces a second press. Measured with the stock kernel: one press of `#` gave 3 DOWN/UP pairs within 260 ms, with gaps of 27 and 9 ms.
- **Lost taps.** If a short tap opens and closes before the sample is taken, the scan sees no change and reports nothing.

No single stock debounce value fixes both. A longer delay reduces ghosts but widens the window in which fast taps are lost.

**Light taps, worst on the 1/4/7/\* row (electrical):** the misses that remain are taps that **produce no interrupt at all**. Stock idles the rows low through the SoC pull-down. A light press on a worn dome has high contact resistance. That resistance forms a **voltage divider** with the pull-down, so the row never crosses the input threshold: no IRQ, lost tap. Software that only listens to the row interrupt cannot see such a tap.

**Fixed by the public stock-compatible kernel** ([axelwar716/cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel), **Fixed (public kernel)**). CAT's own kernel binary can't be fixed by root: the driver is built in and the kernel only accepts modules signed with CAT's key. The public kernel replaces only `boot` and changes the driver as follows (its README has the details):

- **Sampling:** keys are sampled on the edge by a `SCHED_FIFO` thread; an hrtimer re-samples every 1 ms until the contacts settle, and every 5 ms while a key is held. A press is 6 ms of cumulative closure; gaps under 30 ms don't split it; bursts under 6 ms are noise. A release is 30 ms fully open.
- **Electrical:** the row pull-down is turned off (written to TLMM `GPIO_CFG`) and each row is actively driven low for 1 µs before every read and before the IRQ is re-armed, so light taps register without ghosts. The pull-down is restored in suspend so a floating row can't wake the phone.
- Rows are re-read after the IRQ is re-armed (the dual-edge emulation can miss an edge). Idle is IRQ-only, no polling.
- Tunables and counters are in `/sys/module/matrix_keypad/parameters/`.

### Stock software defects around the keypad

- **Row "discharge" that never runs.** The vendor's row "discharge" (driving a row as output between column reads) is always refused by gpiolib, because the line is an IRQ. It never moves the line electrically. It does print "tried to set a GPIO tied to an IRQ as output" about 25 times per scan: about 3,565 lines per 2 s of typing, enough to push earlier boot messages out of the kernel log.
- **Every key logged.** Every key change is written to the kernel log with its row, column and keycode.

---

## Keypad backlight

- **Hardware:** TLMM **GPIO 130**, active-high, 2 mA pad. It can only switch an external driver stage (likely a transistor fed from the battery/VPH side; unconfirmed without the board). The DT declares no current limit and no regulator for it.
- **Binary, no PWM.** GPIO 130 has no PWM or GP-clock function in the pin mux (only GPIO and an SPI function), and it is not on a PMIC LED/PWM block (the PM8916 has no WLED; its LPG drives the main backlight). The light is either fully on or off. **True dimming needs a hardware mod.** Software PWM of the GPIO from a kernel timer is viable but costs CPU wakeups (untested). Otherwise the only lever is how long it stays on.
- **Current while on: about 28 mA** (measured, 22.7–33.4 mA interval; this is a large share of open-lid use).
- **Stock kernel driver.** An OEM LED-class driver creates `/sys/class/leds/keyboard_light`:
  - max 255; **any value > 0 means on**;
  - trigger `none` (the `keyboard_light-trigger` it names is registered by nobody);
  - the stock kernel config lists `LEDS_MSM_GPIO_KEYBOARD_LIGHT` as not set, yet the driver is in the stock kernel.
- **DT leftover.** The disabled NFC node in the DT (a leftover from the reference design; no NFC chip is fitted) also lists GPIO 130 as its firmware line. GPIO 130 belongs to the keypad light; see [hardware.md](hardware.md).
- **Stock rule.** The public kernel README describes the rule as: **on only while the flip is open and the main screen is on**, off with the lid closed or the screen off. The stock kernel driver itself is a plain on/off device, so the decision is made elsewhere in CAT's software. Which stock component switches it has not been traced. It is **not** the vendor lights HAL.
- **Why a GSI leaves it off.** The stock vendor's lights HAL (`light-qti`) exposes only backlight, battery, notifications and attention, not the KEYBOARD or BUTTONS lights. Android also reports no physical keyboard. As a result, AOSP's own keyboard/button light logic never fires and nothing in Android writes the LED: on a GSI the keypad stays dark. Community scripts write the sysfs node as root by polling the main backlight. That works, but it is a polling loop.

---

## Lid (Hall sensor)

### Hardware and DT

| Item | Value |
|---|---|
| DT node | `/soc/hall`, `compatible = "hall-switch"` |
| Signal | TLMM **GPIO 44** (`linux,gpio-int`), flagged active-low; pin config pull-up |
| Interrupt | `interrupts = <44 0x2003>`: both edges, one-shot |
| Wake | `linux,wakeup`: opening the lid wakes the phone from suspend |
| Supply | `vddio-supply` = PM8916 **`l5`** (DT range 1.65–3.3 V). `l5` also feeds the eMMC I/O, so it can never be switched off to save power. |
| Driver | `hall_sensor` (Qualcomm-derived OEM driver; the same driver family is in other QM215 phones' GPL kernels), built in |
| Input device | name `hall_sensor`, phys `/dev/input/hall_dev` |
| Switch | `/sys/class/switch_hall/hall_switch/state` (the class is named `switch_hall` because extcon already owns `switch`) |

### Polarity

| Lid | GPIO 44 level | `hall_switch/state` | Linux/AOSP `SW_LID` convention |
|---|---|---|---|
| Open ("hall far") | high | **1** | 0 |
| Closed ("hall near") | low | **0** | 1 |

The stock switch uses **1 = open**, the opposite of the Linux/AOSP lid convention (1 = closed). Any code that mixes the two gets the lid inverted.

### What the stock driver does (and why Android does not see the lid)

- **Wrong event type.** The input device declares only `EV_SW`/`SW_LID`, but on every change it emits `KEY_SLEEP`/`KEY_WAKEUP`. The device has no `EV_KEY` capability, so the input core **drops these events**. `SW_LID` therefore stays at 0 ("open") forever, whether the lid is open or closed.
- **Through the standard input path, Android never learns the lid state.** Stock Android 11 gets around this with CAT's own code: its power manager reads the `hall_switch` state through the switch uevent. A GSI or any plain AOSP build has no such code, so **with the stock kernel the lid does nothing on a GSI**.
- **300 ms busy-wait.** The stock driver busy-waits **300 ms inside its IRQ handling** (and again in its workqueue) before it publishes the new state. The switch therefore changes about 300 ms after the magnet edge, and a CPU spins for that time on every lid move.
- **Outer screen hook.** The driver also passes the new state to the outer-screen SPI panel driver. That is how the stock kernel powers the outer panel up and down; see [displays.md](displays.md).
- **Lid-close hangs** seen when the outer screen is driven come from the SPI panel path, not from the Hall driver; see [displays.md](displays.md).

**Reporting a real `SW_LID` works (public kernel)** and needs a kernel change only: no framework patch, because AOSP already handles `SW_LID`. The public kernel reports it on an input device named `s22_lid` (1 = closed), keeps the stock `switch_hall` switch and `/dev/input/hall_dev` for cover-screen apps, and replaces the 300 ms busy-wait with delayed work.

---

## Physical keys: stock vs. a generic AOSP GSI

Stock key mapping: the stock firmware has **no keypad-specific key layout**. The matrix keypad falls back to CAT's `Generic.kl`, which maps:

- 248 → `MICMUTE`
- 522 → `STAR`, 523 → `POUND`
- 525 → `PTT`
- 580 → `APP_SWITCH`
- 524 is left unmapped.

The PMIC keys use a stock `qpnp_pon.kl` that maps **116 → `ENDCALL`**. At Linux level the red key is a power key, but stock Android receives it as **End call**, not Power.

CAT's private keycodes **300 (PTT)** and **301 (MICMUTE)** collide with keycodes that Android 13 assigns to other keys. Stock key maps therefore cannot be reused as-is on newer Android; the keys must be mapped to standard keycodes.

The GSI column below is AOSP 13 behaviour (checked against LineageOS 20 with no device-specific changes, plus a TrebleDroid Android 16 GSI for the key mapping). A GSI uses AOSP's own `Generic.kl`, so:

- digits, `*`, `#`, C, Back, Green and Recents work;
- **248 (speaker key) and 525 (orange key) are unmapped** and arrive as keycode 0;
- the red key is **Power**.

| Key | Stock (CAT Android 11) | Generic AOSP GSI |
|---|---|---|
| **Any front key, screen off** | Turns the screen on (stock treats all front keys, D-pad, Back, Home, Recents, Enter and Delete as wake keys) | Does **not** turn the screen on: AOSP's `Generic.kl` has no `WAKE` flags and AOSP 13 only treats a few keys as wake keys |
| **Red** | Ringing: **rejects**. In a call: **hangs up, even with the screen off**. Short press with the screen on: screen off. Long press (500 ms): power menu. Double-press gestures (camera) also work. | Power key. Ringing: **only silences**. In a call: **does not hang up** by default. Short press: screen on/off. Long press: power menu. |
| **Green** | Short: call button (dialer, or the in-call screen during a call). In the dialer with a number: calls. Ringing: answers. Long: voice command. | Same framework handling (answers when ringing). The AOSP-derived dialer in LineageOS 20 does **not** place a call with the green key, only with OK (likely the same in other AOSP dialers). |
| **Speaker key (248)** | Short: does nothing except wake the screen. **Long (800 ms) in a call: speakerphone on/off** with an on-screen notice. | Unmapped (keycode 0): does nothing |
| **Orange key (525)** | Short: nothing. **Long press (500 ms) and double press (< 500 ms)** run a configurable action: wake device, notifications, recent apps, calls, flashlight, or an app. With no long-press action set, it opens Settings → Programmable key. Works the same with the lid closed (the "calls" action only answers with a headset). | Unmapped (keycode 0): does nothing |
| **Vol− + orange** | Screenshot | Not available (orange unmapped). Screenshot is Vol− + red (Power). |
| **Home** | Short: home (waits 300 ms for a double press). Double: app drawer. Long: Assistant. | Short: home. Long: Assistant. Double: nothing. |
| **Recents** | Short: recents. **Long (500 ms): opens/closes the notification shade.** Back + Recents: unpin a pinned screen. | Short: recents. Long: nothing. No Back + Recents combination. |
| **Back** | Back. Long: nothing. | Back |
| **OK** | Confirm. In the dialer: calls. | Confirm |
| **Digits on the home screen** | Open the dialer with that digit | On Launcher3-based launchers: open search, not the dialer |
| **Dialer, long press** | 1 → voicemail; 2–9 (as the only digit) → speed dial; `*` → `+`; Delete → clear the field | No hardware long-press handling in the AOSP-derived dialer; `*` long press does not give `+` |
| **Digits / `*` / `#` in a call** | Show the DTMF keypad | Same (in-call UI) |
| **Volume** | Volume. With the lid closed and the screen off: wakes the phone and passes the key on. Vol− + Vol+: accessibility shortcut. | Volume. Vol− + Vol+: accessibility. Vol+ + Power: ringer mute (AOSP 13). |
| **Lid close** | **Hangs up a call if no headset is connected.** The main screen goes off and the outer screen takes over ([displays.md](displays.md)). The keypad is marked hidden. | With the stock kernel: nothing (no lid event). With a kernel that reports `SW_LID`: AOSP's default lid behaviour is "do nothing" (closing does not sleep the phone, it only counts as user activity), unless the ROM or the user changes that setting. No call handling. |
| **Lid open** | **Answers a ringing call** (a stock setting, on by default) and wakes the phone. The keypad is marked visible. | With a kernel that reports `SW_LID`: wakes the phone. No call handling. The keypad is always reported as hidden. |

**Stock key behaviour works (own build)** on a ROM built for this phone, not on a plain GSI: the red key ends and rejects calls, any front key wakes the screen, green dials from the dialer, speed dial 2–9, `*` long press gives `+`, Delete long press clears the field, and digits on the home screen open the dialer. Not yet verified in a real call: the speaker-key long press and the red key ending a call with the screen off.

**Stock lid call handling** (closing ends a call, opening answers) is viable on a ROM built for this phone: it worked there with the cover app used earlier, now retired. A plain GSI only gives the AOSP behaviour above.

---

## Keyboard apps (IMEs)

- **Stock IME:** a Kika-based T9/multitap keyboard. Its shortcuts:
  - `#` short = next input mode; `#` long = language/candidates panel;
  - `*` = symbol list;
  - long press on a digit = the literal digit;
  - long Delete = clear the field.
- **Debounce stacks.** An IME's own "accidental key repeat protection" (debounce) is applied **on top of** whatever the kernel already does. On a kernel that debounces properly, an extra IME debounce drops fast intentional repeats of the same key (multitap) and makes typing worse. Set it to **0 ms / off** in that case (regular TT9: *Accidental Key Repeat Protection* → *Off*).
- **IME-level filters cannot fix the stock problem.** A filter inside the IME (TT9's repeat protection, or the optional "multi-press" filter of some T9 apps such as TyNi T9) only drops repeats of the same key inside a time window:
  - it cannot bring back a tap the kernel never reported;
  - it does not act outside text fields (dialer, menus);
  - it can drop real fast repeats.

  The same limit applies to a root userspace filter that grabs the keypad and re-emits events: it removes ghosts in every app but cannot recover lost taps.
- **Upstream TT9 slowness in ABC/multitap (general cause).** Upstream TT9 queries the editor's text on each multitap press and waits for the answer with a timeout. In slow apps (for example the Google search app) those queries time out, and letters get lost, accepted too early, or separated by an invented space. Its debounce is measured at processing time, not at the key's real time. So two presses that were queued behind a stall look 0 ms apart, and the second one is dropped. While hidden, the IME also runs at background priority, so the first key in a newly opened field can lag. These are IME-side issues, separate from the kernel keypad problem.
- **TT9 vs. stock shortcuts.** TT9 matches the stock IME on `#` (short = mode, long = language). In TT9, `*` short is shift (not symbols), and TT9 v64 has no "long Delete = clear".
- **TT9 S22 (public, bundled in the `stock-stable-5` image).** A fork of TT9 v64.0 for this keypad: its own debounce is 0 (it trusts the kernel's), ABC multitap runs in memory, the 800 ms multitap timer runs from the last real press, and the IME threads keep foreground priority while the keyboard is hidden. Measured: 500 injected presses all correct and 0 multitap timeouts, against 1 wrong letter and 23 timeouts for upstream TT9 in the Google app. It needs the rebuilt kernel (on the stock kernel the bouncing comes back). With any other keyboard app, set its own debounce to 0 / off.

---

## Viable / not viable

| Item | Verdict | Why |
|---|---|---|
| Fix doubled letters and lost taps on stock firmware | **Works (public kernel)**, stock-compatible | Only `boot` is replaced: stock config, stock DTB, CAT's signed vendor modules load unchanged; checked on CAT's stock Android 11 (`stock-stable-4`). 248 fast presses on 4/7 → 0 doubled or extra letters, 0 row IRQs idle; ~490 real presses × 62 timing variants → 0 lost, 0 ghosts. CAT's own kernel binary can't be patched by root (driver built in, signed modules only) |
| Fix them by changing only the stock debounce value (DT) | **Viable, partial** | Fewer ghosts, but a wider lost-tap window. One late sample per edge cannot do both. |
| Register light taps on worn domes (1/4/7/\* row) | **Works (public kernel)**, row biasing | The tap is lost electrically (pull-down divider), not by timing. Done in the public kernel; see its README. |
| More column drive current to catch light taps | **Viable, no gain** | Negligible next to a worn dome's contact resistance |
| Polling the keypad at a fixed period instead of IRQs | **Viable**, worse than IRQs | Misses more light taps than edge interrupts and keeps the CPU from sleeping |
| Binary-patching the stock kernel's keypad driver | **Viable**, very large effort | No source for the built-in driver; a bad image is undone by reflashing `boot`. Superseded by the public kernel |
| Userspace / IME filter as the fix | **Viable, partial** | Removes some ghosts; cannot recover taps the kernel never reported |
| Fixing IME-side multitap losses | **Works** (TT9 S22, public) | 500 presses: 0 errors, 0 timeouts, against 1 error + 23 timeouts with upstream TT9 |
| Keypad kernel-log spam (~3,565 lines per 2 s of typing) | **Fixed (public kernel)** | The vendor row "discharge" that gpiolib always refused is removed |
| Dimming the keypad backlight | True dimming: **viable only with a hardware mod**; software PWM from a kernel timer: viable, untested | GPIO 130 is on/off only: no PWM or clock function on the pin. Software PWM costs CPU wakeups |
| Saving keypad-light power | **Works** (own build): off 8 s after the last key | ~28 mA (≈1.4 %/h) while lit; on/off only, so on-time is the only lever. Switching by ambient light: viable, untested |
| Keypad light through Android's lights HAL on the stock vendor | **Viable** by replacing the vendor's lights HAL | The stock `light-qti` exposes no KEYBOARD/BUTTONS light |
| Automatic keypad light on a GSI | **Works (public kernel)** | Kernel-side (the public kernel does it) or a root script |
| Lid detection on a GSI with the stock kernel | **Viable** with a root daemon reading the stock switch node; costs wakeups | The stock driver's lid events are dropped; a third-party build does lid handling in userspace |
| Lid detection on a GSI with a kernel that reports `SW_LID` | **Works (public kernel)** | AOSP reads `SW_LID` natively; done in the public kernel |
| Removing the stock 300 ms busy-wait in the Hall IRQ | **Works (public kernel)** | Deferred work instead of spinning; done in the public kernel |
| Cutting the Hall supply to save power | **Not viable** | `l5` also feeds the eMMC I/O, and the Hall must stay a wake source |
| Stock key behaviour (red ends/rejects, any front key wakes, green dials, speed dial, `*` long = `+`, Delete long clears, digits open the dialer) | **Works** (own build) on a ROM built for this phone; **not** on a plain GSI | Speaker-key long press and red key with the screen off not yet verified in a real call |
| Stock lid call handling (closing ends, opening answers) | **Viable** on a ROM built for this phone (worked with the retired cover app); **not** on a plain GSI | AOSP has no lid call handling |
| Closing apps on lid close | **Works** (own build), lightly tested | Background apps closed with the lid shut; never during a call; 0 wakeups at rest; tested with simulated lid events |
| CAT's private keycodes 300/301 on Android 13+ | Not usable as is; **Works** (own build) remapped to standard keycodes | They collide with keycodes Android 13 assigns to other keys |
