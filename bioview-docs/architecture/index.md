# Architecture

BioView is split into three installable packages and runs as several
cooperating OS processes.

## Packages

| Package | Contents |
| --- | --- |
| `bioview-common` | Wire protocol, configuration datatypes, signal schemes, bounded-queue helpers, the known-issue catalogue. Imported by both of the others. |
| `bioview-server` | The headless server, the per-device backends (USRP, BIOPAC, dummy) and the acquisition pipeline. |
| `bioview-client` | The Monitor and Configurator GUIs, the client-side protocol handler, and the launcher that starts everything. |

`bioview-common` never imports the other two. The client never imports the
server package at module load time, so a client-only install stays importable
with no device drivers present.

## Processes

```
  Monitor (Qt)          Configurator (Qt)
        \                     /
         \  TCP 8998 control /
          \ TCP 8999 data   /
           \               /
            BioView server  (one per machine)
                   |
        multiprocessing.Queue per device
                   |
   +---------------+---------------+
   |               |               |
 USRP backend   BIOPAC backend   Dummy backend      (one process each)
   |
 Tx / Rx / process workers                          (threads or processes)
```

Every box above is a separate OS process. That is deliberate:

* **UHD and PyQt never share an interpreter.** UHD's Python bindings hold the
  GIL for long stretches during `recv`; running them in the GUI process makes
  the UI stutter and, worse, makes a UHD overflow look like a frozen window.
* **A device driver that crashes takes down one backend, not the session.**
* **The server outlives any one window,** so a Monitor and a Configurator can
  drive the same hardware at the same time.

## Who owns what

* **Hardware, device configuration and streaming state are server-wide.** There
  is one set of attached devices, so every connected client sees the same
  device status and the same stream.
* **Recordings are written by the client.** The server streams everything it
  acquires; the client tees each chunk to disk and to the plots. Saving on the
  client keeps the write off the acquisition path and puts the file on the
  machine the operator is sitting at.
* **The client decides what to plot.** The server forwards every source, and
  the Monitor routes rows to plots using the per-chunk source list.

## Further reading

* [Launching and process lifetime](launching.md)
* [The server](server.md)
* [The client](client.md)
* [The streaming path](streaming.md)
* [Wire protocol](../reference/protocol.md)
