# bluetooth-audio-watchdog

A small user-level systemd service that detects a silent failure mode in the
Linux Bluetooth audio stack and tells you (and you alone — there's no fleet,
no telemetry) what's actually wrong.

## The failure it catches

BlueZ can mark a Bluetooth headset as `Connected: true` while the audio
transport (`org.bluez.MediaTransport1`, i.e. A2DP/HFP) never gets established.
When that happens:

- AVRCP (media control) comes up, so the device looks "connected" in GNOME.
- No PipeWire sink appears, so there is no audio output to pick.
- Nothing in BlueZ, WirePlumber, or GNOME notices or surfaces the gap.

AirPods on Linux land in this state regularly (paired in Find-My BLE mode,
held by another iCloud-bonded device, asleep in beacon mode, …). This
watchdog polls BlueZ and, when a device sits in that state past a threshold,
classifies *why* and fires a desktop notification + journald entry with a
remediation that matches the actual cause.

## Install

```sh
# clone wherever you like
git clone <repo-url> ~/src/bluetooth-audio-watchdog
cd ~/src/bluetooth-audio-watchdog

# symlink the binary and the user unit into the standard locations
mkdir -p ~/.local/bin ~/.config/systemd/user
ln -s "$PWD/bin/bluetooth-audio-watchdog"           ~/.local/bin/bluetooth-audio-watchdog
ln -s "$PWD/systemd/user/bluetooth-audio-watchdog.service" \
                                                    ~/.config/systemd/user/bluetooth-audio-watchdog.service

systemctl --user daemon-reload
systemctl --user enable --now bluetooth-audio-watchdog
```

Requires `busctl`, `journalctl`, `notify-send`, `systemd-cat` — all present
on a default Ubuntu/GNOME desktop.

## Configuration

Set via `Environment=` in the unit, or by editing the unit override
(`systemctl --user edit bluetooth-audio-watchdog`):

| Variable | Default | Meaning |
| --- | --- | --- |
| `BT_WATCHDOG_THRESHOLD` | `15` | Seconds a device must sit in "connected, no transport" before alerting. |
| `BT_WATCHDOG_POLL`      | `3`  | Poll interval in seconds. |
| `BT_WATCHDOG_AUTOFIX`   | `0`  | If `1`, also call `Device1.Disconnect` on a stuck device after alerting. Off by default — alert-only is safer because legit transient states (AirPods returning to case briefly) can look identical for a few seconds. |

## How it classifies

After threshold the watchdog reads:

- `BREDR.Paired` / `LE.Paired` from `busctl introspect`
- `journalctl -u bluetooth` in the last 2 min, filtered by MAC, counting
  AVDTP "Connection reset by peer" lines (`avdtp_err`) and any AVDTP
  activity (`avdtp_any`)

and picks one of four branches:

| Branch | Trigger | Diagnosis | Suggested fix |
| --- | --- | --- | --- |
| **A** | `BREDR.Paired = false` and `LE.Paired = true` | Only an LE/Find-My bond exists; A2DP can never come up. | `bluetoothctl remove <mac>` then re-pair with the AirPods in your ears and the case button held until the LED is solid white. |
| **B** | `avdtp_err > 0` | Peer is rejecting the A2DP channel. | Turn Bluetooth off on other Apple devices on the same iCloud (Settings, not Control Center), take AirPods out of case, reconnect. |
| **C** | `avdtp_any = 0` (no AVDTP attempt at all) | AirPods are asleep in BLE beacon mode and not responding to BR/EDR. | Make sure no other iCloud-bonded device is claiming them, put them in ears (in-ear sensor keeps them awake), then `bluetoothctl connect <mac>`. |
| **D** | otherwise | A2DP/HFP didn't come up despite a clean pair. | `bluetoothctl disconnect <mac>; sleep 3; bluetoothctl connect <mac>`; check `journalctl -u bluetooth` for the AVDTP error. |

The `- Find My` suffix in the BlueZ device alias is intentionally **not** used
as a signal — it just reflects whatever the AirPods are currently
broadcasting on BLE, which idle AirPods always do. The pairing record is the
authoritative source.

## Subcommands

```sh
bluetooth-audio-watchdog                       # poll loop (what the unit runs)
bluetooth-audio-watchdog --self-test           # 8 classify() cases, exit 0/1
bluetooth-audio-watchdog --show-all            # print all 4 branch texts, no popups
bluetooth-audio-watchdog --show-all notify     # also fire 4 real desktop popups
bluetooth-audio-watchdog --alert B <mac> [name]  # emit one branch end-to-end
```

`--self-test` runs the pure `classify()` function against canned inputs and
is the right thing to wire into CI or a pre-commit hook if you edit the
heuristic.

## Logs

```sh
journalctl --user -u bluetooth-audio-watchdog -f    # service stdout/stderr
journalctl -t bt-audio-watchdog                     # structured warning entries
```

## Limitations

- The classifier reads BR/EDR/LE pair state via `busctl introspect` parsing.
  If BlueZ changes its bearer-interface shape, the awk extraction can fail
  silently — `classify()` then treats both as unpaired and falls into branch
  C, which is the least-bad default.
- Only the user-session DBus is monitored; system-level bluetoothd state is
  what's actually read, but `notify-send` requires a graphical session.
- AVDTP log heuristics depend on bluez writing those exact debug lines,
  which it does in BlueZ 5.85 (Ubuntu 26.04); other versions may need the
  grep patterns adjusted.
