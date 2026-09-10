# Reporting Bugs and Issues

BioView is organized into multiple repositories, and it is pertinent issues are reported to the correct place -

| Symptom | Repository |
| --- | --- |
| A device does not initialize, stream, or behaves wrongly | [`bioview-server`](https://github.com/bioview/bioview-server/issues) |
| A window, plot, control or recording problem | [`bioview-client`](https://github.com/bioview/bioview-client/issues) |
| Configuration parsing, the wire protocol, signal schemes, DPIC | [`bioview-common`](https://github.com/bioview/bioview-common/issues) |
| The installer, packaging, or a bundled dependency | [`bioview-installer`](https://github.com/bioview/bioview-installer/issues) |
| Anything on this site | [`bioview-docs`](https://github.com/bioview/bioview-docs/issues) |

## Reporting Bugs

Before opening a bug report, confirm it reproduces, and reduce it as far as you can. Check `server.log` in the BioView cache directory. A GUI-spawned server is detached and windowless, so that file is the only record of the device side, and it is usually where the actual cause is. See [Troubleshooting](../about/troubleshooting.md).

### Files to Include

* **Version.** Report the BioView version you currently are running.
* **Platform.** OS and version.
* **Device.** Model, connection (USB or Ethernet), and whether the Configurator lists it. If possible, include the version of the driver.
* **Steps to Reproduce**, including expected behavior and what happened instead.
* **A minimal configuration file**, trimmed to what still reproduces the fault.
* **Screenshots**, especially if it is a UI bug.

### Not sure whether it is a bug?

Several behaviors that look wrong are deliberate and documented: a partially started rig refuses to stream, a renamed USRP keeps announcing its old name until power-cycled, and dropped chunks are counted rather than logged per event. Search the docs first, and open a GitHub Discussion if the behavior is surprising — it may be a documentation gap, and worth reporting as one.

## Discussions
For an idea that is not yet a proposal — an approach you want a second opinion
on, or a question about whether something is already possible — open a GitHub
Discussion rather than an issue.
