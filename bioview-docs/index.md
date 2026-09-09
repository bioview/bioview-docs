# BioView

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg?style=for-the-badge&label=License&labelColor=%23084594&color=%234292c6)](https://www.gnu.org/licenses/gpl-3.0)
[![Latest release](https://img.shields.io/badge/Release-0.9.6-green?style=for-the-badge&labelColor=%23005a32&color=%2341ab5d)](setup/downloads.md)
[![Platforms](https://img.shields.io/badge/Windows%20%7C%20macOS%20%7C%20Linux-informational?style=for-the-badge&label=Platforms&labelColor=%234a1486&color=%236a51a3)](setup/downloads.md)

BioView is a cross-platform acquisition application for biomedical and
human–computer interface instrumentation. It drives Ettus USRPs, BIOPAC MP units
and host audio inputs from one process, starts them together, and writes what
they produce into a single self-describing recording.

```bash
bioview
```

That opens the Monitor and, behind it, the server that talks to your hardware.

## How it fits together

BioView runs as a **headless server** that owns the devices, and one or more
**client windows** that connect to it over TCP. The server can be on another
machine; several windows can share one server; and a Monitor and a Configurator
can drive the same rig at the same time.

* **[Monitor](reference/ui.md)** — the acquisition window: live plots, device
  controls, recording, routines and annotations.
* **[Configurator](usage/configurator.md)** — lists attached hardware and edits
  per-device properties, with no configuration file needed.
* **[Server](architecture/server.md)** — device backends, the acquisition
  pipeline, and the wire protocol.

Every device group runs in its own OS process, so UHD's bindings never share an
interpreter with Qt and a driver that falls over takes one backend down rather
than the session. [Architecture](architecture/index.md) explains why that shape
was chosen.

## What it acquires

| Device | Notes |
| --- | --- |
| [Ettus USRP](reference/usrp.md) | B200/B210 and compatible, through UHD. Several radios in one group form a virtual MIMO device. CW, FMCW and pulsed-Doppler schemes; direct-path interference cancellation. |
| [BIOPAC](reference/biopac.md) | MP36, MP150, MP160 through BHAPI. Windows only, and you supply the licensed driver. |
| [Microphone](reference/microphone.md) | Any host input through PortAudio, recorded in the same file and on the same clock as everything else. |
| Dummy | A synthetic device that exercises the whole pipeline, including calibration and DPIC, with no hardware attached. |

## Start here

* [Download an installer](setup/downloads.md)
* [Installing from source, and the hardware drivers](setup/installation.md)
* [Quick start](usage/quickstart.md)
* [Running a session](usage/session.md)
* [Architecture](architecture/index.md)
