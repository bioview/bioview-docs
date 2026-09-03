# BIOPAC

BioView acquires from BIOPAC MP36, MP150 and MP160 units through the
`BIOPACBackend` server backend. Two constraints come with it:

* You must have a legally purchased copy of the **BIOPAC Hardware API (BHAPI)**.
  BioView will never ship those files — they are BIOPAC Inc.'s property.
* BHAPI is Windows-only, so BIOPAC support is Windows-only even though the rest
  of BioView is not.

## Prerequisites

* A compatible MP unit.
* Windows.
* `mpdev.dll`, discoverable in the working folder, on `$PATH`, on the OS install
  drive, or at an explicit `mpdev_path` in the configuration.

## Configuration

```json
"BIOPAC": {
  "type": "BIOPAC",
  "model": "MP36",
  "samp_rate": 1000,
  "channels": [1, 1, 0, 0],
  "hardware": {
    "BIOPAC_MP36": { "channels": [1, 1, 0, 0], "labels": ["ECG", "Resp"] }
  }
}
```

See [Configuration files](configuration.md) for every key. Sources are named
`<group>: <label>` — `BIOPAC: ECG` for the above.

Only *enabled* channels produce sources, and mpdev packs one value per enabled
channel into the sample buffer. The number of acquired columns is therefore the
number of set entries in the mask, not the length of the mask.

Every acquired sample reaches the display, so the **display rate is the sample
rate**; `disp_ds` only sets the acquisition chunk size, capped at 100 ms of data
so a low `disp_ds` at a high sample rate cannot make the plot lag by a second.

## Acquisition: daemon versus polling

BHAPI offers two ways to get samples, and which one is available decides whether
a high sample rate is achievable.

**`receiveMPData` — the buffered stream read (preferred).** A daemon thread
inside the DLL continuously downloads and caches samples from the MP unit;
`receiveMPData` blocks until the requested number of points exists and returns
them *in order*. The emitted timeline matches the hardware clock exactly and one
DLL call covers a whole chunk. `startMPAcqDaemon` must be called **before**
`startAcquisition`, and the two transfer styles must not be mixed within one
acquisition.

**`getMostRecentSample` — per-sample polling (fallback).** Used only when the
DLL does not export `receiveMPData` or the daemon will not start. It returns
whatever value the device happens to be holding, so it neither paces itself nor
guarantees distinct consecutive samples: the Python loop has to hit the sample
rate on its own, and at 1 kHz a loop plus a driver round-trip per sample often
cannot.

Falling short does not look like dropped data — it looks like a plot that
scrolls slowly, because the samples that do arrive are stretched across an
x-axis drawn from the *nominal* rate. BioView measures the achieved rate in the
polling path and warns when it drifts too far behind real time.

## Memory Integrity

The most common reason a BIOPAC unit is discovered but will not open is Windows
**Memory Integrity** (hypervisor-enforced code integrity) refusing the BIOPAC
driver. BioView checks for it when reporting the failure and says so.

Note the distinction the check draws: turning Memory Integrity off leaves it
*configured off but still running* until the machine restarts. The policy
registry key reports only the configured value and would suggest the problem was
solved while the driver is still being refused. The remedy in that state is a
reboot, not another settings change.

The query is bounded and cached — it only ever runs to explain a failure, so it
must not become one. It runs in the server process, not the backend subprocess,
where the underlying WMI call has been seen to hang indefinitely.

## BHAPI reference

| Function | Description |
| --- | --- |
| `connectMPDev` | Establish device connection (USB or UDP). |
| `setAcqChannels` | Activate specific analog input channels. |
| `setSampleRate` | Define the sampling interval. |
| `startMPAcqDaemon` | Start the buffering daemon behind `receiveMPData`. |
| `startAcquisition` | Begin live data collection. |
| `receiveMPData` | Buffered, in-order block read. |
| `getMostRecentSample` | Latest sample with timestamp (polling fallback). |
| `stopAcquisition`, `disconnectMPDev` | Cleanup. |

```c
int connectMPDev(int deviceCode, int connectionType, const char* portName);
```

* **Device codes** — `103` MP36/MP36R, `150` MP150, `160` MP160.
* **Connection types** — `10` USB (auto-detect), `20` Ethernet/UDP (MP150/160).
* **Port** — `"auto"`, a COM port such as `"COM3"`, or a UDP address such as
  `"169.254.111.111"`.

BHAPI signals failure by returning a non-zero result code. BioView wraps every
call and renders the code with its meaning — for example
`MPDRVERR (code 2): the MP device driver did not respond` — so a failure
explains itself instead of surfacing a bare number.

Make sure the DLL's architecture matches the Python interpreter (32- vs 64-bit),
and that firewall and driver permissions allow device access.
