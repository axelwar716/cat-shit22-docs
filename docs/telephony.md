# Modem, telephony, IMS/VoLTE

Scope: the CAT S22 Flip's cellular modem, its firmware and carrier profiles, how stock telephony (IMS/VoLTE, caller ID, SMS) behaves, especially with Spanish carriers, and what works or is viable on non-stock software.

Related pages: [hardware.md](hardware.md) covers the SoC and chips. [power.md](power.md) covers modem idle behaviour (paging, duty cycle, the cost of IMS re-registration, LTE vs 2G at idle).

---

## Modem and firmware

| Item | Value |
|---|---|
| SoC / modem | Qualcomm QM215 with its integrated LTE modem (Qualcomm markets it as an X5 LTE modem with VoLTE support) |
| UE category | LTE cat 4, no carrier aggregation observed |
| Modem firmware (MPSS) | `MPSS.JO.3.3-00078`, build flavour `8937.genns.prodQ` (`SDM439_GENNS_PACK`), production variant, dated June 2022 |
| SIM | Single SIM |
| Firmware package | Qualcomm's generic modem package, not a carrier-certified build |

- **The modem firmware image is signed and can't be modified.** Every image inside the modem partition carries Qualcomm's per-segment hash and a production certificate chain bound to the OEM and model. Secure boot is fused on this phone, so an edited modem image does not load. Unlocking the bootloader doesn't change this: it only relaxes Android Verified Boot for `boot`/`system`/`vendor`, not the firmware chain (see [platform.md](platform.md)).
- **The modem's configuration, by contrast, can be changed.** The modem's configuration lives in its own storage (EFS), not in the signed image, and with root it can be read and written from Android.
- **The modem can't be downgraded.** Anti-rollback is fused too. An older modem (or sbl1/TZ/RPM) image than the one on the phone won't boot. Never flash older firmware.
- **There is no public EDL (emergency download) loader for this phone.** In EDL mode (`05c6:9008`) the phone identifies over Sahara v2 with HWID `0x001060e100190002` (MSM ID `0x001060E1`, OEM ID `0x0019`, i.e. Bullitt, model `0x0002`) and root-key hash `PK_HASH = 51825348f62d3597cd229601da673eb75399c2c5b7ff5260eab794475eef4e20`. Its key hash doesn't match any public firehose loader (2,723 checked). Firehose loaders from other QM215 phones are rejected during the Sahara handshake, and they can't be re-signed. EDL is therefore not a safety net. With an intact boot chain, the tested way out of EDL is holding the power button for about 15 s (or pulling the removable battery; see [platform.md](platform.md#emergency-download-mode-edl-9008-gpt-layout-and-recovery-limits)).
- **The modem's own storage (EFS) is encrypted on disk.** The radio calibration and band configuration load from it at runtime, so an offline firmware dump doesn't give you the supported bands. With root you can read them from the running modem instead.
- **The IMEI lives in that same modem storage.** With root, changing it is technically within reach through the same path as any other modem setting. It wasn't attempted here, and it's illegal in several countries. VoLTE doesn't need it (see below).

## Bands

The unit studied is the **T-Mobile US variant** (TMO build properties, T-Mobile apps, remote SIM-lock (RSU) apps, `simlock rsu` in the modem).

| Source | LTE bands |
|---|---|
| CAT's official US spec sheet | 2, 4, 5, 12, 13, 25, 26, 41, 66, 71 |
| Some spec aggregators | Add 1, 3, 7, 20, 28. These probably list what the chip supports, not what this model ships with |
| **Measured on the phone in Spain (Vodafone network)** | **B1, B3, B20 and B28 all camped** |

- **LTE works on Spanish bands even though the US spec sheet omits them.** On the Vodafone network the phone camps on B1, B3, B20 and B28 and moves between them. It does this with no change to the modem.
- Without a SIM, the phone also camped on UMTS B8 (Orange Spain), another band the official sheet doesn't list.
- B7 and B8 LTE weren't seen in use, so they are unconfirmed.
- The spec-sheet WCDMA bands are 1/2/4/5 and GSM is quad-band (850/900/1800/1900).

## Carrier profiles in the modem

- The modem carries **127 software carrier profiles and 12 hardware profiles**. CAT customised only the T-Mobile US profile. The rest are Qualcomm reference profiles, not certified with each carrier.
- For Spain there are IMS-enabled profiles for Movistar, Vodafone and Orange. Yoigo, DIGI and MVNOs have no profile of their own and use a generic one.
- **The modem picks the profile itself, from the SIM**, among the profiles stored in it. Android plays no part in the choice.

## Stock behaviour with Spanish carriers

- **The phone is not SIM-locked to T-Mobile in practice.** A Spanish SIM worked on stock Android 11, so bands, SIM lock and IMEI were not a problem.
- **IMEI allow-lists didn't block IMS.** The S22 Flip isn't on any Spanish carrier's VoLTE-approved device list, yet the Vodafone network registered IMS for it.
- **Stock VoLTE with Spanish carriers: enabled in CAT's stock carrier configuration for Vodafone (including Lowi) and Orange, not for Movistar/O2, Yoigo or DIGI.** Whether IMS actually registered on stock wasn't checked with logs.
- **Without VoLTE, voice in Spain falls back to 2G.** Vodafone switched 3G off in 2024 and keeps 2G until about 2030. Movistar plans to close 2G/3G at the end of 2027. MasOrange closes 3G on 31 Dec 2027 and keeps 2G until 2030. These dates come from the press. Once 2G goes, a phone without working VoLTE has no voice service in Spain.
- VoWiFi: see [VoLTE / IMS on non-stock software](#volte--ims-on-non-stock-software).

## IMS stack on stock

How the pieces fit on CAT's stock Android 11, in behavioural terms:

- **The IMS client is Qualcomm's and lives in the vendor and the modem, not in AOSP.** SIP registration and call signalling run in the modem. The vendor partition runs the radio interface (RIL) daemon and Qualcomm's IMS daemons (QMI, IMS data and RTP media daemons). These run on stock whether or not VoLTE is in use.
- **The Android side is a Qualcomm ImsService plus a Qualcomm telephony service** that CAT ships in its own system image. Both are tied to CAT's build.
- **Stock carrier configuration declares VoLTE per carrier.**
- The kernel side of IMS (IPsec/XFRM, IPv6 netfilter, RMNET data, IPA, QMI transport) is all in the stock kernel config.
- With VoLTE on LTE, the network keeps a separate IMS data connection (PDN) up beside the internet one. With Wi-Fi connected and "mobile data always active" off, Android drops the mobile internet PDN, and only the IMS PDN stays up.

## Caller ID (hidden caller number)

- **Status: fixed** (the "caller-ID fix"). It is a modem configuration change, not part of any ROM. Incoming VoLTE calls on a **Lowi (Vodafone) SIM** show the caller's number again. Reversible.
- **Tested carrier: Lowi (Vodafone).** Measured on this phone; it is the only carrier tested.
- **Stock symptom:** on CAT's stock firmware with a Lowi (Vodafone) SIM, some incoming calls show the number and others show "Unknown". Stock was never fixed.
- **Other phones show the same thing.** Public user reports describe it on other Qualcomm phones and ROMs with several European carriers, including Lowi (Vodafone).
- **Where VoLTE is off,** calls go over 2G with circuit-switched caller ID. That stops working when 2G closes, so it isn't a fix.

### Cause

- **Generic cause (inferred, strong):** on some calls the network doesn't send the caller's number in the identity field the modem reads, so Android gets no number. Calls that carry the field show the number; the others show "Unknown".

### Fix

**Fixed** with a change to the modem's configuration, which is not part of any ROM. Steps are not published.

> **Warning: very risky.** This writes to the modem's own storage (EFS). A wrong write or an interrupted transfer can leave the phone **without radio**: no calls, no data, no SIM, possibly no IMEI. This phone has no EDL recovery path ([Modem and firmware](#modem-and-firmware)), so the damage may be **permanent**. Tested on one phone with one SIM. Entirely at your own risk.

## SMS

- **SMS over IMS registers in the modem together with voice on LTE.** Android sees the MMTEL (voice/SMS) feature as ready.
- **The network also offers SMS over the circuit-switched domain (SGs) as a fallback**, so SMS probably doesn't depend on IMS registration. That would explain SMS working on software where IMS isn't registered.

## Modem idle

Idle-power facts belong to [power.md](power.md): paging cycle, duty cycle per band and signal level, IMS re-registration and tracking-area timers, 2G vs LTE at idle, the effect of removing 3G, and wakeups caused by the modem. One short version: what raises idle with a SIM is the serving cell and signal level, not IMS or 3G.

---

## VoLTE / IMS on non-stock software

The stock vendor keeps the whole Qualcomm IMS stack: modem, vendor daemons and HAL interfaces. A ROM or GSI on top of it has to bring the Android half, and that's where VoLTE usually breaks.

- **VoLTE on Android 13: works (own build).** IMS registers with a Spanish carrier on LTE and calls go over VoLTE, with caller ID fixed as above. **No third-party telephony or IMS app is used**: no downloaded IMS package, no GSI IMS add-on or settings app, no root daemon. On Android 13 GSIs it is reported working (public user reports). General reason it doesn't just work out of the box: **the IMS stack belongs to the vendor, and an AOSP build or GSI doesn't give it the access and configuration it needs by default.** Voice then falls back to 2G (CSFB) even while the network offers IMS.
- **VoLTE on Android 14+: viable on a ROM built for this phone; fails on unmodified GSIs.** The IMS stack is the vendor's and doesn't change with the Android version, so a ROM built for this phone has no known blocker. GSIs fail for a packaging reason: the IMS packages distributed for Android 14 and newer exist only as 64-bit (arm64) builds, while this phone's userspace is 32-bit, so they don't install. Public reports match this: LTE works on Android 14+ GSIs, VoLTE calls don't. A public third-party Android 15 build for this phone reports VoLTE (HD icon) with a patched copy of the stock 32-bit IMS client and broad SELinux permissions. It publishes no logs or call tests, so it's unverified.
- **The kernel isn't what blocks VoLTE.** A kernel rebuilt with the stock config keeps all the networking IMS needs.
- **Hidden caller number: fixed** with a modem configuration change, independent of the ROM ([Fix](#fix)).
- **Video calling over IMS (ViLTE) and RCS:** not evaluated, and there's no RCS ImsService on stock.
- **VoWiFi:** reported working by users in community chat groups. Not verified here.

## Viable / not viable

| Item | Verdict | Why (general) |
|---|---|---|
| LTE on Spanish bands (B1/B3/B20/B28) | **Works** on stock | Observed in use despite the US spec sheet |
| VoLTE on stock Android 11 with Spanish carriers | Untested here (enabled in CAT's carrier config for Vodafone/Lowi and Orange) | Registration on stock not checked with logs |
| VoLTE on an Android 13 ROM | **Works** (own build), no third-party telephony or IMS app | Verified in logs: IMS registered on LTE, calls delivered over VoLTE |
| VoLTE on Android 13 GSIs | Reported working | Public user reports; not verified here |
| VoLTE on an Android 14+ ROM built for this phone | Viable | Same vendor IMS stack; no known blocker |
| VoLTE on Android 14+ GSIs | Fails on unmodified GSIs; viable with the stock 32-bit IMS client added | The IMS packages distributed for Android 14+ are 64-bit only; one third-party Android 15 build reports it working (unverified) |
| VoWiFi | Untested here; reported working by users in chat groups | Not verified here |
| Hidden caller number with VoLTE (Lowi, Vodafone) | **Fixed**; intermittent on stock | A modem configuration change, ROM-independent, high risk ([Fix](#fix)). The same symptom is reported on other Qualcomm phones |
| Incoming VoLTE call with the lid closed (calls and SMS only) | **Works** (own build) | Rings over VoLTE with only the IMS connection up |
| Modifying the modem firmware image | Not viable | Signed, secure boot fused; would need CAT's signing key or a TrustZone bypass, and neither is public |
| Changing the modem's configuration | **Works** | Outside the signed image |
| Downgrading the modem firmware | Not viable | Anti-rollback fused |
| EDL as unbrick/backup path | Not viable | The boot ROM only accepts a firehose loader signed for this OEM/model (secure boot fused); none is public |
| Changing the IMEI | Viable with root, untested (not attempted) | Stored in the modem's configuration; illegal in several countries |
| Reading the supported band list | Viable with root, from the running modem | Not from an offline dump: modem storage is encrypted on disk |
