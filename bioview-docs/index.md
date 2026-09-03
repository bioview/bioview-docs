# BioView

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg?style=for-the-badge&label=License&labelColor=%23084594&color=%234292c6)](https://www.gnu.org/licenses/gpl-3.0)
[![PyPI - Version](https://img.shields.io/pypi/v/bioview?style=for-the-badge&label=Version&labelColor=%23005a32&color=%2341ab5d)](https://pypi.org/project/bioview)
![PyPI - Downloads](https://img.shields.io/pypi/dm/bioview?style=for-the-badge&label=Downloads&labelColor=%234a1486&color=%236a51a3)

BioView is a cross-platform application for biomedical and human–computer
interface instrumentation control, including Ettus USRPs and BIOPAC devices.

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

## Start here

* [Installation](setup/installation.md)
* [Quick start](usage/quickstart.md)
* [Running a session](usage/session.md)
* [Architecture](architecture/index.md)
