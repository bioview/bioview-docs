# Contribution Guidelines

## Development setup

### Prerequisites

* **Git**, on `PATH`. On Windows use Git Bash or PowerShell.
* **Python 3.12 or 3.13** (the packages require `>=3.12, <3.14`).
* **UHD with Python bindings**, if you are working on the USRP backend. It is
  usually supplied by the Ettus installer rather than pip.
* **BHAPI (`mpdev.dll`)**, Windows only, if you are working on the BIOPAC
  backend.

None of the three is required. The `DUMMY` backend exercises streaming, saving,
calibration and DPIC with nothing attached, and the microphone backend needs
only whatever input the host already has.

### Setting up

Using git, checkout all 3 repositories: `bioview-common`, `bioview-server` and `bioview-client`. Install them in a virtual environment and use ```poetry``` to install each project.

In my personal experience, it is very useful to set up a proper ```launch.json``` file for running debug configurations in VS Code (or any of its derivative IDEs). Alternatively, you may run it using the CLI

```bash
python -m bioview_client.launch --role monitor
python -m bioview_client.launch --role configurator
```

Each window starts, or reuses, its own localhost server. There is no need to spawn a separate server. However, to run the server by hand anyway -

```bash
python -m bioview_server.server --local
```

A window will then find and reuse it rather than starting one of its own.

## Code Style

BioView is organized into three installable packages, which are organized as below -

```bash
bioview-common/bioview_common/
├── constants/         # ports, timeouts, queue depths
├── datatypes/         # DataSource, configuration classes, worker base classes
├── diagnostics/       # the known-issue catalogue
├── protocol/          # Command / Response / status enums
├── signal_schemes/    # CW, FMCW, pulsed Doppler, calibration, DPIC
└── utils/             # network framing, bounded-queue policies, filtering

bioview-server/bioview_server/
├── server.py          # sessions, command dispatch, data fan-out
├── datatypes/         # Backend base class (IPC contract)
├── common/            # display and save workers shared by all backends
└── device/            # usrp/, biopac/, dummy/ -- one package per backend

bioview-client/bioview_client/
├── launch.py          # roles, shared-server lifetime
├── handler.py         # front-end agnostic protocol client
├── workers.py         # off-thread scan / init / receive / save workers
├── monitor.py         # acquisition window
├── configurator.py    # device configuration window
└── components/        # Qt widgets
```

### Comments and documentation

Explanations belong in docs, not in the source. In code, keep comments to short technical remarks — **at most two lines** — that say something the code cannot: why a lock is taken without blocking, why a copy is unconditional, why a value is bounded. Design rationale, file formats, protocol details and physical models go under `bioview-docs/` and are linked from a one-line docstring.

### Style

* [PEP 8](https://peps.python.org/pep-0008/), enforced by `ruff` (line length
  89, `E,W,F,I,UP,B,SIM,C90`).
* Type annotations on public functions and classes.
* Formatting is `ruff format` (Black-compatible: double quotes, spaces, magic
  trailing commas respected).

`pre-commit` runs these before a commit. Please do not push past the hooks.

### Tests

Each package has its own suite:

```bash
cd bioview-common && pytest -q
cd bioview-server && pytest -q
cd bioview-client && pytest -q          # Qt tests run offscreen
pytest tests/hardware --hardware        # needs attached devices
```

Prefer tests that describe behaviour a user would notice — the existing names read that way on purpose (`test_a_plotted_source_that_disappears_is_unplotted`).
