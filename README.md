# MRCOOL EZ Pro (MRCOOL Comfort generation) in Home Assistant — local control, no cloud

**TL;DR:** The current-generation MRCOOL EZ Pro mini-split (model `EZPRO-09-HP-11511DE-O`, 9K BTU / 115V, sold with the *MRCOOL Comfort* app) **can** be integrated into Home Assistant with full local control. The trick is to re-provision its Midea-based Wi-Fi dongle (`US-OSK109`) into a **fresh MSmartHome (SmartHome) account**, then use the `midea_ac_lan` integration, which fetches the V3 token/key from *your own* account once — after that, control is 100% local LAN. As far as we could find (HA forums, GitHub, Reddit — searched October 2026), this is the **first publicly reported success** for this EZ Pro generation. Corrections welcome.

---

## Why this model is "impossible" (until now)

- The EZ Pro ships in two connectivity generations: the older `16D` units (Cielo "SmartHVAC" app) and the current `11DE` units (**MRCOOL Comfort** app, Midea platform). This write-up is about the **11DE / Comfort** generation.
- The MRCOOL Comfort account system is walled off from Midea's public clouds — logging a Comfort account into Midea cloud endpoints fails (the infamous **error 3102**), so the usual "log in with the app account, grab the token" route is dead.
- The Wi-Fi dongle is Midea's **US-OSK109** (SK109 family, FCC ID 2AS2HMZNA23) — a V3-protocol device on port 6444. A V3 device only reveals its token/key to the cloud account that owns it.
- Before this attempt, there were **zero** public success *or* failure reports for integrating the EZPRO Comfort generation with HA.

## What does NOT work (save yourself the time)

1. **Bluetooth discovery in MSmartHome** — the search spins forever. This dongle pairs in **AP mode only**; there is no Bluetooth pairing to find.
2. **Scanning the QR code on the dongle/box with MSmartHome** — the app rejects it. That QR is a *MRCOOL Comfort* provisioning code; Midea's app doesn't recognize it.
3. **mill1000's `midea_ac` integration auto-setup** — LAN discovery finds the unit fine, but its automatic token retrieval uses a built-in *shared* cloud account. For a device owned by another account, the cloud answers `Code: 9999, system error` and the config flow aborts with `cloud_connection_failed`. (This integration is great — the auto-token path just can't work for tenant-owned devices like this one.)

## The working path

### 1. Create a FRESH SmartHome account (important)

Register a new account in the **MSmartHome / SmartHome** app (Midea's official app) with an email address that has **never been registered in any Midea-family app** (not MRCOOL Comfort, not Midea Air, not NetHome Plus). This is the known workaround for the 3102 account wall. Your AC is going to live in this account from now on.

### 2. Put the indoor unit into AP mode

With the unit powered on, point the remote at it and press the **LED / Display button 7 times** (the classic Midea provisioning gesture). The indoor unit's display shows **`AP`** — the dongle is now broadcasting its own hotspot named `net_ac_XXXX`.

> ⚠️ Entering provisioning mode re-binds the unit: the **MRCOOL Comfort app will lose control of it** (the unit itself keeps cooling/heating normally, and the physical remote keeps working). Keep the Comfort app installed as a fallback.

### 3. Add it in SmartHome — manually

- `+` → **Add device manually** → brand **Midea** → **Split-type AC**.
- On the model screen choose the generic **Split-type AC** (not `ES/EH` — that's for Midea's own ES/EH model lines).
- Follow the prompts: join the `net_ac_XXXX` hotspot when asked, then give it your home Wi-Fi (**2.4 GHz only**) credentials.
- Done when the unit appears on the SmartHome home screen. **Verify real control**: power, setpoint, and both indoor and outdoor temperatures should read correctly.

(The dongle accepted provisioning into a plain SmartHome account without complaint — the feared OEM tenant-lock did not materialize.)

### 4. Add it to Home Assistant via `midea_ac_lan`

Install the [`midea_ac_lan`](https://github.com/wuwentao/midea_ac_lan) custom integration (HACS or manual), restart HA, then:

- **Settings → Devices & services → Add integration → Midea AC LAN**
- Choose **Discover automatically**, enter the unit's LAN IP (discovery shows it as `net_ac_XXXX`).
- Select the air conditioner, then log in with **your new SmartHome account** — and make sure the **Server is set to "SmartHome"** (not the default 美的美居). This one-time login is what fetches the V3 token/key for *your* device.
- On the appliance form, confirm: protocol **3**, port **6444**. Submit.

### 5. Result

You get a full `climate` entity with **local polling/control**, and the entity attributes carry the whole telemetry firehose, e.g.:

- indoor temperature, **outdoor temperature**, target temperature, mode, fan speed, swing
- real-time power (W), cumulative energy (kWh), compressor frequency/current
- indoor/outdoor coil temperatures, error code

Set-temperature and mode changes from HA take effect on the unit immediately (with the usual beep). Because token/key are retrieved under your own account and stored by the integration, day-to-day control does not depend on Midea's cloud.

## Caveats

- **One-way door (probably):** after re-provisioning, MRCOOL Comfort no longer controls the unit. Re-pairing back to Comfort *should* work via the same AP-mode procedure, but that return path is itself unreported for the EZ Pro — screenshot your setup before you start.
- The fresh-email rule matters. If your SmartHome account was ever touched by another Midea-family app, make another one.
- The one cloud touch is the token/key fetch in step 4. Everything after that is LAN-local.
- Tested on: EZPRO-09-HP-11511DE-O with US-OSK109 dongle, Home Assistant 2026.9.4, midea_ac_lan 2026.9.2. Other EZ Pro sizes with the same Comfort/OSK109 kit should behave the same — reports welcome.

---

*Found this because a homeowner wanted his mini-split in the same system as the rest of his house — lights, curtains, robot vacuum, door locks, and a watchdog that turns the AC on if the room overheats while his cat is home alone. If this saves you the evening we spent, it did its job.*
