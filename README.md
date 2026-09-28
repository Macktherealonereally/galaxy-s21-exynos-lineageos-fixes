# Galaxy S21 Exynos (SM-G991B / o1s) on LineageOS 23.2: VoLTE, eSIM, NFC, Wi-Fi, WPA3 hotspot and 60 fps video fixes

*Unofficial LineageOS 23.2 (Android 16) for the Samsung Galaxy S21 5G Exynos 2100 (SM-G991B, codename `o1s`), built from the [exy2100](https://github.com/exy2100) trees. Last updated 2026-09-28.*

This page collects the fixes we made on top of the exy2100 lineage-23.2 trees, and where they were submitted. It is written for S21 owners, ROM builders, and their AI agents who search for an error string. Each fix is listed as **symptom (the exact log line) → root cause → fix → how it was verified → where the patch is.**

Short version: **VoLTE (with HD voice), SMS over IMS, eSIM (including carrier apps installing their eSIM), NFC, WPA3, the WPA3 hotspot, 60 fps video recording and the Wi-Fi country code all work on our builds.** None of it needs Samsung's proprietary IMS. Everything is open source and submitted upstream, with PR links below.

> Target device: **SM-G991B** (S21 5G Exynos, `o1s`), on stock firmware base **G991BXXSJHZC2**. The common fixes (`universal2100-common`, kernel) likely also apply to the S21+ (`t2s`), the S21 Ultra (`p3s`) and the S21 FE Exynos (`r9s`), but we only tested o1s.

---

## Contents

1. [Status](#status)
2. [VoLTE / VoWiFi (open-source IMS)](#volte--vowifi-open-source-ims)
3. [eSIM (tsds2 slot switch + OpenEUICC)](#esim)
4. [NFC](#nfc)
5. [Wi-Fi: WPA3, WPA3 hotspot and country code "99"](#wi-fi)
6. [60 fps video recording](#60-fps-video-recording)
7. [SELinux enforcing](#selinux-enforcing)
8. [Smaller fixes](#smaller-fixes)
9. [Known limitations](#known-limitations)
10. [How to reproduce (build)](#how-to-reproduce)
11. [Where the patches are](#where-the-patches-are)

---

## Status

On our signed `userdebug` builds, SELinux **enforcing**, tested on one SM-G991B between 2026-09-25 and 2026-09-28:

| Feature | Upstream exy2100 build (20260621) | With these fixes | Notes |
|---|---|---|---|
| Boot, display 48–120 Hz, touch | ✅ | ✅ | |
| Wi-Fi WPA2 | ✅ | ✅ | |
| Wi-Fi **WPA3** (client) | ❌ "Check password" | ✅ | kernel `SAE_OFFLOAD` flag |
| Wi-Fi country code | ❌ stuck at `99` | ✅ follows SIM (e.g. `NL`) | overlay + kernel |
| Wi-Fi WPA3 **hotspot** (WPA3 and WPA2/WPA3) | ❌ | ✅ | [hostapd in-dongle SAE + overlay](#wpa3-hotspot-no-wpa3-option-or-clients-dropped-with-reason-17) |
| Mobile data, CS calls, SMS | ✅ | ✅ | |
| **VoLTE** calls (in/out, two-way audio) | ❌ | ✅ | open-source IMS stack |
| **HD voice (AMR-WB)** | ❌ | ✅ | see [HD icon note](#hd-icon-on-some-calls) |
| **SMS over IMS** | ❌ | ✅ | SMSC shim |
| VoWiFi | ❌ | ⚠️ built in, not tested | |
| EVS / "HD+" | ❌ | ❌ | [not provided](#evs--hd-voice-not-provided) |
| **eSIM** (select eUICC, read EID) | ❌ | ✅ | RIL slot-switch shim + OpenEUICC |
| eSIM install via QR/activation code | ❌ | ✅ | OpenEUICC (set ES10x MSS 255, see below) |
| eSIM install via **carrier app** | ❌ | ✅ (travel eSIM tested) | OpenEUICC patches |
| Dual SIM (physical + eSIM active together) | ❌ | ✅ | |
| Switch eSIM profiles without a reboot | ❌ | ✅ (~3 s) | OpenEUICC "Send refresh command to modem" ON, see [tips](#how-to-switch-sim-2-between-tray-and-esim) |
| **NFC** (tags) | ❌ never powers on | ✅ | kernel 32-bit ioctls |
| NFC battery drain (`ABNORMAL_POWER(DPD)`) | ❌ wakeup every ~11 s | ✅ | eSE node permissions |
| VoIP mic on speakerphone (WhatsApp etc.) | ❌ | ✅ | `txse4.bin` firmware |
| Fingerprint | ✅ | ✅ | enroll from Settings. Enrolling inside Setup Wizard fails |
| Camera **60 fps video** (1080p, all cameras) | ❌ | ✅ | [camera provider + overlay](#60-fps-video-recording) |
| Samsung Pay / Secure Folder / Samsung Pass | ❌ | ❌ | [Knox fuse](#knox-fuse) |
| USB-C → 3.5 mm DAC | ⚠️ | ⚠️ | detected but silent (audio offload), not fixed here |

---

## VoLTE / VoWiFi (open-source IMS)

### Symptom

On LineageOS the S21 has no VoLTE: no VoLTE toggle, calls on 2G/3G only, and in logcat:

```
RILJ: Unable to complete updateImsRegistrationInfo because service IMS is not available. [PHONE0]
```

The exy2100 XDA thread lists this as "Samsung's implementation doesn't work on AOSP".

### Fix: the open-source IMS stack

We use krazey's **ImsStack + ImsMedia + CarrierSettings** (Android 17 code, built in Android 16 QPR2 compatibility mode). This is the same design that LineageOS 24.0 ships as `hardware/lineage/generic-ims`. You need five things:

1. **Device integration** (guarded, so it is a no-op without the repos): ImsStack, ImsMedia, CarrierSettings, Iwlan, QNS, overlays (`config_use_voip_mode_for_ims`, `config_ims_mmtel_package=com.android.imsstack`, VoLTE/WFC available), and the optional IPsec algorithms (`xcbc(aes)`, `cmac(aes)`, `rfc3686(ctr(aes))`, `rfc7539esp(chacha20,poly1305)`).
2. **RIL shim for the SMSC.** The old blob fixup nulled the SMSC, and SMS over IMS then fails because RP-DATA needs a valid SMSC. A small `libsec-ril.so` shim loads the real RIL as `libsec-ril-impl.so` and rewrites `GET_SMSC_ADDRESS` into `+E.164`. The same shim patches the One UI version fallback so the A556 RIL runs `UiccEnablement` (ISIM/USIM state).
3. **Carrier enablement.** `carrier_volte_available_bool` must be true in CarrierConfig. krazey's CarrierSettings deliberately leaves out service enablement, so your carrier needs a CarrierConfig entry. We ship a `vendor.xml` RRO for KPN (NL). **Gotcha:** if you build such an RRO without `aapt2 --keep-raw-values`, `mnc="08"` is compiled to the integer `8` and never matches `CarrierIdentifier.getMnc()` = `"08"`. The symptoms are:
   ```
   ImsStack: Cellular service config is not available
   ```
   and `dumpsys carrier_config` showing `carrier_volte_available_bool=false`.
4. **IMS APN.** The carrier needs an `ims` APN, and internet APNs with `type="*"` must not swallow the IMS request. Otherwise IMS attaches to the internet PDN, which has no P-CSCF, and never registers. For KPN (204-08) the fix is submitted to LineageOS `vendor/apn`. **After changing APNs: Settings → Network → APNs → ⋮ → Reset to default**, because the APN database survives updates.
5. **Uplink audio on outgoing calls.** Symptom: *outgoing VoLTE call, you hear the other side, they hear nothing* (incoming calls are fine). Telecom puts the call into `MODE_IN_CALL` first, and the Exynos audio HAL opens the capture stream for the modem path (`Primary reconfig as Quad-Mic`, usage `recording`), then keeps it after the switch to VoIP mode. There are two fixes:
   - Telephony starts SIM calls in VoIP audio mode (`PhoneAccount.EXTRA_ALWAYS_USE_VOIP_AUDIO_MODE`) when `config_use_voip_mode_for_ims` is set;
   - ImsMedia reopens the capture stream once when the mode becomes `MODE_IN_COMMUNICATION`.

**Verified:**
- Lebara NL on KPN: IMS registers.
- Incoming and outgoing VoLTE with two-way audio, **AMR-WB (HD) negotiated in both directions**.
- SMS over IMS works in both directions.

**PRs:**
- device (RFC): [exy2100/android_device_samsung_universal2100-common#6](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/6)
- RIL shim: [exy2100/android_device_samsung_universal2100-common#5](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/5) + [exy2100/proprietary_vendor_samsung_universal2100-common#1](https://github.com/exy2100/proprietary_vendor_samsung_universal2100-common/pull/1)
- KPN APN: [LineageOS Gerrit 505199](https://review.lineageos.org/c/LineageOS/android_vendor_apn/+/505199)
- Telephony: [LineageOS Gerrit 505200](https://review.lineageos.org/c/LineageOS/android_packages_services_Telephony/+/505200)
- ImsMedia: [krazey/ImsMedia#1](https://github.com/krazey/ImsMedia/pull/1)

### HD icon on some calls

We traced a report that the "HD icon shows on the other phone but not on the S21 on outgoing calls". The S21 stack is fine: both the display path and the codec are correct.
- On calls between networks (e.g. KPN ↔ another operator), the codec on each leg is negotiated separately. The network can answer the S21's offer with AMR-NB only (`183 Session Progress` with only `AMR/8000`), which gives `NegotiateSdp(): AudioQuality[1]`, and then no HD icon on the S21. That is correct.
- Samsung One UI on the *other* phone hides its HD icon when the remote side is not known to be HD-capable (`remoteMmtelCapa=false`), even when it runs AMR-WB itself.
- To check your own call, grep logcat for `NegotiateSdp(): AudioQuality[`: `[2]` = AMR-WB (HD), `[1]` = AMR-NB.

### EVS / "HD+" voice: not provided

Stock One UI can use EVS ("HD+") with some carriers. The open-source stack negotiates AMR-WB, not EVS, because EVS is **patent-encumbered**. We don't distribute an EVS implementation or instructions for one. AMR-WB HD voice works.

---

## eSIM

The S21 (SM-G991B) has an NXP eUICC that **shares the second SIM interface with SIM tray 2**. Stock calls this `SEC_FLOATING_FEATURE_COMMON_CONFIG_EMBEDDED_SIM_SLOTSWITCH=tsds2`: you use *either* the second nano-SIM *or* the eSIM, next to SIM 1.

### Symptom 1: eSIM never appears, slot mapping fails with INTERNAL_ERR

```
RilRequest: [xxxx]< SET_LOGICAL_TO_PHYSICAL_SLOT_MAPPING error: com.android.internal.telephony.CommandException: INTERNAL_ERR
UiccSlot: ... mIsEuicc=false ... CARDSTATE_ABSENT
```

**Cause:**
- The exy2100 tree ships a newer Samsung RIL (from the Galaxy A55, "A556"). Its `SimManager::DoSetSlotMapping` only knows the A55's MEP eSIM layout, so it forwards physical slot 2 (the eUICC) to the modem as an ordinary mapping, and the modem rejects it.
- The stock S21 RIL instead runs a SIM low-level-control sequence (`ExecuteSlotSwitch`). That code is still inside the A556 RIL; it's just never called.

**Fix:** `libsec-ril-slotswitch`, a small interposer library loaded by the RIL shim, hooks `SimManager::DoSetSlotMapping`.
- For the tsds2 case it runs the stock slot-switch sequence **on SIM 2's `SimManager` instance** (`SimManager::mInstance[1]`). The modem applies the switch to the SIM interface of the RIL instance that receives the command, so running it on instance 0 power-cycles SIM 1 instead.
- Every other request goes to the original code.

### Symptom 2: eSIM selection lost after every reboot

```
avc: denied { set } for property=persist.ril.esim.slotswitch ... scontext=u:r:rild:s0 ... tcontext=u:object_r:default_prop:s0
```

**Fix:** label `persist.ril.esim.` as `radio_prop`, as stock does.

### Symptom 3: carrier app fails to install its eSIM ("error 012", "installation failed")

The carrier app uses `EuiccManager.downloadSubscription()`. With OpenEUICC as the LPA, several things fail one after another:
- `EuiccUiDispatcherActivity: Could not resolve activity for intent: Intent { act=android.service.euicc.action.RESOLVE_NO_PRIVILEGES ... }`: there was no consent dialog.
- Then `onGetDownloadableSubscriptionMetadata` returned an error within ~10 ms, because OpenEUICC's `shouldIgnoreSlot()` is inverted for devices whose only eUICC is reported as *removable* (Samsung's RIL reports the S21's as removable).
- Then `onDownloadSubscription()` was not implemented at all.

**Fix:** OpenEUICC patches:
- the inverted comparison;
- metadata lookup plus download;
- the carrier-privilege access rules passed through to the platform;
- `EuiccResolutionActivity` (the consent dialog);
- don't report a finished download as failed when only the enable step failed.

Submitted upstream: [PeterCxy/OpenEUICC#358](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/358), [PeterCxy/OpenEUICC#360](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/360).

### Symptom 4: download fails at the last step with 6A80

```
Error code: ES10B_ERROR_REASON_UNDEFINED
Last HTTP response (from server): ... "status": "Executed-Success" ... "boundProfilePackage": ...
Last APDU response (from SIM): 6A80
```

In logcat:

```
es10b_load_bound_profile_package -1, reason 255
ProfileDownloadException(lpaErrorReason=ES10B_ERROR_REASON_UNDEFINED ... lastApduResponse=[106, -128])
```

**Cause:**
- SGP.22 §2.5.5 requires each Bound Profile Package segment of up to 255 bytes to be sent in **one** APDU.
- OpenEUICC's default ES10x MSS is 63 ("Most Compatible"), so the ~183-byte first segment goes out as 3 chained APDUs.
- The S21's NXP eUICC (EID prefix 89043051) rejects the first one with `6A80`.

**Fix, pick one:**
- **No rebuild:** OpenEUICC → Settings → Info → tap *App Version* 7× → Developer → **ES10x MSS → "High Efficiency" (255)**. Then **force-stop OpenEUICC**, because it only reads the setting when it opens its channel.
- Device tree: overlay OpenEUICC's `config_es10x_mss_default` to 255 (OpenEUICC ≥ 7c05c06).
- Our lpac-jni patch: 255-byte blocks for LoadBoundProfilePackage only, [PeterCxy/OpenEUICC#359](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/359).

255-byte APDUs pass through the Samsung RIL without problems (all `9000`).

**Verified 2026-09-27:**
- A travel eSIM was installed through its own app via `EuiccManager`: `es10b_load_bound_profile_package 0`, PIR sent, and the profile enabled.
- Physical SIM 1 and the eSIM (as SIM 2) were both in service on LTE at the same time.
- The selection survived reboots.

### How to switch SIM 2 between tray and eSIM

One UI has a toggle for this; LineageOS doesn't (yet). The switch is the standard `setSimSlotMapping` call:
- **eSIM:** `[{port 0, phys 0} → logical 0, {port 0, phys 2} → logical 1]`
- **tray 2:** `[... {port 0, phys 1} → logical 1]`

You can make that call with OpenEUICC's slot-mapping screen or with a privileged tool. Once switched, it persists.

**Tips:**
- **Never re-request an eSIM code just because the carrier app showed an error.** Check whether `es10b_load_bound_profile_package 0` appeared in the log first. If it did, the profile is already on the chip: enable it in OpenEUICC, or reboot. Every new download attempt counts against the carrier's limit.
- **Switching eSIM profiles without a reboot:** keep OpenEUICC's developer option **"Send refresh command to modem"** switched **ON**. It is on by default. To find it: Settings → Info → tap *App Version* 7× → Developer Options.
  - With it on, Disable/Enable sends a REFRESH to the eUICC. The RIL reports `UNSOL_SIM_REFRESH`, and SIM 2 disappears and comes back on the new profile within **~3 s**. The tsds2 mux stays on the eSIM.
  - The Settings → SIMs toggle works this way too.
  - With it **off**, Android and the chip get out of sync: a later enable fails with ES10c result 2 (`profileNotInDisabledState`) and "Can't switch SIMs". A reboot brings them back in sync.
- **No "Disable" button in OpenEUICC?** OpenEUICC hides Disable/Delete for the active profile on a *removable* eSIM as a safeguard, and Samsung's RIL reports the S21's built-in eUICC as removable. Enable **Settings → Advanced → "Allow Disabling / Deleting Active Profile"**.
- If OpenEUICC shows `SwitchingProfilesRefreshException`, check whether the profile *was* switched anyway before retrying.

**PRs:**
- [exy2100/android_device_samsung_universal2100-common#5](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/5) (RIL shim + slot switch + prop label)
- [exy2100/proprietary_vendor_samsung_universal2100-common#1](https://github.com/exy2100/proprietary_vendor_samsung_universal2100-common/pull/1) (vendor blob rename)
- [exy2100/android_device_samsung_o1s#3](https://github.com/exy2100/android_device_samsung_o1s/pull/3) (o1s: OpenEUICC + `android.hardware.telephony.euicc`)
- [PeterCxy/OpenEUICC#358](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/358), [PeterCxy/OpenEUICC#359](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/359), [PeterCxy/OpenEUICC#360](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/360) (OpenEUICC)

---

## NFC

### Symptom 1: NFC never turns on

The SN100U at I2C `0x2b` answers NACK. The kernel shows `pn547_dev_ioctl: bad ioctl cmd:4004e901`. (XDA 4790286 #138.)

**Cause:** the driver defines its ioctls with `uint64_t` (`0x4008E901`), but the AOSP NXP HAL that LineageOS uses sends the 32-bit encoding (`0x4004E901`).

**Fix:** accept both encodings (Flopster101's Floppy2100 commit "nfc: pn547: Support 32-bit IOCTLs"). [exy2100/android_kernel_samsung_universal2100#8](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/8)

### Symptom 2: `ABNORMAL_POWER(DPD)` every ~11 seconds, battery drain

```
[pn547] pn547_dev_read: sec_nfc: ABNORMAL_POWER(DPD): 00000002 0C600240
```

`nfc_wake_lock` is held ~2 s per message: **5m37s over 168 wakeups in a 30-min screen-off test.** `dumpsys secure_element` shows `eSE1 mIsConnected:false`.

**Cause:**
- The eSE only enters Deep Power Down once `/dev/p61` has been opened and closed, because the release sets SPI CS high.
- The stock rc files give `/dev/p61` to `system`, but LineageOS's AOSP SE HAL runs as `secure_element`, so its `open()` fails with EACCES. It is a DAC failure, so **no avc denial** shows up.

**Fix:** chown `/dev/p61`, `/dev/p3` and `/dev/st54spi` to `secure_element` (rc + ueventd), as LineageOS s5e9925 does.

**Result:** 0 pn547 IRQs in 120 s screen-off, no more `ABNORMAL_POWER`. [exy2100/android_device_samsung_universal2100-common#2](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/2)

### Symptom 3: `NfcEventLog: java.io.IOException: Failed to create directory for /data/nfc/event_log.binpb.new`

**Cause:** `/data/nfc` is unlabelled (`system_data_file`).

**Fix:** label it `nfc_data_file`. [exy2100/android_device_samsung_universal2100-common#2](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/2)

---

## Wi-Fi

### WPA3: "Check password" / `AUTH_FAILURE_WRONG_PSWD` on WPA3 and WPA2/WPA3 networks

```
nl80211: MLME connect failed: ret=-22 (Invalid argument)
```

This appears right after `sae passphrase set successfully`.

**Cause:** since kernel 5.3, cfg80211 rejects `NL80211_ATTR_SAE_PASSWORD` unless the driver advertises `NL80211_EXT_FEATURE_SAE_OFFLOAD`. bcmdhd does SAE in firmware but didn't advertise it.

**Fix:** advertise it. This is the same fix as in LineageOS's S22 kernel. Already open upstream: [exy2100/android_kernel_samsung_universal2100#7](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/7).

### Country code stuck at "99"

`dumpsys wifi`:

```
mTelephonyCountryCode: NL
mDriverCountryCode: 99
isDriverSupportedRegChangedEvent: true
```

**Cause:**
- The overlay `config_wifiDriverSupportedNl80211RegChangedEvent=true` (copied from another tree; stock and AOSP use `false`) makes the framework wait for an `NL80211_CMD_WIPHY_REG_CHANGE` event.
- bcmdhd_101_16 (self-managed regdomain) never sends that event.

**Fix:**
- Set the overlay to `false`. [exy2100/android_device_samsung_universal2100-common#3](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/3)
- Also port Pixel's `wl_notify_regd()` so the driver reports the real country to cfg80211. [exy2100/android_kernel_samsung_universal2100#9](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/9)

Either change alone fixes the framework side.

**Verified:** `mDriverCountryCode: NL`, and the kernel logs `wl_notify_regd : regd notified: NL`.

### WPA3 hotspot: no WPA3 option, or clients dropped with reason 17

**Symptoms:**
- Settings → Hotspot → Security only offers WPA2-Personal. `dumpsys wifi` shows `SupportedFeatures=122`, i.e. no `SOFTAP_FEATURE_WPA3_SAE`. (XDA 4790286 lists this as "Hotspot WPA 3 security".)
- If WPA3 is enabled with only the overlay, clients authenticate and are then dropped, in a loop. hostapd has no PMK for them: `No PSK for STA trying to use SAE with PMKSA caching`.
- With the first hostapd fix but H2E still on, WPA3-only clients fail right after 4-way message 3 with **reason 17**. The client's log shows:
  ```
  WPA: RSNXE mismatch between Beacon/ProbeResp and EAPOL-Key msg 3/4
  RSNXE in Beacon/ProbeResp - hexdump(len=0): [NULL]
  RSNXE in EAPOL-Key msg 3/4 - hexdump(len=3): f4 01 20
  ```

**Cause:**
- The BCM4375 firmware runs SAE **inside the dongle**. Its feature list has `sae`, but no `idauth` (4-way offload) and no `extsae` (SAE frames to the host).
- So hostapd has two jobs:
  - give the dongle the password (Broadcom vendor command `BRCM_VENDOR_SCMD_BCM_PSK`);
  - take the PMK back from bcmdhd's `BRCM_VENDOR_EVENT_SAE_KEY` and run the 4-way handshake itself.
- LineageOS's `external/wpa_supplicant_8` only had this `CONFIG_BRCM_SAE` support for the **client** side (wpa_supplicant), not in hostapd. Samsung's stock hostapd has it.
- The second problem was H2E. Android's hostapd config uses `sae_pwe=2`, so hostapd put an RSNXE (SAE H2E) into message 3/4. The beacon is built by the firmware, though, and carries **no** RSNXE. The firmware also only does hunting-and-pecking SAE.
- The kernel needs no change: Samsung's bcmdhd_101_16 already has the AP-side SAE handling.

**Fix:**
- **hostapd** (LineageOS `external/wpa_supplicant_8`, one change): support bcmdhd in-dongle SAE for SoftAP under `CONFIG_BRCM_SAE`. It adds:
  - the password vendor command before `START_AP`;
  - a new `EVENT_SAE_KEY` → PMKSA;
  - a PMKSA fallback for SAE stations without a PMKID;
  - `sae_pwe=0` for this mode, as stock does. hostapd logs `SAE: bcmdhd in-dongle SAE does not support H2E; using sae_pwe=0`.
  - It is built only when the device sets `$(call soong_config_set_bool,wpa_supplicant_8,board_wlan_bcmdhd_sae,true)` (universal2100-common already does), and it only runs for WPA3/SAE hotspots. [PR-22]
- **Overlay** (universal2100-common): `config_wifi_softap_sae_supported=true`, as stock's `SoftapOverlayWpa3` does. **Only together with the hostapd change.** On its own, WPA3 shows up but nobody can connect. [PR-23]

**Verified 2026-09-28** (build "Goshawk 10.3"):
- `SupportedFeatures` 122 → 126.
- A Galaxy Z Flip 8 joined WPA2-Personal, WPA2/WPA3 transition (`key_mgmt` 0x402) and WPA3-Personal (0x400). The 4-way handshake completed each time (`EAPOL-4WAY-HS-COMPLETED`), and the internet worked.

**Other Samsung Exynos/bcmdhd devices** whose firmware does in-dongle SAE (the Galaxy S10/S20/S21/S22 families; the LineageOS trees `exynos9820-common`, `x1s`, `y2s`, `z3s`, `c2s` and `b0s` already set `board_wlan_bcmdhd_sae`) can use the same hostapd change. They only need to set `config_wifi_softap_sae_supported=true` in their WifiOverlay and test. An independent S22 Ultra analysis reached the same conclusion: [hacenbm144-spec/b0s-wpa3-fixes](https://github.com/hacenbm144-spec/b0s-wpa3-fixes).

---

## 60 fps video recording

### Symptom

Every AOSP camera app (Aperture, Open Camera…) records at **30 fps**, even when set to 60. `ffprobe` shows `avg_frame_rate=30/1`. The public fps ranges stop at 30 on every camera:

```
android.control.aeAvailableTargetFpsRanges (10014): int32[16]
  [15 15 15 20 ]
  [20 20 24 24 ]
  [8 30 10 30 ]
  [15 30 30 30 ]
```

The exy2100 XDA thread lists this as a known bug that upstream considered "won't fix". The r9s approach (adding 60 to the Aperture overlay) doesn't work either; its own commit message says "Still not working".

### Cause

It isn't the codecs or `media_profiles` (those are identical to stock). It is a gate **inside Samsung's camera HAL** (`libexynoscamera3.so`):
- The HAL has real 60 fps modes: 1080p60 on every camera, listed in the vendor tag `samsung.android.scaler.availableVideoConfigurations`.
- `ExynosCameraConfigurations::m_adjustPreviewFpsRange` only accepts a 31–119 fps range from a **Samsung client**, i.e. when the session parameter `samsung.android.control.cameraClient` marks the Samsung camera app or SDK. Otherwise it **silently keeps 30 fps**.
- On AOSP, that session parameter never reaches the HAL. `android.request.availableSessionKeys` is empty (Samsung keeps its list in a vendor tag), so the framework drops all Samsung session tags.

### Fix

- **Camera provider** (`hardware/samsung`, opt-in via `samsungCameraVars.high_fps_video`): [PR-19]
  - advertise the HAL's fixed high fps modes (`[60,60]`);
  - make the AE target fps range a session key;
  - for >30 fps sessions only, add `cameraClient=2` plus `recordingMin/MaxFps` to the session parameters, as the stock camera stack does.
  - ≤30 fps sessions are untouched.
- **universal2100-common:** enable the flag. [PR-20]
- **o1s:** Aperture overlay that offers 60 fps **only at 1080p**, the size where the HAL has a 60 fps mode on every camera. [PR-21]

**Verified 2026-09-28** (build "Goshawk 10.2"):
- Aperture 1080p60, rear camera: **329 frames / 5.48 s = 60.0 fps**, mean frame interval **16.66 ms**, **0 gaps > 25 ms**.
- Front camera 1080p60: 60.0 fps. The 30 fps control clip: 30.0 fps.
- logcat: `CamDev-HighFps: patchSessionParams: high fps session [60, 60], cameraClient 2`.

**Notes:**
- Switching between 30 and 60 briefly restarts the preview.
- The encoder bitrate still follows the 30 fps profile.
- 4K60 is allowed by the provider on the rear main/ultra-wide cameras, but it is not validated, so Aperture hides it.

---

## SELinux enforcing

Our builds run **enforcing**. The fixes needed are in [exy2100/android_device_samsung_universal2100-common#4](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/4):
- **SysInput HAL:** it had no domain. In enforcing mode init refuses to start it:
  ```
  init: File /vendor/bin/hw/vendor.samsung.hardware.sysinput-service (labeled "u:object_r:vendor_file:s0") has incorrect label or no domain transition from u:r:init:s0 to another SELinux domain defined.
  ```
  The PR gives it a proper HAL domain (`hal_samsung_sysinput`).
- health HAL/charger (`boot_status_prop`, `IThermal`, `/proc/last_kmsg`), power HAL → thermal, SE HAL → `vendor_persist_nfc_prop`, EDEN NN → `gpu_mm_min_clock`, vold → UFS host `uevent`.

---

## Smaller fixes

| Symptom | Fix | Where |
|---|---|---|
| Mic doesn't work in VoIP calls (WhatsApp, Signal…) on **speakerphone** | ship `vendor/firmware/txse4.bin` from stock (the mixer path `txse1-txse4-txse2` needs it) | open: [o1s#2](https://github.com/exy2100/android_device_samsung_o1s/pull/2) + [vendor_o1s#1](https://github.com/exy2100/proprietary_vendor_samsung_o1s/pull/1) |
| `getMicrophones()` returns nothing | `audio_board_info.xml` with 2 mics, values UNKNOWN (no routing change) | [exy2100/android_device_samsung_o1s#4](https://github.com/exy2100/android_device_samsung_o1s/pull/4) |
| Long-running video: MFC iovmm leak | Samsung "media: mfc: clear sgt to prevent iovmm leak" | [exy2100/android_kernel_samsung_universal2100#10](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/10) |
| No last_kmsg after a crash | `console-ramoops` region | [exy2100/android_kernel_samsung_universal2100#11](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/11) |
| Clean checkout doesn't build; incremental OTA `KeyError: '/vendor_dlkm'`; `AttributeError` in releasetools; `/vendor_dlkm` not mounted; first boot after flash goes to recovery | lineage.dependencies, fstab, releasetools, module loading, `formattable` | open: [universal2100-common#1](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/1), [o1s#1](https://github.com/exy2100/android_device_samsung_o1s/pull/1) |
| Idle/deep-sleep drain reported on the 20260621 build | probably "Remove Exynos PASR driver" (the dev's announced deep-sleep fix; already in exy2100 kernel HEAD, but not in a release build). We carry it on our kernel, but haven't measured the effect yet. | upstream `2b525e0f8` |

---

## Known limitations

### Knox fuse

The first boot of any custom image trips the Knox warranty fuse (`ro.boot.warranty_bit=1`) **permanently**. Unlocking the bootloader alone does not trip it. After that, **Samsung Pay/Wallet, Samsung Pass and Secure Folder never work again on that phone**, not even after going back to stock.

### HD icon on some calls

See [above](#hd-icon-on-some-calls). On calls between networks the S21 leg may really be AMR-NB (the network's choice), and the other phone's HD icon only describes *its* leg. This is not a bug in the ROM.

### EVS / HD+ voice

Not provided, because EVS is patent-encumbered. AMR-WB HD voice works.

### Other

- Fingerprint enrolment fails inside Setup Wizard. Enrol from Settings afterwards.
- USB-C DACs: reported on XDA as detected but silent (Samsung audio offload path). Not addressed here.
- After a reboot with both SIMs in use, SIM 1 once stayed out of service until an airplane-mode toggle.
- After enabling a new eSIM profile with OpenEUICC's "Send refresh command to modem" turned off, you need a reboot before the new line appears. Keep it on (see [eSIM tips](#how-to-switch-sim-2-between-tray-and-esim)).
- `dumpsys secure_element` still shows `eSE1 mIsConnected:false`. This has no user-visible effect, and the eSE power issue itself is fixed.

---

## How to reproduce

This is a high-level outline. The exy2100 o1s README (PR #1) has the exact steps.

1. **Firmware:** flash stock **G991BXXSJHZC2** (BL/AP/CP/CSC) first. Other bases (HZA6, HYD5) hang at the logo.
2. **Sources:**
   ```sh
   repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs
   ```
   Then add a local manifest with:
   - the exy2100 device, common, kernel and vendor repos;
   - the exy2100 forks (`device/lineage/sepolicy`, `hardware/samsung`, `hardware/lineage/interfaces`, `lineage-sdk`, `packages/apps/Settings`, `device/samsung_slsi/sepolicy`, `hardware/samsung_slsi-linaro/exynos`), with `remove-project` for the LineageOS originals;
   - the LineageOS `hardware/samsung_slsi-linaro/{config,graphics,interfaces,openmax,codec2,exynos5,sgpu}` repos.
3. **For VoLTE and eSIM** add a second local manifest:
   ```xml
   <remote name="krazey" fetch="https://github.com/krazey" />
   <remote name="estkme" fetch="https://github.com/estkme-group" />
   <remote name="angry"  fetch="https://gitea.angry.im/PeterCxy" />
   <remove-project name="platform/packages/modules/ImsMedia" optional="true" />
   <project path="packages/modules/ImsMedia"  name="ImsMedia"        remote="krazey" revision="cb1dd74e9d5e7b7693dfc5870037607eb58d8ea3" />
   <project path="packages/modules/ImsStack"  name="ImsStack"        remote="krazey" revision="5ff784d265a94b8227f8fcc93bf915f7635283f0" />
   <project path="packages/apps/CarrierSettings" name="CarrierSettings" remote="krazey" revision="2d00ce6100921547482bb3ecda690ec346d64be2" />
   <project path="packages/apps/OpenEUICC"    name="openeuicc"       remote="estkme" revision="master" sync-s="true" />
   <project path="prebuilts/openeuicc-deps"   name="android_prebuilts_openeuicc-deps" remote="angry" revision="67a341e9cfbaed43da7e5a281f5cc7b8893fdc55" />
   ```
   krazey force-pushes, so pin commits. openeuicc-deps HEAD needs SDK 37, which lineage-23.2 doesn't have.
4. **Apply the patches** from the PRs listed below that aren't merged yet (fetch the PR branches or `git am` them).
   `external/wpa_supplicant_8` and `hardware/samsung` are not device repos: a `repo sync` drops local commits there. Until the Gerrit changes are merged, keep them on a fork with a `remove-project` in your local manifest, or re-apply the patch after each sync.
5. **Build:**
   ```sh
   source build/envsetup.sh && breakfast o1s && m bacon
   ```
   For daily use, sign with your own release keys. Our builds are `userdebug`, enforcing and release-signed.
6. **Install:**
   - Flash LineageOS recovery in download mode with **auto-reboot off**.
   - Boot straight into recovery. Stock restores its own recovery on the first normal boot.
   - Run **Format data** and `adb sideload` the OTA.
   - Incremental OTAs work with the releasetools/fstab fixes from PR #1. Anything that writes to `/product` or `/system` outside the OTA breaks incrementals.

---

## Where the patches are

Placeholders like [PR-19] are replaced by links once each PR or Gerrit change is opened. Until then, the patches for the newest fixes are in this repo:
- 60 fps: `patches/lineage-hardware-samsung/` (applies to both LineageOS and exy2100 `hardware/samsung`), `patches/universal2100-common/camera-high-fps-video/`, `patches/o1s/`;
- WPA3 hotspot: `patches/lineage-wpa-supplicant-8/`, `patches/universal2100-common/wpa3-softap/`.

| Ref | Repo | Topic | Status |
|---|---|---|---|
| — | exy2100 universal2100-common [#1](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/1) | build/OTA/vendor_dlkm fixes | open |
| — | exy2100 o1s [#1](https://github.com/exy2100/android_device_samsung_o1s/pull/1) | lineage.dependencies, README | open |
| — | exy2100 o1s [#2](https://github.com/exy2100/android_device_samsung_o1s/pull/2) + vendor_o1s [#1](https://github.com/exy2100/proprietary_vendor_samsung_o1s/pull/1) | txse4 (VoIP speaker mic) | open |
| — | exy2100 kernel [#7](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/7) | WPA3 | open |
| [#8](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/8) | exy2100 kernel | NFC pn547 32-bit ioctls | open |
| [#2](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/2) | exy2100 universal2100-common | NFC eSE permissions + /data/nfc | open |
| [#3](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/3) | exy2100 universal2100-common | Wi-Fi country overlay | open |
| [#9](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/9) | exy2100 kernel | Wi-Fi `wl_notify_regd` | open |
| [#4](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/4) | exy2100 universal2100-common | SELinux enforcing fixes | open |
| [#5](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/5) | exy2100 universal2100-common | RIL shim + eSIM slot switch | open |
| [#1](https://github.com/exy2100/proprietary_vendor_samsung_universal2100-common/pull/1) | exy2100 vendor universal2100-common | `libsec-ril-impl` rename | open |
| [#3](https://github.com/exy2100/android_device_samsung_o1s/pull/3) | exy2100 o1s | OpenEUICC + eUICC feature | open |
| [#4](https://github.com/exy2100/android_device_samsung_o1s/pull/4) | exy2100 o1s | audio_board_info.xml | open |
| [#10](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/10) | exy2100 kernel | MFC iovmm leak | open |
| [#11](https://github.com/exy2100/android_kernel_samsung_universal2100/pull/11) | exy2100 kernel | console-ramoops | open |
| [OpenEUICC#358](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/358) | OpenEUICC (gitea.angry.im) | `shouldIgnoreSlot` fix | open |
| [OpenEUICC#359](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/359) | OpenEUICC | BPP 255-byte blocks (6A80) | open |
| [OpenEUICC#360](https://gitea.angry.im/PeterCxy/OpenEUICC/pulls/360) | OpenEUICC | carrier-app download + consent UI (#99) | open |
| [LineageOS Gerrit 505199](https://review.lineageos.org/c/LineageOS/android_vendor_apn/+/505199) | LineageOS vendor/apn | KPN IMS APN | in review |
| [#6](https://github.com/exy2100/android_device_samsung_universal2100-common/pull/6) | exy2100 universal2100-common | VoLTE integration (RFC) | open |
| [LineageOS Gerrit 505200](https://review.lineageos.org/c/LineageOS/android_packages_services_Telephony/+/505200) | LineageOS Telephony | VoIP audio mode for IMS calls | in review |
| [#1](https://github.com/krazey/ImsMedia/pull/1) | krazey/ImsMedia | uplink capture reopen | open |
| [PR-19] | LineageOS hardware/samsung (Gerrit) | camera provider: Samsung high fps video modes (60 fps) | prepared |
| [PR-20] | exy2100 universal2100-common | enable high fps video | prepared |
| [PR-21] | exy2100 o1s | Aperture: 60 fps at 1080p only | prepared |
| [PR-22] | LineageOS external/wpa_supplicant_8 (Gerrit) | hostapd: bcmdhd in-dongle SAE SoftAP (WPA3 hotspot) | prepared |
| [PR-23] | exy2100 universal2100-common | WifiOverlay: WPA3-SAE SoftAP (needs PR-22) | prepared |

**Credits:**
- ata-kaner and the exy2100 contributors, for the lineage-23.2 trees.
- Flopster101 (Floppy2100), for the NFC ioctl fix and the MFC backport.
- krazey, for the open-source IMS stack.
- PeterCxy and the OpenEUICC/lpac contributors.
- LineageOS universal9830/s5e9925 maintainers, whose SMSC shim and eSE fix we followed.
- Tim Zimmermann, for the original bcmdhd SAE support in LineageOS's wpa_supplicant, and hacenbm144-spec, for the independent S22 Ultra WPA3 hotspot analysis.
- XDA users kyzer.android (NFC ioctl, txse4 root causes), ✦andrew!^~^, AMAZING2545, and everyone who reported bugs in thread 4790286.

---

## License

This guide is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The files in `patches/` are licensed under the license of the project they apply to (OpenEUICC: GPL-3.0; LineageOS/AOSP: Apache-2.0).
