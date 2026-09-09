# Upgrading

BioView is still pre-1.0. Configuration keys and the wire protocol can change
between releases, and there is no compatibility shim across versions.

## The one rule

**Every part of a session must be on the same version.** The three packages are
released together and always carry the same version number, and the client and
server negotiate no version at connect time — a mismatch shows up as a command
the far side does not recognise, or as a source list that does not line up.
That matters in two places:

* A remote server. Upgrade the machine running the server and the machines
  running the windows in the same pass.
* A source install. `bioview-common`, `bioview-server` and `bioview-client` all
  move together; do not mix a tag on one with `main` on another.

Recordings are not affected. The `.bvr` header carries its own source
descriptors and configuration snapshot, so a file written by an older release
stays readable.

## From an installer

Download the new installer and run it.

=== "Windows"

    Run the new `BioView-<version>-Setup.exe`. It upgrades in place, keeps the
    `.bvi` association, and does not need the old version uninstalled first.

=== "macOS"

    Replace the app in `/Applications` with the one from the new `.dmg`. Clear
    the quarantine attribute again if Gatekeeper objects — see
    [Download](../setup/downloads.md).

=== "Linux"

    ```bash
    flatpak install --user BioView.flatpak
    ```

    Flatpak replaces the existing installation with the newer bundle.

Close every BioView window first. The server is a separate process and outlives
the window that spawned it; an installer replacing files under a running server
is asking for a half-upgraded rig.

## From source

```bash
pip install --upgrade \
  "bioview-common @ git+https://github.com/bioview/bioview-common.git@v0.9.6" \
  "bioview-server @ git+https://github.com/bioview/bioview-server.git@v0.9.6" \
  "bioview-client @ git+https://github.com/bioview/bioview-client.git@v0.9.6"
```

For an editable checkout, pull all three repositories to the same tag. Only
dependency changes need the install re-run; source edits take effect
immediately.

!!! warning "Do not let NumPy 2 in"

    All three packages pin `numpy>=1.26,<2` because the UHD bindings are built
    against the NumPy 1.x ABI. An upgrade that pulls NumPy 2 into the
    environment leaves the USRP workers deadlocking on import with the GIL held,
    and the radio never answers `START_STREAMING`. There is no runtime guard.
    If you have upgraded other packages in the same environment, check
    `pip show numpy` before blaming the hardware.

## After upgrading

1. Open the Configurator and press **Discover**. Backends that failed to load
   are listed there with the reason — the fastest way to see that a driver did
   not survive the upgrade.
2. Load an existing configuration. Unknown keys are ignored rather than fatal,
   but a key that has been renamed will silently stop having an effect, so check
   the [configuration reference](../reference/configuration.md) against what
   changed in the [changelog](changelog.md).
3. Record ten seconds and confirm the plot scrolls in real time. A trace running
   fast or slow means the display rate and the acquisition rate disagree, which
   is the symptom a decimation change produces.

## Rolling back

Every past release keeps its installers on the [releases
page](https://github.com/bioview/bioview-installer/releases), and every tag
stays on the package repositories. Downgrading is the same operation as
upgrading, to an older version.
