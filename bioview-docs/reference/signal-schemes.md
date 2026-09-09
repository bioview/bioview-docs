# Signal schemes

A signal scheme decides what a USRP (or the dummy RF backend) transmits, and
what the demodulator subtracts on the way back. It is selected per device group
with `"signal_scheme"`.

All schemes live in `bioview_common.signal_schemes` and share one base class, so
the receive pipeline, calibration overlay and runtime parameter updates work
identically across them.

## `cw` (default)

A continuous tone per Tx channel at that channel's `if_freq`, with a per-channel
amplitude and static phase. The receiver downconverts by the same IF, low-passes
at `if_filter_bw`, and reports amplitude and differential phase per Tx/Rx pair.

This is what DPIC and the standard biomedical measurements use.

```json
{ "signal_scheme": "cw" }
```

## `fmcw`

A repeating linear frequency chirp.

```json
{
  "signal_scheme": "fmcw",
  "fmcw": {
    "chirp_start_hz": 50000,
    "chirp_end_hz": 150000,
    "chirp_duration_s": 0.001,
    "idle_time_s": 0.001
  }
}
```

The instantaneous phase is `2π(f₀t + ½kt²)` with `k` the sweep rate. The Tx
cycle is `chirp_duration_s + idle_time_s`, which lets the transmitter replay one
pre-built cyclic buffer instead of synthesizing every sample.

## `pulsed_doppler`

Gated pulses at a fixed pulse repetition interval.

```json
{
  "signal_scheme": "pulsed_doppler",
  "pulsed_doppler": {
    "pulse_width_s": 1e-5,
    "pri_s": 1e-3,
    "doppler_if_hz": 100000
  }
}
```

## Calibration overlay

Any scheme can carry a gated AM burst envelope for amplitude calibration. It is
an overlay, not a scheme, so it composes with CW, FMCW and pulsed Doppler alike.

```json
"calibration": {
  "enabled": false,
  "shape": "triangle",
  "num_pulses": 5,
  "pulse_duration_s": 0.1,
  "packet_spacing_s": 1.0,
  "envelope_freq_hz": 10.0,
  "modulation_depth": 0.2,
  "inject_channels": [0],
  "record_reference": true
}
```

| Key | Meaning |
| --- | --- |
| `shape` | `triangle`, `sawtooth` or `rectangle`. |
| `num_pulses` | Pulses per burst packet. |
| `pulse_duration_s` | Duration of one pulse, i.e. the shape's period. |
| `packet_spacing_s` | Period between burst packets: burst start to burst start. |
| `envelope_freq_hz` | Shape frequency, used only when `pulse_duration_s` is absent. |
| `modulation_depth` | AM depth applied to the carrier. |
| `inject_channels` | **Global** Tx indices that carry the burst. |
| `record_reference` | Advertise a `CalRef_*` source per injecting Tx. |

A burst lasts `num_pulses * pulse_duration_s` and repeats every
`packet_spacing_s`; the remainder of that interval is silence. A burst longer
than its own repeat interval is clamped to it. `pulse_duration_s` and
`envelope_freq_hz` name the same quantity from opposite ends — the duration
wins when both are given, and the default 0.1 s is exactly `1 / 10 Hz`, so the
two agree out of the box. `envelope_freq_hz` is kept because that is what the
reference `B210_2CHANNEL` calls it.

Three things to know:

* **The burst timings are live.** `num_pulses`, `pulse_duration_s` and
  `packet_spacing_s` are spin boxes on the RF settings panel's second
  calibration row, alongside the shape and depth, and take effect on the next
  Tx buffer.

* **`inject_channels` is in global Tx indexing** but a scheme only ever sees its
  own device's channels, so the factory always translates it — the default `[0]`
  included. Without that a second device silently injects the burst on its own
  local Tx0 as well as on global Tx0.
* **The overlay can be toggled while streaming.** The Monitor's checkbox routes
  through the Tx command queue rather than mutating the scheme object, which
  would race waveform generation. Enabling calibration also disables the cyclic
  Tx buffer, since the waveform is no longer periodic over a short cycle.

## Phase conventions

`tx_phase_static()` is the programmed Tx phase without the running
`2πf_IF·n/fs` carrier ramp; that is what the demodulator subtracts, because the
receive downconversion has already removed the ramp. `tx_phase_at()` includes
it. Subtracting the ramp twice turns the recorded phase channel into a linear
ramp instead of a channel measurement.
