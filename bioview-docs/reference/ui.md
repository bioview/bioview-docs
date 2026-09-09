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
| Experiment  | USRP1       | USRP2      >> (scrolls sideways)  |
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

The grid is sized at startup to hold every `display_sources` entry, so a
configuration asking for five plots opens at 2x3 rather than the default 2x2.
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
* **Activity** — what the session is doing right now.

The bar is a readout, not a control: only **More…** takes the mouse, and the
labels, dots and progress bars beside it are transparent to it, so crossing the
strip does not put anything into a hover state. **More…** carries an ordinary
button frame rather than being flat -- a flat button is invisible until the
cursor reaches it and then paints a bare rectangle of highlight into an empty
strip, which reads as the *bar* lighting up rather than the button.

### Activity messages

State changes are announced in the bar as well as in the log, which is closed by
default:

| Message | When |
| --- | --- |
| `Initialization started…` | Initialize pressed; stays up until it finishes |
| `Initialized successfully!` | Every device group came up |
| `Initialized with errors (n/m device groups ready)` | Some came up, some did not |
| `Initialization failed` | No device group came up |
| `Discovering devices…` | Discover pressed |
| `Streaming started` / `Streaming stopped` | The stream changed state |

Each level has its own colour: blue for something in progress, green for
success, red for failure, orange for a warning, and a muted yellow for the
stream starting and stopping. Streaming is not reported in green -- it is the
routine thing the application does, and green is kept for something having gone
right.

A message about something still running stays up until it is replaced; a
finished-state message clears itself after four seconds.

A **partial** initialization also raises a modal naming each group that failed
and why, using the same wording the log and the Configurator use. Nothing else
distinguishes a half-initialized rig from a healthy one until a plot stays flat,
and discovery is excluded — a device merely absent from a scan is not a failure.

## Mark Event

Writes a timestamped annotation into the active recording's metadata trailer.
It needs a save target and an active recording; annotations attach to a
recording, and there are no sidecar files.

## Settings strip

One panel per configuration block, laid out side by side rather than stacked
into tabs. The window's width decides how many are visible at once — one below
900 px, two below 1300, three below 1750, four above — and any further panels
are reached by scrolling the strip sideways. Panels always divide the visible
width exactly, so a half-visible panel means there is more to the right.

Device panels expose that backend's runtime parameters — gains, IF frequencies,
amplitudes, phases, channel masks, the calibration overlay, the channel map,
DPIC balance. Changes are applied live and are recorded, timestamped, into the
active recording's metadata.

The channel map is frozen while streaming, since it changes how many rows the
backend emits. The calibration overlay is not: it is designed to be toggled
live.

**Balance** starts the DPIC search and returns immediately; the button reads
*Balancing…* until the server reports the run has finished, and the rest of the
UI — including Stop — stays usable throughout. Stopping the stream aborts a
running balance rather than waiting out its time budget. The inject Tx is
retuned to the measure Tx's IF for the run, and the IF box updates to match:
the receive chain band-passes around the measure Tx, so a cancellation tone
anywhere else cannot reach it.

## Log

Debug level traces every control exchange — what was asked, what came back and
how long it took. The device-status poll is excluded, since it repeats every
couple of seconds and would bury everything else.

Errors that BioView recognises are rendered through the shared issue catalogue,
so a cause such as a driver blocked by Memory Integrity reads the same here as
it does in the Configurator.
