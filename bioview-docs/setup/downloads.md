# Download

Every release is built by CI on a native runner for each platform and attached
to a GitHub Release. The current release is **0.9.6**.

<div class="grid cards" markdown>

- :material-download: **[All installers for the latest
  release](https://github.com/bioview/bioview-installer/releases/latest)**

- :material-history: **[Every past
  release](https://github.com/bioview/bioview-installer/releases)**

</div>

## What to download

| Platform | Asset | Notes |
| --- | --- | --- |
| Windows 10/11, x64 | `BioView-0.9.6-Setup.exe` | Inno Setup installer. Includes the UHD driver and FPGA images. |
| macOS, Apple Silicon | `BioView-0.9.6-arm64.dmg` | Built on macOS 14. |
| macOS, Intel | `BioView-0.9.6-x86_64.dmg` | Built on macOS 13. |
| Linux, x86_64 | `BioView.flatpak` | Single-file bundle; needs a Flatpak runtime. |

Each bundle is self-contained: `bioview-common`, `bioview-server`,
`bioview-client`, Python, Qt and UHD 4.10.0.0 are all inside it. There is no
separate Python installation to manage and nothing to `pip install`. BIOPAC is
the one exception — its driver is licensed hardware software and can never be
redistributed, so it is installed separately on Windows.

The bundles are large for that reason: Qt plus UHD plus the FPGA images that
ship with it.

## One binary, three roles

All three installers put down the same executable. It dispatches on `--role`:
the default opens the Monitor and starts a hidden localhost server for it in a
separate process; `--role configurator` opens the Configurator; `--role server`
is what the launcher spawns for itself. The server and the GUI are kept in
separate interpreters deliberately — UHD's bindings hold the GIL for long
stretches inside `recv`, and sharing a process with Qt turns that into a frozen
window. See [Launching](../architecture/launching.md).

---

## Windows

1. Run `BioView-0.9.6-Setup.exe`.
2. SmartScreen will warn that the publisher is unrecognised — the installers are
   not code-signed. Choose **More info › Run anyway** if you trust the source.
3. The installer needs no administrator rights (`PrivilegesRequired=lowest`) and
   installs per-user by default.

It creates Start Menu entries for **BioView Monitor** and **BioView
Configurator**, an optional desktop icon, and registers the `.bvi` configuration
extension so double-clicking a configuration opens the Monitor with it loaded.

### Afterwards

* **USRP over USB** may need a one-time WinUSB driver. The Ettus UHD installer
  or [Zadig](https://zadig.akeo.ie/) both provide it. The bundled UHD supplies
  the library and FPGA images, not the USB driver binding.
* **BIOPAC** needs BHAPI (`mpdev.dll`) purchased from BIOPAC Systems Inc. and
  installed separately. See [BIOPAC](../reference/biopac.md).

---

## macOS

1. Open the `.dmg` for your architecture and drag **BioView** to Applications.
2. The app is ad-hoc signed rather than notarized, so Gatekeeper refuses it on
   first launch. Right-click the app and choose **Open**, then confirm; or clear
   the quarantine attribute:

   ```bash
   xattr -dr com.apple.quarantine /Applications/BioView.app
   ```

Do this once — subsequent launches are normal.

UHD is compiled from source into the bundle, so USRPs work with no separate
driver install. BIOPAC does not: BHAPI is Windows-only, and so BIOPAC support is
Windows-only even though nothing else in BioView is.

---

## Linux

The Flatpak bundle is not on Flathub; install the downloaded file directly.

```bash
flatpak install --user BioView.flatpak
flatpak run org.bioview.BioView
```

If this is the first Flatpak on the machine, add Flathub and the runtime first:

```bash
sudo apt install flatpak            # or your distribution's equivalent
flatpak remote-add --if-not-exists --user flathub \
    https://flathub.org/repo/flathub.flatpakrepo
flatpak install --user flathub org.freedesktop.Platform//24.08
```

The bundle declares `--device=all` so it can reach USB USRPs, and
`--filesystem=home` so recordings can be written to a folder you choose. A USRP
that is visible to the host but not to the sandbox is almost always a udev
permissions problem on the host rather than a Flatpak one — check that your user
can open the device outside the sandbox first.

Ubuntu LTS and Debian stable are the tested targets. Fedora and RHEL are not
supported: they have no prebuilt UHD Python bindings, and the Flatpak builds UHD
from source anyway, so the constraint is really about how much of the pipeline
has been exercised there rather than a hard incompatibility.

---

## Verifying the install

Open BioView. The status bar should report **BioView Server Connected** within a
second or two; that means the launcher found or started a server and the client
authenticated to it. Then open the Configurator (Start Menu entry, `--role
configurator`, or `bioview-configurator` from a source install) and press
**Discover**: it lists what is attached and, separately, any backend that failed
to load along with the reason. A USRP with no driver shows up there rather than
just being absent.

No hardware to hand? A `DUMMY` device group exercises the whole acquisition
path — streaming, saving, calibration and DPIC — with nothing plugged in. See
[Quick start](../usage/quickstart.md).

## Installing from source

Running from source is the right choice if you are modifying BioView, if you
need a platform no installer covers, or if you want to use a system UHD instead
of the bundled one. See [Installation](installation.md) for driver and Python
prerequisites, and [Development setup](../contributing/build-instructions.md)
for the editable-install workflow.

## Building the installers yourself

`bioview-installer` drives all three builders from a single `build.toml`. The
same builders run in CI on tag push. See its README and
[Development setup](../contributing/build-instructions.md).
