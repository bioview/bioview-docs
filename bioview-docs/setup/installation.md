# Installation

Most people should take a prebuilt installer: it carries Python, Qt and UHD
inside it, so there is nothing to configure before the app opens. See
[Download](downloads.md).

This page covers the other case — running BioView from source — and the hardware
drivers that no installer can supply for you.

## Supported platforms

BioView itself imposes no operating system requirement. Its device drivers do,
and that is what the list below really describes.

| Platform | USRP | BIOPAC | Microphone |
| --- | --- | --- | --- |
| Windows 10/11 (x64) | Yes | Yes | Yes |
| Ubuntu LTS, Debian stable and derivatives | Yes | No | Yes |
| macOS (Apple Silicon and Intel) | Yes | No | Yes |

BIOPAC support is Windows-only because BHAPI is. Fedora, RHEL and their
derivatives are untested: they have no prebuilt UHD Python bindings, so UHD has
to be built from source there. Non-LTS Ubuntu releases frequently lack a
matching `python3-uhd`.

A `DUMMY` device group needs none of the above. It runs the full acquisition
pipeline — streaming, recording, calibration, DPIC — against a synthetic
channel, which is how the RF path is developed and tested with no radio
attached.

## Hardware drivers

### UHD (for USRPs)

BioView drives Ettus USRPs through UHD's **Python bindings**: `import uhd` must
succeed in the interpreter the *server* runs in. If it does not, the USRP
backend is recorded as unavailable with the reason and reported in the
Configurator — it does not silently disappear from the device list.

The bindings are not on PyPI for every platform, which is why the delivery
differs per system.

=== "Ubuntu / Debian"

    ```bash
    sudo apt update
    sudo apt install libuhd-dev uhd-host python3-uhd
    sudo uhd_images_downloader          # FPGA images
    ```

    `python3-uhd` installs into the system interpreter. A virtual environment
    created with `--system-site-packages` can see it; one created without cannot.

=== "Windows"

    The Ettus installer supplies `libuhd`, the bindings and the FPGA images:
    [files.ettus.com/binaries](https://files.ettus.com/binaries/). Alternatively
    `pip install uhd` inside your environment — the official wheel carries the
    DLLs, the bindings and the images together, and it is what the Windows
    installer bundle is built from.

    USB-attached USRPs also need a WinUSB driver binding, from the Ettus
    installer or [Zadig](https://zadig.akeo.ie/).

=== "macOS"

    No wheel exists. Build UHD from source with the Python API enabled:

    ```bash
    brew install cmake boost libusb ninja pkg-config
    cmake -DENABLE_PYTHON_API=ON ...
    ```

    `bioview-installer/scripts/build_macos.sh` does exactly this for the release
    bundle and is the reference for the flags that work.

Whichever route, the UHD version, the bindings and the FPGA images must all
match. Releases pin **UHD 4.10.0.0**.

!!! warning "UHD requires NumPy 1.x"

    The UHD Python bindings are built against the NumPy 1.x ABI and declare
    `numpy>=1.11,<2.0` themselves. Under NumPy 2 the transmit and receive
    workers deadlock on pybind11's lazy NumPy C-API import, with the GIL held,
    and the radio never answers `START_STREAMING`. All three BioView packages
    therefore pin `numpy>=1.26,<2`. There is no runtime guard for this — the pin
    is the fix. Do not install NumPy 2 into a BioView environment.

### BHAPI (for BIOPAC)

BIOPAC MP36, MP150 and MP160 units are driven through the **BIOPAC Hardware API**,
which must be purchased from BIOPAC Systems Inc. and installed with its own
installer. BioView will never ship those files. `mpdev.dll` needs to be
discoverable — in the working folder, on `PATH`, on the OS install drive, or at
an explicit `mpdev_path` in the configuration.

On Windows, Memory Integrity (Core Isolation) blocks the BIOPAC driver. If you
have just turned it off, the setting reads as off while the driver is still
loaded until the machine restarts; BioView tells the two states apart and says
which one you are in. See [BIOPAC](../reference/biopac.md).

### Audio input

Nothing to install. The microphone backend goes through PortAudio via
`sounddevice`, a regular dependency of `bioview-server` rather than an extra —
the backend registry probes it at import, and a missing optional package would
have looked like broken hardware. See [Microphone](../reference/microphone.md).

The microphone backend is newer than the 0.9.6 installers; track `main` to use
it today, or wait for the next release.

## Running from source

### Python

BioView requires **Python 3.12 or 3.13** (`>=3.12, <3.14`). Use a virtual
environment; if you are relying on a distribution-packaged `python3-uhd`, create
it with `--system-site-packages` so the bindings remain importable.

```bash
python3.12 -m venv bioview-env
source bioview-env/bin/activate          # Linux/macOS
bioview-env\Scripts\activate             # Windows
```

### Packages

BioView is three packages, and install order matters: `bioview-common` is
imported by both of the others without being declared as their dependency, so it
goes first.

```bash
pip install "bioview-common @ git+https://github.com/bioview/bioview-common.git@v0.9.6"
pip install "bioview-server @ git+https://github.com/bioview/bioview-server.git@v0.9.6"
pip install "bioview-client @ git+https://github.com/bioview/bioview-client.git@v0.9.6"
```

Use `@main` in place of the tag to track development — the three packages are
released together and always carry the same version, so mixing a tag with `main`
across them is not a supported combination.

BioView is not published on PyPI. The release artifacts are the platform
installers; the packages themselves are consumed from git.

If you are going to edit the code, use the development setup instead — it does
the same three installs in editable mode and writes matching VS Code launch
configurations. See [Development setup](../contributing/build-instructions.md).

### First run

```bash
bioview                  # Monitor (starts or reuses a localhost server)
bioview-configurator     # Configurator
```

Both entry points route through the launcher, so there is no separate step to
start a server. Continue with [Quick start](../usage/quickstart.md).
