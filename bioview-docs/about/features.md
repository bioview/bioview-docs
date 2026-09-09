# Features

## Acquisition

* **Server/client architecture.** A headless server owns the hardware; the GUIs
  connect to it over TCP. The server may be on another machine, and several
  windows can share one.
* **Ettus USRPs.** Real-time transmit and receive through UHD, including several
  radios flattened into one virtual MIMO group with a configurable channel map.
* **BIOPAC.** MP36, MP150 and MP160 units alongside the RF measurement, through
  the buffered `receiveMPData` stream where the driver supports it.
    * *Requires a copy of the BIOPAC Hardware API purchased from BIOPAC Systems
      Inc.; Windows only.*
* **Microphone.** Any host input, as an ordinary device rather than a side
  channel — the audio lands in the recording sample-aligned with the rows beside
  it, so there is no `.wav` to line up afterwards.
* **Dummy backend.** Runs the entire pipeline, calibration and DPIC included,
  against a synthetic channel with nothing attached.
* **Synchronized start.** Every device in a session starts together or not at
  all. If one fails, the ones already running are stopped again — a partially
  started rig records data that cannot be aligned.

## Signal processing

* **Signal schemes.** CW, FMCW and pulsed Doppler, sharing one base class so the
  receive pipeline, live parameter updates and the calibration overlay work
  identically across them.
* **Calibration overlay.** A gated AM burst envelope that composes with any
  scheme, toggleable while streaming, with an optional recorded reference row per
  injecting transmitter.
* **DPIC.** Direct-path interference cancellation by a four-sweep coarse-to-fine
  search over injection phase and amplitude — a port of the reference LabVIEW VI,
  including its dwell times. Run on demand or automatically at Start; it is
  asynchronous end to end, so Stop stays live throughout and aborts it.

## Visualization and recording

* **Bounded, decimated display path.** Each chunk is stride-decimated to at most
  one screen's worth of points, and plot buffers are sized from each source's
  advertised display rate, so render cost tracks the grid size rather than the
  acquisition rate.
* **Self-describing recordings.** One `.bvr` file per run holds the samples, the
  ordered source descriptors, the device configuration at start, every live
  parameter change and every annotation. No sidecar files.
* **Event annotation.** Timestamped marks written into the recording itself.
* **Experiment routines.** Fixed-duration timed modes with optional audio, video
  or text instructions, each recording to its own labelled file.

## Operation

* **Device configuration utility.** The Configurator lists attached hardware,
  reports backends that failed to load and why, and assigns USRP names without
  writing to device EEPROM.
* **Live parameter changes.** Gains, IF frequencies, amplitudes, phases, channel
  masks, the calibration overlay and the channel map are editable from the
  settings strip, applied live and timestamped into the recording.
* **Shared server lifetime.** A Monitor and a Configurator can drive one rig at
  once; the server only retires when the last window does.
* **Explained failures.** A shared catalogue of known issues means a cause such
  as a driver blocked by Memory Integrity is described the same way, with the
  same remedy, wherever it surfaces.
* **Single-binary installers.** Windows, macOS and Linux bundles carrying Python,
  Qt and UHD. See [Download](../setup/downloads.md).
