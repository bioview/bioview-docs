# USRP

BioView drives Ettus USRPs (B200/B210 and compatible) through the `USRPBackend`
server backend, using UHD's Python bindings.

## Prerequisites

* UHD with Python bindings (`import uhd` must work in the server's interpreter).
* FPGA images for the device. The frozen bundle ships them and points
  `UHD_IMAGES_DIR` at itself.

If `import uhd` fails, the USRP backend is recorded in `UNAVAILABLE_BACKENDS`
with the reason and reported to the Configurator — it does not silently vanish
from the device list.

UHD's Windows binaries are built against NumPy 1.x. Importing them under
NumPy 2.x works but prints a mismatch traceback from inside `import uhd`;
BioView detects that and says plainly what it is.

## Device names

`uhd.find` reports a radio's EEPROM name. BioView layers a **user-assigned
alias** on top of it, keyed on the radio's serial and stored on this machine —
no EEPROM is ever written. That makes renaming safe and reversible.

Two caches back this:

| Cache | Maps | Used for |
| --- | --- | --- |
| `usrp_serial_numbers` | name → serial | Turning the name in a configuration file into something `uhd.find` can address. |
| `usrp_device_aliases` | serial → assigned name | Overlaid during discovery. |

Because the alias is applied *at discovery*, configuration files, the channel
map and the serial cache all just see the chosen name; nothing downstream knows
an alias is involved.

Renaming is the one editable property the Configurator exposes today. Note that
a renamed radio keeps announcing its old name over the wire until it is power
cycled — BioView's alias is already in effect, but the device itself has not
caught up.

`uhd.find` results are cached for a few seconds, and the alias is overlaid while
building that cache, so a rename drops the cache; otherwise the very next
listing would still report the old name.

## Concurrency

`uhd.find()` is **not safe to call concurrently**. The server calls it from
per-client command threads as well as from the device-init path, and two
overlapping calls have taken the process down with a Windows access violation.
It is serialized behind a lock, which also makes the discovery cache's
read/refresh atomic so a second caller waits for the first result instead of
re-running discovery against a half-cleared cache.

## Configuration

```json
"USRP": {
  "type": "USRP",
  "signal_scheme": "cw",
  "samp_rate": 1000000,
  "carrier_freq": 1000000000,
  "hardware": {
    "MyB210_3": {
      "tx_channels": [0, 1],
      "rx_channels": [0, 1],
      "if_freq": [100000, 110000],
      "tx_gain": [40, 40],
      "rx_gain": [40, 40],
      "tx_amplitude": [1.0, 1.0],
      "tx_phase": [0.0, 0.0],
      "if_filter_bw": 5000
    }
  },
  "channel_map": { "layout": "full_nxn", "dpic": [] }
}
```

Group-level keys (`samp_rate`, `carrier_freq`, `signal_scheme`) apply to every
radio in the group; per-channel lists are in each device's local channel order.

Several radios in one group form a virtual MIMO device — see
[Channel maps and MIMO](channel-map.md), [Signal schemes](signal-schemes.md) and
[DPIC](dpic.md).

## Runtime parameters

Gains, amplitudes, phases, the calibration overlay and the DPIC weights can all
be changed while streaming. They go through each transmit worker's command
queue rather than mutating the scheme object, which would race waveform
generation on another thread.

Assumptions BioView makes about a USRP group: two working channels per device,
default data formats, an internal clock and timing reference, and unit-amplitude
waveforms.

## Notes from the field

* B210s behave poorly with the default frame size; BioView's default receive
  frame size is 1024.
* Keeping the receive buffer small produces spikes in the data from filtering
  edge effects.
* Starting a stream spawns transmit, receive and process workers per device. On
  Windows each is a full process spawn, which is why `START_STREAMING` is
  allowed 90 seconds server-side and 120 seconds on the wire.
