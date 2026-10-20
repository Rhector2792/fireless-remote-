# Fireless Remote

A custom Android remote for **Amazon Fire TV** — built because the official Fire TV app only works over your home network. Fireless Remote gives you **two independent connection modes**, works on any Android 9+ phone, and needs **zero external dependencies** (plain Kotlin + the Android SDK).

| | Bluetooth mode | WiFi / ADB mode |
|---|---|---|
| D-pad, OK, BACK | ✓ | ✓ |
| Play/Pause, Rewind, Fast-Forward | ✓ | ✓ |
| Type into search boxes | ✓ (sends real keystrokes) | ✓ (`input text`) |
| HOME, MENU | — (keyboard HID cannot send them) | ✓ |
| Needs a shared WiFi? | **None at all** | Phone + Fire TV on one network |
| Phone requirement | Phone with working HID-device profile (test yours) | Any Android 9+ phone |
| Install on Fire TV side | Nothing | Nothing (just enable ADB debugging) |

---

## Why this project exists

- The physical Fire TV remote talks to the stick over **Bluetooth**, but Amazon's remote protocol is proprietary — third-party apps can't speak it.
- The official *Fire TV* app requires your home WiFi and does not work on a private phone-hotspot without internet.
- Android 9+ ships a public API — `BluetoothHidDevice` — that lets a phone **act as a Bluetooth keyboard**. Fire TV Sticks natively pair Bluetooth keyboards, so the phone can become the remote with no app on the TV side.
- Not every phone supports the HID-device role (several OEMs disable it), so the app also ships a **WiFi/ADB fallback** with a hand-written ADB client (classic protocol + RSA authentication) — the same tech `adb` itself uses.

## How each mode works

### Bluetooth mode
1. The app registers the phone as a HID device with a composite report map:
   - **Report ID 1** — boot-compatible keyboard (D-pad arrows, Enter = OK, Backspace = BACK)
   - **Report ID 2** — consumer-control page (Play/Pause, Rewind, Fast-Forward as flag bits)
2. Fire TV: *Settings → Controllers & Bluetooth Devices → Other Bluetooth Devices → Add Bluetooth Devices* → select the phone → confirm the PIN.
3. Button presses are sent as HID reports. No network is ever involved.

### WiFi / ADB mode
1. Enable ADB debugging on the Fire TV once: *Settings → My Fire TV → Developer Options → ADB debugging: ON*.
2. The phone scans its current network (plus common hotspot subnets) for port `5555`.
3. The app implements the **classic ADB protocol** from scratch: `CNXN` handshake, `AUTH` token exchange (RSA-2048, SHA1withRSA signatures), `OPEN`/`OKAY`/`WRTE`/`CLSE` stream handling — and sends `input keyevent <code>` shell commands for every button press.
4. First connection shows an authorization prompt on the TV; accept with **Always** and the app never asks again.
5. An optional **private hotspot** mode (via `startLocalOnlyHotspot`) keeps phone and TV on the same network even without a router. Note: a hotspot has no internet, so streaming only works in normal mode.

## Feature matrix

| Button | Code path |
|---|---|
| D-pad (hold = auto-repeat) | Keyboard usages `0x4F–0x52` / `input keyevent 19–22` |
| OK (center) | Enter `0x28` / keyevent `23` |
| BACK | Backspace `0x2C` / keyevent `4` |
| HOME, MENU | keyevent `3`, `82` (WiFi mode only) |
| ⏯ ⏪ ⏩ | Consumer bits / keyevents `85`, `89`, `90` |
| Typing / search | HID keystrokes per char / `input text` (spaces auto-escaped, unsupported chars filtered) |

Haptic feedback on every press, hold-to-repeat on the D-pad, screen stays awake while the remote is open.

## Honest limitations (no magic can fix these)

- **No power button and no Alexa voice** in either mode — Amazon's remote protocol (BLE + proprietary auth) cannot be emulated by a stock phone
- **HOME / MENU unavailable in Bluetooth mode** — plain HID keyboards cannot send them
- Bluetooth mode depends on the phone OEM enabling the HID-device role (MIUI often ships it disabled)

## Check if YOUR phone supports Bluetooth mode

Install the free **Bluetooth HID Profile Tester** ([Play Store](https://play.google.com/store/apps/details?id=com.rdapps.bluetoothhidtester)) — it tells you in seconds. Reference data: [community compatibility list](https://github.com/raghavk92/Android_Bluetooth_HID_Device_Profile_CompatibilityList).

## Build it yourself

```bash
# Requirements: JDK 17, Android SDK (platform 34, build-tools 34)
./gradlew assembleDebug      # debug APK
./gradlew assembleRelease    # release APK (signed with the included keystore)
```

- Output lands in `app/build/outputs/apk/…`
- `app/fireless-release.keystore` signs the release build — **replace it with your own keystore** if you fork this project (see `app/build.gradle.kts`)
- No AndroidX, no libraries, no internet permission beyond ADB itself

## Project structure

```
app/src/main/java/com/fireless/remote/
├── MainActivity.kt            remote UI (D-pad, media, haptics, auto-repeat)
├── SetupActivity.kt           connection wizard (scan / hotspot / mode switching)
├── HidConstants.kt            HID report descriptor + usage constants
├── HidPeripheralManager.kt    phone-as-Bluetooth-keyboard (BluetoothHidDevice API)
├── adb/
│   ├── adbProtocol.kt         frame constants (CNXN/AUTH/OPEN/OKAY/CLSE/WRTE)
│   ├── AdbClient.kt           hand-written ADB protocol client + RSA auth
│   └── AdbKeyPair.kt          RSA-2048 key generation & signing
└── remote/
    ├── RemoteController.kt    mode switching, key routing, connection state
    ├── HotspotSupport.kt      hotspot credentials extraction (OEM-proof)
    └── NetworkScanner.kt      subnet detection + ADB port scanning
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| "registerApp FAILED" | Your phone's OEM disabled the HID-device role — use WiFi mode, or root + Magisk HID-enabler module |
| Hotspot fails to start | Turn **Location services ON** + grant Location permission (Android requirement, not ours) |
| "No ADB device found" | Fire TV: ADB debugging ON + same network as the phone |
| Connect hangs forever | The authorization prompt is waiting on the TV screen — accept it there |
| Keys not received | Re-accept the TV prompt (remove old pairing in Fire TV settings first) |

## FAQ

**Why no voice search?** Alexa voice runs through Amazon's own encrypted protocol. Not emulatable — that's the one thing only the original remote does.

**Is this safe for my Fire TV?** It only sends standard `input keyevent` commands — the same thing every approved testing tool sends. Nothing is installed or modified on the stick.

**WiFi password sharing?** Never. The app never sees your password. If you change your WiFi password later, just reconnect your phone and stick like any other device.

## Credits

- [google/python-adb](https://github.com/google/python-adb) — reference for the ADB protocol framing and auth flow
- [relay-hid](https://github.com/Abdk4Moura/relay-hid), [wearmouse](https://github.com/ginkage/wearmouse) — reference patterns for `BluetoothHidDevice` usage
- [raghavk92's compatibility list](https://github.com/raghavk92/Android_Bluetooth_HID_Device_Profile_CompatibilityList) — HID-device profile support data

## License

Add your chosen license before publishing (e.g. MIT) — delete this section if the project stays private.
