# The client

The client package is three things: a front-end agnostic protocol handler, and
two Qt windows built on top of it.

## `bioview_client.handler.Client`

A `QThread` that owns the control and data sockets and exposes the protocol as
Qt signals. It is deliberately GUI-agnostic — anything that can connect to a
`pyqtSignal` can drive BioView.

Responsibilities:

* **Server discovery.** A LAN scan probes every host on the local /24 plus
  loopback, on a bounded thread pool so a 254-host sweep cannot starve the app.
* **Connection and authentication**, dispatched off the UI thread.
* **Device discovery, initialization and disconnection.**
* **Streaming control**, and the client-side recording that goes with it.
* **Live parameter updates.**

Two details worth knowing:

* A connection watchdog peeks the control socket every couple of seconds. A
  server that goes away is otherwise only noticed on the next command, and
  nothing then clears the connected state — the window sits there believing it
  is connected while every action fails.
* The address the client actually reached the server on is kept, not the NIC
  address the server advertises in its discovery payload. That address may be
  unroutable from here (VPN, several NICs) or be refused outright by a `--local`
  server.

### Command tracing

Every control exchange is traced at debug level with what was asked, what came
back and how long it took:

```
-> START_STREAMING (Experiment={4 key(s): ...}, USRP={29 key(s): ...})
<- START_STREAMING: SUCCESS -- Data streaming started successfully in 812 ms
```

`GET_DEVICE_STATUS` is excluded: it repeats every couple of seconds for the
whole length of a device operation and would bury everything else. The poll loop
reports its own progress instead.

## Monitor

`bioview_client.monitor.BioViewMonitor` is the acquisition window: the plot
grid, the settings panel, the log, the status bar and the routine controls.

* Data arrives as `(data, sources)` over a **queued** connection, so bursts are
  marshalled to the UI thread one event at a time and never re-enter the
  receiving path.
* Plot sources are reconciled, not just repopulated. When the server's advertised
  source list changes — enabling a BIOPAC channel, say — a source that has gone
  away is unplotted and its grid cell released, while one that is still plotted
  keeps its tick.
* Sources are named `Device: Source` (for example `BIOPAC: Ch1`,
  `USRP: Tx1Rx1`) everywhere they appear. Channel labels are only unique within
  a device, so a bare label is ambiguous as soon as two devices stream at once.
  The plot title carries exactly the name the selector shows.

## Configurator

`bioview_client.configurator` lists every device attached to the server and lets
per-device properties be edited — the role NI's USRP Configuration Utility plays
for LabVIEW. See [Using the Configurator](../usage/configurator.md).

## Recording

Saving happens on the client. `DataSaver` runs on its own thread and appends
full-rate chunks to a `.bvr` file, so disk I/O never blocks the data receiver.
The format is documented in [the .bvr format](../reference/bvr-format.md).

The data socket and its receiver stay up for the whole session. Stopping a
stream only pauses the server's backends; tearing the socket down on stop is
what used to break restart.
