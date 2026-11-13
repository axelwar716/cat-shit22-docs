# Power, battery and sleep

Scope: the battery cell and fuel gauge, how the stock platform sleeps and wakes, what keeps the SoC awake on stock and on AOSP-based ROMs, modem and Wi‑Fi idle behaviour, CPU/GPU/DDR power, the hardware energy floor, and which power ideas are viable or not.

Related pages: display and backlight power in [displays.md](displays.md); Bluetooth music, audio and NFC power in [audio-bt-nfc.md](audio-bt-nfc.md); radio and IMS in [telephony.md](telephony.md); chips, rails and GPIOs in [hardware.md](hardware.md); kernel, vendor and Android layers in [platform.md](platform.md). Reference battery results to compare builds against: [section 11](#11-reference-results-for-comparison).

All mA figures below are hardware, datasheet or stock-platform figures. The phone has no current sensor (see below), so most absolute currents are estimates and are marked as such.

---

## 1. Battery cell and charger

| Item | Value | Notes |
|---|---|---|
| Capacity | 2000 mAh | Rated (BTE-2000, 3.8 V, 7.6 Wh, 2050 mAh typical). The stock profile and the gauge use 2000 mAh, and every mA and %/h figure in this repo is on that base (1 mA = 0.05 %/h). Android's battery stats print "Estimated 1000 mAh", which is a framework default, not a measurement. |
| Cell profile in the stock DT | Q2805 | OCV table, resistance and accumulator tables used by the gauge. The profile's battery resistance is a fixed 222 mΩ (plus a 50 mΩ "capacitive" term); the gauge never learns it. |
| Charger | PM8916 internal linear charger (`qpnp-linear-charger`) | Linear, so input current ≈ battery current + system current. On a plain USB port (SDP) it is limited to 500 mA. The thermal-mitigation levels map to fixed charge currents (1400 / 1100 / 600 / 500 / 0 mA). In the "warm" temperature regime the stock DT charges to 4.20 V at 1000 mA. |
| Empty threshold | ~3.4 V | Lowering it would add only ~1.4 % of capacity. |

Charger behaviour worth knowing:

- **Turning charging off in software does not cut the input.** With charging disabled the charger keeps powering the system from USB (the charge counter stays frozen while the phone runs). The charger driver exposes no "input suspend" control on either the battery or the USB supply. Any battery measurement must be made with the cable physically unplugged.
- **With USB connected the phone never deep-sleeps.** The USB controller holds the 19.2 MHz XO, its bus vote pins the DDR at 595.2 MHz, and the USB LDOs (L7, L13) stay in high-power mode. Bus, rail and current readings taken on a cable are not representative of battery operation. The developer option "Stay awake while charging" adds a second reason not to sleep.
- **Battery temperature is suspiciously constant.** About 98 % of ~1,150 readings fell between 25.0 and 25.9 °C while the XO and PMIC sensors read 31–37 °C, and it reads lower while charging than while discharging. Inferred (strong): the thermistor line sees a fixed resistor; confirming it needs the line's resistance measured. If so, the temperature-based charge protections and the gauge's temperature compensation never act.
- The `skin-therm-adc` thermal zone returns nonsense (0.9 to 30.8 °C with a stable XO temperature). Do not base any policy on it.
- The cycle counter is not a health indicator. It counts very slowly and the driver resets it whenever the shutdown state-of-charge it saved is invalid, which happens after every unclean reboot or flash.

## 2. The fuel gauge: how it reports and where it lies

The S22 Flip uses the PM8916 **VM-BMS** ("voltage-mode" battery monitoring), driven by the kernel driver `qpnp-vm-bms` and a closed vendor daemon, `vm_bms`. **There is no current sensor anywhere: no coulomb counter, no shunt, no per-rail metering.** Everything except the voltage comes from a model.

### What each battery value actually is

| Value | What it is | Is it a measurement? |
|---|---|---|
| Voltage FIFO | Hardware average of VBAT: 128 samples × 70 ms ≈ 9 s, delivered in batches of 5 roughly every 50 s while the SoC is awake. Empty during suspend. | Yes (voltage) |
| Sleep voltage accumulator | Hardware average of VBAT over the whole last sleep, published once per resume with ~0.3 mV resolution. | Yes. The best rest signal the phone has. |
| `current_now` | Written by the closed daemon as (modelled OCV − last FIFO voltage) / ~0.27 Ω, the profile's 222 mΩ plus its 50 mΩ capacitive term. It reproduces exactly, with zero offset. | **No.** It is rescaled voltage. |
| `voltage_ocv` | The daemon's internal OCV estimate. It integrates its own `current_now`, with a time constant of ~30 min above 60 %, 2–3 h in the 20–45 % plateau, and it is frozen during suspend. | No |
| `capacity` (%) | State of charge of that model. | No |
| `charge_counter` | Full-charge capacity × state of charge, i.e. the gauge percentage in another unit (steps of about 20 mAh). | **No.** It is not a coulomb counter and inherits every bias of the %. |
| `resistance` | Fixed profile value (222 mΩ), never learned. | No |

The VM-BMS does not wake the SoC during suspend: on suspend the driver forces its low-power mode and disables its FIFO interrupt. It does a small amount of work on every resume.

### Where it is unreliable

- **It does not see idle drain.** Because the model's OCV is frozen while the phone sleeps, the percentage and `charge_counter` underestimate idle consumption by roughly 1.4× to 2.4×, then catch up in jumps. During long idle periods `current_now` can climb for no physical reason (one night it went from 137 to 737 mA while the phone was asleep).
- **It sticks at the top.** Starting from 100 % ("Full"), the percentage and `charge_counter` stay pinned for a while, then recalibrate in a jump (for example 99 → 92 % in 38 minutes). For a clean figure, start below ~90 % or discard the first hour after unplugging.
- **It stalls and then jumps at the bottom.** Below about 20 % (≈400 mAh) the gauge holds still and then drops in one go (one example: 20 → 12 % in 12 minutes with the screen off at ~3.69 V, which is impossible at the real drain). Readings below ~20 % are not usable for current estimates.
- **It is optimistic in the low zone.** The OCV table is almost flat between ~20 and 45 % (1.7–4 mV per %), so small voltage errors become large % errors. Examples: the gauge showed 31 % when the rest voltage corresponded to ~10 %; another run ended at 89 % shown versus ~74 % by voltage. Runtime estimates based on % in the lower half are not reliable.
- **Use %/h, not mA.** A %/h figure read from the charge counter does not depend on capacity; mA figures are that %/h on the rated 2000 mAh. Comparisons made with the same method on the same phone are valid.
- **After charging, the voltage relaxes.** More than 85–90 % of the voltage step after a load change happens in under 30 s, but after a full charge the relaxation biases the next ~1.5 h of rest readings upward by a few mA.
- **Android's per-app battery figures are meaningless.** The device ships no real power-profile values, so battery stats in mAh (screen, radio, apps) are fictional. Their time counters (awake time, radio active time, wakelocks) are still useful.

Kernel builders: the Qualcomm 4.9 base tree compiles a 4-row limit for one of the battery lookup tables, while the stock Q2805 profile has 6 rows. With that limit the VM-BMS fails to probe, the battery stays at a fake 50 % (the linear charger's fallback) and `current_now` reads 0. This is the `ACC_IBAT_ROWS` 4 → 6 change already documented in the public [kernel README](https://github.com/axelwar716/cat-shit22-kernel).

## 3. How the platform sleeps

- **Sleep states.** The QM215 can suspend Linux and reach **VDD_MIN** (XO shut down, rails at retention, DRAM (likely LPDDR3) in self-refresh). With the lid closed, no cable and software that lets Linux suspend, the SoC spends roughly 84–91 % of the time in VDD_MIN in the measured nights (91 % in the 2026-10-18 night). The RPM (resource power manager) is not the limiting factor: it enters VDD_MIN as soon as every master releases its votes.
- **Stock boot.** The stock command line boots with low-power modes disabled; a stock boot script re-enables them later. By the time Android is up, CPU idle states work normally.
- **What stays powered in sleep.** The Wi‑Fi, USB and eMMC supply rails turn off (the eMMC VCC rail, L8, is cut about 84 % of the time at rest). The RPM keeps S3, S4, L2, L5 and L7 on. The I/O rail L5 also feeds the Hall sensor and the eMMC I/O, so it cannot be cut. The display, CPU and audio-codec rails stay up.
- **Screen on.** With the main screen on, every PM8916 buck (S1–S4) runs in PWM and every enabled LDO is in high-power mode. The reason: the 4.9 regulator core adds up the static loads that devices declare in the DT, whether or not those devices are active. For example, the analog audio codec declares 550 mA on the 2.05 V buck even when silent. The AP never releases the XO while the main panel is on, because the panel works in video mode and the DSI/MDP path needs the clock (see [displays.md](displays.md)).
- **PM8916 GPIO3** enables an external 1.2 V LDO that the board needs. Driving it low resets the phone instantly.
- **Signed firmware.** RPM, TrustZone, modem, Wi‑Fi/BT (PRONTO) and the DSPs run production-signed firmware, with secure boot, anti-rollback and debug disable fused. Their power behaviour can only be influenced through the votes and requests the Linux side sends them.

### Cost of one wakeup

A full suspend/resume cycle keeps the SoC awake for roughly **0.29 s, almost independent of what triggered it** (Wi‑Fi packet or RTC alarm). Measured on a 4.9 kernel built to match stock, with stock-equivalent settings:

| Phase | Median | What it is |
|---|---|---|
| Suspend path | ~106 ms | File-system `sync` (~52 ms), task freezing, device callbacks, taking CPUs 1–3 offline |
| Resume path | ~81 ms | Bringing the CPUs back, device callbacks, and ~38 ms of post-suspend notifiers, mostly the thermal core re-evaluating 23 of the 24 thermal zones (including several PMIC ADC reads) |
| AOSP suspend service wait | ~102 ms | Fixed base delay of Android's suspend service before it tries again |

At typical awake currents this is about **4–8 µAh (≈0.06–0.11 J) per wakeup** (estimate). The kernel's 2-second "alarm too close" hold (`alarmtimer`) adds a fixed 2 s awake whenever an alarm is due less than 2 s after suspend starts, but it is rare.

Once wakeups are down to a few per minute, they make up only about 5–20 % of idle drain. **At rest, the fixed floor dominates, not the wakeups** (see section 9).

## 4. What keeps the SoC awake (stock kernel, stock vendor, AOSP)

These come from the stock kernel, the stock vendor or AOSP itself, so they affect stock and AOSP-based ROMs alike unless a ROM changes them.

**Per-resume overhead from the kernel and early userspace**

- **Thermal uevent storm.** On every resume the thermal core re-evaluates 23 of the 24 thermal zones. The 14 zones using the `user_space` governor (10 TSENS "-usr" zones and 4 ADC zones) each emit a uevent, and bringing CPUs 1–3 back online emits 3 more: 17 per resume by code. About 30 per cycle were counted; the rest are unattributed.
- **`ueventd` relabelling.** After cold boot, AOSP's `ueventd` runs a recursive SELinux `restorecon` on the sysfs path of every `change` and `online` uevent, which costs ~5.5 ms of CPU per event. With the storm above, `ueventd` was the largest CPU consumer per wakeup cycle.
- **Every uevent wakes the vendor health HAL.** It listens for uevents with a wake-up epoll flag, so each one also briefly holds off suspend.
- **AlarmManager time-change timer** (AOSP, examined on Android 13). The 4.9 kernel signals a "clock was set" event on every resume. AlarmManager's time-change timerfd is registered with a wake-up flag, so it adds 2 wake events per resume, each holding off suspend for ~25 ms.
- **Unknown wake reasons.** The stock kernel never reports the IRQ that woke it to the wake-reason log, so Android's battery stats always record "unknown". The resume IRQ is only visible in the kernel log with a debug mask set.

**Periodic sources**

- **Health HAL alarm.** The vendor health HAL wakes the phone every 10 min on battery (every 60 s on a charger) to refresh the battery state. Cost ≈0.03–0.05 mA (estimate). It is also what triggers low-battery shutdown, so it should stay.
- **Kernel housekeeping while awake.** The Qualcomm `rq_stats` module sends a sysfs notification every 50 ms that nothing listens to, and per-cgroup PSI averaging runs as non-deferrable work (6.5 wakeups/s with ~170 cgroups). Both add CPU-cluster wakeups whenever the SoC is awake. About 27–29 cluster wakeups per second with the screen off and Android awake have no visible cause (the tick) and remain unexplained.
- **eMMC PM QoS.** The stock DT applies the eMMC's PM QoS request to all four cores, so each I/O request blocks core and cluster power collapse for its duration plus 10–20 ms. The effect is small (estimated ≤1 mA, only during active use).
- **Wi‑Fi verbose logging.** When it's on, two Wi‑Fi framework threads wake about 2.7 times per second. On Android 13 `settings get global` returns `null` for it even when it's on; the real state is in `dumpsys wifi`.
- **Launch boost.** The stock vendor's app-launch boost disables CPU power collapse for its duration (by design of the vendor performance configuration).

**Radios**

- **Wi‑Fi RX wakelock.** Every received packet that reaches the host takes a 50 ms wakelock in the stock `prima` driver. Usually it ends before the suspend service's own wait, so it does not lengthen the cycle. Packets that arrive while suspend is starting cause aborts: each abort pays for the `sync` and freezing without ever sleeping.
- **Modem indications.** While Android considers a screen on, telephony keeps every modem indication enabled (signal strength and so on), and the modem wakes the AP every few seconds. In one capture without a SIM it woke the AP every 5.12 or 10.24 s exactly, which matches standard UMTS (3G) DRX cycle lengths exactly (inferred, strong; without a SIM it camped on UMTS, see [telephony.md](telephony.md#bands)). Once Android goes non-interactive, telephony filters those indications and modem-driven wakeups almost disappear.

**Android framework (AOSP 12 and later) while it believes a screen is on**

None of these run once Android is truly non-interactive, which is why the lid-closed problem in section 5 matters so much:

- no Doze and no deferral of non-wakeup alarms;
- Wi‑Fi stays out of the driver's suspend mode (no broadcast/multicast filtering), keeps polling RSSI every 3 s and keeps doing connected scans;
- the link bandwidth estimator polls traffic stats every second and the modem every few seconds;
- `FaceDownDetector` keeps the accelerometer running at 5 Hz (batched every 2 s);
- BLE scanning stays in screen-on mode;
- the Power HAL stays in "interactive" mode.

Two more sensor wakers: on the lock screen, SystemUI's proximity listener wakes the sensor DSP about 44 times per second (zero once unlocked), and location's stationary throttling keeps the significant-motion sensor armed.

## 5. Why Lineage and GSIs don't deep-sleep with the lid closed

There are two separate causes, and both come from the layers above the hardware.

**1. Android never learns that the lid closed.** The stock Hall driver declares the standard Linux lid switch but never emits it. Stock CAT software reads the lid through its own path (a sysfs switch where 1 means open, plus CAT's framework code). On the stock kernel, an AOSP framework or a GSI therefore never gets a lid event. AOSP's default lid behaviour is also "do nothing". So closing the flip does not put an AOSP build to sleep by itself. A kernel that emits the standard lid switch fixes this part; the public kernel already does, as described in its README. See [hardware.md](hardware.md) for the Hall sensor.

**2. With the outer screen exposed, Android stays awake.** On non-stock system images the stock vendor compositor does not bring up the outer screen, so on a plain GSI Android does not see it at all. On a ROM or cover app that does expose it, closing the lid with the outer display on leaves the device **awake and interactive** while the main display is off:

- no `SCREEN_OFF` broadcast is sent, Doze never starts, and every screen-on behaviour in section 4 keeps running;
- Linux never even attempts to suspend: `suspend_stats` shows 0 successes and 0 failures. The CPU cluster and RPM still use their idle states between interrupts (VDD_MIN counts go up), but the system as a whole never suspends.

Switching the outer screen off together with the main one avoids it, but defeats the purpose of a cover screen.

On stock Android 11, CAT's own framework handles the closed lid. With the lid closed it forces the outer screen on at full brightness and, after about 8 s without activity, applies its dim policy and puts the system to sleep, which also turns the outer panel off. This is inferred from the stock software and was not traced on a running stock phone.

**Works (own build)** (the "lid-sleep fix"). The outer-screen clock can be drawn by the kernel without Android being awake, as the public kernel already does.

## 6. Modem idle (LTE, IMS)

Facts below are modem-firmware and network behaviour on a European LTE network, read from the modem's own diagnostics and RPM counters.

- **The network sets the paging cycle: 640 ms.** The phone can ask for a shorter cycle but never a longer one. The modem's XO-off/on rhythm matches it (measured period 591–617 ms).
- **The cell matters more than anything else.** On a strong mid-priority band (B1 at about −95 dBm) the modem is awake 33–37 ms per paging cycle (~94 % XO off). On the high-priority band at its coverage edge (B3 at about −116 dBm) it is awake 48–54 ms per cycle (~91 % XO off). Those ~3 extra points of modem duty cycle are estimated at 2–3 mA (estimate from the measured duty cycle; nothing on the phone can resolve 2 mA).
- **Reselection priorities can park the phone at the edge of a weak band.** Operator reselection rules that favour a high-priority band at low thresholds can keep the phone camped at that band's coverage edge, oscillating between bands (one sequence: B3 → B20 → B1 → B3 in ~35 min) and measuring all configured neighbour frequencies all the time. Excluding bands on the phone side would be viable, but the expected gain is below what the gauge can resolve in a night.
- **IMS in idle is cheap.** IMS re-registers about every 2 h (the network grants about 7,200 s whatever the phone asks for) and the periodic tracking-area update runs every ~54 min. Together they are estimated at about 0.3 mA. No RRC connections were seen in 14 minutes of idle. IMS registration is not what raises idle drain.
- **A registered SIM costs a few mA over airplane mode** (paging plus mandatory neighbour measurements, fixed as long as a SIM is registered). The exact figure is unresolved: modem counters suggest more than the gauge shows.
- **Removing 3G changes nothing in idle.** Same cell, with and without UMTS allowed: 94.5 vs 94.4 % XO off on B1, and 91.2–91.3 vs 91.4–92.2 % on B3. That is noise.
- **"LTE only" saves nothing measurable.** In a real overnight test the phone never left LTE. Drops to 2G cluster in the first minutes after boot (partly while waiting for the PIN). Toggling airplane mode returns to LTE in about a second without passing through 2G.
- **"Camp on 2G, use 4G for calls" is not possible.** A call arrives on the network the phone is camped on, and LTE and GSM idle cost the same order of magnitude (~5–10 mW, ≈1.3–2.6 mA).
- **Mobile data off saves nothing while Wi‑Fi is connected.** With "mobile data always active" disabled, Android drops the cellular internet data connection when Wi‑Fi is up, leaving only the IMS data connection (0–208 bytes of traffic in the test blocks). Turning mobile data off at lid close only removes background traffic; the pure idle saving is ≈0.
- **Connected mode is also set by the network.** Connected-mode DRX (10 ms on-duration, 100 ms inactivity, 320 ms long cycle) and the RRC release timer (~7 s of inactivity) are network settings and already tight. The phone is LTE category 4 with no carrier aggregation, and it does not advertise the power-preference indication, so it cannot ask for a more power-saving DRX.
- **In active LTE use, connected time costs more than transmit power.** Over 16 h, transmit time was 16.6 min (3.2 min above 15 dBm), while the radio was RRC-connected for ~3 h 48 min. The browser alone held 2 h 59 min of radio-active time. The way to save is fewer and shorter bursts of traffic, not lower TX power.

More on IMS, VoLTE and the RIL in [telephony.md](telephony.md).

## 7. Wi‑Fi idle, and Wi‑Fi vs LTE

The Wi‑Fi/BT/FM radio is the PRONTO core inside the SoC (WCN36xx family), driven by CAT's signed `prima` module (`pronto_wlan.ko`).

- **Stock configuration** enables idle power save (IMPS/BMPS), ARP offload, modulated DTIM 3 and multicast address-list filtering. In driver suspend mode the radio listens to every 3rd DTIM. Beacon wakeups line up on a 3-beacon grid (368 of 371 in one capture), which confirms the listen interval of 3. With the screen on, it listens to every beacon.
- **Beacon listening is the Wi‑Fi idle floor.** Associated and idle, the chip wakes ~3 times per second for ~30 ms to hear beacons. That is radio activity outside VDD_MIN, not AP wakeups, and is estimated at roughly 1.4–3.8 mA. The firmware forwards a beacon to the host only when a monitored information element has changed.
- **The driver's suspend mode filters broadcast.** In a 50-packet UDP broadcast test, 35–41 packets reached the host with suspend mode off and 0 with it on. The framework only turns this mode on when the screen is off, which ties back to section 5.
- **ADB over Wi‑Fi keeps waking the phone,** about once per second. In one capture ADB packets were 91 % of the packets that woke it. Power figures taken with ADB over Wi‑Fi connected are not representative.
- **Multicast you subscribe to gets through.** The stock firmware passes subscribed multicast groups. If the mDNS daemon is running (ADB over Wi‑Fi starts it), mDNS traffic causes about half of all Wi‑Fi wakeups.
- **APF (Android Packet Filter) is not available.** APF would have to run inside the signed Pronto Wi‑Fi firmware, and CAT's `prima` driver has no APF command (only the legacy packet filter), so framework-side packet filtering cannot move into the firmware.
- **Without a WoWLAN configuration, every suspend drops Wi‑Fi.** If the wireless stack has no WoWLAN configuration when the system suspends (seen after a framework restart re-created the interface), the kernel disconnects from the AP on every suspend, and Android does not reconnect while the screen is off.
- **Wi‑Fi off.** The vendor Wi‑Fi HAL does not unload the driver module when Wi‑Fi is turned off. The chip stays in IMPS and remains alive for Bluetooth and FM. Whether it fully releases the XO in that state is unconfirmed.
- **Wi‑Fi vs LTE in active use.** On this phone the choice of network does not change active-use consumption measurably: differences between arms were within block-to-block noise in controlled A/B tests. Longer uncontrolled runs hint at a small advantage for Wi‑Fi, but that is not conclusive. At rest, keeping Wi‑Fi associated adds the beacon floor above; avoiding that floor trades off push notifications while the lid is closed.

## 8. CPU, GPU, DDR, CPR

Hardware context: QM215, 28 nm, 4× Cortex-A53 in one cluster on their own buck (PM8916 S2, 1.05–1.35 V) with closed-loop CPR (Core Power Reduction). GPU is an Adreno 308-class part. CX and MX (logic and memory domains, shared with the GPU, display, bus and modem) are arbitrated by the RPM.

- **Frequency tables depend on the fuse bin.** The unit studied is fused to CPU speed bin 2, the shortest table: 960 / 1094.4 / 1209.6 MHz. The GPU is in the lowest bin: 400 / 270 MHz plus the 19.2 MHz idle level. "Only two GPU levels" comes from the fuse, not from software. Other units may be fused differently.
- **960 MHz is the lowest CPU operating point.** The PLL can go lower, but a lower point would vote the same voltage floor, so the saving would scale with frequency only, not with voltage squared, and it can cost more energy per task. Viable, no gain.
- **Real CPU load is small.** Measured: lid closed 0.4–0.7 %, main screen idle ~1.2 %, scrolling ~15 %, browser loop 17–20 % (averaged over 4 cores). Anything that only touches CPU core voltage has little to work with.
- **In active use, the voltage that matters is the shared CX/DDR domain, not the CPU rail.** Higher CPU states pull the bus and CX up with them. This is why frequency caps can pay off a little in active use and CPU undervolting cannot (see the table).
- **GPU undervolt from Linux is not possible.** The GPU votes a CX corner to the RPM; there is no GPU voltage under Linux control.
- **DDR and the displays.** With the main screen static, the display engine asks for less bandwidth than the lowest DDR step, and the outer screen only votes bandwidth during an SPI transfer. The display does not keep the DDR fast. What raises DDR with the CPU awake is CPU-side activity and, on a cable, USB.
- **CPR margin exists but is worth little.** CPR already adapts the voltage per chip in closed loop and keeps compensating for temperature. What is left above the floor is a small foundry and control-loop margin. Estimated value: about 1 % at rest and with the lid closed, a few percent under heavy browsing. That is below what the gauge can resolve, and undervolting carries stability risk. With the lid closed and the SoC suspended, CPU voltage is irrelevant.
- **Load and idle tuning.** Parking cores (`core_ctl`), timer slack and LPM prediction bring marginal or no gains. The 4.9 kernel has no `s2idle` for `mem`, so the per-suspend CPU hotplug cost (~10 ms) can't be avoided.
- **The "load average" of ~4–5 at idle is cosmetic.** Four kernel threads (DSI event, eMMC command queue, two framebuffer threads) sleep uninterruptibly and Linux counts them as load. They use no CPU.

## 9. Energy floor and ceilings (hardware)

**Lid closed, Linux suspended, outer screen dark, Wi‑Fi off.** Component budget (estimates from datasheets and measured residency, below what the gauge resolves; datasheet figures marked):

| Component | State | mA |
|---|---|---|
| SoC in VDD_MIN + DRAM (likely LPDDR3) self-refresh + PMIC/RPM | 84–91 % of the time (measured) | 3–6 (estimate) |
| Outer LCD controller (ST7735S family) | Powered and scanning, backlight off | 0.9 typical, 2 max (datasheet). 0.015 in sleep-in |
| Outer panel LDOs (L6, L17) held in high-power mode by declared loads | — | 0.05–0.3 (estimate) |
| eMMC | Sleep, VCC cut, VCCQ low-power | 0.05–0.2 (estimate) |
| Touch controller (CHSC5448) | Deep sleep | ~0.01 (datasheet) |
| Speaker amp (AW88194) | Software power-down | 0.001–0.05 (estimate) |
| USB-C CC controller (WUSB3801) | Dual-role toggling | 0.02–0.1 (estimate) |
| Hall IC | Active | 0.005–0.02 (estimate) |
| Remaining wakeups (health HAL ~5/h) | — | 0.02–0.05 |
| Registered SIM: LTE paging + neighbour measurements | — | a few mA, depending on cell and band (section 6) |
| Wi‑Fi associated (if left on) | Beacon listening | ~1.4–3.8 (estimate) |

- Outside the SoC, the largest controllable item in that floor is an outer panel left scanning in the dark. The rest is in the µA range or inside signed firmware.
- Once Linux suspends properly, the AP is asleep almost all of the time with the lid closed, and what remains is the modem/cell and the fixed floor. At that point the software ceiling is essentially reached.
- **Screen on.** The bulk is the video-mode display path and the backlights, which is fixed hardware (see [displays.md](displays.md)). Everything the AP can change on the rails adds up to ~1.5–5 mA at best (estimate), below what the gauge resolves.
- **A newer kernel, a newer Android or a 64-bit userspace would not lower the floor.** The floor is set by the LCDs, the radio, the PMIC and the battery, and the closed vendor blobs fix the rest of the stack. There is no measured or code-based reason to expect lower drain from a version change. The 64-bit options are compared in [platform.md](platform.md#64-bit).
- Bluetooth audio playback is a separate regime: the BT/Wi‑Fi chip never releases the XO while streaming, so there is no deep sleep during music. See [audio-bt-nfc.md](audio-bt-nfc.md).

## 10. Lid-closed drain by build

What each relevant change achieved with the lid closed, unplugged. %/h counts only the time the lid was shut; mA is the same figure on the rated 2000 mAh (1 mA = 0.05 %/h). Where several changes went in together, the gain is not split between them. Rows measured by voltage and by charge counter are not directly comparable: the counter under-reads idle drain.

| Date | Build / change | Conditions | Lid-closed drain | Method |
|---|---|---|---|---|
| 2026-10-04 | Own build before the lid-sleep fix, ADB over Wi‑Fi | Outer display exposed, Android interactive, never suspends | ~2–3 %/h | Awake-time counters (estimate) |
| 2026-10-04 | **Lid-sleep fix**, no ADB | Outer clock shown, Android non-interactive; Doze after 2 min, awake 3.4–3.8 % | **0.3–0.9 %/h** | Awake-time counters (estimate) |
| 2026-10-12 | Public kernel (stock-stable-4) on the own ROM | 35.7 h, awake 3.4 % | ~0.5 %/h (10.6 mA); by voltage likely 13–25 mA (≈0.65–1.25 %/h, estimate) | Charge counter; lower bound, the gauge under-reads idle |
| 2026-10-16 | Own build, outer backlight left on all night | Wi‑Fi on | 2.3 %/h (45.4 mA) | Voltage |
| 2026-10-17 | **Outer backlight off after 8 s (public kernel) plus lid-closed idle tuning** (several changes, not split) | Airplane mode, no SIM; wakeups ÷78 | **0.6 %/h (11.7 mA)** | Voltage |
| 2026-10-18 | + outer-screen image-retention fix | Airplane mode, no SIM; 91 % of the time in VDD_MIN | 0.6 %/h (12.0 mA), no change within noise | Voltage |
| 2026-10-19 | Same, with SIM | LTE + IMS, Bluetooth on, music-app notifications over LTE | 0.95 %/h | Charge counter, 506 min |
| 2026-10-20 | Same, with SIM | LTE + IMS only, weak band | **0.5–0.7 %/h** | Charge counter, 6 h 33 min |

- **Current working figure: ~0.6 %/h with a SIM on LTE + IMS, about 7 days with the lid closed.**
- Bluetooth music with the lid closed is a separate regime (the radio chip holds the XO): ~10.9 %/h over a full test on LTE; in the same test, offline playback of downloaded music with no data traffic ~7–9 %/h. See [audio-bt-nfc.md](audio-bt-nfc.md).
- Screen-on changes (frequency caps, 20 Hz, keypad-light timeout) are not lid-closed changes and are not listed. Active-use results are in section 11.

## 11. Reference results for comparison

A reproducible baseline for anyone building their own kernel, vendor or ROM for this phone: what the own build achieves today, measured with the phone's own gauge, and exactly how, so another build can be measured the same way and compared. Builds are named by date; "current" is the build in daily use since 2026-10-22.

### Headline (current build)

- **Lid closed (standby), SIM on LTE + IMS: ~0.6 %/h (≈12 mA), about 7 days.** Measured 0.5–0.7 %/h on 2026-10-20; 0.95 %/h (≈19 mA, about 4.4 days) with Bluetooth on and a music app polling over LTE. Changes made after 2026-10-20 target open-lid use or cost nothing at rest; lid-closed drain was not re-measured.
- **Continuous synthetic use, lid open, screen on: 187 mA (LTE only) and 194 mA (Wi‑Fi + LTE) ≈ 8.5–9.7 %/h in active use, depending on network. 100 → 20 % took 8 h 26 min.** Measured on the night of 2026-10-22.
- **Lid open, screen off: ≈0.6 %/h**, the same floor as lid closed (low resolution, see below).
- **Bluetooth music with the lid closed: ~10.9 %/h on LTE; ~7–9 %/h (~150 mA) offline, with downloaded music and no data traffic** (both from the same test, 2026-10-19).

### Lid closed (standby)

%/h counts only the time the lid was shut, from the charge counter.

| Date | Build | Conditions | Duration | Result | Evidence |
|---|---|---|---|---|---|
| 2026-10-20 | 2026-10-18 | SIM, LTE + IMS, current lid-closed tuning, weak high-priority band at its coverage edge | 6 h 33 min | **0.5–0.7 %/h** (≈10–14 mA) | Measured. Started from "Full", 1 % resolution |
| 2026-10-19 | 2026-10-18 | SIM, LTE + IMS, Bluetooth on, not playing, music app installed and polling over LTE about every 10 min | 506 min | 0.95 %/h (≈19 mA) | Measured. ~4 mA of it attributed to the music app (Hypothesis, battery-stats model) |
| 2026-10-18 | 2026-10-18, before VoLTE | No SIM, airplane mode, 91 % of the time in VDD_MIN | 11.4 h | 0.6 %/h (12.0 mA) | Measured, by rest voltage (not charge counter) |

Longer history by change is in [section 10](#10-lid-closed-drain-by-build).

### Active use (synthetic workload, lid open)

Unplugged at 100 %, workload running continuously until the gauge reached 20 %. mA per band = 200 mAh divided by the time the charge counter took to fall 200 mAh; the four bands of each half are averaged. %/h is the logger's own summary per state over the same time.

| Night | Build | First half (100 → 60 %): network, mA per 200 mAh band → mean | Second half (60 → 20 %): network, mA per band → mean | Logger summary | 100 → 20 % | Evidence |
|---|---|---|---|---|---|---|
| 2026-10-22 → 23 | **Current** (2026-10-22) | LTE only: 170 · 194 · 216 · 176 → **187 mA** | Wi‑Fi + LTE: 194 · 184 · 191 · 207 → **194 mA** | LTE 256 min 9.37 %/h; Wi‑Fi + LTE 250 min 9.58 %/h | **8 h 26 min** (185 cycles) | Measured |
| 2026-10-21 → 22 | 2026-10-21 | Wi‑Fi + LTE: 149 · 168 · 180 · 159 → **163 mA** | LTE only: 210 · 205 · 189 · 230 → **207 mA** | Wi‑Fi + LTE 294 min 8.15 %/h; LTE 233 min 10.26 %/h | 8 h 47 min (193 cycles) | Measured |
| 2026-10-20 → 21 | 2026-10-18 | LTE only: 220 · 207 · 249 · 204 → **219 mA** | LTE only: 223 · 216 · 223 · 236 → **224 mA** | 11.53 %/h, ~10.9 %/h without a gauge jump at 20 % (mean of the modelled current: 229.6 mA) | 7 h 15 min from 99 % (159 cycles) | Measured |
| 2026-10-19 → 20 | 2026-10-18 | LTE only, whole run (bands not computed) | — | 11.73 %/h over 450 min (integrated modelled current: 232 mA) | 7 h 21 min (161 cycles) | Measured |

What changed between builds: 2026-10-21 added fewer frames drawn, the outer-screen call screen and the 8 s keypad-light timeout; 2026-10-22 added Wi‑Fi and background I/O tuning. The LTE-only first half went from 219 mA (2026-10-20) to 187 mA (2026-10-22), −15 %, without a split between those changes.

- **Wi‑Fi + LTE vs LTE only.** In both crossed nights the Wi‑Fi + LTE half was lower on the same band of the battery: 163 vs 187 mA (100 → 60 %) and 194 vs 207 mA (60 → 20 %). Solving the two nights together gives about −18 mA (~9 %) for Wi‑Fi and about +25 mA for the lower half of the battery (lower voltage, more current for the same power). That is within the ±25 mA night-to-night noise per band, and a 66-minute block A/B at 100–92 % (same workload; Wi‑Fi, LTE and LTE + Data Saver, two 10-minute blocks each in mirrored order) found no difference: Wi‑Fi 176, LTE 174, LTE + Data Saver 173 mA (modelled current). Read it as a moderate hint, not a result.
- **Battery level matters.** The same workload drew 201 mA in the first hour (99 → 90 %, ~4.17 V) and 223–249 mA later in the same night. Compare the same bands of the battery, not whole-night means.
- **Hours of use.** 8.5–9.7 %/h extrapolates to about 10.5 h from 100 to 0 %. The gauge is optimistic in the lower half (section 2), so treat that as an upper bound; the measured 100 → 20 % times above are the safer figure.

### Other states (older figures, still indicative)

| State | Date / build | Result | Evidence |
|---|---|---|---|
| Lid open, screen off, data on, after the workload | 2026-10-20 → 21 / 2026-10-18 | ≈0.6 %/h (197 min, 2 counter steps) | Measured, but below 20 % where the gauge stalls: indicative only |
| Screen off in total (lid open + lid closed), LTE | 2026-10-19 → 20 / 2026-10-18 | 100 mAh in 508 min ≈ 12 mA ≈ 0.6 %/h; AP awake 0.2 % of the time | Measured, same caveat |
| Bluetooth music, lid closed, LTE | 2026-10-19 / 2026-10-18 | ~10.9 %/h over the lid-closed time of the whole test (a streaming app syncing over LTE at times); same test, offline playback of downloaded music with no data traffic: ~150 mA (140–190), 7–9 %/h | Measured (offline figure from a 9-min window) |
| Main screen on and static | 2026-10-14 / before the frequency caps | ~115–150 mA (≈5.8–7.5 %/h), voltage-block bench; may include the keypad light | Measured, ~14 mA resolution |
| Mixed cycle with Wi‑Fi, two 20 s lid closes per ~235 s cycle | 2026-10-17 | 100 → 32 % in 9 h 01 min (~7.5 %/h); not the same workload as the nights above | Measured |

### How it was measured

**Instrument.** The phone's own fuel gauge, read from the battery's `charge_counter` (µAh) by a logger running on the phone as root. There is no current sensor on this phone and nothing external was used: `charge_counter` is the gauge's state of charge in µAh, moving in steps of about 20 mAh (section 2). mA is derived from how long the counter takes to fall; %/h from the counter over the rated 2000 mAh. The gauge's modelled `current_now` is logged too, but only as a cross-check.

**Logger.** Waits for the cable to be pulled and starts a few seconds later. It samples about every 10 s while the phone is awake: counter, %, voltage, modelled current, lid state, main-backlight level, CPU frequencies, radio technology and data counters. It also records each state change. States: workload running (screen on, lid open), lid open with the screen off, and lid closed, read from the Hall sensor's switch. At every state change it writes the counter and %, so time spent in each state can be summed even though the logger cannot sample while the phone sleeps (its timers stop in suspend). It takes a wakelock only around those transitions. It stops on replug, and in active-use runs the workload stops at 20 %.

**Lid-closed standby.** Close the lid, leave the phone untouched, open it hours later. %/h = counter drop over lid-closed time only. The result inherits every gauge bias: the counter under-reads idle drain by about 1.4–2.4× and catches up later (section 2), so these are lower bounds and only comparable with figures taken the same way. Start below ~90 % or discard the first hour after unplugging (the gauge sticks at "Full"), stop above 20 %, and run at least 6 h.

**Active-use workload.** A root script drives the phone through injected input, in a fixed, seeded order so every run does the same thing; only the web content differs. One cycle, about 164 s:

1. Browser: types a search into the address bar, waits on the results page (~20 s), opens the first result, scrolls it 5–10 times with pauses of a few seconds, then goes home and force-stops the browser.
2. Messages: opens a conversation with a dummy number, types a short sentence word by word with typing pauses, deletes it (nothing is sent, no draft is kept), exits.
3. Home screen with the screen on and untouched for 60 s.

The script itself uses about 1.5 % of one core; the whole system about 53 % of one core per cycle (≈13 % of the four cores). Mobile traffic was about 3 MB per cycle on LTE (3.2 MB in the 2026-10-21 night). The screen timeout is raised while it runs so the main screen stays on throughout. It does **not** exercise the keypad backlight (input is injected, so the light never comes on; ~28 mA while lit in real typing), the touch controller, the outer screen, camera, audio, Bluetooth, location or calls, so it underestimates real typing. In the nights above the lid stayed open during the workload.

**Conditions.** Unplugged the whole time (with USB connected the phone never deep-sleeps, and turning charging off in software does not cut the input). No ADB over Wi‑Fi (it wakes the phone about once per second). Indoors, room temperature (the battery sensor reads a near-constant ~25 °C, section 1). Main brightness 24/255 (37/255 on the 2026-10-19 night). A Spanish MVNO SIM, LTE + IMS registered, at home on a weak high-priority band (RSRP mostly −118 to −108 dBm). Wi‑Fi associated to a home access point in the Wi‑Fi + LTE halves (with Wi‑Fi up, Android drops the mobile internet connection and keeps only IMS); in the LTE-only halves Wi‑Fi was turned off. Bluetooth off (it switches itself off after 5 min unused).

**Resolution and limits.**

- 1 % = 20 mAh; a counter step is about 20 mAh, so a 200 mAh band is about 10 steps. Bands are timed from the 10 s samples.
- Below ~20 % (≈400 mAh) the gauge holds still and then drops in one go (one night: 20 → 12 % in 12 min with the screen off). Nothing below 400 mAh is used for mA.
- At the top the gauge sticks at "Full" for a while; the first band can read low.
- The counter under-reads idle drain; standby figures are lower bounds.
- Night-to-night noise is about ±25 mA per band. Differences smaller than that between single nights are not results; cross the order (one night A then B, the next B then A) and compare the same bands.
- Absolute mA assume the rated 2000 mAh; comparisons made with the same method on the same phone hold regardless.

**To compare another build.** Charge to 100 %, unplug with the lid open, run an equivalent workload continuously to 20 %, log `charge_counter` with timestamps and compute mA per 200 mAh band; compare band by band with the table above, with the network split and order stated. For standby, measure lid-closed time only from the counter, over at least 6 h, between ~90 % and 20 %.

---

## Viable / not viable

| Idea | Verdict | Why |
|---|---|---|
| Deep sleep with the lid closed when the outer screen is also an Android display | **Works** (own build) | Lid-sleep fix: ~2–3 %/h → 0.3–0.9 %/h with the lid closed. Without it Android stays interactive while the outer display is on (`suspend_stats` 0/0). A kernel-drawn clock needs no Android. |
| Lid detection on AOSP/GSIs | **Works (public kernel)** | Needs a kernel that emits the standard lid switch (the stock Hall driver doesn't). |
| Deep sleep with the lid closed on a GSI, kernel clock on the outer screen | **Works (public kernel)** | On a GSI the outer screen isn't an Android display, so nothing keeps Android interactive. AOSP's lid behaviour must be set to sleep (`lid_behavior=1`; VoltageOS already does). Lid-closed drain not measured on a GSI |
| Accurate battery current from the phone itself | **Not viable** without extra hardware | No current sensor; `current_now` is derived from voltage, and `charge_counter` is the % in mAh. |
| Battery stats in mAh per app | **Viable** with a measured power profile; **Works** (own build), approximate | No device power-profile values on stock; only the time counters mean anything. The own build carries figures from gauge benches, still model-based (one night: 2,178 mAh modelled vs 1,860 mAh real); an external meter would be needed for accuracy. |
| Rest-drain estimates from voltage | **Viable, with limits** | The hardware sleep-voltage accumulator plus the OCV table work. Below ~45 % the flat OCV table amplifies voltage errors. |
| Lid-closed standby drain | **Works** (own build) | 2.3 → 0.6 %/h step of 2026-10-17 (with the public kernel's 8 s outer-backlight timeout; not split, see [section 10](#10-lid-closed-drain-by-build)). ~0.6 %/h on LTE + IMS; calls and SMS only while the lid is closed (no push notifications while closed). Stock wakeup sources are in [section 4](#4-what-keeps-the-soc-awake-stock-kernel-stock-vendor-aosp) |
| Reporting the real wake IRQ to Android | **Works** (own build), diagnostic only | The stock kernel just never logs it. No drain change. |
| Wi‑Fi dropping during sleep | **Fixed** (own build) | Without it every suspend disconnected and Android didn't reconnect with the screen off |
| Fewer Wi‑Fi wakeups | **Works** (own build) | Wi‑Fi wakeups 10.0 → 4.6/min (−54 %); estimated 1.5–2.5 mA (0.08–0.13 %/h) |
| Extra Wi‑Fi firmware tuning for idle (beacon filter, packet filters, longer DTIM skip) | **Viable, marginal** | Fractions of a mA. |
| APF packet filtering | **Not viable** | APF runs inside the signed Pronto Wi‑Fi firmware; CAT's `prima` driver has no APF command (only the legacy packet filter). |
| Removing 3G | **Works** (tested), no gain in idle | Same modem duty cycle on the same cell. |
| "LTE only" | **Viable, no gain in idle** | The phone already stays on LTE; it only avoids the brief 2G drops right after boot. |
| Idle on 2G, calls on 4G | **Not viable** | Calls arrive on the network the phone is camped on; idle cost is similar. |
| Longer paging/DRX, longer RRC tail, eDRX/PSM, fast dormancy | **Not viable** | Set by the network, or not available in LTE for this device. |
| Excluding weak high-priority bands at home | **Viable, marginal** | 2–3 mA estimated, below what the gauge resolves in a night. Less capacity away from home. |
| Lower modem/PA voltage | **Not viable** | Set inside the signed modem firmware and RF calibration. |
| Disabling RX diversity | **Viable**, not worthwhile | Would go through the modem configuration; diversity is worth ~3 dB in weak coverage. |
| CPU/GPU frequency caps | **Works** (own build), modest gain in active use | Higher CPU states drag the shared CX/DDR domain up. Measured 10–21 mA less in active use (≈0.5–1.05 %/h); none with the lid closed. |
| Memory/storage bandwidth tuning | **Works** (own build), small | Average DDR bandwidth 1,374 → 827 MB/s, time at top DDR 9.3 → 0 %, jank 7.8 → 5.6 %; current change not resolvable |
| Active-use tuning | **Works** (own build) | LTE active-use night 219 → 187 mA (−15 %, ~11 → ~9.4 %/h); not split between changes |
| Bluetooth switching off when unused | **Works** (own build) | Off after 5 min with nothing connected or scanning |
| CPU operating point below 960 MHz | **Viable, no gain** | Same voltage floor: only a linear saving, and possibly more energy per task. |
| CPR trim / CPU undervolt | **Viable, no measurable gain** | CPR already adapts per chip. The leftover margin is worth about 1 % at rest and a few percent at heavy load, which can't be measured. Stability risk for no measurable gain. |
| GPU undervolt | **Not viable** | The GPU votes a CX corner to the RPM; no voltage control from Linux. |
| Changing RPM/PMIC behaviour in firmware | **Not viable** | Signed production firmware. The RPM already reaches VDD_MIN, so there is nothing to gain. |
| Moving rails to low-power modes from the AP | **Viable, no practical gain** | ~1.5–5 mA at best with the screen on (estimate); unmeasurable; touches stock DT and vendor modules. |
| Mainline kernel / newer Android / 64-bit for battery | **Viable, no battery gain** | The floor is hardware; blobs constrain the rest. The one reported mainline figure (third-party, about 1 %/h lid closed) is not below 0.6 %/h. 64-bit options in [platform.md](platform.md#64-bit). |
| Lid or key handling by root daemons instead of the kernel | **Viable**, costs wakeups | Each lid or key event, or each poll, wakes userspace; kernel-side handling avoids it (not measured) |
| Custom Power HAL for battery | **Works** (own build), no battery gain | It helps smoothness, not suspend or idle. |
| Removing resident vendor daemons or services | **Viable, no battery gain** | They barely use CPU (most <6 s in 87 min). It frees RAM, not battery. |
| Feature flags or developer options for battery | **Viable, no gain** | None affect power measurably. |
| Larger zRAM, swap, KSM, smaller ART heap | **Viable, no gain** | No memory pressure measured that would justify them. |
| Lowering the empty-voltage threshold | **Viable, not worth it** | ~1.4 % more capacity. |
| Charge-voltage limit for battery longevity | **Viable** | The charger has a 4.20 V regime. Costs an estimated 8–12 % of runtime per charge; untested. |
