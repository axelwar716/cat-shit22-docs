# Hardware and electrical

Scope: what is inside the CAT S22 Flip and how it is wired, as far as it can be established without a schematic. That covers the SoC, the PMIC and its rails, every identified chip with its bus, address and GPIOs, pin-map highlights, battery and gauge hardware, memory, thermal sensors, ESD behaviour, and what the hardware physically cannot do.

Related pages: [displays.md](displays.md) (both panels in depth), [keypad-lid.md](keypad-lid.md) (keypad, Hall sensor, keyboard light), [audio-bt-nfc.md](audio-bt-nfc.md) (audio path, Bluetooth, NFC, pogo pins), [telephony.md](telephony.md) (modem, IMS), [power.md](power.md) (sleep, wakeups, battery life), [platform.md](platform.md) (boot chain, partitions, kernel, vendor).

## How to read this page

- All of it comes from the stock device tree (DTB + DTBO as flashed on firmware v30), the stock kernel and vendor binaries, read-only runtime state (sysfs, debugfs, PMIC registers), public datasheets and teardown photos.
- **There is no schematic.** The manufacturer filed the schematic, block diagram and BOM with the FCC as permanently confidential, and the public teardown photos from third parties do not have the resolution to follow copper traces. Anything about the physical route between a SoC pin and a part is inferred from the DT and drivers.
- **A DT node with `status = "okay"` does not prove the part is fitted.** The S22 DT is partly a Qualcomm reference (QRD) template. Several touch controllers, a fingerprint sensor and NFC are declared in it but are not on the board. Each chip below is marked by evidence:
  - **Confirmed**: the DT declares it and a driver binds to it at runtime, or the chip identifies itself.
  - **Declared**: it is in the DT, with no proof that it is physically present.
  - **Inferred**: deduced from the DT, the drivers or datasheets, not traced on the board.
- "Measured" is used only for things measured on the hardware or on stock software.

## Board and SoC identity

| Item | Value | Evidence |
|---|---|---|
| SoC | Qualcomm **QM215** (`qcom,qm215`, `msm-id` 0x182) | Stock DTB |
| SoC family | MSM8917-derived. The clock controller is `qcom,gcc-qm215` in the 8917/8952 clock driver, the pin controller is the MSM8917 TLMM (134 GPIOs) and the display block uses the shared msm8937/8917 MDSS description. It is not an MSM8937: the 8937 name only appears in shared DT includes. | DT + kernel |
| CPU | 4x ARM Cortex-A53 (`CPU part 0xd03`), one cluster. On stock the kernel and userspace are 32-bit (ARMv7, `armeabi-v7a`) | `/proc/cpuinfo` |
| CPU frequencies | 960, 1094.4 and 1209.6 MHz exposed on this unit (frequency table selected by the fused speed bin) | cpufreq, fuses |
| GPU | Adreno 308 (the kernel driver reports `Adreno306v33`). Only **two** power levels, 400 and 270 MHz, because of the fused GPU bin | kgsl |
| Base clock | 19.2 MHz XO (crystal oscillator) | Clock tree |
| Board | DTBO entry 27, `qcom,board-id = <0x1000b 0x04>`, model string `QRD`. The boot image carries 27 DTBs and the DTBO partition 30 entries | Boot / DTBO images |
| ODM | Contract manufacturer, inferred (strong) from the stock factory test app and several kernel symbols. No board or service document seen | Stock system / kernel / vendor |
| PMIC | Qualcomm **PM8916**, on SPMI (slave IDs 0 and 1). It is the only PMIC: no PMI or external charger | DT |
| Subsystems | RPM (power manager), MSS (modem), LPASS/ADSP (audio DSP + sensors), Pronto/WCNSS (Wi‑Fi, Bluetooth, FM) | DT, PIL |

**Firmware.** SBL1, RPM, TZ, aboot, devcfg, cmnlib and the modem are signed with a production chain (debug disabled, OEM-specific IDs). The ADSP is authenticated at load time (inferred, strong). Secure boot is fused. Unlocking the bootloader only relaxes Android Verified Boot for `boot`/`system`/`vendor`. It does not affect the RPM, TZ, modem, ADSP or Wi‑Fi firmware images, which cannot be modified (configuration outside the images, such as modem EFS/NV and Wi‑Fi ini/NV files, is editable with root). Details are in [platform.md](platform.md).

## PMIC: PM8916

| Block | What it does on the S22 | Driver (stock kernel) |
|---|---|---|
| SMPS S1–S4, LDO L1–L18 | All system rails (table below). Managed through RPM votes | `rpm-smd-regulator` |
| S2 + SPM/CPR | CPU cluster supply (APC), with closed-loop CPR voltage scaling | `cpr-regulator`, SPM |
| Linear charger | The only charger. One input (`usb-chgpth`, USBIN), no DCIN | `qpnp-linear-charger` |
| VM-BMS | Voltage-mode fuel gauge with the Q2805 battery profile. It does **not** measure current | `qpnp-vm-bms` |
| VADC / ADC_TM | Voltages and thermistors (channel list below) | `qpnp-adc-voltage`, `qpnp-adc-tm` |
| PON | Power key (KPDPWR) and RESIN inputs | `qpnp-power-on` |
| LPG (PWM), 1 channel | Main display backlight PWM, routed out through MPP4 | `pwm-qpnp` |
| Vibrator | Internal vibrator driver for the ERM motor | `leds-qpnp-vibrator` |
| Charge LED | Red charging LED driven by the charger (`qcom,chgr-led-support`) | `leds-qpnp` / charger |
| RTC | Real-time clock and wake alarms | `qpnp-rtc` |
| Analog audio codec | Earpiece, headset and microphone analog front end (PDM link to the SoC) | Vendor modules (stock `.ko`) |

### PMIC GPIOs and MPPs

| Pin | Use on the S22 |
|---|---|
| GPIO1 | No consumer, no IRQ |
| GPIO2 | Only the clock-request line of the disabled NFC template node |
| GPIO3 | Driven high. Enables an external 1.2 V LDO that the DT names `disp_vdda_eldo1` (a reference-design name, fed from S3) with no Linux consumer. **Driving it low reboots the phone instantly** (measured), so whatever it powers is essential. It is likely a system 1.2 V rail, possibly for the DRAM (unconfirmed) |
| GPIO4 | No consumer, no IRQ |
| MPP1, MPP3 | No consumer, no ADC channel |
| MPP2 | Skin thermistor input (`skin_therm`) |
| MPP4 | Digital output carrying the LPG1 PWM to the main display backlight. Set up by the bootloader |

### VADC channels

| Channel | Signal |
|---|---|
| 0x00 | `usb_in` (USB input voltage) |
| 0x02 | `ireg_fb` (charger current feedback) |
| 0x05 | `vcoin` |
| 0x06 | `vbat_sns` (battery voltage) |
| 0x07 | `vph_pwr` (system supply) |
| 0x08 | PMIC die temperature |
| 0x09 / 0x0a | References |
| 0x0b | `chg_temp` |
| 0x11 | `skin_therm` (MPP2) |
| 0x30 | `batt_therm` (battery thermistor) |
| 0x31 | `batt_id` |
| 0x32 / 0x3c | XO thermistor |
| 0x36 | PA thermistor |

## Rail map

Voltages are what the hardware reports, from the PMIC's own regulator registers where available. "AP" is the application processor (Linux). Rails the AP does not vote for are held by the RPM or the other subsystems (modem, Wi‑Fi). A rail with no Linux consumer is **not** free: the RPM or the modem may still use it.

| Rail | Type | Voltage | Consumers declared in the DT | Screen on (main display) | Idle / suspended (lid closed) |
|---|---|---|---|---|---|
| S1 | SMPS, PWM, 3.2 MHz | 1.125 V | CX digital core: clock controller, modem, LPASS, Pronto (level/corner votes) | On, SVS corner | Kept at the RPM retention level; the SoC reaches VDD_MIN |
| S2 | SMPS, PWM, 3.2 MHz | ~1.1–1.3 V depending on CPU frequency (CPR) | CPU cluster (APC) through the SPM | On | CPU power-collapsed |
| S3 | SMPS, PWM, 1.6 MHz | 1.275 V (briefly up to 1.425 V) | Parent of the low-voltage LDOs (inferred) and of the external 1.2 V LDO enabled by PMIC GPIO3. Not voted by the AP | On | Kept on by the RPM |
| S4 | SMPS, PWM, 1.6 MHz | 2.05 V | Codec charge pump / PA supply (declares a 550 mA static load); parent of the 1.8 V LDOs (inferred) | On | Kept on by the RPM |
| L1 | LDO | 0.968 V nominal | None | Off | Off |
| L2 | LDO | 1.1625 V | MX memory rail: modem, Pronto, camera CSI | On (HPM) | Kept on by the RPM |
| L3 | LDO | 1.35 V | Wi‑Fi/BT RF (`iris-vddrfa`) | On (HPM) | Held by other masters |
| L4 | LDO | 1.744 V nominal | None | Off | Off |
| L5 | LDO | 1.8 V | I/O rail: eMMC VCCQ, touch VIO, Hall sensor, audio codec I/O, unused fingerprint template | On (HPM) | **Always on** (RPM). Cannot be switched off |
| L6 | LDO | 1.8 V | DSI PHY/PLL and main panel VDDI, outer panel VDDIO, camera VIO | On (HPM) | Off when both panels are off |
| L7 | LDO | 1.8 V | PLLs (modem, Pronto, high-frequency PLL), Wi‑Fi RF XO/digital, USB 1.8 V | On (HPM) | Kept on by the RPM |
| L8 | LDO | 2.9 V | eMMC VCC | On only while the eMMC works | **Off** whenever the eMMC sleeps (most of the idle time) |
| L9 | LDO | 3.3 V | Wi‑Fi PA (`iris-vddpa`) | On with Wi‑Fi | Off with Wi‑Fi off |
| L10 | LDO | 2.7 V nominal | Camera autofocus actuator (`cam_vaf`) | Off unless the camera runs | Off |
| L11 | LDO | 2.7 V nominal | microSD VDD | Off without a card | Off |
| L12 | LDO | 1.65–2.95 V nominal | microSD I/O | Off without a card | Off |
| L13 | LDO | 3.05 V | USB PHY 3.3 V, microphone bias | On only with USB connected (or mic bias in use) | Off |
| L14, L15 | LDO | 1.8 V nominal | No Linux consumer | Read off | Read off |
| L16 | LDO | 2.7 V nominal | Camera analog (`cam_vana`) | Off unless the camera runs | Off |
| L17 | LDO | **3.0 V in hardware** (Linux requests 2.85 V) | Main panel VCI, outer panel VDD, touch controller VDD | On (HPM) | On while the outer panel or touch is powered |
| L18 | LDO | 2.6 V nominal | None | Off | Off |
| `usb_vdig` | Fixed | 1.2 V | USB PHY digital | With USB | Off |

Notes on the rail map:

- **With the main display on, every SMPS runs in forced PWM and every enabled LDO in high-power mode (HPM).** The cause is in the stock DT: consumers declare large static loads (100 mA per display consumer on L6 and L17, 550 mA from the audio codec on S4, 325 mA from the eMMC on L5), and the 4.9 regulator core adds up the declared load of every consumer, enabled or not. Even with every reducible item counted, what the AP could trim this way is estimated at a few mA at most, below what the phone's gauge can resolve.
- **L17 sits at 3.0 V in hardware while Linux asks for 2.85 V.** The DT `init-voltage` is 3.0 V. Another master's vote is the likely cause (unresolved). L17 is an LDO from the battery side, so the output voltage barely changes the battery current.
- **L5 vote counter.** L5's `use` count grows with every resume (from single digits to hundreds after a night, and into the thousands over days). It is an unbalanced enable/disable vote in a driver, apparently on the audio codec path. It has no electrical cost, because L5 is always on anyway.
- **L6 and L17 stay in HPM during sleep** while a display driver holds them enabled with its declared 100 mA load. The RPM forces S3, S4, L2, L5 and L7 on; L6 and L17 are not in that set and can go off or to low-power mode.
- **DSI PHY regulator.** The stock DT runs the DSI PHY's internal regulator in LDO mode. The Qualcomm QM215 reference design uses DCDC mode with the same settings. Whether the S22 board supports DCDC is unknown.
- **USB connected changes everything.** USB votes for L7 HPM, L13, the USB PHY and high DDR/NoC clocks, and keeps the XO on. Rail or sleep observations made with a cable plugged in do not represent battery operation.

## RPM and sleep states (hardware view)

- **The SoC does reach VDD_MIN** (XO shutdown plus minimum voltage) with the display off and no USB cable. It does not with USB connected.
- **With the main display on, the SoC never releases the XO** and never enters VDD_MIN. The main panel is a video-mode panel without frame memory, so the DSI PLL and escape clock need the XO and the display engine needs CX at its lowest operating corner. This is structural, not a software choice. See [displays.md](displays.md).
- The CPU idle path uses core power collapse and cluster (L2) power collapse. The stock eMMC DT entry places a PM QoS latency vote on all four cores for every I/O request plus 10–20 ms: 13 µs in high-performance mode, 651 µs in low-power mode. While that vote is active, cluster power collapse (11.2 ms exit latency) is not allowed.
- The stock kernel command line contains `lpm_levels.sleep_disabled=1`, but at runtime the parameter reads `N`. Stock does suspend normally.
- While Bluetooth audio is streaming, the Pronto (Wi‑Fi/BT) subsystem never releases the XO. The cause sits in the Bluetooth stack policy, not in the chip. See [audio-bt-nfc.md](audio-bt-nfc.md).
- The RPM is not what limits idle consumption. With the lid closed and the SoC asleep, what remains is the modem, the cell and the parts that stay powered (see [power.md](power.md#9-energy-floor-and-ceilings-hardware)).

## Buses

| Bus | Controller | SoC pins | Devices |
|---|---|---|---|
| I²C-3 | BLSP1 QUP3, `i2c@78b7000` | TLMM 10, 11 | Touch controller 0x2e (plus template ghosts 0x14, 0x24, 0x38) |
| I²C-4 | BLSP1, `i2c@78b8000` (disabled in Linux) | TLMM 14, 15 | Owned by the ADSP: sensor bus |
| I²C-5 | BLSP2 QUP1, `i2c@7af5000` | TLMM 18, 19 | Speaker amp 0x34, USB-C controller 0x60, FSA4480 0x42 (declared, not detected), NFC 0x28 (disabled template) |
| SPI (spi6) | BLSP2 QUP2, `spi@7af6000`, 50 MHz | TLMM 20–23 | Outer display |
| CCI | Camera control interface `cci@1b0c000` | TLMM 29–32 | Both camera sensors |
| DSI0 | MDSS DSI, 2 data lanes | Dedicated | Main display |
| SDCC1 | `sdhci@7824900`, 8-bit | Dedicated | eMMC |
| SDCC2 | `sdhci@7864900`, 4-bit | Dedicated | microSD |
| PDM | Digital codec link | TLMM 69–74 | PM8916 analog codec |
| MI2S | `pri_mi2s` function | TLMM 85–88 | Speaker smart amplifier I²S |
| UART | BLSP1 UART1 | TLMM 0, 1 | Declared |

## Chip inventory

| Part | Function | Bus / address | GPIOs (TLMM unless noted) | Supplies | Evidence |
|---|---|---|---|---|---|
| Qualcomm QM215 | SoC: 4x A53, Adreno 308, modem, LPASS/ADSP, Pronto | — | — | S1 (CX), S2 (APC), L2 (MX), L3, L7 | Confirmed |
| Qualcomm PM8916 | PMIC, charger, gauge, codec, LPG, vibrator, RTC | SPMI | — | Battery / VPH | Confirmed |
| Sitronix **ST7701S** | Main display controller, 480x640, MIPI DSI video mode, no frame memory | DSI0, 2 lanes | TE 24, reset 60 | L17 (VCI), L6 (VDDI, PHY) | Confirmed. Two alternative panels (GC9503V, JD9161Z) and an HX8399C entry are also in the DT |
| Main backlight boost | External boost driver, single control input (enable + PWM) | — | PM8916 LPG1 → MPP4 | — | Inferred; **part not identified** (no name in the DT, bootloader, teardowns or FCC files) |
| Sitronix **ST7735S**-family (DT calls it `st7789v2`) | Outer display controller, 128x128, RGB565 over SPI | SPI6, 50 MHz | DC 68, TE 124 (the panel never sends it), reset 125, backlight enable 12 | L17 (VDD), L6 (VDDIO) | Confirmed panel; controller family inferred from its init sequence (stock ST7735S defaults). Silicon ID not read |
| Chipsemi **CHSC5448** | Main touchscreen controller, 5 points, glove mode | I²C-3 0x2e | IRQ 65 (falling edge), reset 64 | L17 (VDD), L5 (VIO) | Confirmed (identifies itself). Own RISC‑V firmware, embedded in the stock kernel and checked at boot |
| Awinic **AW88194** | Speaker smart amplifier (boost + DSP) | I²C-5 0x34, I²S on TLMM 85–88 | Reset 66, IRQ 59 | Battery | Confirmed (identified at probe) |
| Awinic **AW8155** | Hearing-aid coil (HAC) amplifier; modes are set by pulses on its enable pin | — | Enable 38 | Battery | Declared + driver; held off in normal use |
| ON Semiconductor (Fairchild) **FSA4480** | USB-C analog audio switch | I²C-5 0x42 | — | — | **Not detected**: no driver binds and the stock audio driver logs `no fsa4480 chip` |
| PM8916 analog codec + SoC digital codec | Earpiece, headset, microphones | PDM, TLMM 69–74 | USB-C audio enables 8 and 93, US/EURO switch 63 (declared in the codec pin control) | S4, L5, L13 (mic bias) | Confirmed |
| WillSemi **WUSB3801** | USB Type-C CC controller (dual-role, prefers sink, accessory support) | I²C-5 0x60 | IRQ 17 | No supply in the DT | Confirmed. Pins: VDD, GND, CC1, CC2, VBUS detect, INT, SDA, SCL, ID. The DT also gives it `reset-gpio` 12, the same line as the outer backlight; the driver does not use it |
| Qualcomm USB 2.0 HS PHY/controller | USB | `usb@78db000` | — | L7 (1.8 V), L13 (3.3 V), 1.2 V fixed | Confirmed |
| Pronto (in SoC) + WCN36xx-family RF companion | Wi‑Fi, Bluetooth, FM | WCNSS `a000000`, pins 76–80 | — | L9 (PA), L3 (RF), L7 (XO/digital), L2, S1 | Confirmed subsystem. The RF companion's exact part (likely WCN3615) is not read |
| Modem (MSS, in SoC) | LTE/IMS; GNSS also lives in this firmware domain (not mapped) | PIL `4080000` | Modem-owned pins | RPM | Confirmed |
| GalaxyCore **GC5035** | Camera sensor (5 MP class by part number) | CCI | See camera row below | L6, L16, L10 | Confirmed (probes) |
| GalaxyCore **GC02M2** | Camera sensor (2 MP class by part number) | CCI | See camera row below | L6, L16 | Confirmed (enumerated) |
| Flash LED driver | Rear LED: torch and flash | — | `FL_EN` 33, `FL_NOW` 34 | — | Confirmed behaviour; **part not identified** |
| Sensortek **STK8BA53** | Accelerometer | ADSP sensor bus (I²C-4) | — | ADSP domain | Enumerated by the sensor HAL |
| Sensortek **STK3X1X** | Ambient light + proximity | ADSP sensor bus | — | ADSP domain | Enumerated by the sensor HAL |
| TDK InvenSense **ICP-10100** | Barometer | ADSP sensor bus | — | ADSP domain | Enumerated by the sensor HAL |
| Hall sensor | Lid open/closed | — | IRQ 44 | L5 | Confirmed function; **IC not identified** |
| Keypad matrix | 5x5 matrix | — | 10 GPIOs (pin map below) | — | Confirmed |
| Keyboard backlight | LEDs switched by an external transistor | — | 130 | VPH (inferred) | Confirmed |
| ERM vibration motor | Haptics | PM8916 vibrator | — | Internal | Confirmed |
| eMMC, 16 GB | Internal storage, HS400 with command queue | SDCC1 | — | L8 (VCC), L5 (VCCQ) | Confirmed. Vendor not recorded |
| microSD slot | Removable storage | SDCC2 | Card detect 67 | L11, L12 | Declared |

**Template parts that are not on this board:**

| Declared in the DT | Why it is not real |
|---|---|
| Goodix GT1X (I²C-3 0x14), Cypress CYTTSP5 (0x24), FocalTech FTS (0x38) | They share reset 64 / IRQ 65 with the CHSC5448. Their drivers are compiled into stock but never bind |
| FPC1020 fingerprint | Its reset is on TLMM 124, which is the outer display's TE line. No sensor on the phone |
| NXP NQ NFC (I²C-5 0x28) | `disabled` in the DT and NFC is off in the kernel config. Its IRQ pin (17) is the USB-C controller's IRQ and its firmware pin (130) is the keyboard light. See [audio-bt-nfc.md](audio-bt-nfc.md) |
| Sensor HAL configuration | The vendor's sensor definition file is the Qualcomm reference board's. Its bus addresses do not prove this board's wiring |

## Camera control lines

| Signal | `camera@0` | `camera@1` |
|---|---|---|
| MCLK | 26 | 28 |
| Reset | 36 | 40 |
| Standby | 35 | 39 |
| Digital enable | 25 (shared) | 25 (shared) |
| Supplies | VIO L6, VANA L16, VAF L10 (actuator) | VIO L6, VANA L16 |

Which sensor (GC5035 or GC02M2) is wired to which node has not been traced.

## Flash LED

| Pins | Result |
|---|---|
| 33 alone | Torch, steady |
| 34 alone | Nothing |
| 33 + 34 | Flash mode. The LED chip cuts it by itself after about 1.5 s to protect the LED |

Measured on the phone. The kernel public README describes the same behaviour.

**Flashlight fix: Fixed (public kernel).** GPIO 33 (`FL_EN`) alone gives a steady torch; 34 (`FL_NOW`) alone does nothing; 33 + 34 is flash mode, which the LED chip cuts by itself after about 1.5 s. Qualcomm's `qm215_gpio_flash` driver drove both pins for the flashlight, so a kernel rebuilt from the Qualcomm base turned it off after about 3 s. The public kernel (`stock-stable-5`) now uses 33 for the torch and 33 + 34 for the camera flash; the flashlight stays on for the whole 14 s test on Android 13, 15 and 16. CAT's stock kernel never had this bug.

## GPIO / pin map highlights

| TLMM | Function |
|---|---|
| 0, 1 | BLSP1 UART1 (declared) |
| 8, 93 | USB-C analog audio enables (codec pin control) |
| 10, 11 | I²C-3 (touch) |
| **12** | **Outer display backlight enable.** Binary. 8 mA drive at runtime (the DT sleep state declares 2 mA). Also declared as the WUSB3801 reset (unused) |
| 14, 15 | I²C-4, sensor bus owned by the ADSP. Do not touch |
| 16 | NFC VEN in the template; unused input |
| **17** | WUSB3801 interrupt (the NFC template's IRQ) |
| 18, 19 | I²C-5 (amp, USB-C, FSA4480) |
| 20–23 | SPI to the outer display |
| 24 | Main display TE (`mdp_vsync`) |
| 25, 26, 28, 35, 36, 39, 40 | Camera control (table above) |
| 29–32 | CCI (camera I²C) |
| 33, 34 | Flash LED `FL_EN` / `FL_NOW` |
| 38 | AW8155 enable |
| **44** | **Hall sensor**: input with pull-up, active low, interrupt on both edges, wakeup source |
| 45, 62, 97, 126, 127 | Keypad rows (interrupt inputs) |
| 59 | AW88194 IRQ |
| 60 | Main display reset. It reads 0 while the panel is on and showing an image, so the panel reset is either not on this pin or goes through an inverter (unresolved) |
| 63 | Headset US/EURO switch (codec pin control) |
| 64, 65 | Touch reset / IRQ |
| 66 | AW88194 reset |
| 67 | microSD card detect (pull-up) |
| 68 | Outer display D/C |
| 69–74 | PDM link to the PM8916 codec |
| 76–80 | Wi‑Fi/BT RF interface (`wcss_wlan`) |
| 85–88 | I²S to the speaker amp |
| 89, 90, 96, 99, 129 | Keypad columns (outputs) |
| 91 | Vol+ (`gpio-keys`, pull-up) |
| 95 | In the DSI pin-control group; function on the board unconfirmed |
| 124 | Outer display TE (also the FPC template's reset) |
| 125 | Outer display reset |
| **130** | **Keyboard backlight** (binary). Also the NFC template's firmware pin |

**Pins with no Linux owner:** 37, 41–43, 46, 47, 49, 51–58, 61, 75, 81–84, 92, 98, 100–123, 131–133. No IRQ, extcon, key or kernel consumer uses them. That does not make them free or floating: some may be set by the bootloader or owned by the modem.

**Reference-design backlight pins are not used here.** On the QRD, 93/94 (and 98 on the 8917 MTP) are backlight enables. On the S22, 93 belongs to audio, 94 is unclaimed and 98 is an input.

### Keypad matrix

| | TLMM |
|---|---|
| Columns (outputs, 2 mA, active high) | 96, 99, 90, 89, 129 |
| Rows (interrupt inputs) | 126, 127, 45, 97, 62 |

- Row 45 is the `1 4 7 *` row.
- On stock the rows idle low through the SoC's internal pull-down. The input threshold is a fixed digital threshold.
- Stock DT values: `debounce-delay-ms = 3`, `col-scan-delay-us = 5`.
- The power/end key and Vol− are not in the matrix. They are on the PM8916 PON inputs (KPDPWR and RESIN respectively, per the stock key layout). Vol+ is on TLMM 91.

The stock double-press and lost-tap problems come from how the stock driver samples this matrix, and from a worn dome forming a resistive divider with the row pull-down. That analysis is in the [public kernel README](https://github.com/axelwar716/cat-shit22-kernel#keypad-matrix_keypad-rewrite) and in [keypad-lid.md](keypad-lid.md).

### Hall sensor (lid)

| Property | Value |
|---|---|
| Line | TLMM 44, pull-up, `ACTIVE_LOW`, interrupt on both edges, `linux,wakeup` |
| Supply | L5 (`vddio`, DT range 1.65–3.3 V) |
| Stock driver | `hall_sensor` (`hall-switch` compatible) |
| Stock state | `/sys/class/switch_hall/hall_switch/state`: **1 = open, 0 = closed**. Updated about 300 ms after the edge (busy-wait in the stock IRQ path) |
| Stock input device | `hall_sensor` (phys `/dev/input/hall_dev`). It declares `SW_LID` but emits `KEY_SLEEP`/`KEY_WAKEUP`, which the input core drops because the device has no `EV_KEY`. **With the stock kernel `SW_LID` never changes from 0**, so AOSP never learns the lid state. The stock framework reads the switch node instead |

Behaviour and integration are covered in [keypad-lid.md](keypad-lid.md).

### Pin conflicts inherited from the template

| Pin | Real use | Template claim |
|---|---|---|
| 12 | Outer backlight | WUSB3801 reset. Enabling that reset would fight the backlight |
| 17 | WUSB3801 IRQ | NFC IRQ |
| 124 | Outer display TE | FPC reset |
| 130 | Keyboard backlight | NFC firmware/download |
| 64, 65 | CHSC5448 reset/IRQ | Three other touch controllers |

## Displays and backlights (hardware summary)

Full details are in [displays.md](displays.md).

| | Main display | Outer display |
|---|---|---|
| Controller | ST7701S | ST7735S family (DT says ST7789V2) |
| Resolution | 480x640 | 128x128 |
| Link | MIPI DSI, 2 lanes, video mode | SPI at 50 MHz, RGB565 (32,768 bytes per full frame) |
| Frame memory | **None.** Every refresh is a full scan driven by the SoC | Has GRAM. With static content nothing is sent over SPI |
| Supplies | L17 (VCI), L6 (VDDI and DSI PHY) | L17 (VDD), L6 (VDDIO) |
| Backlight | PWM from PM8916 LPG1 (about 9.4 kHz, 9-bit) out of MPP4 into an external boost with a single logic input | Single enable line, TLMM 12. **Binary**: levels 4, 80 and 255 give the same light, and a low line leaves no residual glow |
| Backlight cost | Brightness 255 vs 12: about +76 mA (measured) | — |
| Controller power (datasheet) | — | About 0.9 mA typical (2 mA max) while scanning; 0.015 mA in sleep-in |

Stock outer-backlight logic: the driver only toggles the GPIO. The 1–255 level is forwarded to an LED trigger that nothing is attached to. Stock contains no backlight-controller driver (no AW/KTD/LM36/TPS-class part) and no PWM consumer for the outer panel.

## Keyboard backlight

- Switched by an external transistor from TLMM 130 (2 mA drive). The pin has no PWM function, so the light is **on or off only**.
- Draws **about 28 mA while on** (measured, 22.7–33.4 mA confidence interval).
- Android's stock light HAL on this vendor does not expose keyboard or button lights. See [keypad-lid.md](keypad-lid.md).

## Touch controller

- The CHSC5448 runs its own firmware, which ships embedded in the stock kernel (65,492 bytes) and is checked at boot. It is outside the Qualcomm firmware chain.
- Its deep sleep draws about 10 µA (datasheet). The stock driver suspends it with the main display blank and ignores outer-display events (framebuffers 239 px wide or less).
- **Touch scanning is a large item with the screen on.** Putting the controller in deep sleep while the main screen is on saved about 10–11 mA (measured), but touch then needs a key press to wake.
- **ESD heartbeat:** the stock driver polls the controller every 4 s for ESD recovery. Turning that heartbeat off makes no measurable difference: +0.2 mA, 95 % interval −9.5 to +9.9 mA (measured).
- The stock driver does not manage its regulators: L17 and L5 stay on because other drivers hold them.

## ESD findings

| Finding | Detail |
|---|---|
| Main panel ESD check | The stock display driver reads the panel status (DCS `0Ah`) every few seconds and resets the panel on a bad reply |
| 8-colour idle mode | The ST7701S's 8-colour idle mode makes the status read return unexpected values (0x9C / 0xDC), so the ESD check fails and resets the panel in a loop. The mode is unusable with the stock ESD check (measured); see [displays.md](displays.md#viable--not-viable) |
| Touch ESD heartbeat | 4 s poll, no measurable current (above) |

## USB-C, charging input and pogo pins

- **Single charging path:** USB-C VBUS → PM8916 USBIN → linear charger. There is no DCIN and no second charger on any bus. BC1.2 detection is done by the USB PHY driver.
- **No 5 V output.** The DT has no VBUS/OTG supply and no 5 V regulator, and the linear charger registers no boost. The phone cannot power USB devices, so OTG only works with self-powered devices (inferred from DT and source; not tested with a bus-powered device).
- **Pogo pins** (two gold contacts on the back): power only, VBUS and GND in parallel with the USB-C connector. A third party measured 5.2 V across them while charging over USB-C. Nothing in hardware or stock software detects a dock: no GPIO, IRQ, ADC channel or driver. Two contacts cannot carry USB data. With the contacts live while charging over USB-C, a metal object across them shorts the charger input. Details in [audio-bt-nfc.md](audio-bt-nfc.md).
- The WUSB3801's VBUS-detect bit could tell "power without a CC partner" (pogo only) from a USB-C cable. No stock software reads it.

## Battery and gauge hardware

| Item | Fact |
|---|---|
| Rating | 2000 mAh (BTE-2000, 2050 mAh typical, 4.35 V charge). The gauge and the stock profile use 2000 mAh |
| Battery profile | Q2805 (in the DT), with its OCV table and resistance values |
| Gauge type | **PM8916 VM-BMS, voltage mode. There is no current sensor**, neither for the battery nor per rail |
| `current_now` | Not a measurement. It is computed from the voltage: (model OCV − measured voltage) / 0.271 Ω, where 0.271 Ω is the profile's default plus capacitive resistance. It fits the data exactly (R² = 1.0000) |
| `capacity` (%) | The output of a slow observer: time constant about 30 min above 60 %, 2–3 h in the 20–45 % plateau, frozen during suspend. It under-reports idle drain by 1.4–2.4x and catches up in jumps. Below about 20 % it can stall and then drop several points at once (measured) |
| `charge_counter` | The same % expressed in mAh (20 mAh steps). Not a coulomb counter |
| Cycle count | Not usable as a health metric: it resets to 0 when the stored shutdown SOC is invalid |
| Battery thermistor | `batt_therm` on the VADC, 100 kΩ pull-up to 1.8 V. It reads 25.0–25.9 °C in 98 % of samples, even with the PMIC and XO at 31–37 °C. **Inferred (strong):** the line has a fixed resistor instead of a real thermistor (it also reads lower while charging than while discharging). If so, the charger's temperature protections and the gauge's temperature compensation never act. Confirming it needs the line's resistance measured |
| Charging | Linear (CC/CV) from USBIN. In the "warm" temperature band the stock driver charges to 4.2 V at 1000 mA. An SDP (computer port) gives 500 mA |
| Empty cutoff | About 3.4 V |
| Charge LED | Red LED driven by the PM8916 charger |
| Charging off ≠ input off | Disabling charging in software does not disconnect the input: the charger keeps supplying the system. The kernel exposes no input suspend |

Battery-life figures are in [power.md](power.md#10-lid-closed-drain-by-build).

## Memory and storage

| Item | Fact |
|---|---|
| RAM | 2 GB. `MemTotal` about 1.88 GiB after hardware and kernel reservations. DRAM type is likely LPDDR3 (unconfirmed) |
| Software profile | Stock is Android Go (low-RAM profile), 32-bit userspace and zygote, 240 dpi |
| eMMC | 16 GB, 8-bit HS400 with command queueing, clock scaling 50–200 MHz. VCC (L8) is cut while the eMMC sleeps; VCCQ (L5) stays on |
| microSD | SDCC2, 4-bit, card detect on TLMM 67 |
| Kernel memory features | Linux 4.9 has PSI, memcg, zRAM (LZ4) and the cgroup freezer. It has no MGLRU, DAMON or zRAM recompression, and none of those is practical to backport |

Partition layout and the size of `super` are in [platform.md](platform.md).

## Thermal

**Thermal zones** (24, as exposed by the stock DT and drivers):

| Group | Zones |
|---|---|
| Thermistors on the VADC | `xo-therm-adc`, `xo-therm-buf-adc`, `pa-therm0-adc`, `skin-therm-adc` |
| CPU | `cpuss-usr`, `apc1-cpu0..3-usr`, `apc1-cpu0..3-step`, `penta-cpu-max-step` |
| GPU | `gpu0-usr`, `gpu0-step` |
| Subsystems | `mdm-core-usr` (modem), `q6-usr` (ADSP), `camera-usr`, `aoss0-usr`, `aoss0-lowf` |
| PMIC / battery | `pm8916_tz`, `bms`, `battery` |

**Cooling devices** (10): CPU frequency x4, GPU devfreq, CX regulator floor (`regulator-cx-cdev`), battery (charge current), modem PA, modem processor, modem VDD.

- A zone name does not prove a separate physical sensor. Thermistor positions on the PCB are unknown.
- The stock `thermal-engine` (vendor) runs three monitors on `xo-therm-adc` with a 1 s sampling period while the screen is on, covering charging, CPU hotplug and CPU frequency (limit around 56 °C). It does not run during suspend.
- `skin-therm-adc` returns implausible values (0.9–30.8 °C while the XO thermistor is stable). It is unusable for policy (measured).
- See the battery thermistor inference above.

## What the hardware cannot do

| Limit | Why |
|---|---|
| Dim the outer display backlight | It is a single enable line (TLMM 12), with no PWM or analog control routed to it |
| Dim the keyboard backlight (true dimming) | A GPIO switching a transistor, and the pin has no PWM function. Software PWM from a kernel timer is viable but costs CPU wakeups (untested) |
| Drive the main backlight with constant current | The PMIC PWM goes into a logic input of an external boost. No analog output exists |
| Keep the main display on with the SoC in deep sleep | Video-mode panel without frame memory: the SoC must refresh it continuously (XO, DSI PLL, display engine) |
| Partial update / self-refresh on the main panel | The ST7701S variant used has no display RAM and no command mode |
| Measure current | Voltage-mode gauge only. No coulomb counter and no per-rail sensing |
| Supply 5 V over USB-C or the pogo pins | No boost or OTG regulator on the board |
| NFC | No NFC chip, and the template's pins are used by other functions |
| Dock detection over the pogo pins | Power-only contacts with no sense line |
| Bluetooth A2DP offload | Not available on this chip combination: the CPU encodes every audio frame |
| Run the GPU at more than two levels | Fused bin: only 400 and 270 MHz |
| Switch off L5 or PMIC GPIO3 | L5 feeds the eMMC I/O, Hall and touch I/O. GPIO3's LDO is essential (the phone reboots instantly) |
| Modify RPM/TZ/modem/ADSP/Wi‑Fi firmware images | Production-signed with secure boot fused (configuration outside the images, such as modem EFS/NV and Wi‑Fi ini/NV, is editable with root) |

## Viable / not viable

| Item | Verdict | General reason |
|---|---|---|
| Re-tuning PMIC rail modes (SMPS auto/PFM, LDO low-power) from Linux | Viable, not worth it | All the AP can change adds up to an estimated 1.5–5 mA with the screen on, below what the gauge resolves. Several rails are owned by the RPM or the modem |
| Lowering L17 or L6 voltage | Viable, no gain | They are LDOs, so battery current barely depends on their output voltage. L17 is shared by both panels and touch |
| DSI PHY regulator in DCDC mode | Viable only if the board has the DCDC inductor (unconfirmed) | At most a fraction of a mA |
| Outer panel at rest while dark | **Works** (own build), 0.6 → 0.6 %/h with the lid closed (no change within noise) | Stock already powers it down after its backlight timeout. The controller's own datasheet figures are 0.9 mA scanning vs 0.015 mA at rest, below what the gauge resolves |
| Touch controller asleep while the main screen is on | Viable, usability cost | About 10–11 mA saved (measured), but touch then needs a key press to wake |
| CPU/GPU frequency caps | **Works** (own build) | Software policy within the existing operating points; measured 10–21 mA less in active use (≈0.5–1.05 %/h), none with the lid closed. See [power.md](power.md#8-cpu-gpu-ddr-cpr) |
| CPR undervolt | Viable, not worth it | A small voltage margin exists, but the expected gain is around or below what the gauge resolves, with stability risk. See [power.md](power.md#8-cpu-gpu-ddr-cpr) |
| GPU undervolt | Not viable from Linux | The GPU votes a CX corner to the RPM; there is no GPU-specific voltage control |
| Main display below about 20 Hz | Viable, flicker risk, small gain | Estimated 0.9–2.2 mA; flicker was measured at 20–22 Hz with some timings (a-Si panel without frame memory). Details in [displays.md](displays.md#viable--not-viable) |
| Dimmable outer or keyboard backlight | True dimming: viable only with a hardware mod; software PWM from a kernel timer: viable, untested | Binary enable lines. Software PWM costs CPU wakeups while lit (see [displays.md](displays.md#viable--not-viable), [keypad-lid.md](keypad-lid.md#viable--not-viable)) |
| Identifying the backlight boost, flash LED driver and Hall IC | Unknown | No part names in DT, bootloader, teardowns or FCC documents; needs board photos or continuity tests |
| NFC by adding a chip | Viable only with new hardware | Needs a chip, an antenna, free pins (the template's IRQ and firmware pins are taken) and a HAL that the vendor lacks |
| Pogo dock detection | Viable in software only as a "VBUS without CC partner" signature | Of no practical use: there is no dock that powers the contacts |
| USB OTG powering devices | Not viable without extra hardware | No 5 V boost |
| Modifying signed firmware images (RPM, TZ, modem, ADSP, Wi‑Fi) | **Not viable** | Secure boot fused, production signatures. Configuration outside the images (modem EFS/NV, Wi‑Fi ini/NV) is editable with root |
| Lid-closed deep sleep | **Works** (own build): ~0.6 %/h with a SIM on LTE + IMS | The SoC reaches VDD_MIN with the display off and no USB. Software-side causes on other ROMs are in [power.md](power.md#5-why-lineage-and-gsis-dont-deep-sleep-with-the-lid-closed) |
| Flashlight that stays on | **Fixed (public kernel)** | Torch on GPIO 33 only, flash on 33 + 34 (see [Flash LED](#flash-led)) |
| Camera video recording with a rebuilt kernel | **Fixed (public kernel)** | `msm_vidc` asked for frames 16 KiB larger than the stock v30 vendor allocates (1080p: 3,153,920 vs 3,137,536 bytes), so the first frame was rejected (`qbuf` −22); the encoder firmware only needs 3,133,568 bytes. Now 1080p/720p at ~30 fps with sound on Android 13, 15 and 16; screen recording was never affected |
| Touchscreen with a rebuilt kernel | **Works (public kernel)** | CHSC5448 driver from a third-party GPL kernel: all 84 stock symbols, 5 points, glove mode, ESD heartbeat; the touch firmware is never reflashed |
| Hearing-aid coil amp AW8155 | Driver **Works (public kernel)**; HAC audio untested | Modes set by pulses on its enable pin |
