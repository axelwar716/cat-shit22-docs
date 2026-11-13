# Stock firmware, boot, kernel, vendor and ROM viability

Scope: what CAT ships on the S22 Flip (firmware v30 with Android 11 Go, the boot chain, the 4.9 kernel and the vendor partition), the limits these set, and which kinds of platform work are viable or not viable on this phone.

Related pages: [hardware.md](hardware.md) (SoC, PMIC, chips, buses), [power.md](power.md) (sleep, idle and active consumption), [telephony.md](telephony.md) (modem, IMS/VoLTE).

---

## Stock firmware at a glance

| Item | Stock value |
|---|---|
| Android | Android 11 **Go edition**, firmware v30, build `LTE_S02113.11_N_S22Flip_0.030.03` |
| Security patch | 2022-06-05 (vendor and kernel from June 2022) |
| SoC | Qualcomm QM215 (MSM8917-derived; shares Qualcomm's msm8937/8917 platform code), 4× Cortex-A53 |
| ABI | **32-bit everywhere**: ARMv7 kernel (`uname -m` = `armv7l`), `armeabi-v7a` userspace, `zygote32`. The CPU is 64-bit capable, but no shipped software uses it (see [64-bit](#64-bit)) |
| RAM / storage | 2 GB (`MemTotal` about 1.88 GiB), 16 GB eMMC |
| Kernel | Linux `4.9.227-perf`, built June 2022 with a non-public Clang 10.0.7 |
| Treble | Yes. VNDK 30, vendor manifest at FCM target level 4 |
| Memory profile | `ro.config.low_ram=true` (Android Go low-RAM profile) |
| Bootloader chain | `BOOT.BF.3.3.2-00078` (sbl1), `TZ.BF.4.0.5-00264`, `RPM.BF.2.2-00253`, modem `MPSS.JO.3.3-00078` |
| Variant studied | T-Mobile US (see [telephony.md](telephony.md)) |

The contract manufacturer (ODM) is inferred (strong) from the factory test app and kernel symbols. No board or service document confirms it (see [hardware.md](hardware.md)). Bullitt Group, which made CAT phones, entered administration in 2024, and the CAT phone licence later moved to another company. No further firmware or source releases should be expected.

## Partitions and storage layout

- **Not A/B.** There is one `boot`, one `super`, no `_a`/`_b` partitions and no slot suffix. The two metadata slots inside `super` are not two bootable systems. Nothing falls back automatically: a bad `boot` or `super` is fixed by reflashing.
- **Dynamic partitions.** `super` is 2,684,354,560 bytes. Its group `qti_dynamic_partitions` allows 2,680,160,256 bytes and holds `system`, `vendor` and `product`.

| Logical partition (stock v30) | Size |
|---|---|
| `system` | ~1.07 GB |
| `vendor` | ~283 MB |
| `product` | ~1.22 GB |

- With stock `vendor` and `product` left in place, `system` can only grow by ~108 MB. That's why GSIs on this phone need `product` deleted (a public community guide for GSIs does this).
- `userdata` is f2fs with inline-crypto file-based encryption. `metadata` is ext4.
- Data from stock Android 11 can't be reused by an Android 13 ROM, so switching between them needs a data wipe.
- `boot`, `recovery`, `dtbo`, `vbmeta*` and the firmware partitions are written from the bootloader's fastboot. The logical partitions in `super` are written from userspace fastboot (fastbootd).
- **No `fastboot boot`.** The stock bootloader refuses to boot a kernel from RAM (tested with four image variants), so testing a kernel means flashing `boot`. Through lk2nd it is reported working by third parties, who boot images from RAM that way. Writing `boot` takes under a second, but keep a copy of the previous image.
- No custom recovery is available for this phone.

## Boot image and device tree

- `boot` uses Android boot header v2 with 2,048-byte pages: `zImage` + ramdisk + **27 device trees appended**.
- The device tree in use is the QM215 one (`qcom,qm215`, msm-id `0x182`). The DTBO entry in use carries board-id `0x1000b 0x04`.
- Much of the device tree is Qualcomm's reference template. Parts that aren't fitted (Goodix touch, fingerprint, NFC and several alternative touch controllers) are declared anyway. Details are in [hardware.md](hardware.md).
- The stock kernel command line includes `lpm_levels.sleep_disabled=1`. **It doesn't stop sleep**: the vendor's post-boot script reverts it, and the phone reaches the SoC's lowest power state.

## Bootloader, secure boot and verified boot

- **The bootloader can be unlocked.** A public community guide covers unlocking and backing up. When it's unlocked, the kernel command line reports `unlocked` / `orange`.
- **Trust the command line, not the properties.** Userspace properties can report `ro.boot.flash.locked=1` / `green` while the bootloader is actually unlocked (seen on a rooted unit).
- **Secure boot is fused.** The `sec` partition (`mmcblk0p4`, 16 KiB) only holds a factory `secdat` template (`magic ca51723b296f122a`, `"default dat file"`), and reading the physical QFPROM fuse region (`0x00058000`) through `/dev/mem` triggers a synchronous XPU abort and instant reboot. Querying TrustZone directly via the SCM call `TZ_INFO_GET_SECURE_STATE` (`svc = 0x6`, `cmd = 0x4`) returns **`resp = 0x00000050`** (`ret = 0`, `is_secure_device = 0`), and `/dev/ramdump_smem` is absent in Android because `scm_is_secure_device()` evaluates to true. Under Qualcomm's Little Kernel convention (`platform/msm_shared/include/scm.h`, where bit `0` = active/blown and bit `1` = not active), `0x00000050` (`0b0101_0000`) decodes as:

| Bit | Mask | Value in `0x50` | Qualcomm LK / TrustZone field | State on this phone |
|---|---|---|---|---|
| Bit 0 | `0x01` | `0` | `SECBOOT_FUSE` (secure boot authentication) | **Active (fused / enforced)** |
| Bit 1 | `0x02` | `0` | `SHK` (secondary hardware key) | **Provisioned (fused)** |
| Bit 2 | `0x04` | `0` | `DEBUG_DISABLED` | **JTAG / debug disabled by fuse** |
| Bit 3 | `0x08` | `0` | `ANTI_ROLLBACK` | **Active (fused)** |
| Bit 4 | `0x10` | `1` | FEC configuration | Not active |
| Bit 5 | `0x20` | `0` | `RPMB_ENABLED` | **RPMB provisioned and enabled** |
| Bit 6 | `0x40` | `1` | `DEBUG_RE_ENABLED` | **Debug NOT re-enabled** |

- **All firmware stages are signed with a 3-certificate production chain.** Every pre-kernel stage and subsystem image carries Qualcomm's hash segment (`PT_NULL` with `p_flags = 0x02200000`, `QCOM_MI_PBT_HASH_SEGMENT`) and the same 3-certificate DER X.509 chain (issued 9 Oct 2021):
  1. **Root CA (self-signed, 1,055 B):** `CN=QCT Root CA 1, O=QUALCOMM, OU=CDMA Technologies, OU=General Use umx Key (for testing 13 only)`. Its SHA-256 is **`51825348f62d3597cd229601da673eb75399c2c5b7ff5260eab794475eef4e20`**, matching the `PK_HASH` reported by the boot ROM over Sahara bit for bit.
  2. **Intermediate CA (1,040 B):** `CN=QUALCOMM Attestation CA, O=QUALCOMM, OU=CDMA Technologies` (SHA-256 `c44e6c941796839fda9065263edfd9313351564814e826ef0dcb08c05884bda4` across all partitions).
  3. **Leaf attestation certificate (1,163 B):** `CN=SecTools User, O=SecTools`, carrying `OU=02 001060E100190002 HW_ID`, `OU=04 0019 OEM_ID`, `OU=06 0002 MODEL_ID`, `OU=07 0001 SHA256`, `OU=03 0000000000000002 DEBUG` (`2` = production debug disabled) and a per-stage `SW_ID`:

| Partition (`mmcblk0pN`) | Format / ISA | Version string (`QC_IMAGE_VERSION_STRING` / variant) | `SW_ID` (`OU=01`) | Leaf cert offset & SHA-256 |
|---|---|---|---|---|
| `mmcblk0p15` (`sbl1`) | ELF32 ARM | `BOOT.BF.3.3.2-00078` (`PAASANAZA`, `OEM=mmaxServer2`) | `0x00` (SBL1) | `0x1268` · `819224351491b28694714662a37c5e821805e771f29d978d0184a7f56c59ee37` |
| `mmcblk0p17` (`rpm`) | ELF32 ARM | `RPM.BF.2.2-00253` (`AAAAANAAR`, `OEM=mmaxServer2`) | `0x0A` (RPM) | `0x11a8` · `9b80994bce22bfcdffaf8782ab24fad64d7c0629f747660d6d8390de3262ac26` |
| `mmcblk0p19` (`tz`) | ELF64 AArch64 | `TZ.BF.4.0.5-00264` (`ZALAANAAA`, `OEM=CRM`) | `0x07` (TZ) | `0x12c8` · `0cc2c9f15952b9a70eb1d1a17356ec2fbd6c3ddf225c117e5b085ec6082d41d7` |
| `mmcblk0p21` (`devcfg`) | ELF64 AArch64 | TrustZone bus/GPIO config (no plaintext version string) | `0x05` (DEVCFG) | `0x11a8` · `87f98fe43f95fb433b1c18ed28549b60d55ef6f57069dcbfc5de3196bebd33c7` |
| `mmcblk0p43` (`aboot`) | ELF32 ARM | Little Kernel (`app/aboot/aboot.c`, `target/msm8952`) | `0x09` (APPSBL) | `0x11a8` · `970d9f86fb7d2e2afd61a5215da94f4ded433e0e6b621204478fd57fa6539733` |
| `mmcblk0p1` (`modem`) | FAT16 (`NON-HLOS`) | `MPSS.JO.3.3-00078-SDM439_GENNS_PACK-1.419755.4.425674.1` (`8937.genns.prodQ`), `ADSP.VT.3.0-00168-00000-1`, `CNSS.PR.4.0.3-00314`, `VIDEO.VE_ULT.3.1-00049` | `0x01` (MBA), `0x02` (modem), `0x04` (ADSP), `0x0C` (cmnlib), `0x0D` (WCNSS), `0x0E` (Venus) | MBA `0xe808` (`4ea95f32…`) · Modem `0x72788` (`58d098b7…`) |

- **Unlocking only relaxes Android Verified Boot** for `boot`/`dtbo`/`vbmeta`/`super`. Inside its payload, `aboot` (`mmcblk0p43`) embeds four additional X.509 certificates (`0x8118c` `O=Android LK`, `0x8162c` `O=Internet Widgits Pty Ltd`, `0x828e5` and `0xb01e4` `O=Android/OU=Android/CN=Android`) used for the AVB / dm-verity layer. Unlocking (`verifiedbootstate=orange`) tells `aboot` to continue when `AvbSlotVerify` fails; it does not touch the hardware-anchored PBL → `sbl1` → `tz`/`rpm`/`devcfg`/`aboot` signature chain.
- Nothing before the kernel can be changed: `aboot` is signed like the rest of the chain. Once the bootloader is unlocked, `aboot`'s fastboot can be **used** to flash `boot`, `dtbo`, `recovery`, `vbmeta*` and `super`. What can be changed is the kernel, device tree and command line, and then the kernel's power votes to the RPM.
- **Never relock the bootloader with modified software.** A locked `aboot` verifies images against CAT's AVB key inside the signed bootloader (`"Security Error: This phone has been flashed with unauthorized software & is locked."`) and blocks fastboot flashing in the locked state (`"Partition flashing is not allowed"`). Bricks reported by users were with modified images.
- **Never flash older sbl1/TZ/RPM/modem images.** Anti-rollback is fused (`0x50`, bit 3 = `0`).
- **Don't read the QFPROM fuse region through `/dev/mem`.** It is protected by XPU, and reading it aborts and reboots the phone immediately.
- **There is no EDL safety net.** No public loader signed for this OEM/model exists (see below and [telephony.md](telephony.md#modem-and-firmware)). A full partition copy taken while Android runs with root is the only practical backup. That copy contains per-unit calibration and identifiers (`modemst1`, `modemst2`, `fsg`, `fsc`, `persist`), so never restore those partitions to another phone.

---

## Emergency Download Mode (EDL 9008), GPT layout and recovery limits

### 1. Sahara v2 identity measured in EDL (`05c6:9008`)

When the phone is in Qualcomm Emergency Download mode, it enumerates on USB as `05c6:9008` (*Qualcomm HS-USB QDLoader 9008*) speaking **Sahara v2** (`version=2, min=1, max_cmd_pkt_size=0x400, mode=0 image_tx_pending`). Switching the greeting to **Sahara command mode** (`SAHARA_MODE_COMMAND`, read-only without uploading any programmer) returns the SoC's hardware identity and root-key hash:

| Sahara command / parameter | Measured value | Breakdown / meaning |
|---|---|---|
| `USB VID:PID` | `05c6:9008` | Qualcomm HS-USB QDLoader 9008 (PBL / `sbl1` Sahara) |
| `SAHARA_EXEC_CMD_MSM_HW_ID_READ` (`HWID`) | `0x001060e100190002` | `MSM_ID = 0x001060E1` (QM215 / MSM8917) · `OEM_ID = 0x0019` (Bullitt / CAT) · `MODEL_ID = 0x0002` |
| `SAHARA_EXEC_CMD_OEM_PK_HASH_READ` (`PK_HASH`) | `51825348f62d3597cd229601da673eb75399c2c5b7ff5260eab794475eef4e20` (repeated 3× across the 3 root-key slots) | SHA-256 of the self-signed `QCT Root CA 1` certificate anchored in fuses to authenticate boot stages and Firehose programmers |
| `SAHARA_EXEC_CMD_GET_SOFTWARE_VERSION_SBL` (`SBLVER`) | `0x00000000` | SBL software version reported in command mode |

### 2. What puts the phone into EDL 9008

There are two distinct paths into `05c6:9008`:

1. **Software-requested entry with an intact eMMC boot chain (volatile / temporary EDL):**
   - Running `adb reboot edl` from Android writes the emergency-download cookie to the PMIC PON / shared IMEM registers and reboots; the PBL or `sbl1` sees the cookie and starts Sahara on `05c6:9008` before loading `aboot`.
   - `aboot` (`mmcblk0p43`) also contains the `dload` transition strings `"HALT: reboot into dload mode..."`, `"SCM call to check dload mode failed: %x"`, `"dload mode key sequence detected"` and `"Failed to reboot into dload mode"`, while `sbl1` (`mmcblk0p15`) contains `"boot_dload.c"`, `"USB: qhsusb_al_check_for_pbl_dload"`, `"usb:: going for EDL"`, `"sbl1_sahara.c"` and `"Sahara: Hello pkt sent"`. In all software-triggered cases, the eMMC GPT and signed partitions remain untouched.
2. **Forced fallback by the PBL or `sbl1` (persistent EDL / boot-chain brick or hardware strap):**
   - **Failure before `aboot`:** if the mask ROM (PBL) cannot read a valid GPT on the eMMC, cannot find `sbl1` (`type GUID dea0ba2c-cbdd-4805-b4f9-f428251c3e98`), or `sbl1` fails hash/RSA signature or anti-rollback verification, the PBL falls straight into USB EDL (`05c6:9008`). Once `sbl1` runs, it locates the next stages by their primary GPT type GUIDs—`sbl1` embeds the exact little-endian type GUIDs of **`sbl1`** (`dea0ba2c-…`), **`rpm`** (`098df793-…`), **`tz`** (`a053aa7f-…`), **`devcfg`** (`f65d4b16-…`) and **`aboot`** (`400ffdcd-…`). If any of those stages is missing, corrupted or fails TrustZone/PBL authentication, `sbl1` invokes `boot_error_handler.c` → `boot_dload.c` (`"usb:: going for EDL"`) and stays in 9008.
   - **Why corrupting `boot`, `dtbo`, `vbmeta`, `recovery` or `super` does NOT trigger EDL 9008:** as long as `sbl1`, `rpm`, `tz`, `devcfg` and `aboot` are intact and the bootloader is unlocked, the boot chain reaches `aboot`. When `aboot` fails to load or verify `boot`, `dtbo` or `vbmeta`, it executes its recovery path (`"ERROR: Could not do normal boot. Reverting to fastboot mode."` / `"Non Multi-slot: Unbootable entering fastboot mode"`) and stops in **Fastboot Mode** (`18d1:d00d`), not EDL 9008.
   - **Hardware EDL entry (USB D+ strap or board test points):** `sbl1` contains `"USB D+ check, Start"` / `"USB D+ check, Delta"`, which in Qualcomm's boot code detects an EDL cable (USB `D+` shorted to `GND` at power-on), and QM215/MSM8917 boards also have `FORCE_USB_BOOT` / eMMC clock test points. Neither hardware method has been needed or traced on this board.

### 3. Exiting EDL 9008 without flashing (when the boot chain is intact)

- **Software reset trap (measured):** once the phone has entered Sahara command mode, **no USB or Sahara command reboots it out of EDL**:
  - Sending `SAHARA_RESET_REQ` (`cmd_reset()`) returns `SAHARA_RESET_RSP` (`RESET True`), but does **not** reboot the SoC—the Sahara state machine simply sends a new `HELLO` packet waiting for an image on `05c6:9008`.
  - Sending `RESET_STATE_MACHINE` or switching to `image-tx` mode does not reboot the phone either.
  - Issuing a **USB port reset** from the host causes the PBL's USB controller to stop responding (`device not accepting address`), leaving the phone unenumerated and hung until a physical reset.
  - While sitting in EDL 9008, the phone **does not run normal battery charging** (charging control lives in `aboot` and the Android kernel).
- **Physical exits that work:**
  1. **Hold the power button for ~15 seconds (measured, Works):** in the stock v30 device tree, stage S2 of the PM8916 power-key input (`KPDPWR_N`) is configured with `s2-type = 7` (**hard reset**, unlike a third-party dump where `s2-type = 1` warm reset). Holding power for ~15 s makes the PM8916 cycle power to the SoC in hardware, clearing the PBL Sahara state and booting normally.
  2. **Unplug USB and remove the 2000 mAh battery for a few seconds:** because the battery is removable, unplugging USB (so `VBUS` does not keep the PMIC alive) and pulling the battery cuts `VBAT` completely to the PM8916 and QM215, clearing volatile SRAM/PON state.
  3. **Stop in Fastboot Mode (`Power + Volume Down`):** if the boot chain (`sbl1`–`aboot`) is healthy but `boot`, `dtbo`, `vbmeta` or `super` is broken, holding **Power + Volume Down** while turning the phone back on stops in `aboot`'s **FASTBOOT MODE** (`18d1:d00d`) so those partitions can be reflashed.
- **When button/battery resets do NOT exit 9008:** if the GPT or any signed pre-kernel stage (`sbl1`, `rpm`, `tz`, `devcfg`, `aboot`) is corrupted on the eMMC, every cold boot fails verification again and drops straight back into `05c6:9008`.

### 4. Why EDL 9008 cannot be used to flash or unbrick (Firehose loader audit)

Because `SECBOOT_FUSE` is blown (`0x50`, bit 0 = `0`), the PBL verifies any Firehose programmer (`prog_emmc_firehose_*.mbn`) before executing a single instruction in RAM: its root certificate must hash to `PK_HASH = 51825348f62d3597…`, its leaf certificate must carry `SW_ID = 0x03` (Firehose host tool) and `HW_ID = 0x001060e100190002` (`OEM_ID 0x0019` Bullitt, `MODEL_ID 0x0002`), and its hash segment (`0x02200000`) must match every ELF segment.

All known public loader repositories and S22 firmware packages were audited against those requirements:

| Source / file inspected | Scope / contents | Result against `HWID 0x001060e100190002` & `PK_HASH 51825348f62d3597…` | Verdict |
|---|---|---|---|
| Public Firehose loader collection | 878 programmers | 0 matches for `PK_HASH 51825348f62d3597…`; for `001060e1` only 2 loaders of another OEM exist (`OEM_ID 0x0016`) | **Rejected by PBL** |
| Global index of public loaders | 2,723 loaders (1,526 unique) across 7 public collections | 0 matches for `PK_HASH 51825348f62d3597…` (SHA-256 or SHA-384) and 0 loaders for `OEM_ID 0x0019` (Bullitt/CAT); `001060e1` only has loaders of two other OEMs (`0x0042`, `0x0016`) | **Absent from public collections** |
| QM215 Firehose loader from a public collection (372,524 B, SHA-256 `2ff021dd…`) | ELF32 Firehose (`BOOT.BF.3.3.2-00055`, May 2020) | Signed by another OEM (`OU=04 0004 OEM_ID`, `OU=02 201060E100040000 HW_ID`, `PK_HASH 168d0bad52b5d5cb…`) | **Rejected (wrong OEM & PK hash)** |
| MSM8917/QM215 Firehose loader from a public collection (365,452 B, SHA-256 `772622e9…`) | ELF32 Firehose for MSM8917/QM215 | Signed by another OEM (`OU=04 0016 OEM_ID`, `OU=02 001060E100160000 HW_ID`, `PK_HASH d5f6c6ec4f4092dc…`) | **Rejected (wrong OEM & PK hash)** |
| Third-party stock dump from another S22 unit (8 partitions) | Contains only `boot`, `dtbo`, `vbmeta`, `vbmeta_system`, `dsp`, `modem`, `super`; **no** `.mbn` Firehose loader and no `sbl1`/`rpm`/`tz`/`aboot` | **No loader included** |
| Firehose XML exploits (`peek`/`poke`, `CVE-2019-2317`) | Vulnerabilities in the Firehose XML command parser | Sahara runs inside the PBL *before* Firehose loads; XML exploits require a validly signed Firehose ELF to be accepted and running in RAM first | **Not applicable without a signed loader** |

Because the ELF segments are covered by the RSA-signed hash table and `OEM_ID`/`HW_ID` sit inside the leaf certificate signed by `QUALCOMM Attestation CA`, foreign QM215 loaders cannot be patched, nor can `sbl1`'s certificate chain (`SW_ID = 0x00`) be transplanted onto a Firehose binary (`SW_ID = 0x03`).

### 5. Stock v30 GPT partition table, `*bak` slots and restoration paths

The 16 GB eMMC (`15,758,000,128` bytes = `30,777,344` 512-byte LBAs = `15,388,672` 1-KiB blocks) carries a 56-partition GPT (`EFI PART` revision `0x00010000`, primary header at `LBA 1`, entries at `LBA 2..33`, usable range `LBA 34..30777310`, backup entries at `LBA 30777311..30777342`, backup header at `LBA 30777343`). Below are the critical boot, firmware and system partitions measured on stock v30 (`LTE_S02113.11_N_S22Flip_0.030.03`):

| Node (`/dev/block/…`) | GPT name | LBA start .. end | Size (`KiB` / `bytes`) | GPT type GUID | Stock v30 `sha256_raw` | Primary vs `*bak` status |
|---|---|---|---|---|---|---|
| `mmcblk0p1` | `modem` | `16384 .. 188415` | `86,016 KiB` (`88,080,384 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `25b79220c70187b7…` | No `*bak` partition |
| `mmcblk0p2` | `DDR` | `188416 .. 188479` | `32 KiB` (`32,768 B`) | `20a0c19c-286a-42fa-9ce7-f64c3226a794` | `50e9fe24b9942b10…` | No `*bak` (trained DDR parameters) |
| `mmcblk0p4` | `sec` | `191552 .. 191583` | `16 KiB` (`16,384 B`) | `303e6ac3-af15-4c54-9e9b-d9a8fbecf401` | `d97cd192d0a410fa…` | No `*bak` (template `secdat`) |
| `mmcblk0p5` | `cmnlib` | `191584 .. 193631` | `1,024 KiB` (`1,048,576 B`) | `73471795-ab54-43f9-a847-4f72ea5cbef5` | `4d0cf22e35941bd0…` | Bit-identical to `cmnlibbak` |
| `mmcblk0p6` | `cmnlibbak` | `193632 .. 195679` | `1,024 KiB` (`1,048,576 B`) | `73471795-ab54-43f9-a847-4f72ea5cbef5` | `4d0cf22e35941bd0…` | Same hash & same type GUID as `cmnlib` |
| `mmcblk0p7` | `cmnlib64` | `195680 .. 197727` | `1,024 KiB` (`1,048,576 B`) | `8ea64893-1267-4a1b-947c-7c362acaad2c` | `1c376898db68032f…` | Bit-identical to `cmnlib64bak` |
| `mmcblk0p8` | `cmnlib64bak` | `197728 .. 199775` | `1,024 KiB` (`1,048,576 B`) | `8ea64893-1267-4a1b-947c-7c362acaad2c` | `1c376898db68032f…` | Same hash & same type GUID as `cmnlib64` |
| `mmcblk0p9` | `keymaster` | `199776 .. 201823` | `1,024 KiB` (`1,048,576 B`) | `e8b7cf6e-5694-4627-8a2a-899b09f2dbea` | `384aaf59472db000…` | Bit-identical to `keymasterbak` |
| `mmcblk0p10` | `keymasterbak` | `201824 .. 203871` | `1,024 KiB` (`1,048,576 B`) | `e8b7cf6e-5694-4627-8a2a-899b09f2dbea` | `384aaf59472db000…` | Same hash & same type GUID as `keymaster` |
| `mmcblk0p11` | `devinfo` | `203872 .. 205919` | `1,024 KiB` (`1,048,576 B`) | `1b81e7e6-f50d-419b-a739-2aeef8da3335` | `30e14955ebf13522…` | All-zero on raw eMMC; unlock state lives in RPMB |
| `mmcblk0p15` | `sbl1` | `213016 .. 214039` | `512 KiB` (`524,288 B`) | `dea0ba2c-cbdd-4805-b4f9-f428251c3e98` | `efa11b0bfda6d747…` | Bit-identical to `sbl1bak` |
| `mmcblk0p16` | `sbl1bak` | `214040 .. 215063` | `512 KiB` (`524,288 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `efa11b0bfda6d747…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p17` | `rpm` | `215064 .. 216087` | `512 KiB` (`524,288 B`) | `098df793-d712-413d-9d4e-89d711772228` | `4ee68b08e7922f57…` | Bit-identical to `rpmbak` |
| `mmcblk0p18` | `rpmbak` | `216088 .. 217111` | `512 KiB` (`524,288 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `4ee68b08e7922f57…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p19` | `tz` | `217112 .. 221207` | `2,048 KiB` (`2,097,152 B`) | `a053aa7f-40b8-4b1c-ba08-2f68ac71a4f4` | `ab47f2a7c6210e89…` | Bit-identical to `tzbak` |
| `mmcblk0p20` | `tzbak` | `221208 .. 225303` | `2,048 KiB` (`2,097,152 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `ab47f2a7c6210e89…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p21` | `devcfg` | `225304 .. 225815` | `256 KiB` (`262,144 B`) | `f65d4b16-343d-4e25-aafc-be99b6556a6d` | `48f2a1e8c7c87400…` | Bit-identical to `devcfgbak` |
| `mmcblk0p22` | `devcfgbak` | `225816 .. 226327` | `256 KiB` (`262,144 B`) | `f65d4b16-343d-4e25-aafc-be99b6556a6d` | `48f2a1e8c7c87400…` | Same hash & same type GUID as `devcfg` |
| `mmcblk0p23` | `dsp` | `226328 .. 259095` | `16,384 KiB` (`16,777,216 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `f37412ebc2e81247…` | No `*bak` partition |
| `mmcblk0p27` | `misc` | `330776 .. 332823` | `1,024 KiB` (`1,048,576 B`) | `82acc91f-357c-4a68-9c8f-689e1b1a23a1` | `30e14955ebf13522…` | Bootloader Control Block (`boot-recovery`, `boot-fastboot`) |
| `mmcblk0p43` | `aboot` | `1146880 .. 1148927` | `1,024 KiB` (`1,048,576 B`) | `400ffdcd-22e0-47e7-9a23-f16ed9382388` | `ff5411fc1bc3aa91…` | Bit-identical to `abootbak` |
| `mmcblk0p44` | `abootbak` | `1148928 .. 1150975` | `1,024 KiB` (`1,048,576 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `ff5411fc1bc3aa91…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p45` | `dtbo` | `1150976 .. 1167359` | `8,192 KiB` (`8,388,608 B`) | `24d0d418-d31d-4d8d-ac2c-4d4305188450` | `59c4c70c21fd10fe…` | Bit-identical to `dtbobak` |
| `mmcblk0p46` | `dtbobak` | `1167360 .. 1183743` | `8,192 KiB` (`8,388,608 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `59c4c70c21fd10fe…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p47` | `vbmeta` | `1183744 .. 1183871` | `64 KiB` (`65,536 B`) | `4b7a15d6-322c-42ac-8110-88b7da0c5d77` | `94917aab80a51c37…` | Bit-identical to `vbmetabak` |
| `mmcblk0p48` | `vbmetabak` | `1183872 .. 1183999` | `64 KiB` (`65,536 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `94917aab80a51c37…` | Same hash; **inactive** generic type GUID (`ebd0a0a2-…`) |
| `mmcblk0p49` | `vbmeta_system` | `1184000 .. 1184127` | `64 KiB` (`65,536 B`) | `1344859d-3a6a-4c14-a316-9e696b3a5400` | `f614009f9a18a0b0…` | **Differs in 427 B** from `vbmeta_systembak` (v30, `2022-06-05`, rollback `1654387200`) |
| `mmcblk0p50` | `vbmeta_systembak` | `1184128 .. 1184255` | `64 KiB` (`65,536 B`) | `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` | `1d13342070dad4a3…` | Retains previous **v27** (`0.027.03`, `2021-09-05`, rollback `1630800000`); inactive GUID |
| `mmcblk0p51` | `boot` | `1184256 .. 1315327` | `65,536 KiB` (`67,108,864 B`) | `20117f86-e985-4357-b9ee-374bc1d8487d` | `ede091b6c01e234b…` | No `*bak` partition |
| `mmcblk0p52` | `recovery` | `1315328 .. 1446399` | `65,536 KiB` (`67,108,864 B`) | `9d72d4e4-9958-42da-ac26-bea7a90b0434` | `14f3e56ae0468867…` | No `*bak` partition |
| `mmcblk0p55` | `super` | `1654784 .. 6897663` | `2,621,440 KiB` (`2,684,354,560 B`) | `97d7b011-54da-4835-b3c4-917ad6e73d74` | `e6708df8feba3a20…` | Dynamic partitions (`system`, `vendor`, `product`) |

- **Why the `*bak` partitions do not save a corrupted `sbl1` or `aboot`:** in `sbl1bak` (`p16`), `rpmbak` (`p18`), `tzbak` (`p20`), `abootbak` (`p44`), `dtbobak` (`p46`), `vbmetabak` (`p48`) and `vbmeta_systembak` (`p50`), the GPT type GUID is set to the generic data GUID `ebd0a0a2-b9e5-4433-87c0-68b6b72699c7` rather than an active boot GUID. Qualcomm's OTA updater uses the `*bak` slots for staging/rollback during an OTA (as seen in `vbmeta_systembak` still holding the v27 image after the v30 OTA) and swaps the GPT type GUIDs during the update transaction. If `sbl1` or `aboot` is overwritten in place while its primary type GUID still points to `p15`/`p43`, the boot ROM will not automatically fail over to `sbl1bak`/`abootbak`.
- **Restoration paths by scenario:**
  1. **Phone in EDL 9008 by software command or hang, with `sbl1`–`aboot` intact (Works):** hold Power for ~15 s (or unplug USB and pull the battery). If `boot`, `dtbo`, `vbmeta` or `super` is broken, stop in `aboot` Fastboot Mode (`18d1:d00d`)—use a Linux host, as macOS fails to configure the `18d1:d00d` USB interface (`< waiting for any device >`)—and flash stock `vbmeta`, `vbmeta_system`, `dtbo`, `boot`, `recovery` and `super` (`fastboot -S 256M flash super super-v30-sparse.img` or from `fastbootd`), plus `fastboot -w` when crossing Android versions. With the bootloader unlocked, `aboot` blocks fastboot writes to critical loader partitions (`"Critical partition flashing is not allowed"`) while allowing `boot`/`dtbo`/`vbmeta*`/`recovery`/`super`.
  2. **Phone stuck in persistent EDL 9008 due to corrupted GPT or `sbl1`/`rpm`/`tz`/`devcfg`/`aboot`:**
     - **Over USB (EDL / Sahara / Firehose):** **Not viable** (no public Firehose programmer signed for `OEM_ID 0x0019` / `PK_HASH 51825348f62d3597…`).
     - **Direct hardware eMMC access (ISP in-circuit wiring or chip-off, with a prior stock v30 dump):** **Viable, untested.** The boot chain and system partitions (`gpt-first-4MiB`, `gpt-last-4MiB`, `sbl1`, `rpm`, `tz`, `devcfg`, `cmnlib`, `cmnlib64`, `keymaster`, `aboot`, `boot`, `dtbo`, `vbmeta*`, `super`) are not encrypted per-unit on the eMMC and can in principle be written directly over the eMMC bus (`CLK`, `CMD`, `DAT0`, `VCC`/`VCCQ`) bypassing the SoC's PBL. Per-unit partitions (`modemst1`, `modemst2`, `fsg`, `fsc`, `persist`) and `mmcblk0rpmb` are tied to the individual phone and must never be overwritten from another unit.

### 6. Can the SoC boot chain (PBL / `sbl1` / TrustZone / `aboot`) be jailbroken?

A boot-chain jailbreak would mean running unsigned code before the Linux kernel (`PBL` → `sbl1` → `rpm`/`tz`/`devcfg` → `aboot`) or bypassing `SECBOOT_FUSE` (`0x50`) to load an unsigned Firehose programmer or modified `sbl1`/`tz`/`rpm`/`aboot`/`modem`/`adsp` images:

1. **Vector 1 — BootROM (PBL) USB Sahara exploit (boot-ROM style): Not viable.** The PBL sits in mask ROM inside the QM215 silicon. Bugs of that class belong to other vendors' USB boot ROMs, not to Qualcomm's Sahara state machine. No public memory-corruption vulnerability exists in the QM215/MSM8917 PBL or Sahara v2 handler that bypasses hash-segment (`0x02200000`) and X.509 (`PK_HASH = 51825348f62d3597…`, `HWID = 0x001060e100190002`) verification, and Firehose XML bugs (`peek`/`poke`, `CVE-2019-2317`) only exist inside the Firehose binary after the PBL has already authenticated and loaded it.
2. **Vector 2 — Historical `sbl1` / QSEE TrustZone exploits (`CVE-2017-5626`, `CVE-2016-5349`, `CVE-2015-6639`) and downgrades: Not viable.** All stock v30 bootloader and TrustZone stages (`BOOT.BF.3.3.2-00078`, `RPM.BF.2.2-00253`, `TZ.BF.4.0.5-00264`, `devcfg`, `cmnlib*`, `keymaster`, `aboot`) were built and signed on **9 Oct 2021** (`OEM_IMAGE_UUID_STRING = ..._20211009_1304`), years after the 2015–2018 SBL1 ELF-segment and QSEE privilege-escalation bugs were patched. Downgrading to an older vulnerable build is blocked by the blown **`ANTI_ROLLBACK`** fuse (`0x50`, bit 3 = `0`), enforced in `sbl1` (`boot_rollback_version.c`) and `tz` (`/secboot/anti_rollback` + RPMB rollback counters), and no older image from another vendor carries the `0x0019` signature.
3. **Vector 3 — Debug policy blobs (`apdp` / `msadp` / `dpo`) and JTAG: Not viable.** In `0x50`, `DEBUG_DISABLED = 0` (blown) and `DEBUG_RE_ENABLED = 1` (not re-enabled), and every leaf certificate has `OU=03 0000000000000002 DEBUG` (disabled). On stock v30, `mmcblk0p36` (`apdp`, 256 KiB), `mmcblk0p37` (`msadp`, 256 KiB) and `mmcblk0p38` (`dpo`, 8 KiB) are **completely empty (all zero bytes)**; enabling debug through `apdp` would require a debug policy ELF signed by Bullitt's private root key (`PK_HASH 51825348f62d3597…`).
4. **Vector 4 — Modifying `aboot` (`mmcblk0p43`) or `fastboot flashing unlock_critical`: Not viable for signature bypass.** Although root in Android can write to `/dev/block/mmcblk0p43`, `sbl1` verifies `aboot`'s hash segment and X.509 certificate (`SW_ID = 0x09`) on every boot; flipping a single bit in `aboot` drops the phone into permanent EDL 9008. `aboot` does contain the command `flashing unlock_critical` (at `0x895b0`) and stores `device_info` (`ANDROID-BOOT!`, `is_unlocked`, `is_unlocked_critical`) in **RPMB** via TrustZone (`rpmb.c` / `rpmb_listener.c`, which is why the raw 1 MiB `devinfo` partition `mmcblk0p11` reads as all zeros), but `unlock_critical` only governs which partitions `aboot` lets `fastboot` write—it does **not** disable `sbl1` or PBL signature checks. Legacy Little Kernel boot-header integer overflows (`CVE-2014-9798` family) are also patched in the v30 `aboot` (`"Integer overflow detected in bootimage header fields"`, `"Boot image buffer address overlaps with aboot addresses"`, `"Signature read buffer address overlaps with aboot addresses"`).
5. **Vector 5 — Hardware voltage fault injection (*voltage glitching*): Viable only with invasive lab hardware, untested.** Glitching `VDD_CORE` / `VDD_MX` precisely during eMMC read or Sahara RSA verification can in theory skip a branch in the PBL or `sbl1`, but requires board teardown, shield/decoupling-capacitor removal, trigger wiring to an FPGA glitcher/oscilloscope and blind silicon characterization with high physical risk and no published parameters for this phone.
6. **What DOES work without a boot-chain jailbreak — chainloading from `boot` (Works / Reported working):** once unlocked via `fastboot flashing unlock`, the stock signed `aboot` logs `"State: Unlocked, AvbSlotVerify returned %s, continue boot"` and jumps in non-secure `EL1` (`SVC`) to whatever image is flashed in `boot` (`mmcblk0p51`), including its built-in CAF `Image.gz` decompression and 64-bit jump path. That allows flashing a custom kernel directly (**Works**) or chainloading a secondary open-source bootloader (`lk2nd`) inside `boot` (**Reported working** on the mainline Linux port, providing a modern Fastboot with working `fastboot boot` from RAM and booting 64-bit mainline kernels) without touching `sbl1`/`tz`/`rpm`/`aboot`. Because execution starts after `aboot` in non-secure `EL1`, neither a custom kernel nor `lk2nd` can modify TrustZone (`EL3`), RPM firmware or the signed modem/ADSP/WCNSS images.

## Kernel

- **CAT never released the kernel source** for the S22 Flip, and none is publicly available. Community Android trees for the phone all use the prebuilt stock kernel.
- **The whole stock kernel has been decompiled here, with its real symbol names recovered.** Most of it is Qualcomm's public 4.9 source, so together with that source it reads close to CAT's own: the decompile fills in CAT's additions, and the OEM drivers CAT didn't publish were checked or rewritten against it. It isn't the original source (no comments, local names or build files), and nothing from it is published.
- **The stock kernel isn't bare.** It has `PREEMPT`, WALT, `NO_HZ_IDLE`, CPUFreq/CPUIdle with per-core and per-cluster power collapse, `schedutil`, bus DCVS, suspend, zram, memory cgroups, PSI and binder. Debug options (`DEBUG_FS`, `FTRACE`, `SCHED_DEBUG`) are on, with no proven overhead in normal use.
- **The stock kernel loads 25 signed modules** (24 audio, plus the `pronto_wlan` Wi-Fi driver). They're signed with CAT's RSA-4096/SHA-512 key, signature enforcement is on, and they're bound to the exact 4.9 ABI (`vermagic 4.9.227-perf`, modversions).
- **A rebuilt 4.9 kernel works.** It runs daily on this phone, and a public rebuild exists: [axelwar716/cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel). Its README documents the kernel base (Qualcomm `LA.UM.9.8.c26`, 4.9.227), why only Clang reproduces the stock module CRCs, how the 25 stock modules resolve all 730 imports, which OEM drivers CAT didn't publish (touch, Hall, USB-C, audio amps, keypad light) and where they were recovered from, and the stock keypad sampling problem. The rest of this section only adds what that README doesn't cover.
- **Video recording and the flashlight break on every rebuild from the Qualcomm base**, on every ROM; CAT's stock kernel is fine. **Fixed (public kernel)** in `stock-stable-5`: details in [hardware.md](hardware.md#flash-led) and the kernel README.
- **Qualcomm's public 4.9 tree built with CAT's stock config is not a stock-equivalent kernel.** CAT's config has 42 options whose code only exists in CAT's unpublished source, so the build tool silently drops them. 16 of them were enabled, among them the lid (Hall) sensor, the USB-C controller, the keypad light and the audio amplifiers. The touch driver has no source there either. The public kernel adds them back (see its README).
- **Mainline as a daily kernel: reported working as a Linux phone, but even at its best it would not cut battery drain measurably.** Third parties report a mainline Linux system running daily on this phone without Android: VoLTE calls, texts, mobile data, Wi-Fi, Bluetooth, both screens, cameras and lid-closed suspend. Its reported overnight figure is **about 1 %/h with the lid closed** (read from voltage, because the gauge freezes). That is not below the rebuilt 4.9's 0.6 %/h; network, SIM and Wi-Fi differ, so it is not a like-for-like comparison. For Android, the best case is assumed below: suspend, radio, both screens and every driver working as well as on the rebuilt 4.9.
  - **Lid closed (~12 mA, 0.6 %/h):** the rebuilt 4.9 already keeps the AP asleep 99.8 % of the time, with the SoC in VDD_MIN 84–91 % of it. What is left is the SIM's LTE paging (modem firmware and the network) and the retention floor of the SoC, DRAM and PMIC (hardware and RPM firmware), the same under any kernel. What a kernel decides, awake time plus wakeups, is about 0.1–0.25 mA at most (estimate), around 0.01 %/h.
  - **Screen on (187 mA in the synthetic test):** most of it is the video-mode main panel path and its backlight, the time LTE stays connected, and the CPU, GPU and bus on the same frequency and voltage tables, set by the SoC and its fused CPR. The four A53s are a single cluster, so a newer energy-aware scheduler has no smaller cores to move work to. What the AP can still change on the rails adds up to ~1.5–5 mA at best (estimate, below what the gauge resolves), and it can be done on 4.9 as well.
  - **Its memory features** (MGLRU, a newer zram) target smoothness under RAM pressure, not drain. Android 13's app freezer already works on the own build.
  - **Its only real gain is software lifetime** (newer kernel, security fixes). It also can't run CAT's vendor, so for Android radio/IMS, camera, audio and the outer screen would all have to be redone. Suspend reaching VDD minimisation needed several power-management fixes that exist only in the device-specific tree of that port, not in the upstream tree.
- **Kernel 4.19 / GKI: not worthwhile.**
  - GKI's module interface is only stable within one GKI family, so it can't host CAT's 4.9 modules.
  - Upstream 4.19 reached end of life in Dec 2024.
  - Nobody is known to run 4.19 on a stock 32-bit Android 11 QM215 vendor.
  - The only public 4.19 attempt for the S22 Flip (Oct 2026) is Qualcomm's generic msm8937 tree at 4.19.325, the last 4.19 release, built as arm64. Its device tree points to a prebuilt kernel that isn't published. Its 64-bit vendor is not CAT's: it mixes S22 files (speaker amp firmware) with blobs from another msm8937 phone, including camera tuning for sensors this phone doesn't have. As of 7 Nov 2026, that effort has also added bring-up commits to the 4.19.325 tree (an outer-screen SPI display driver that sends each frame synchronously without DMA, a touch driver and an audio amplifier driver), plus a set of 64-bit userspace daemons. The S22 device tree files and the Hall-sensor driver that its Makefiles and build scripts reference are not in the published tree, so it doesn't build as published. Its boot-image script disables deep CPU idle and boots with SELinux permissive. There is still no release, no boot log and no power measurement. It depends on a 64-bit vendor that isn't CAT's, so the phone's camera tuning and signed modules are lost and RAM pressure on 2 GB goes up.
  - No published measurement shows a power benefit for 4.19 or mainline over 4.9 on this phone: the one mainline figure (about 1 %/h lid closed, above) is not lower.
- **Updating within 4.9** (4.9.227 to 4.9.337) would bring security fixes, not power savings, at high cost.

## Vendor

- **Rebuilt vendor: works (own build).** The own build runs daily on a vendor rebuilt from stock v30 keeping CAT's proprietary radio/IMS, camera, graphics, outer-screen compositor and firmware blobs.
- **CAT published no newer vendor.** The stock partition is from June 2022 (VNDK 30). That's why the own vendor is rebuilt from v30: one built from another phone's blobs would lose CAT's device-specific ones, which are exactly what makes the phone work.
- **The vendor holds the HALs and firmware for radio/IMS, Wi-Fi, camera, audio and the display compositor.** The compositor sees the outer screen as a second display.
- **What's closed falls into two groups:**
  - **Signed firmware** (bootloader stages, modem, ADSP, Wi-Fi/BT firmware, video codec). The images are untouchable, and nothing in them can be trimmed for power. Configuration kept outside the signed images (modem EFS/NV, Wi-Fi ini/NV files) is editable with root.
  - **Closed but editable vendor code and config** (Qualcomm's perf/boost HAL, `thermal-engine`, IRQ balancer, sensor registry, init scripts). Changes here affect performance peaks and behaviour, not the idle floor.
- **A modified vendor mounts with an unlocked bootloader.** `vendor` isn't protected by dm-verity then. Rebuilding the vendor from scratch is viable but a very large effort: it would mean replacing proprietary radio/IMS, audio, camera, sensors, Wi-Fi and compositor HALs, and crossing the Treble boundary.
- **Stock vendor gaps** (stock facts):
  - There's no `android.hardware.power` HAL, only Qualcomm's own perf HAL.
  - There's no Thermal HAL: Android's thermal service reports "HAL Ready: false", and `thermal-engine` acts on its own.
  - There's no PowerStats HAL.
  - The vendor sets ART heap limits of 196m/512m, which override Android Go's smaller defaults.
  - The vendor's `ro.lmk.use_psi=false` is ignored on Android 13 (SELinux rejects it), but it hangs Android 16 (see the kernel README).

## Android versions and GSIs over the stock vendor

- **Android 13 over the stock-based vendor works.** The own build (LineageOS 20) is the daily ROM. Over the untouched stock vendor and a stock-config 4.9 kernel it's statically compatible too (`checkvintf`: compatible), it boots, and the vendor HALs work.
- **Only 32-bit GSIs work** (`a64` / `arm32_binder64`). `arm64` GSIs don't boot.
- **Android 13, 15 and 16 GSIs work on the public kernel** (**Works (public kernel)**, tested by its author on the stock v30 vendor): a LineageOS 20 GSI (Android 13), VoltageOS 4.5 (Android 15) and TrebleDroid (Android 16), all with SELinux enforcing, 1080p video recording and the flashlight checked. Stock Android 11 was checked with `stock-stable-4`. Android 16 on the stock v30 vendor needs `ro.lmk.use_psi=true`, or `lmkd` exits in a loop and the system hangs (the [kernel README](https://github.com/axelwar716/cat-shit22-kernel) explains it). On a GSI, Android doesn't see the outer screen as a display; only the kernel clock is there.
- **Beyond Android 14 the vendor manifest has to be rewritten.** AOSP dropped the compatibility matrix for FCM level 4 in Android 15. Legacy device trees handle this at the manifest level, with no kernel change.
- **Android 16 needs eBPF features** that the official LineageOS policy only allows backported down to 4.14. For a ROM built for this phone on 4.9, Android 15 is the practical ceiling, and Android 16 only with unofficial patches. An Android 16 GSI already boots on the public kernel with one vendor property fix (above).
- A newer Android isn't expected to bring useful features for this phone, and probably costs more battery (not measured).
- **A public third-party Android 15 build** for this phone runs on CAT's stock kernel and the stock vendor. It replaces kernel-side fixes with root daemons in userspace (keypad debounce, keypad light by lid, outer-screen drawing, PTT key), adds a CPU boost script and runs SELinux permissive. Nothing in it changes the power behaviour described in [power.md](power.md). Daemons that watch the lid or keys add wakeups compared with doing the same in the kernel (not measured). A 64-bit variant of the same approach exists for arm64 GSIs on a 4.19 kernel: its outer-screen daemon draws to the framebuffer from userspace for a 10-second window after the lid closes or the PTT key is pressed, and polls once a second the rest of the time. The same caveat applies: a kernel-side clock needs no such wakeups.
- Lid-closed deep sleep on Lineage/GSIs is a separate topic, covered in [power.md](power.md).
- **VoLTE** on non-stock Android is covered in [telephony.md](telephony.md#volte--ims-on-non-stock-software).
- **microG instead of Google apps: works** (own build). LineageOS 20 already includes the restricted signature spoofing that microG needs, so no framework patch is needed.
- **Apps that check device integrity** (some banking and payment apps): hiding root and spoofing the build properties and fingerprint passes the basic and device integrity levels (viable, untested here). The strong, hardware-backed level only passes with leaked attestation keys, which Google revokes. Some apps add their own root or server-side checks. Relocking isn't an option (see above).

## 64-bit

The QM215's Cortex-A53 cores are 64-bit capable, but everything CAT ships is 32-bit (see the table at the top). There are two different ways to move to 64-bit, and they don't cost the same.

| | A. 64-bit kernel, 32-bit userspace and vendor | B. Everything 64-bit (kernel, system, vendor) |
|---|---|---|
| What it means | An arm64 kernel runs CAT's 32-bit vendor and a 32-bit Android (the A53 runs AArch32 user code) | 64-bit system and vendor; 32-bit apps only as a fallback |
| What it needs | Every kernel module rebuilt as arm64: CAT's 25 signed modules (audio, Wi-Fi) are ARMv7 and can't load. Every vendor driver must accept 32-bit calls from a 64-bit kernel (GPU, display, camera, audio, IPA/RMNET, secure channel). Qualcomm's downstream drivers have these paths; CAT's OEM drivers would have to be checked one by one | A 64-bit vendor. None exists for this phone, so it would come from another phone's blobs and lose CAT's device-specific ones (see [Vendor](#vendor)) |
| Gains | Kernel hardening that Android kernels only offer on arm64 (KASLR, Shadow Call Stack, Clang CFI). All RAM mapped directly: the stock 32-bit kernel uses a 3 GB/1 GB split with highmem, so only about 0.7–0.8 GB is permanently mapped. The mainline work for this SoC family is arm64 | Apps that are only published for arm64. The Qualcomm IMS packages distributed for Android 14+ are arm64-only (see [telephony.md](telephony.md#volte--ims-on-non-stock-software)). Some speed in native 64-bit code |
| Costs | Large porting effort; any driver without a working 32-bit path breaks that function | More RAM on a 2 GB phone: 8-byte pointers, larger code than 32-bit Thumb-2 and, while 32-bit apps remain, a second zygote. Android's low-RAM (Go) profile favours 32-bit userspace for this reason |
| Battery | No change expected: the floor is set by the radio, panels, PMIC and battery ([power.md](power.md#9-energy-floor-and-ceilings-hardware)) | Same floor; likely equal or slightly worse in active use through more memory pressure (not measured) |

- **An `arm64` GSI over the stock vendor isn't viable.** Besides a 64-bit kernel, 64-bit system processes and apps load the vendor's graphics driver and buffer mapper in-process, and the stock vendor only ships 32-bit ones.
- The public 64-bit work for this phone (the 4.19 Android attempt, and mainline Linux running daily as arm64) is described under [Kernel](#kernel).

## 2 GB of RAM

- **Usable RAM is about 1.88 GiB.** The stock vendor sets a **1.5 GiB LZ4 zRAM**, the Android Go low-RAM profile and a 32-bit zygote. The stock kernel has PSI, and with the low-RAM profile Android 13's `lmkd` uses it.
- **zRAM stays mostly empty in normal use.** A bigger zRAM doesn't help, and swap on eMMC would add I/O, latency and wear.
- **Memory pressure costs CPU when browsing.** Reclaim (`kswapd`) becomes a top background CPU user.
- **Android 13's cached-app freezer doesn't work on the stock 4.9 kernel,** which lacks the required cgroup and binder freeze primitives. **Works (own build):** cached apps are frozen in the background without error, and the vendor ABI isn't affected.
- **Not available on 4.9:** MGLRU, DAMON, zRAM recompression and zstd. Backporting them is a large memory-management project with no measured pressure to justify it.
- Removing resident system apps frees RAM, not battery.

## Known traps

Each trap below stands on its own and is general to this platform.

- **Removing some Android 13 components bootloops the phone.**
  - Disabling the SDK sandbox package makes the package manager refuse to start.
  - The adservices APEX (which contains the SDK sandbox), on-device personalization, UWB and Nearby's half-sheet can't be removed cleanly.
  - With privileged-app permission enforcement on, a privileged app without its permission allow-list bootloops the phone.
  - Removing the calendar provider makes a "Do Not Disturb by event" rule crash `system_server`.
- **"Simulate a display with a cutout" (Developer options) bootloops Android 13** over the stock vendor. The setting also applies to the 128×128 outer screen and `system_server` dies. This is likely on any build where the outer screen is an Android display.
- **A root module that replaces a system app survives reflashing the ROM** and hides the ROM's newer copy. After flashing a ROM, check the installed root modules.
- **Replacing a system APK without invalidating the package cache** leaves the old version running ("Dex checksum does not match").
- **Some PMIC lines are essential despite their names.** Driving one of the PM8916's GPIOs low (it enables an external 1.2 V LDO named for the display) reboots the phone instantly. See [hardware.md](hardware.md).
- **In `debugfs/gpio`, the number after `in`/`out` is the mux function, not the level,** and the `gpiochip1` rows belong to the PMIC, not the SoC's TLMM.
- **Piping `dumpsys media.camera` into `head`/`grep` kills the camera provider** (SIGPIPE).
- **Measuring power with USB connected is invalid.** See [power.md](power.md).

---

## Viable / not viable

| Item | Verdict | Why (general) |
|---|---|---|
| Unlocking the bootloader | **Works** | Done on this phone; documented by the community |
| Relocking with modified software | Untested; likely not viable | A locked `aboot` verifies images against CAT's AVB key inside the signed bootloader (`"Security Error: This phone has been flashed with unauthorized software & is locked."`) and blocks fastboot flashing in the locked state; no user-key slot is known. Bricks reported by users were with modified images |
| Modifying or downgrading signed firmware (`sbl1`, TZ, RPM, `aboot`, modem, ADSP, Wi-Fi) | **Not viable** | Production-signed 3-certificate chain (`PK_HASH 51825348f62d3597…`, OEM `0x0019`, model `0x0002`); secure boot and anti-rollback are fused (`0x50`) |
| Reading SoC identity and root-key hash in EDL 9008 (Sahara v2 command mode) | **Works** | Returns `HWID 0x001060e100190002` and `PK_HASH 51825348f62d3597…` read-only without uploading a programmer |
| Exiting EDL 9008 when entered by command (intact boot chain) | **Works** | Hold power ~15 s (PM8916 hard reset, `s2-type = 7`) or unplug USB and pull the removable battery (viable, untested); Sahara reset (`SAHARA_RESET_REQ`) does not reboot the SoC, and a host USB port reset leaves the phone unenumerated |
| Exiting EDL 9008 via `SAHARA_RESET_REQ` or host USB port reset | **Not viable** | `cmd_reset()` only restarts the Sahara greeting after command mode, and a USB port reset hangs enumeration (`device not accepting address`) |
| Recovering from a bad `boot`, `dtbo`, `vbmeta` or `super` image (unlocked bootloader, intact `sbl1`–`aboot`) | **Works** | Boot chain stops in `aboot` Fastboot Mode (`18d1:d00d`), not EDL 9008; reflash stock `vbmeta*`, `boot`, `dtbo` and `super` from a Linux host |
| EDL 9008 as a USB unbrick / flashing path (Sahara / Firehose) | **Not viable** | Boot ROM requires a Firehose loader signed for OEM `0x0019` / model `0x0002` and `PK_HASH 51825348f62d3597…` (secure boot fused); none exists across 2,723 public loaders checked |
| Using Firehose loaders from other QM215/MSM8917 phones (OEM IDs `0x0004`, `0x0016`, `0x0042`) or Firehose XML exploits (`peek`/`poke`) | **Not viable** | Rejected by the PBL during the Sahara certificate check before any Firehose code loads into RAM |
| Restoring corrupted GPT / `sbl1` / `rpm` / `tz` / `aboot` via direct hardware eMMC access (ISP / chip-off) with a prior v30 backup | Viable, untested | Boot partitions are not encrypted per-unit on the eMMC; requires physical wiring to the eMMC bus while preserving the unit's own `modemst1/2`, `fsg` and `persist` |
| SoC boot-chain jailbreak in software (bypassing PBL, `sbl1`, TZ or `aboot` signature checks) | **Not viable** | Secure boot, anti-rollback and JTAG disable are fused (`0x50`); no public QM215 PBL/Sahara exploit exists, Oct 2021 firmware postdates older SBL1/QSEE bugs, `apdp`/`msadp`/`dpo` are empty, and `unlock_critical` (in RPMB) does not bypass signature checks |
| Bypassing secure boot via hardware voltage fault injection (*voltage glitching* on PBL/`sbl1`) | Viable only with invasive lab hardware, untested | Requires board teardown, decoupling removal, FPGA glitcher/oscilloscope and uncharacterized timing, with high physical risk |
| Secondary bootloader chainloading (`lk2nd`) or custom kernel from `boot` (unlocked bootloader) | **Works** (custom kernel) / **Reported working** (`lk2nd`) | An unlocked `aboot` skips AVB blocking and jumps in non-secure `EL1` to any image in `boot` without modifying `sbl1`/`tz`/`aboot` |
| Booting a kernel from RAM (`fastboot boot`) | Not viable with the stock bootloader (tested with four image variants); **Reported working** through lk2nd | Needs a custom lk2nd build. Otherwise flash `boot` |
| Custom 4.9 kernel rebuilt from Qualcomm source | **Works (public kernel)** | Stock modules load unchanged; public example [cat-shit22-kernel](https://github.com/axelwar716/cat-shit22-kernel) |
| Plain Qualcomm 4.9 tree with stock config | Viable with CAT's OEM drivers and config options re-added; **Works (public kernel)** | Without them there is no lid sensor, touch, USB-C controller, keypad light or audio amps: 42 CAT-only config options are dropped (16 of them enabled) and the touch driver has no source |
| Kernel 4.19 / GKI | Viable, not worthwhile | Stock modules bound to 4.9 ABI; 4.19 EOL; no proven benefit |
| Mainline Linux as a daily system (not Android) | **Reported working** | Third-party report: calls, texts, data, both screens, cameras, lid-closed suspend; about 1 %/h lid closed (reported overnight figure) |
| Mainline as Android daily kernel | Viable; no battery gain even at its best | Lid closed, the AP is already asleep 99.8 % of the time and the rest is modem and hardware floor; screen on, the drain is panel, backlight, LTE and the same frequency/voltage tables on one A53 cluster. Can't run CAT's vendor; only gain is software lifetime |
| Rebuilt vendor (from stock v30) | **Works** (own build) | Rebuilt from stock v30 keeping CAT's blobs. `vendor` isn't dm-verity protected with the bootloader unlocked; CAT published nothing newer, and another phone's blobs would lose CAT's device-specific ones |
| Rebuilding the vendor from scratch | Viable, very large effort | Requires replacing proprietary radio/IMS, audio, camera and compositor HALs |
| Android 13 ROM over the stock-based vendor | **Works** (own build, LineageOS 20) | Treble-compatible with VNDK 30 and the 4.9 kernel |
| Android 15 over the stock vendor | **Works (public kernel)** on a GSI (VoltageOS 4.5, SELinux enforcing, video and flashlight OK); viable as a ROM built for this phone | Such a build needs a rewritten vendor manifest; 4.9 still supported |
| Android 16 over the stock vendor | **Works (public kernel)** on a GSI (TrebleDroid, with `ro.lmk.use_psi=true`); viable, unofficial, as a ROM built for this phone | Without that property `lmkd` loops and the system hangs. A ROM built for this phone needs eBPF backports not officially available for 4.9 |
| arm64 GSIs | Not viable with the stock vendor | The stock vendor only has 32-bit graphics libraries, which 64-bit processes load in-process; also needs a 64-bit kernel |
| 64-bit kernel over the stock 32-bit userspace and vendor | Viable, large effort, no battery gain | CAT's signed ARMv7 modules can't load; every module rebuilt and every vendor driver's 32-bit path checked |
| Full 64-bit (kernel, system, vendor) | Viable only from another phone's blobs, with losses | No 64-bit vendor for this phone; more RAM use on 2 GB; no battery gain |
| Android 13 app freezer | **Works** (own build) | Stock 4.9 lacks the required cgroup and binder freeze primitives; works on the own build |
| MGLRU / DAMON / zRAM recompression | Viable, not worthwhile | Not in 4.9; large backport; no measured pressure |
| Bigger zRAM or eMMC swap | Viable, not worthwhile | zRAM mostly empty; eMMC swap adds latency and wear |
| microG instead of Google apps | **Works** (own build) | LineageOS 20 already has the needed spoofing support |
| Apps that check device integrity | Viable with spoofing, untested here | Hiding root plus spoofed properties/fingerprint pass basic and device integrity; strong (hardware-backed) integrity only with leaked attestation keys, which Google revokes; some apps add their own root or server checks |
| Removing the SDK sandbox / adservices APEX | Viable only with framework changes | Removed as is it bootloops Android 13: the package manager requires exactly one SDK sandbox package |
| Debloated system | **Works** (own build), RAM not battery | One resident service alone: ~48 MB RSS freed |
| Dialer with SMS (Messages tab, SMS in call details and search) | **Works** (own build) | Same cold-start time and memory as the unmodified dialer, within noise |
| Returning from a custom ROM to stock v30 | **Works** | Stock v30 `vbmeta` images + stock `super` from fastbootd + data wipe; Android 11 boots |
| CPU/GPU frequency caps, CPR/undervolt | Caps: **Works** (own build); CPR/undervolt: viable, no measurable gain | Details in [power.md](power.md#8-cpu-gpu-ddr-cpr) |
