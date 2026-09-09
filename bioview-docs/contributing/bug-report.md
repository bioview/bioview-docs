# Reporting a bug

## Which repository

BioView is four repositories, and an issue is most useful in the one that owns
the code:

| Symptom | Repository |
| --- | --- |
| A device does not initialize, stream, or behaves wrongly | [`bioview-server`](https://github.com/bioview/bioview-server/issues) |
| A window, plot, control or recording problem | [`bioview-client`](https://github.com/bioview/bioview-client/issues) |
| Configuration parsing, the wire protocol, signal schemes, DPIC | [`bioview-common`](https://github.com/bioview/bioview-common/issues) |
| The installer, packaging, or a bundled dependency | [`bioview-installer`](https://github.com/bioview/bioview-installer/issues) |
| Anything on this site | [`bioview-docs`](https://github.com/bioview/bioview-docs/issues) |

If you are not sure, pick the closest and say so in the issue. Moving one is
cheap.

## Before opening one

Confirm it reproduces, and reduce it as far as you can. A `DUMMY` device group
runs the whole acquisition path with no hardware, so a fault that reproduces
against the dummy backend is one anybody can pick up — that is worth trying
before writing the report.

Check `server.log` in the BioView cache directory. A GUI-spawned server is
detached and windowless, so that file is the only record of the device side, and
it is usually where the actual cause is. See
[Troubleshooting](../about/troubleshooting.md).

## What to include

* **Version.** The one from the announcement bar on this site is the current
  release; report the one you are running. For a source install, the tag or
  commit of all three packages.
* **Platform.** OS and version; for USRP work, the UHD version and how it was
  installed.
* **The device.** Model, connection (USB or Ethernet), and whether the
  Configurator lists it.
* **Steps to reproduce**, and what you expected instead.
* **The relevant part of `server.log`**, and the Monitor's log panel at debug
  level — it traces every control exchange with the command, the reply and the
  elapsed time.
* **The configuration file**, trimmed to what still reproduces the fault.

Screenshots help for anything visual. A `.bvr` file helps for anything about
recorded data — the header carries the source list and the configuration
snapshot, so the file describes the session that produced it.

## Not sure whether it is a bug

Several behaviours that look wrong are deliberate and documented: a partially
started rig refuses to stream, a renamed USRP keeps announcing its old name
until power-cycled, and dropped chunks are counted rather than logged per event.
Search the docs first, and open a GitHub Discussion if the behaviour is
defensible but surprising — that is a documentation gap, and worth reporting as
one.
