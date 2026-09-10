# Installation

## Downloads

Pre-built binaries are built using GitHub CI/CD and available on the [Releases](https://github.com/bioview/bioview-installer/releases) page. Download the installer corresponding to your operating system. *If your OS warns about BioView being a potentially harmful file, allow it to run anyway. We are working on getting code-signing certificates being made available to us.*

!!! info "Note"
    Unless you are a developer, you should be using one of these pre-built installers since they come with all dependencies and drivers bundled in.

## Supported Platforms

BioView itself imposes no operating system requirement. Device drivers, on the other hand, may have OS restrictions, listed below. 

| Platform | USRP | BIOPAC | Microphone |
| --- | --- | --- | --- |
| Windows 11 (x64) | Yes | Yes | Yes |
| Linux (x64) | Yes | No | Yes |
| macOS (Apple Silicon) | Yes | No | Yes |

## Hardware drivers

### Ettus USRPs

BioView drives Ettus USRPs through UHD's Python bindings. Please follow the instructions below to setup UHD so that BioView can recognize it.

!!! warning "UHD Version"
    BioView is built against **UHD 4.10.0.0**, ensure that is the version you install. 

=== "Ubuntu / Debian"

    ```bash
    sudo apt update
    sudo apt install libuhd-dev uhd-host python3-uhd
    sudo uhd_images_downloader          # FPGA images
    ```

    `python3-uhd` installs into the system interpreter. A virtual environment
    created with `--system-site-packages` can see it; one created without cannot.

=== "Windows"

    The Ettus installer supplies `libuhd`, [the bindings and the FPGA images](https://files.ettus.com/binaries/). USB-attached USRPs also need a WinUSB driver binding, which can be acquired from the Ettus installer.

=== "macOS"

    No wheel exists. Build UHD from source with the Python API enabled:

    ```bash
    brew install cmake boost libusb ninja pkg-config
    cmake -DENABLE_PYTHON_API=ON ...
    ```

    `bioview-installer/scripts/build_macos.sh` does exactly this for the release
    bundle and is the reference for the flags that work.

### BIOPAC Hardware API

!!! info "Distribution"
    BIOPAC MP36, MP150 and MP160 units are driven through the BIOPAC Hardware API (BHAPI) which must be purchased from BIOPAC Systems Inc. and installed with its own installer. BioView will **never** ship those files.

`mpdev.dll` needs to be discoverable — in the working folder, on `PATH`, on the OS install drive, or at an explicit `mpdev_path` in the configuration.