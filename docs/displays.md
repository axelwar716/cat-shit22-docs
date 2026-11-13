# Displays (main and outer)

Scope: the CAT S22 Flip's two screens as shipped (firmware v30): panels, video paths, backlights, how the stock software drives them, known defects, Android rendering facts tied to this hardware, and what is viable or not.

Related: [hardware.md](hardware.md) (chips and buses), [power.md](power.md) (sleep, rails, battery cost), [keypad-lid.md](keypad-lid.md) (Hall sensor, keys), [platform.md](platform.md) (stock firmware, kernel and vendor).

---

## Overview

| | Main (inner) screen | Outer (cover) screen |
|---|---|---|
| Panel controller | Sitronix ST7701S | Named ST7789V2 in the DT; init sequence matches the ST7735S family (see below) |
| Resolution | 480 x 640 | 128 x 128 |
| Interface | MIPI DSI, video mode, 2 lanes, 24 bpp | SPI, RGB565 (32,768 bytes per full frame) |
| Framebuffer | `fb0` | `fb2` |
| Frame memory in the panel | None (host must stream every refresh) | Yes (controller RAM; nothing is sent while the image is static) |
| Refresh | 60 Hz, single mode | 27 fps declared in the DT (nominal only) |
| Backlight | PWM from the PMIC into an external boost | On/off enable line (GPIO), no dimming |
| What Android sees on stock | Primary display | External display named "HDMI Screen" |

---

## Main screen

### Panel and video path

- The active panel is a **Sitronix ST7701S, 480 x 640, DSI video mode, 2 data lanes, 24 bpp, 60 Hz**. The bootloader selects it on the kernel command line (`qcom,mdss_dsi_st7701s_vga_video`).
- The stock DT also carries two alternative 480 x 640 video-mode panels (Galaxycore GC9503V and Jadard JD9161Z), presumably for second-source modules. Only the ST7701S is selected on the units examined.
- Stock timing: 720 x 770 total pixels per frame, pixel clock about 33.26 MHz, DSI PLL VCO about 397-399 MHz.
- Display engine: Qualcomm MDSS (`hw_rev` 0x10100000) with 1 VIG pipe, 2 RGB pipes without scaler, 1 DMA pipe and a cursor pipe. The MDP clock sits at its 50 MHz minimum when idle.
- The ST7701S has **no frame memory** in this mode. Every refresh is streamed from DDR by the MDP and sent over DSI, even when nothing on screen changes. The DSI lanes already drop to low-power (LP-11) during blanking; MDSS does not use ULPS inside video mode.
- Consequence (measured on the hardware): **while the main screen is on, the SoC never releases the XO crystal and never reaches VDD-min**, whatever the content. See [power.md](power.md#3-how-the-platform-sleeps).
- **No partial update reaches the panel.** The panel is video-mode and CAT's display HAL declares partial update "not applicable".
- ESD protection on stock: the DT enables a periodic register read of the panel's power-mode register (`0Ah`, expected `0x9C`); a mismatch triggers panel recovery.
- Power: panel VCI from PM8916 **LDO17** (2.85 V requested, 3.0 V read in hardware; shared with the outer screen and the touchscreen), VDDIO and DSI PHY from **LDO6** (1.8 V). Both stay in high-power mode while the screen is on because of load votes declared in the DT. The DSI PHY regulator runs in LDO mode.
- Rough cost of the video path itself (MDP + DSI + DDR + panel logic, without backlight): in the order of tens of mA. Measurements disagree (one bench run gave about 87 mA, which also included backlight, touch and keypad light; a corrected estimate is about 40-55 mA). Not resolved.

### Refresh modes on stock

- SurfaceFlinger is offered **a single 60 Hz mode**. The stock DT does not declare dynamic refresh (DFPS) for the panel.
- Android's idle behaviour on this phone: with a static screen the hardware vsync is switched off and SurfaceFlinger composes only occasionally (a couple of times per minute, for the status bar clock and similar). A blinking text cursor alone keeps the screen redrawing about twice per second.

### Backlight

- **Path:** PM8916 **LPG channel 1** (PWM, 9-bit, about 9.39 kHz) routed out through **MPP4** in digital (DTEST) mode into an **external boost converter with a single control pin** (enable + PWM). MPP4 is configured by the bootloader and not claimed by Linux.
- The boost IC has not been identified (not named in the DT, the bootloader or public teardowns).
- Android writes `lcd-backlight` 0-255; the panel side maps it onto a 1-4095 range. The PM8916 has no WLED block (the `wled` nodes in the DT are template leftovers).
- Measured: brightness 255 draws about **+76 mA** more than 12/255. At very low brightness the backlight is a small part of the screen-on budget; the panel driver IC and matrix are estimated to draw several times more than the backlight at 12/255.
- The panel's own CABC output does not drive the backlight on this board, and the MDP has no assertive-display block. CAT's vendor ships a content-adaptive backlight library that no binary loads.

---

## Outer screen

### Hardware

- **128 x 128 colour LCD on SPI**, exposed by the kernel as **`fb2`**.
- **Controller identity is uncertain.** The DT node is named `qcom,mdss_spi_st7789v2_qvga_cmd`, but the init sequence it sends (frame-rate commands `B1h`/`B2h`/`B3h` with 3/3/6 parameters, power block `C0h`-`C5h`, `FCh`) matches the **ST7735S** defaults, not the ST7789V2 command set. The silicon ID has never been read. Treat ST7789V2 datasheet values (frame-rate dividers, `C6h`, `BCh`) as wrong for this phone.
- Wiring (from the stock DT):

| Signal | Where |
|---|---|
| SPI | BLSP2 QUP2 (`7af6000.spi`, `spi6`), 50 MHz |
| D/C (data/command) | TLMM GPIO 68 |
| Reset | TLMM GPIO 125 |
| TE (tearing effect) | TLMM GPIO 124, declared but **the panel never sends a TE pulse** |
| Backlight enable | TLMM GPIO 12 (the DT declares 2 mA drive and a pull-down in the sleep pin state; it reads 8 mA, no pull, at runtime) |
| Supply | Shares LDO17 with the main panel and touchscreen |

- **Refresh:** the DT declares 27 fps. This is only the number published to the compositor. With static content there is **no SPI traffic and no panel interrupt** (measured: zero transfers over 12 s; one frame in 139 s with the lid closed). The controller holds the image in its own RAM; frames are only sent when content changes. Its internal scan rate (set by `B1h`) has not been measured.
- Controller current (datasheet values, ST7735S family): about 0.9 mA typical (2 mA max) while scanning, about 0.015 mA in sleep-in.

### Backlight: on/off only

- The outer backlight is driven by `mdss_fb_set_bl_brightness` on `fb2`, which ends in the SPI panel driver toggling **GPIO 12**. The sysfs node is `/sys/class/leds/lcd-backlight2/brightness`.
- It is **binary**: any non-zero value sets GPIO 12 high, 0 sets it low. **4, 80 and 255 give exactly the same light** (verified by driver trace and by eye). With GPIO 12 low there is no residual light at all.
- The 1-255 value is additionally forwarded to an LED trigger (the DT backlight type is `bl_gpio_pulse`) to which nothing is bound, so the level is discarded. The stock kernel contains no backlight controller driver and no PWM consumer for this panel; the PMIC's single PWM channel is used by the main screen.
- The node reads 127 at boot (LED class default) until something writes it.
- Leaving the outer backlight on continuously costs roughly **30 mA** (estimate). That is why the stock firmware times it out.
- **Image retention:** if the backlight goes to 0 but the controller is left powered and scanning, the last image stays on the LCD and can be seen with a torch, and a static image is held indefinitely. To avoid this the panel must be put to sleep (sleep-in, `10h`) when brightness reaches 0. Sleep-in keeps the controller RAM but blanks the display, so it cannot be used to keep a clock readable.

### What the stock firmware does with it

Derived from analysis of the stock v30 kernel, compositor and framework. It has **not been traced on a running stock phone**, so timings are inferred from the binaries.

**Lid closed:**
- CAT's framework forces the outer display **on and at full brightness** while the lid is closed (the main display goes off).
- CAT's SystemUI keeps a presentation (clock, battery, signal, calls, notifications) on the outer display for as long as that display exists, so the outer screen never mirrors the main one on stock.
- After **about 8 s without user activity** the framework applies its dim policy and lets the device sleep, which powers off the outer panel. User activity restarts the window. Some physical keys (notably the orange PTT key) are relayed to the cover-screen UI by a CAT system service.
- When the outer panel is powered off with the lid closed, the stock kernel **does not send the display-off/sleep-in commands and does not reset the panel, but cuts the panel supply**, leaving the backlight on. It then arms an **8 s timer**; when it fires, the backlight goes off and the panel is reset.
- The "dimming" that stock users remember is most likely this: **the LCD image fading as the unpowered panel loses it, with the backlight still on.** It is not an intermediate backlight level (none exists).
- Each blank/unblank of the outer display within those 8 s cancels and re-arms the timer. With a clock refreshing, the faded state can therefore last indefinitely.

**Lid open:**
- A brightness-0 request switches the outer backlight off immediately.
- On the lid-open event the stock kernel cancels any pending timer, switches the backlight off and resets the panel.

**Lid sensor:** the stock Hall driver never emits a Linux lid switch event; CAT's framework reads the lid state through its own path (details in [keypad-lid.md](keypad-lid.md)).

### Stock compositor (SDM) and the outer screen

CAT's display HAL (Qualcomm SDM, legacy fbdev variant, modified by CAT) handles `fb2` as a pluggable external "HDMI" display. In behavioural terms:

- **On non-stock system images it does not create the outer display.** The outer screen never appears in Android there; only the kernel framebuffer exists.
- **It fakes a single HDMI hotplug once the boot animation has finished.** It does not look at the kernel's connection state to decide when. If creating the display fails at that moment, it never retries until the compositor process restarts.
- It requires the kernel's `fb2` to present itself as a connected pluggable panel with a mappable, double-buffered framebuffer. If the driver does not report a frame rate, it defaults to 30 Hz (so Android shows 27 or 30 Hz depending on the kernel; both are nominal).
- **All outer-screen composition is done by the GPU.** The compositor copies the GPU's render target into the framebuffer with the CPU and flips buffers. The render target is RGBA8888 while the panel takes RGB565, so the kernel driver has to convert.
- Backlight: the compositor only ever writes **255 on the first frame after power-on and 0 on power-off.** It has no intermediate levels.
- If Android sees the outer display but nothing draws on it, AOSP assigns it the main display's layer stack and **mirrors the main screen onto it**, sending a frame to the outer screen for every main-screen redraw. Each one is a 32 KB SPI transfer and about 6 ms of CPU (measured).
- With the outer display exposed on a non-stock ROM, enabling a display cutout emulation in Developer options bootlooped the phone (the 128 x 128 display breaks it). Simulated secondary displays and desktop/freeform modes carry a similar risk (unconfirmed). Untested on stock.

### Known defects in the SPI panel driver path

When the kernel is rebuilt from Qualcomm's public 4.9 code against CAT's compositor, the SPI panel driver has the defects below. All of them are **Fixed (public kernel)** ([cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel), section "Outer screen"):
- broken framebuffer `mmap`;
- no RGBA8888 to RGB565 conversion;
- panel not presented as a pluggable ("dtv") panel, so the stock compositor ignores it;
- waiting for a TE pulse the panel never sends;
- `lcd-backlight2` writes not reaching the outer screen;
- a hang after closing the lid caused by a corrupted SPI queue (an SPI message reused while still in flight);
- lid-closed power-off not matching stock.

Measured with the fixes: 35.7 h with the lid closed on battery, ~10.6 mA average (~0.5 %/h, charge counter), 0 SPI errors.

### Outer screen and sleep on non-stock ROMs

- On a plain GSI with the stock vendor, Android does not see the outer screen at all (see the compositor above). Only the kernel framebuffer exists.
- On an Android 13 ROM that does expose it, an outer display that stays on keeps the whole system awake with the lid closed: no screen-off broadcast, Doze never starts and Linux never suspends. **Works** (own build, lid-sleep fix): 2–3 %/h → 0.3–0.9 %/h with the lid closed. See [power.md](power.md#10-lid-closed-drain-by-build).

---

## Android rendering on this phone

- **GPU:** Adreno 308 (the kernel's kgsl driver reports `Adreno306v33`), on the QM215 SoC. Levels 400 / 270 MHz plus a 19.2 MHz idle level. See [hardware.md](hardware.md) and [power.md](power.md).
- **No Vulkan HAL** in the stock vendor. The stock vendor has no Android Power HAL (only Qualcomm's perf HAL), so ADPF hints are unsupported.
- **Hardware video decode** in the stock vendor: **AVC (H.264), HEVC and VP8 only.** VP9 and AV1 (what most web video players serve by default) are decoded on the CPU and stutter. Forcing H.264 in a browser brings back hardware decode (about 20 % less whole-phone current in one video test).
- On the main screen, most frames in real use are composed by the display hardware. Full GPU (client) composition shows up mainly on a static screen, via the vendor's idle fallback.
- **Animations:** with the animator duration scale at 0, AOSP drops edge-stretch effects, ripples and system transitions, and Chromium/WebView reports `prefers-reduced-motion` to web pages.
- **D-pad scrolling still animates** even with animations off. AOSP's `Scroller`/`OverScroller` ignore the animation scale, so each programmatic D-pad scroll step animates for 250 ms. Chromium on Android always animates keyboard scrolling unless told otherwise. On a keypad phone this is the normal way to read.
- **WebView** has no clock of its own: it draws on the host app's RenderThread when invalidated, so it follows the display rate and any per-app frame-rate limit.
- AOSP 13 bug (fixed in Android 14 QPR3): when the vendor enables per-app memory cgroups, WebView renderer processes end up in the `webview_zygote` group, and the app freezer can freeze more than intended (hangs, not extra drain).
- Dark themes do not save power on these LCDs; black pixels draw no appreciably less current.

---

## Viable / not viable

| Item | Verdict | Why (general) |
|---|---|---|
| Dimming the outer backlight (intermediate levels) | Viable only with a hardware mod | Hardware is an on/off enable line (GPIO 12); no PWM or backlight driver is wired to it |
| Software PWM of the outer backlight from the CPU | Viable, defeats sleep | Would keep the CPU awake and defeat sleep |
| Readable outer clock with the controller in sleep-in | **Not viable** | Sleep-in blanks the display (RAM is kept, image is not shown) |
| Saving power by lowering the outer's declared 27 fps | Viable, no gain | No periodic traffic exists with static content |
| Lowering the outer controller's internal scan rate | Viable, untested, negligible gain | Sleep-in with the backlight off already removes most of it |
| Outer screen as an Android display on a non-stock ROM | **Works** (own build); unused since the cover app was retired | The stock compositor skips it on non-stock system images; it also needs a kernel driver that meets the compositor's contract (see the public kernel README). The kernel clock now drives the outer screen |
| Kernel-drawn lid clock, no app, any ROM | **Works (public kernel)** | Redrawn every minute by a wake alarm, also in deep sleep (see its README) |
| Lid clock local time | **Works (public kernel)** on Android 13; wrong on Android 15+ | The kernel takes the time zone Android sets, with EU/US DST until then; Android 15+ never sets it, so the clock shows UTC + the DST hour |
| Outer screen held off with the lid open (no mirroring) | **Works (public kernel)** | No power vote, backlight 0; frames are accepted but never sent over SPI |
| Outer screen at rest | **Works** (own build) | Part of the 2.3 → 0.6 %/h step ([power.md](power.md#10-lid-closed-drain-by-build)); image retention while dark fixed, no drain change (0.6 %/h) |
| Cover app on the outer screen (clock, notifications, music, calls, orange key, answer/hang up with the lid) | **Works** (own build), retired for the kernel clock | Its always-on outer backlight cost ~27–31 mA overnight (2.3 vs 0.6 %/h) |
| Deep sleep with the lid closed when the outer display is exposed (AOSP 13) | **Works** (own build) | Lid-sleep fix: ~2–3 %/h → 0.3–0.9 %/h with the lid closed. Without it Android stays interactive while the outer display is on |
| Main screen at a fixed 20 Hz | **Works** (own build) | Same energy as 30 Hz within 1 %, smoother (20.0 vs 15.2 fps, jank 7.9 vs 11.5 %). Against stock 60 Hz, 30 Hz alone saved ~47 mA in use (~2.4 %/h) |
| Main screen below 20 Hz | Viable, flicker risk, small gain | Estimated 0.9–2.2 mA. Flicker was measured at 20–22 Hz with some timings (a-Si panel without frame memory); the stock display driver caps the rate |
| Panel self-refresh / partial update on the main screen | **Not viable** without a different panel | Video-mode panel without frame memory; no drop-in 2.8" MIPI panel with memory is available |
| Constant-current main backlight | Viable only with a hardware mod | The control pin feeds the boost's logic input; MPP4 has no analog output |
| Content-adaptive backlight (panel CABC, MDP CABL) | Panel CABC: **not viable**; MDP CABL: viable, no gain | Panel CABC output is not wired to the backlight; negligible at low brightness |
| Panel 8-colour idle mode | Viable only with the panel ESD check adapted or off (loses ESD recovery); unusable (8 colours) | With the stock ESD check it resets the panel in a loop. ≤1–2 mA (estimate) |
| Single DSI lane, 16 bpp link, ULPS in video mode | Lanes and ULPS: **not viable**; 16 bpp: viable, no gain | Lane count is fixed by hardware; no DDR saving; MDSS does not do ULPS in video mode |
| Clocking the MDP from XO to free GPLL0 | Viable, untested | Only at 20 Hz with no frames going to the outer screen; ~0.1–1.3 mA (estimate) |
| Fewer frames drawn, smoother web video | **Works** (own build) | −47 % frames opening and closing apps; web video 583 vs 732 mA in one test (−20 %) |
| Hardware decode of VP9/AV1 | **Not viable** | The video hardware decodes AVC, HEVC and VP8 only |
| Vulkan rendering | **Not viable** | No Vulkan HAL in the stock vendor, and no Vulkan driver exists for this Adreno 3xx-class (OpenGL ES 3.0-level) GPU |
| Saving power with dark mode | Viable, no gain | LCD, not OLED |
