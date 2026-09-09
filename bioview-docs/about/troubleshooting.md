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
Failed to start streaming -- USRP: USRP did not answer START_STREAMING within 5s
```

If any device fails, the ones that already started are stopped again: a
partially started session records data that cannot be aligned across devices.

Start is not the slow operation — the workers are threads created when the
device was initialized, and resuming them is measured in tenths of a second. The
five-second budget is deliberately tight for that reason: a device that has not
answered is wedged, and waiting longer only delays the error while holding up
every device queued behind it. If something is slow, it is almost always
Initialize (150 s budget), where the radio is actually being opened.

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

## The recording is at the wrong rate

The recorded rate is `samp_rate / (save_ds * disp_ds)`, not `samp_rate /
save_ds`: the client writes the file from the display stream, so both divisors
apply. A recording that came out ten times thinner than expected is almost
always `disp_ds` sitting at its default of 10. Set it explicitly in any
configuration meant to record.

The file itself is not ambiguous about this — the `disp_freq` in the `.bvr`
header is the rate that actually reached disk.

## The microphone recorded the wrong thing

Windows enumerates loopback inputs ("Stereo Mix", "What U Hear") alongside real
microphones. During a routine a loopback records the instruction audio being
played back rather than the participant, and nothing in the file says so. With
no host default input, BioView skips loopbacks by name and warns which input it
took instead; a configuration that names one explicitly is honoured, because the
backend cannot know that was not deliberate. Name the input you want in
`device`. See [Microphone](../reference/microphone.md).

A requested sample rate is also only a request: MME commonly offers a device
only at its native rate. The negotiated rate is what the sources advertise and
what the `.bvr` header records, so check the header rather than assuming the
configured value.

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
