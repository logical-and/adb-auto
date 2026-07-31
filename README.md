# adb-auto

Connect `adb` to Android devices over Wi-Fi without hunting for IP addresses, ports, or pairing codes.

Android randomises the wireless-debugging port on every toggle and every reboot, so yesterday's
`adb connect 192.168.1.50:37401` is dead today. `adb-auto` discovers the current endpoint over mDNS
and connects. When nothing is reachable, it walks you through a QR-code bootstrap instead of failing
with instructions.

```
$ adb-auto
Advertised on the LAN:
  10.10.10.106:43105  SM-F966B
  connected 10.10.10.106:43105

adb devices:
10.10.10.106:43105     device product:q7qxeea model:SM_F966B device:q7q
emulator-5554          device product:sdk_gphone64_x86_64 model:sdk_gphone64_x86_64
```

## Install

```bash
curl -o ~/.local/bin/adb-auto https://raw.githubusercontent.com/logical-and/adb-auto/master/adb-auto
chmod +x ~/.local/bin/adb-auto
```

Requirements:

- `adb` (Android SDK platform-tools) on `PATH`, or set `ADB=/path/to/adb`
- `avahi-utils` for discovery (`sudo apt install avahi-utils`)
- optional, for the QR flows: `python3` plus `segno` (`python3 -m pip install --user segno`)

## Usage

```
adb-auto            Discover and connect. Falls back to the QR bootstrap when
                    nothing is reachable.
adb-auto --qr       Pairing QR only
adb-auto --settings QR that opens Settings on the phone
adb-auto --pair     Pairing-code mode (prompts for the 6 digits)
adb-auto --list     Show what is advertised, connect to nothing
```

### The bootstrap

Run `adb-auto` with nothing connected and it stages the setup, skipping any step you do not need:

1. Shows a QR that opens **Settings** on the phone
2. You turn on Developer options > Wireless debugging
3. It detects the device and tries to connect
4. Only if the device is not paired, shows a **second** QR for the pairing scanner
5. Connects

Step 4 is skipped whenever the phone is still paired with the machine, which is the usual case after
a reboot or a toggle.

## Why the QR opens Settings and not Wireless debugging

Because Android will not allow anything else. A scanned QR is delivered to the system as a
browser-style navigation, and Android only honours activities that declare `CATEGORY_BROWSABLE`.
The Developer options screen does not:

```
$ adb shell am start -a android.settings.APPLICATION_DEVELOPMENT_SETTINGS \
    -c android.intent.category.BROWSABLE
Error: Activity not started, unable to resolve Intent

$ adb shell am start -a android.settings.SETTINGS -c android.intent.category.BROWSABLE
Starting: Intent { act=android.settings.SETTINGS cat=[android.intent.category.BROWSABLE] }
```

`android.settings.ADB_WIRELESS_SETTINGS` does not resolve at all on the devices tested (Samsung
SM-F966B on API 36, and the AOSP emulator). So the QR encodes
`intent://#Intent;action=android.settings.SETTINGS;end` and the last two taps stay manual.

If a device is already attached over USB, `adb-auto` skips the QR entirely and enables wireless
debugging directly with `settings put global adb_wifi_enabled 1`, which needs no root.

## What it handles

**Ports that change.** The endpoint is rediscovered on every run. The last working one is cached and
retried when the phone stops advertising (screen asleep) but keeps the port open.

**Stale adb entries.** Old `ip:port` entries left over from previous toggles are disconnected, so
`adb devices` does not fill up with dead transports. Entries for other IPs and for emulators are
never touched.

**Telling failures apart.** `Connection refused` means wireless debugging is off, so it offers the
Settings QR. A device that answers but rejects the handshake has lost its RSA pairing, so it offers
the pairing QR. These look identical in `adb devices` and need opposite fixes.

**Stale mDNS records.** Avahi keeps serving a cached record for a while after wireless debugging is
switched off, so an advert alone is not trusted: the port is probed before a device counts as ready.

**Multi-homed hosts.** Docker and libvirt bridges echo the same mDNS advert on every virtual
interface, turning one phone into a dozen unroutable candidates. Only real interfaces are considered.

## Discovery

Discovery goes through `avahi-browse`, not `adb mdns services`. On many setups adb's default
`libadbmdns` backend returns an empty list while avahi resolves the same records instantly.

`ADB_MDNS_OPENSCREEN=1` also fixes adb's own discovery, but `adb-auto` deliberately avoids it: that
backend auto-connects devices under an mDNS-name serial
(`adb-<guid>._adb-tls-connect._tcp`), so a phone also reached by `ip:port` appears **twice** in
`adb devices`. That breaks scripts that count attached devices or pick "the non-emulator one".
`adb-auto` actively removes those duplicates.

## Note on pairing codes

Pairing cannot be fully automated in pairing-code mode. Android generates the 6-digit secret and
displays it only on the phone, with no host-side way to read it. Everything else is automatic. Use
`--qr`, where the host generates the secret and the phone's camera reads it, for a flow with nothing
to type.

## License

MIT
