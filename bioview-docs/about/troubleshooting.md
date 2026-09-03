# Troubleshooting

## Where to look first

**`server.log` in the BioView cache directory.** A server spawned by a GUI
window is detached and windowless, so everything it knows about a device that
would not open goes there. It is the only record of the device side.

The Monitor's log panel at debug level traces every control exchange — what was
asked, what came back, and how long it took.

## The window says "no server"

The launcher starts a localhost server for every window, so this normally means
the server died or never bound its ports.

* Another BioView server may already hold 8998/8999. That is fine — the window
  should reuse it. If it does not, something else is on the port; BioView
  distinguishes them with a discovery handshake rather than a bare TCP connect.
* Check `server.log` for a bind error.
* The server binds exclusively, so a second server on the same ports fails to
  start by design rather than running alongside and splitting clients.

## A device is listed but will not initialize

The log names the device and the cause. Recognised causes carry a remedy.

**BIOPAC, Windows Memory Integrity.** The most common one. Memory Integrity
refuses the BIOPAC driver. Turning it off leaves it *configured off but still
running* until the machine restarts, so if you have just flipped the switch, the
remedy is a reboot, not another settings change — BioView distinguishes the two
states and says which one you are in.

**USRP not detected.** Verify the cabling and that `import uhd` works in the
server's interpreter. If the USRP backend failed to import entirely, the
Configurator reports it with the reason.

**A renamed USRP is not found.** BioView's alias takes effect at discovery, so
it should work immediately. The radio itself keeps announcing its old name until
power cycled.

## Streaming fails to start

The error names each device that failed and why:

```
Failed to start streaming -- USRP: USRP did not answer START_STREAMING within 90s
```

If any device fails, the ones that already started are stopped again: a
partially started session records data that cannot be aligned across devices.

A start genuinely can take tens of seconds the first time, because each device's
transmit, receive and process workers are OS process spawns on Windows.

## The plot scrolls too slowly

The x-axis is drawn from the *nominal* sample rate, so a plot that scrolls at
half speed means the acquisition path is only achieving half the requested rate.

For BIOPAC this is almost always the per-sample polling fallback: if the DLL
cannot start the acquisition daemon behind `receiveMPData`, a Python loop plus a
driver round-trip per sample cannot keep up at 1 kHz. BioView measures the
achieved rate in that path and warns when it drifts. See
[BIOPAC](../reference/biopac.md).

## Data drops

Every streaming queue is bounded, and each has a declared policy — the display
queue evicts the oldest chunk to keep latency down, the save queue waits briefly
and only then drops. Drops are counted, not logged per event.

Sustained drops on the save path mean the disk is not keeping up; use an SSD.
Sustained drops on the Rx path mean demodulation is not keeping up, which will
also show as UHD overflows.

## Spikes in the data

Filtering edge effects from a small receive buffer. Increase it.

## High CPU

Reduce the plot grid size and the display window, or lower the display rate. The
plot path decimates each chunk to at most one screen's worth of points, so the
grid size and refresh rate dominate.

## Closing one window killed the server

It should not: the server only shuts down when the last window closes. Every
window registers itself in `windows.json` in the cache directory, and a server is
only terminated by the window that spawned it, when no other window is
registered and the server reports no clients.

If you see this, check that both windows were started through the launcher —
`bioview`, `bioview-configurator`, or `python -m bioview_client.launch --role
...`. All of the supported entry points route through it.
