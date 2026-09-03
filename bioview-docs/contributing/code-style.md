# Code style and architecture

## Repository layout

BioView is three installable packages plus an installer and these docs.

```
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

`bioview-common` never imports the other two. `bioview-client` never imports
`bioview-server` at module load time.

## Where things belong

* **Generic datatypes** — `bioview-common/datatypes/`. Anything both the server
  and a client need to agree on.
* **Device-specific logic** — its own package under `bioview_server/device/`.
* **No business logic in Qt widgets.** Widgets emit signals; the window and the
  handler decide what to do.
* **No device access in the client.** Everything goes through the protocol.

## Comments and documentation

Explanations belong in these docs, not in the source. In code, keep comments to
short technical remarks — **at most two lines** — that say something the code
cannot: why a lock is taken without blocking, why a copy is unconditional, why a
value is bounded. Design rationale, file formats, protocol details and physical
models go under `bioview-docs/` and are linked from a one-line docstring.

Docstrings: one line for anything whose name and signature already say what it
does; a short paragraph only when there is a real contract to state.

## Style

* [PEP 8](https://peps.python.org/pep-0008/), enforced by `ruff` (line length
  89, `E,W,F,I,UP,B,SIM,C90`).
* Type annotations on public functions and classes.
* Formatting is `ruff format` (Black-compatible: double quotes, spaces, magic
  trailing commas respected).

`pre-commit` runs these before a commit. Please do not push past the hooks.

## Tests

Each package has its own suite:

```bash
cd bioview-common && pytest -q
cd bioview-server && pytest -q
cd bioview-client && pytest -q          # Qt tests run offscreen
pytest tests/hardware --hardware        # needs attached devices
```

The client suite sets `QT_QPA_PLATFORM=offscreen`, so it runs headless.
Hardware tests are skipped unless `--hardware` is passed.

Prefer tests that describe behaviour a user would notice — the existing names
read that way on purpose (`test_a_plotted_source_that_disappears_is_unplotted`).
