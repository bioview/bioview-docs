# Changelog

All four repositories — `bioview-common`, `bioview-server`, `bioview-client` and
`bioview-installer` — are versioned and tagged together by `release.sh`, so one
version number describes the whole system. Pushing the installer tag is what
triggers the CI matrix that builds and attaches the platform bundles.

Installers for every release are on the [releases
page](https://github.com/bioview/bioview-installer/releases); the current one is
covered on [Download](../setup/downloads.md).

## Unreleased

* **Microphone backend.** Host audio input through PortAudio, as a device rather
  than a side channel: one row per captured channel down the ordinary display
  path, so speech is recorded sample-aligned with the RF and physiological rows
  beside it. Capture is callback-based (blocking reads are unavailable on
  WDM-KS), the sample rate is negotiated against what the input will actually
  accept rather than resampled, and loopback inputs such as "Stereo Mix" are
  skipped when no default input exists.
* **Instruction audio routing.** `audio_output_device` in the experiment block
  names the output that routine instructions play through, matched exactly and
  then as a case-insensitive substring. A machine with HDMI audio alongside
  speakers has no useful default, and a routine playing to a device nobody is
  listening to leaves nothing behind to show it happened.
* **Instruction files are checked at startup** rather than at playback, and a
  playback failure raises a toast over the window.
* **The plot grid is sized for the configured `display_sources` at startup**, up
  to the 4×3 the layout controls allow, so a configuration asking for five plots
  no longer silently drops the fifth against a 2×2 default.
* **Activity messages in the status bar.** Initialization, discovery and stream
  transitions are announced where the operator is looking, not only in a log
  panel that is closed by default. A partial initialization also raises a modal
  naming each device group that failed and why.

## 0.9.6 — 8 September 2026

* **DPIC rewritten as the reference search.** The closed-form solve was removed;
  cancellation is now the four-sweep coarse-to-fine search from the
  `Pig_2Ch_NCS_BIOPAC_BalanceSignal` LabVIEW VI, including its per-point dwell
  times, with a test asserting parity against the VI's behaviour. See
  [DPIC](../reference/dpic.md).
* **Balance is asynchronous end to end.** The backend runs the search on its own
  thread, the server acknowledges and publishes the outcome for polling, and the
  client polls from a worker. Stop stays responsive throughout and aborts a
  running balance instead of waiting out its time budget.
* **Balance progress is visible.** Every measurement reports its stage, index,
  applied value, metric and both gains; the values land in the matching spin
  boxes and the Balance button carries the stage.
* **Inject IF coercion.** A pair's inject transmitter is retuned onto its measure
  transmitter's IF before the search, because the receive chain band-passes
  around the measure IF and an injection anywhere else cannot reach it.
* **Live channel-map editing.** The map, including the DPIC pair list, can be
  edited from the settings panel and applied without re-initializing the device.
  It stays frozen while streaming, since it decides how many rows the pipeline
  emits.
* **Display rate fixed.** `disp_ds` was plumbed but never applied and RF sources
  never advertised a rate, so traces scrolled at up to fifty times real time.
  Both RF backends now stamp `samp_rate / (save_ds * disp_ds)` onto every source
  they advertise.
* **Environment guards removed.** Runtime probes for dependency versions that the
  pins already guarantee — the NumPy C-API warm-up, the UHD/NumPy ABI check, the
  tune-request fallbacks — were deleted rather than left printing to a stdout
  nobody reads.

## 0.9.5.2 — 4 September 2026

* Simultaneous BIOPAC and USRP acquisition. Each backend was given its own
  response queue: a queue shared between backends let one device consume
  another's reply, which is why a slow USRP start surfaced as an empty
  `Failed to start streaming:`.
* BIOPAC data reached the plots, and at the right rate — the display path now
  carries every acquired sample, with the chunk size capped at 100 ms so a low
  `disp_ds` at a high sample rate cannot make the plot lag.
* A live channel-mask change reaches the plot-source selector: the reply carries
  the new source list, and the selector is reconciled rather than rebuilt, so a
  disabled channel is unplotted and the rest keep their ticks.
* Shared-server lifetime. Windows register themselves in `windows.json` and only
  the last one standing may shut the server down, so closing a Configurator no
  longer takes the Monitor's server with it.

## 0.9.5.1 — 2 September 2026

* Configurator fixes.

## 0.9.5 — 1 September 2026

* **BIOPAC support.** MP36, MP150 and MP160 through BHAPI, using the buffered
  `receiveMPData` stream where the DLL provides it and falling back to
  per-sample polling where it does not — with the achieved rate measured, so the
  fallback's inability to hold 1 kHz is reported rather than silently halving
  the timeline.
* **Multiple concurrent clients.** Each connection is a session with its own
  command thread, so a Monitor and a Configurator can drive one rig at once and
  every reply is addressed to the client that asked.
* **The Configurator.** Config-free device enumeration, per-backend editable
  property schemas, and reporting of backends that failed to load along with the
  reason.
* **USRP naming.** User-assigned aliases keyed on the radio's serial, applied at
  discovery, with nothing written to device EEPROM.
* **Bounded queues throughout the streaming path**, each with a declared drop
  policy and a drop counter rather than an unbounded queue and a dead
  `except queue.Full`.

## 0.9.4.1 — 9 July 2026

* `.bvi` configuration files, asynchronous device operations, and the first of
  the USRP channel-mapping work.

## 0.9.4 — 3 July 2026

* USRP mapping groundwork and assorted USRP fixes.

---

The 0.9.3 series carried the split into three packages and the installer
builders. Those releases are not itemised here; commit-level history is in each
repository.
