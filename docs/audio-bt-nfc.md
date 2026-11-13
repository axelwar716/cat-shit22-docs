# Audio, Bluetooth, NFC and pogo pins

Scope: the audio hardware and stock audio path, the Bluetooth chip and A2DP stack, what keeps the SoC awake while music plays (lid closed included), NFC and the rear pogo contacts of the CAT S22 Flip.

Related pages: [hardware.md](hardware.md) (chip and bus inventory), [power.md](power.md) (sleep states, rails, wakeups), [platform.md](platform.md) (stock kernel, vendor and module signing).

A note on sources: the stock firmware is Android 11 (v30) with CAT's vendor. Most of the Bluetooth audio behaviour below was traced on an AOSP-based system (Android 13) running on a vendor whose Bluetooth audio parts were still **CAT's stock v30 ones**, because that is where the stack source can be read. Where a fact has not been checked on stock Android 11, it says so.

---

## 1. Audio hardware

| Block | Part | Connection | Notes |
|---|---|---|---|
| Analog codec | Inside the **PM8916** PMIC | PDM link from the SoC on TLMM 69–74 | Earpiece, wired-headphone path and microphones. Digital part in the SoC (LPASS). Stock vendor modules: analog codec, digital codec and the `sdm450` machine driver |
| Speaker amplifier | **AWINIC AW88194** smart PA (boost + DSP) | I²C-5, address 0x34; reset TLMM 66; IRQ TLMM 59; I²S audio from the SoC; supplied from VBAT | Chip ID register reads 0x1806. Only drives the loudspeaker |
| Hearing-aid coil amplifier | **AWINIC AW8155** | Enable on TLMM 38 (modes are selected by pulses on the EN line) | Hearing-aid compatibility (HAC) path in the machine driver. Low at idle |
| USB-C analog audio switch | **FSA4480** (declared) | I²C-5, address 0x42 | Declared in the DT, but **does not answer on the unit examined** (the stock audio driver logs that no FSA4480 chip was found) |
| USB-C port controller | **WillSemi WUSB3801** | I²C-5, address 0x60; IRQ TLMM 17 | DRP, preferring sink, with accessory support. Reports an analog audio adapter (Audio Adapter Accessory Mode) to the USB power supply |

Other hardware facts:

- The codec's DT node declares a **static 550 mA load on the S4 buck** that applies even when nothing is playing; together with other static loads it keeps S4 in PWM with the screen on. See [power.md](power.md).
- L13 is voted on demand for microphone / USB.
- The DT also carries AW883xx support: the stock audio modules link against AW883xx symbols, but the unit examined has an AW88194 (the AW883xx path only does a handshake).
- No schematic is public (the FCC filing keeps schematic and BOM confidential), so physical routing beyond what the DT declares is unknown.

### Speaker amplifier tuning (AW88194, ACF)

The public kernel README covers the same topic more briefly.

- The stock kernel has AWINIC's AW881xx driver **V1.8.0.2** built in.
- Factory tuning lives in a single ACF file in the vendor firmware. It holds **two complete tuning sets for the same chip**, named `aw88194` and `aw88194A`, each with one profile, "Music".
- The stock driver picks the set **by chip name**, from the DSP PID it reads **on the first playback, with the I²S clock running**. On the unit examined, the DSP reports the PID that selects `aw88194A`. The product ID register (0x01) alone points to the other set, so it can't be used to choose.
- AWINIC's public driver (v1.9) expects exactly one entry per data type/profile. Given the factory file unchanged, it rejects it as redundant and **the speaker stays silent**. Reading the DSP PID while the chip is powered down returns stale values, so a driver that loads the DSP firmware at boot without the audio clock fails its DSP check.
- Speaker calibration is stored in `persist` (`aw_cali.bin`).
- When playback stops, the driver puts the amplifier in power-down (PWDN and AMPPD).

### Stock routing

- **Earpiece** (handset and voice-handset) and **wired headphones** go through the PM8916 analog codec, **not** through the AW88194.
- **Speaker** and **hands-free** (voice-speaker reuses the speaker path) depend on the AW88194.
- On stock, the amplifier always runs the ACF "Music" profile. The vendor also ships per-scene files (speaker / voice / receiver / FM) for both PIDs, but the stock kernel never loads them. They look like leftovers from an older AWINIC driver.
- The stock mixer paths request a mode-switch control on the speaker path that the stock amplifier driver doesn't implement, so the audio route simply skips it.
- The loaded audio policy declares **no wired headset or headphone device**. A copy of the policy that does declare one exists in the vendor, but it isn't used.
- **Headphone jack:** a 3.5 mm jack is **not confirmed** (public spec sheets contradict each other). Headphones over USB-C (3/4-pole detection through FSA4480 + WUSB3801) are supported by the software stack but **unconfirmed in hardware**, because the FSA4480 doesn't answer on the unit examined.

### Compressed offload (speaker)

- The vendor audio HAL supports **compressed offload to the DSP**. An MP3 played through a `MediaPlayer`-based player opens an offload output to the speaker: 64 KB buffers, ~80 ms start latency, no underruns.
- Offload is only used for the **speaker with MP3/AAC**. It is never used for Bluetooth, for headphones, or for apps that decode their own audio (streaming apps included).
- **By default, AOSP's AudioFlinger holds a partial wakelock ("AudioOffload") for the whole offloaded playback**, so the AP can't suspend even though the DSP has ~4 s of audio buffered. The stock vendor doesn't change that default. With the wakelock disabled, playback continues and no other audio wakelock or kernel wakeup source holds the AP. Whether that is glitch-free on battery with the lid closed has **not been verified**.
- There is no system equaliser or effects chain involved in this path.

---

## 2. Bluetooth hardware and stack

| Item | Fact |
|---|---|
| Chip | Qualcomm **WCNSS / Pronto** core inside the QM215, with a WCN36xx-family RF companion (likely **WCN3615**, part not read). Wi‑Fi, Bluetooth and FM share the core. **2.4 GHz only** |
| Pins / firmware | WCNSS lines on TLMM 76–80. The firmware is proprietary and signed (loaded by PIL) |
| Host transport | HCI over **SMD** channels (`APPS_RIVA_BT_CMD` / `APPS_RIVA_BT_ACL`) through `smd_tty`. The vendor HCI HAL is Qualcomm's `android.hardware.bluetooth` (1.0) service |
| Controller ACL buffer | 7 × 1024 bytes (reported by `HCI_Read_Buffer_Size`) |
| Wi‑Fi coexistence | Wi‑Fi and BT share the same 2.4 GHz radio |
| Unsigned config | The editable configuration files of the WCNSS firmware cover **Wi‑Fi only**. There is no unsigned Bluetooth configuration |

### A2DP: no offload, software encoding

- **There is no A2DP offload or split-A2DP on this platform.** The vendor disables both, and the Pronto core has no hardware path for it. The **application processor encodes every frame in software** inside the Bluetooth process and sends it to the chip over SMD.
- The stock vendor's A2DP audio policy is written for **Qualcomm's Bluetooth stack** (the QTI Bluetooth audio HAL), which is what CAT's Android 11 uses.
- **Codecs.** On an AOSP system with the stock vendor: SBC, AAC and LDAC (LDAC comes from AOSP). **No aptX**, because the vendor ships no aptX encoder. The codec set of CAT's stock Android 11 has not been checked.

### AAC on the AOSP stack (stack facts)

- **Bitrate.** The stack negotiates `min(320 kb/s, sink maximum)`, CBR, MPEG-2 AAC-LC, 44.1 kHz stereo. VBR is off by default even when the sink supports it. With a typical 256 kb/s sink this gives 256 kb/s CBR.
- **Packets.** One AAC frame (1024 samples ≈ 23.2 ms) goes in each media packet: ~43 packets/s, each one roughly 0.75 KB on the air and fitting in a single 3‑DH5 packet. A typical sink MTU is around 1 KB.
- **The controller sends one "packets completed" event per packet**, so the host takes ~43 Bluetooth interrupts per second while AAC streams.
- **Cost per packet outweighs cost per byte.** Controller TX time is ~5.2 ms per media packet, against ~2 ms of actual airtime for its payload. Lowering the bitrate without changing the packet count barely changes controller energy, and **the fdk AAC encoder costs the same CPU from 128 to 320 kb/s**.
- **Two frames per packet.** LATM, as used by fdk-aac, can carry up to two AAC frames in one packet (sub-frames). At 256 kb/s two frames don't fit in a ~1 KB MTU. At around 160 kb/s or lower they would. No source confirms that real headphones accept multi-frame LATM packets.
- **SBC vs AAC.** SBC saves a little encoder CPU, but sends more bits over the air (more radio interrupts). The net energy difference is roughly zero, and AAC sounds better.
- **Sink buffer.** Typical sinks report ~150 ms of buffering (AVDTP delay report). That is the real limit on how far encoding can be batched before audible micro-dropouts appear.

---

## 3. Why the SoC never sleeps deeply while Bluetooth music plays

There are two different things here:

- **Suspend** (Linux suspend-to-RAM). It is **impossible while A2DP plays**: the AOSP A2DP source and AudioFlinger hold wakelocks for the whole stream. That is inherent to software encoding.
- **Hardware deep idle** (crystal oscillator off / VDD_MIN, see [power.md](power.md)). Wakelocks don't block it. The RPM can reach it during cpuidle if no master (APSS, Pronto, ...) holds XO. **While A2DP plays, Pronto never releases XO** (0 % XO-off). With headphones connected but **paused** (link in sniff), Pronto releases XO most of the time (roughly 60–90 %, depending on Wi‑Fi activity) and the system reaches VDD_MIN.

### Where the Bluetooth music wakeups come from

1. **The AOSP Bluetooth stack forbids sniff mode while streaming.** This is the root cause of the chip staying awake.
   - When an A2DP stream starts (phone-initiated or headset-initiated), the stack's **AV (A2DP) state machine** disables sniff "unconditionally during streaming". It writes a link policy that only allows hold mode. It re-enables sniff when the stream stops.
   - The Pronto firmware **obeys the host policy**. The same sniff request is accepted when the policy allows it, and is rejected with `0x11` (unsupported feature or parameter value) right after the host has blocked sniff. The "exits from sniff without a host command" seen at stream start are the controller applying that policy.
   - So the link stays in active mode. The controller is awake but idle about **75 % of the time** while streaming (from the controller's own activity-energy reports), and XO never turns off.
   - **The block is in the host stack, not in the chip or its signed WCNSS firmware.**
2. **The vendor Bluetooth audio path polls.** Between AudioFlinger and the Bluetooth process, audio goes through the vendor's Bluetooth audio HAL and session library (AOSP 11 code compiled into the vendor). When the shared queue (FMQ) is full, the writer **sleeps 1 ms and retries**, which allows up to ~1000 wakeups per second. The HAL's short output period (10 ms) also makes AudioFlinger add a FastMixer thread. This is the AOSP-HAL path that AOSP-based systems use. Whether the QTI variant used by stock Android 11 behaves the same has not been checked.
3. **One controller event per AAC frame** (~43/s). These events arrive at a **random phase**, 5–34 ms after each encoder burst. Each one wakes the SMD IRQ, a kworker, the HCI HAL reader and two stack threads.
4. **Independent timers that are not aligned.** The encoder timer, the mixer threads, the player app's own timers, and (on Android 13 SystemUI) the lock-screen media seek bar all fire on their own clocks. The seek bar polls the playback position every 100 ms even with the screen off. Together they give about 100 cluster wakeups per second.
5. **Plumbing, not maths.** The actual audio work (decoding plus AAC encoding plus PCM conversion) is a small fraction of one core. Most of the CPU time goes into passing buffers between threads and processes (binder, FMQ, HCI, SMD).

What does **not** matter: CPU frequency (960 MHz is already the lowest OPP, see [power.md](power.md)), the stack's queue sizes, automatic flush (never set), AFH and QoS, and absolute volume (AVRCP absolute volume is active, so the host does not scale PCM).

### Music with the lid closed, in general

- What keeps the SoC awake while playing: the app, `audioserver`, the vendor audio HAL, the Bluetooth process and the Pronto core. The AP sits in cpuidle (L2 power collapse between bursts), never in suspend.
- Doze, the app freezer and a closed lid do **not** stop BT music on an AOSP system (playback runs as a foreground media service, and the audio wakelock belongs to a system UID).
- Speaker playback through compressed offload is the only music path in which the AP could suspend between buffers.
- Music over SCO (the call path) is technically possible, but it is phone quality (mono, 8/16 kHz).

---

## 4. Bluetooth on non-stock (AOSP / GSI) systems: causes only

These are not stock bugs. They describe what happens when an AOSP system runs on the stock v30 vendor.

- **Only GATT starts, so no audio profiles.** Neither the Android 11 vendor nor a generic AOSP build enables the classic audio profiles (A2DP source, HFP AG, AVRCP target) for this device.
- **Music comes out of the speaker even when headphones are connected.** The vendor's A2DP audio policy loads the QTI Bluetooth audio HAL. That HAL only talks to Qualcomm's stack, not to the AOSP stack, so the A2DP output never opens and AudioPolicy falls back to the speaker. TrebleDroid GSIs ship a "System Wide BT HAL" toggle for exactly this mismatch.
- **Headsets stuck at "Connecting".** The June 2026 AOSP security patch has a regression in encryption-change handling (the CTKD hardening). It breaks classic audio connections that the phone initiates. This is not specific to the S22; the public LineageOS 20 tree has it too.

---

## 5. Music streaming apps on this phone (general facts)

- The userspace is **32-bit only** (`armeabi-v7a`), so every client needs a 32-bit build. The official client's 9.1.x branch still publishes one. Its lite version no longer signs in.
- The official client **decodes inside its own process** and outputs 16-bit PCM at 44.1 kHz on a normal (non-fast, deep-buffer) track through Oboe. So there is no resampling, and DSP offload never applies to it.
- Volume normalisation and the limiter cost a measurable amount of CPU.
- **Offline**, a song started from search has no "next track" without a network connection, so playback just ends. Playback from a downloaded playlist continues.
- After a large download, the client keeps processing in the background for a while, which raises lid-closed consumption noticeably.
- With the lid closed on LTE, the client wakes the cellular radio periodically (about every 10 minutes) even when it isn't playing.
- Only the official client (among maintained clients) plays downloaded music offline. Clients that need a network connection stop when Wi‑Fi or data goes away.

---

## 6. NFC

**The S22 Flip has no NFC.**

- **Kernel:** NFC support is not enabled in the stock config. The DT node `nq@28` (`qcom,nq-nci`, I²C-5 address 0x28) is `disabled`.
- **Vendor and system:**
  - There is no NFC HAL service and no VINTF entry, and Android does not declare the NFC feature.
  - The vendor still carries the generic Qualcomm/NXP reference package (NXP PN5xx/SN100 firmware and configs, and an init service whose binary doesn't exist). That package ships with every reference board, so it doesn't prove that a chip exists.
- **The template's NFC pins have been reused:**

| NFC template signal | Pin | Real use on the S22 |
|---|---|---|
| IRQ | TLMM 17 | WUSB3801 USB-C controller IRQ |
| FIRM / download | TLMM 130 | Keypad backlight |
| VEN | TLMM 16 | Unused |
| CLKREQ | PM8916 GPIO2 | Unused |
| Bus | I²C-5, 0x28 | Free |

  An NFC chip wired as the template describes would clash with the USB-C IRQ and the keypad light.
- **Power cost: none.** The only trace is one failed `exec` at boot.
- Whether the PCB has an empty NFC footprint is unknown (it would need a macro photo of the board).

---

## 7. Pogo pins

The back cover has **two gold-plated pogo contacts**. CAT's spec sheet says they are "for compatible charging docks". No official dock was ever sold.

- **They carry power only: VBUS and GND, in parallel with the USB-C port's VBUS.** Confirmed by a public measurement: a third-party answer read 5.2 V DC between them while the phone charged over USB-C; the DT, the drivers and the spec sheet agree. Looking at the back with the pogos at the bottom, + is on the left. The centres are 8 mm apart.
- The PM8916 linear charger has a **single input (USBIN)**: there is no DCIN and no second charger.
- **Nothing detects a dock.** There is no GPIO, IRQ, ADC channel, extcon, driver, service or factory app for it. With only two contacts there is no room for data (USB needs D+/D−, and Type-C needs CC).
- **The pins are live at 5 V while charging over USB-C.** A metal object across them shorts the charger.
- **Charging through the pogos alone** (unmeasured, from the code): the Type-C controller sees no CC partner, and BC1.2 detection would most likely classify the source as a computer port (SDP), so charging would be slow (100–500 mA).
- The phone **cannot output 5 V**, neither on USB-C nor on the pogos. There is no boost regulator or OTG VBUS supply. USB OTG therefore only works with self-powered devices (not tested with a bus-powered stick).

---

## Viable / not viable

| Item | Verdict | Why |
|---|---|---|
| Speaker with the public AWINIC driver | **Works (public kernel)** | AW881xx v1.9.0; the stock ACF tuning is picked by chip name (`aw88194` / `aw88194A`) like stock V1.8.0.2; the first start reloads with I²S running and reads the DSP PID ("dsp check" passes); calibration in `persist` |
| Wired headphones over USB-C | Viable in software, untested; hardware unconfirmed | WUSB3801 + FSA4480 drivers (3/4-pole detection) are in the public kernel; the FSA4480 doesn't answer on the unit examined and the stock policy has no wired device |
| AP suspend during speaker offload playback | Viable; **works** on a cable (own build), suspend untested on battery | Plays without keeping the CPU awake on the cable; not verified on battery with the lid closed |
| A2DP offload / split A2DP | Not viable | No hardware path in Pronto; the vendor disables it |
| Suspend while BT music plays | Not viable | No hardware A2DP path, so the AP must encode every frame; the stack and AudioFlinger hold wakelocks by design |
| Lower AP CPU use during Bluetooth music | **Works** (own build, AP side only) | Lid closed: CPU 20.8 → 12.1 % of 4 cores, AP XO-off 4.5 → 24 %, 0 lost packets; the BT chip still never releases the XO while streaming. Music with the lid closed, one test (2026-10-19): ~10.9 %/h over the whole test on LTE; offline playback of downloaded music, no data traffic: ~150 mA ≈ 7–9 %/h |
| Pronto releasing XO while BT music plays | Viable, untested | Blocked by the AOSP stack's sniff policy (host side, changeable). Whether the firmware and headsets keep A2DP stable in sniff is untested |
| Two AAC frames per packet | Viable in stack and encoder, untested on headphones | Fits the MTU only at about 160 kb/s or less; sink support for multi-frame LATM is unknown |
| Lower AAC bitrate alone as a power saving | Viable, not worthwhile | Per-packet cost dominates; encoder CPU doesn't change with bitrate |
| SBC instead of AAC for power | Viable, not worthwhile | Net energy about zero, worse sound |
| Lower CPU clock for music | Viable, no gain | 960 MHz is already the lowest OPP |
| Unsigned Bluetooth firmware configuration | Not viable | Doesn't exist: Bluetooth settings live only in the signed firmware; the editable WCNSS files are Wi‑Fi only |
| BT audio on AOSP / GSI with the stock vendor | **Works** (own build): AAC 44.1 kHz 256 kb/s, HFP, AVRCP absolute volume; viable on GSIs | The blockers in section 4 are configuration and stack issues, not hardware |
| Headsets stuck at "Connecting" (June 2026 AOSP CTKD regression) | **Fixed** (own build) | Regression in the AOSP stack, not specific to the S22 (section 4) |
| NFC | Viable only with new hardware | No chip; template pins reused for USB-C and keypad light; would need new hardware, and payment apps also check device integrity (spoofable with caveats, see [platform.md](platform.md#android-versions-and-gsis-over-the-stock-vendor)) |
| Pogo dock detection, data, buttons or 5 V output | Viable only with new hardware | Two power-only contacts, no detection input, no boost; anything more needs new hardware |
