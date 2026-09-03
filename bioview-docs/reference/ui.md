# Monitor UI

## Layout

```
+--------------------------------------------------------------+
| Initialize | Start | [routine v] | [x] Save | Stop            |
+---------------------------+----------------------------------+
| Experiment log            | Mark event                        |
+---------------------------+----------------------------------+
|                                                              |
|                     Plot grid (rows x cols)                  |
|                                                              |
+--------------------------------------------------------------+
| Experiment | USRP | BIOPAC   settings tabs                    |
+--------------------------------------------------------------+
| status bar: server picker, device indicators, routine progress|
+--------------------------------------------------------------+
```

The plot grid and the panels above it sit in a splitter; plots get expansion
priority and start at roughly half the window height. The window opens
**maximized**, not fullscreen, so the title bar and taskbar stay visible.
`F11` enters true fullscreen and `Esc` leaves it, returning to maximized.

## Command bar

| Control | Behaviour |
| --- | --- |
| **Initialize** | Bring up the device groups in the loaded configuration. Disabled until a server is connected. |
| **Start** | Begin free-running acquisition. |
| **routine** | Pick a declared timed mode. Starting one requires connected devices, no active stream, and a valid save target — routines always save. |
| **Save** | Record while streaming. Requires both a file name and a save folder; enabling it without them warns and reverts. |
| **Stop** | Stop the stream, and tear down any running routine with it. |

Grid layout and the save target are locked while streaming.

## Plot sources

The **Plot Sources** selector in the Experiment tab lists every source the
server advertises, named `Device: Source` — `BIOPAC: Ch1`, `USRP: Tx1Rx1`.
Channel labels are only unique within a device, so the device prefix is what
keeps two devices' channels apart. The plot title carries exactly the same name.

Ticking a source assigns it the lowest free grid cell; unticking releases the
cell. When the server's source list changes — enabling a BIOPAC channel, say —
the selector is reconciled rather than rebuilt blindly: sources that have gone
away are unplotted and their cells released, and surviving sources keep their
ticks. Shrinking the grid unplots whatever no longer fits and unticks it.

**Display Time** sets the visible window in seconds. Plot buffers are sized from
each source's advertised display rate, so a 1 kHz BIOPAC channel and a decimated
RF channel both fill the same window correctly.

## Status bar

* **Server picker** — discovered servers, and connect/disconnect. A local server
  is found and connected automatically; the picker is for reaching a remote one.
* **Device indicators** — one dot per device group: green connected, orange
  disconnected, blinking yellow connecting, blue streaming.
* **Routine progress** — the running routine's label and countdown.

## Mark Event

Writes a timestamped annotation into the active recording's metadata trailer.
It needs a save target and an active recording; annotations attach to a
recording, and there are no sidecar files.

## Settings tabs

One tab per configuration block. Device tabs expose that backend's runtime
parameters — gains, IF frequencies, amplitudes, phases, channel masks, the
calibration overlay, the channel map, DPIC balance. Changes are applied live and
are recorded, timestamped, into the active recording's metadata.

The channel map is frozen while streaming, since it changes how many rows the
backend emits. The calibration overlay is not: it is designed to be toggled
live.

## Log

Debug level traces every control exchange — what was asked, what came back and
how long it took. The device-status poll is excluded, since it repeats every
couple of seconds and would bury everything else.

Errors that BioView recognises are rendered through the shared issue catalogue,
so a cause such as a driver blocked by Memory Integrity reads the same here as
it does in the Configurator.
