# Running a session

A worked example: a B210 taking a CW measurement alongside two BIOPAC channels,
with two timed routines.

## The configuration

```json
{
  "Experiment": {
    "type": "EXPERIMENT",
    "panel_width": 0.25,
    "enable_save": true,
    "save_dir": "./recordings",
    "file_name": "usrp_biopac_session.bvr",
    "display_sources": ["Tx1Rx1", "BIOPAC Ch1"],
    "timed_modes": [
      { "label": "Baseline", "duration": "3m" },
      {
        "label": "Task",
        "duration": "90s",
        "instruction": {
          "type": "text",
          "file": "instructions/task.txt",
          "font_size": 28,
          "line_gap": 2.0
        }
      }
    ]
  },
  "USRP": {
    "type": "USRP",
    "panel_width": 0.5,
    "signal_scheme": "cw",
    "samp_rate": 1000000,
    "carrier_freq": 1000000000,
    "save_ds": 100,
    "disp_ds": 1,
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
  },
  "BIOPAC": {
    "type": "BIOPAC",
    "panel_width": 0.25,
    "model": "MP36",
    "samp_rate": 1000,
    "channels": [1, 1, 0, 0],
    "hardware": { "BIOPAC_MP36": { "channels": [1, 1, 0, 0] } }
  }
}
```

The USRP claims half the settings strip and the other two a quarter each
(`panel_width`), because a two-channel RF panel is a far wider grid than the
five rows of the experiment block.

`save_ds: 100` with `disp_ds: 1` records the RF rows at 10 kHz. Both divisors
apply to the file, not just the plot, so leaving `disp_ds` at its default of 10
would have written 1 kHz instead — see [the streaming
path](../architecture/streaming.md). BIOPAC needs no such care here: it is
already at 1 kHz and every acquired sample reaches the display.

This produces six sources:

| Source | From |
| --- | --- |
| `USRP: Tx1Rx1`, `USRP: Tx1Rx2`, `USRP: Tx2Rx1`, `USRP: Tx2Rx2` | `full_nxn` over 2 Tx × 2 Rx |
| `BIOPAC: Ch1`, `BIOPAC: Ch2` | the two enabled analog channels |

## Running it

```bash
bioview --config-file usrp_biopac_session.json
```

1. **Initialize.** Both groups come up. This is the slow step — opening the
   B210 means USB enumeration, FPGA and CODEC bring-up and clock locking, and
   `uhd.find` alone takes seconds. If one group fails, the log names it and a
   modal says which and why; the other still initializes, but streaming will
   refuse to start with a partial rig, since data that cannot be aligned across
   devices is not worth recording.
2. `Tx1Rx1` and `BIOPAC Ch1` are plotted for you the moment they are
   advertised, because `display_sources` asked for them. Tick any others you
   want and set the grid to 3×2.
3. Pick **Baseline** from the routine dropdown and press **Start**. The status
   bar shows the countdown; the stream stops itself at 3 minutes and the
   recording is written to `usrp_biopac_session_Baseline.bvr`.
4. Pick **Task** and start again. The instruction popup appears for its
   duration, and the recording lands in `usrp_biopac_session_Task.bvr`.

Free-running acquisition is the same, minus the dropdown: it records to
`usrp_biopac_session.bvr` and runs until you press Stop.

## Adjusting while streaming

Gains, IF frequencies, amplitudes, phases, the BIOPAC channel mask and the
calibration overlay can all be changed live from the settings tabs. Every change
is timestamped into the active recording's metadata, so the file records what
the rig was doing at each moment.

Changing the BIOPAC channel mask adds or removes streams. The server replies
with the new source list, and the plot-source selector follows: a channel you
just disabled disappears and gives its plot cell back; the rest keep their
ticks.

The channel map and the save target are locked while streaming.

## Two windows at once

Opening the Configurator while the Monitor is streaming is fine — both attach to
the same server, and closing either one leaves the other running. The server
only goes away when the last window does.
