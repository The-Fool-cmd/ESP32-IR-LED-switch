# ESP32 IR desk light switch

ESPHome firmware for a wave-to-toggle LED strip: an IR proximity sensor on one GPIO, a
dimmable strip on another, and Home Assistant on the other side of Wi-Fi.

## How it works

Everything lives in `desk_light.yaml` and runs on the device:

1. Boot, connect to Wi-Fi, come up on the Home Assistant API.
2. Watch the IR sensor on GPIO18.
3. On a wave, toggle the strip on GPIO19 with a 1000 (adjsutable) ms fade.
4. Stay awake and connected — the device is mains-powered and always on.

The strip is also a normal ESPHome `light` entity, so Home Assistant (or anything talking
to the API) can set brightness and state directly. The wave sensor is just one more way to
toggle it; it doesn't own the state.

`restore_mode: ALWAYS_OFF` means the strip comes up off after a reboot or a power blip,
regardless of what it was doing before.

## Hardware

- Any ESP32 devkit (`board: esp32dev`).
- IR proximity / break-beam sensor → **GPIO18**, digital output, active-high.
- LED strip via a MOSFET or driver module → **GPIO19**, PWM at 1 kHz.

The sensor is debounced in firmware: it has to read high for 50 ms before it counts as a
wave (`delayed_on`), and low for 250 ms before it can trigger again (`delayed_off`). That
second number is what stops one slow hand pass from toggling the light several times.

## Setup

```
cp secrets.yaml.example secrets.yaml
```

Fill in `secrets.yaml`:

- `wifi_ssid` / `wifi_password` — your network.
- `api_key` — 32-byte base64 encryption key for the Home Assistant API
  (`openssl rand -base64 32`).
- `ota_password` — password for over-the-air updates.

If Wi-Fi doesn't come up, the device starts an open fallback AP called
**Desk Light Fallback** with a captive portal for entering credentials.

## Flashing

First flash has to be over USB:

```
esphome run desk_light.yaml --device /dev/ttyUSBx
```

To find the port: `udevadm monitor --subsystem-match=tty --property`, and watch for the
`add` event when you plug the board in.

After that, OTA works — the device stays awake and on the network, so subsequent updates
can go over the air:

```
esphome run desk_light.yaml
```

Logs, either way:

```
esphome logs desk_light.yaml
```

## Configuration

Everything tunable is inline in `desk_light.yaml`:

- `output.frequency` (1000Hz) — PWM frequency for the strip. Raise it if you see flicker
  on camera; some drivers don't like going too high.
- `default_transition_length` (500ms) — fade time on toggle.
- `restore_mode` (ALWAYS_OFF) — state after boot. `RESTORE_DEFAULT_OFF` will bring back
  whatever it was before the reboot instead.
- `filters.delayed_on` / `delayed_off` on the wave sensor — debounce and retrigger
  spacing, described above.
- `pin.inverted` on GPIO18 — flip this if your sensor pulls its output *low* on detect.
