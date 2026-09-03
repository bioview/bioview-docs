# Features

* **Server/client architecture.** A headless server owns the hardware; the GUIs
  connect to it over TCP. The server may be on another machine, and several
  windows can share one.
* **Real-time USRP control.** Ettus USRPs for high-throughput transceiving,
  including multi-radio virtual MIMO groups.
* **BIOPAC integration.** Synchronized physiological data alongside RF
  measurement.
    * *Requires a copy of the BIOPAC Hardware API purchased from BIOPAC Systems
      Inc.; Windows only.*
* **Live visualization.** A configurable plot grid fed by a decimated display
  path, so the render cost does not scale with the acquisition rate.
* **Synchronized acquisition.** Every device in a session starts together or not
  at all — a partially started rig records data that cannot be aligned.
* **Signal schemes.** CW, FMCW and pulsed Doppler, with an optional gated burst
  calibration overlay that composes with all of them.
* **DPIC.** Closed-form direct-path interference cancellation, run on demand or
  automatically at Start.
* **Experiment routines.** Fixed-duration timed modes with optional audio, video
  or text instructions.
* **Event annotation.** Timestamped marks stored in the recording itself.
* **Self-describing recordings.** One `.bvr` file per run holds the samples, the
  source descriptors, the device configuration at start, every live parameter
  change, and every annotation.
* **Device configuration utility.** The Configurator lists attached hardware and
  assigns device names without writing to device EEPROM.
* **Explained failures.** A shared catalogue of known issues means a cause such
  as a blocked driver is described the same way, with the same remedy, wherever
  it surfaces.
