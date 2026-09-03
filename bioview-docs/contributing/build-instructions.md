# Development setup

## Prerequisites

* **Git**, on `PATH`. On Windows use Git Bash or PowerShell.
* **Python 3.12** (the packages require `>=3.12, <3.14`).
* **UHD with Python bindings**, if you are working on the USRP backend. It is
  usually supplied by the Ettus installer rather than pip.
* **BHAPI (`mpdev.dll`)**, Windows only, if you are working on the BIOPAC
  backend.

## Setting up

From a checkout containing `bioview-common`, `bioview-server` and
`bioview-client` side by side:

```bash
python setup_dev.py
```

That creates `.venv`, installs the three packages into it in editable mode plus
the test tooling, and writes `.vscode/launch.json` and `.vscode/settings.json`
pointing at that interpreter. It is stdlib-only, so it runs on a bare
interpreter before anything is installed.

Install order matters and the script handles it: `bioview-common` is imported by
both of the others but is not declared as a dependency of either, so it goes
first.

Options:

```bash
python setup_dev.py --recreate      # rebuild the venv from scratch
python setup_dev.py --run-tests     # verify by running the suites
python setup_dev.py --with-biopac   # (re)assert the Windows BIOPAC dep (wmi)
python setup_dev.py --with-uhd      # try to pip-install the UHD bindings
python setup_dev.py --no-vscode     # skip writing .vscode/
```

Editable installs mean source edits take effect immediately; only dependency
changes need a re-run.

## Running it

```bash
python -m bioview_client.launch --role monitor
python -m bioview_client.launch --role configurator
```

Each window starts, or reuses, its own localhost server. There is no separate
"start the server first" step.

To debug the server, use the VS Code configurations: both have
`"subProcess": true`, so the debugger attaches to the server the window spawns
and breakpoints in server code are hit.

To run the server by hand anyway:

```bash
python -m bioview_server.server --local
```

A window will then find and reuse it rather than starting one of its own.

## Testing

```bash
cd bioview-common && pytest -q
cd bioview-server && pytest -q
cd bioview-client && pytest -q
```

The server suite includes end-to-end tests against the dummy RF backend, so it
exercises the full acquisition path with no hardware attached. Tests needing
real devices live in `bioview-server/tests/hardware` and only run with
`--hardware`.

## Packaging

`bioview-installer` builds the frozen single-binary bundles. It freezes
`scripts/pyinstaller_entry.py`, which hands off to `bioview_client.launch:main`;
the same binary re-execs itself with `--role server` for the child server.

See `bioview-installer/` and `release.sh`.
