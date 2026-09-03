# Configuration files

A BioView configuration is a single JSON object. Each top-level key is either
the experiment block or one **device group**; the key is the group's id and
appears in every source name, log line and status row.

```json
{
  "Experiment": { "type": "EXPERIMENT", "...": "..." },
  "USRP":       { "type": "USRP",       "...": "..." },
  "BIOPAC":     { "type": "BIOPAC",     "...": "..." }
}
```

`type` selects the configuration class: `EXPERIMENT`, `USRP`, `BIOPAC` or
`DUMMY`. A file with no `EXPERIMENT` block still gets a default one, so the
plot-source selector and the save controls always exist.

## Experiment block

| Key | Default | Meaning |
| --- | --- | --- |
| `enable_save` | `false` | Record to disk while streaming. |
| `save_dir` | `null` | Folder for recordings. |
| `file_name` | `""` | Base name; a numeric suffix is added if it already exists. |
| `display_sources` | `[]` | Sources ticked in the plot selector at startup. |
| `timed_modes` | `[]` | Routines, see below. |

Saving requires both `file_name` and `save_dir`. Enabling it without them is
refused and the checkbox reverts.

### Timed modes (routines)

A routine pairs a fixed duration with an optional instruction presented while
data is collected. Routines always save; the recording is named
`<file_name>_<label>.bvr`. Free-running ("unlimited") mode is always available
alongside them.

```json
"timed_modes": [
  {
    "label": "Baseline",
    "duration": "3m",
    "instruction": {
      "type": "text",
      "file": "instructions/baseline.txt",
      "font_size": 28,
      "line_gap": 2.0
    }
  },
  {
    "label": "Task",
    "duration": 90,
    "instruction": { "type": "audio", "file": "instructions/task.mp3", "loop": false }
  }
]
```

* `duration` — a number of seconds, a unit-suffixed string (`"3m"`, `"90s"`,
  `"1h"`), or a clock string (`"mm:ss"`, `"hh:mm:ss"`).
* `instruction.type` — `audio`, `video` or `text`.
* `loop` — audio and video only; default is play once.
* `font_size`, `line_gap` — text only. Omit `line_gap` to show all lines at once.

Audio needs no window; text and video open a popup.

## USRP block

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

`hardware` is keyed by device name (see [USRP](usrp.md) for how names are
assigned). Several radios in one group form a **virtual device**: their channels
are flattened into global Tx and Rx indices in the order the devices appear, and
`channel_map` and `dpic` are written in those global indices.

Per-channel parameters are lists indexed by that device's local channel order.

See [Channel maps and MIMO](channel-map.md) and
[Signal schemes](signal-schemes.md).

## BIOPAC block

```json
"BIOPAC": {
  "type": "BIOPAC",
  "model": "MP36",
  "samp_rate": 1000,
  "channels": [1, 1, 0, 0],
  "hardware": { "BIOPAC_MP36": { "channels": [1, 1, 0, 0], "labels": ["ECG", "Resp"] } }
}
```

| Key | Default | Meaning |
| --- | --- | --- |
| `model` | `MP36` | `MP36`, `MP150` or `MP160`. |
| `samp_rate` | `1000` | Hz. Also the display rate: every acquired sample reaches the plots. |
| `channels` | `[1,1,1,1]` | Enable mask, one entry per analog channel. |
| `labels` | — | Per-channel names; defaults to `Ch1`…`ChN` over the *enabled* channels. |
| `mpdev_path` | `null` | Explicit path to `mpdev.dll` when it is not otherwise discoverable. |
| `connection_type` | `10` | mpdev connection code. |
| `port` | `"auto"` | mpdev port. |
| `disp_ds` | `10` | Sets the acquisition chunk size, capped at 100 ms of data. |
| `save_ds` | `1` | Save decimation. |

Parameters listed under `hardware` win over the top-level copy, so an edit made
in the UI is written to both. See [BIOPAC](biopac.md).

## Dummy block

The virtual device. Without `hardware`/`channel_map` it synthesizes
phase-shifted sine waves; with them it runs the same CW, calibration and DPIC
pipeline as the USRP backend against a virtual MIMO channel model — which is how
the RF path is tested with no radio attached.

Dummy devices are excluded from the Configurator's listing unless explicitly
asked for: that window lists hardware that is actually attached, and a simulated
device sitting among the real ones is misleading.
