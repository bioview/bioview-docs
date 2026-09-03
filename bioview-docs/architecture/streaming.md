# The streaming path

```
UHD recv  ->  rx queue  ->  process worker  -+->  save queue    ->  HDF5 (server, optional)
                                             |
                                             +->  display queue ->  display worker
                                                                        |
                                                            server data output queue
                                                                        |
                                                                  TCP data socket
                                                                        |
                                                     client DataStreamer -+-> DataSaver (.bvr)
                                                                          +-> PlotGrid
```

## Chunks

A chunk is one receive buffer — roughly 40 ms at 1 MSps with the default USRP
buffering. Every stage below counts in chunks.

The wire payload is
`[Total Length (4)][Header Length (4)][JSON header][raw float32 data]`. The
header carries the ordered list of source descriptors, one per row, so the
client can route each row to the right plot and the right save column without
relying on a global ordering agreed in advance. Samples go out as float32: no
plot resolves more, and it halves the streamed volume. The server-side save path
stays float64.

## Bounded queues and drop policy

Every streaming queue is bounded. On an unbounded queue a consumer that stalls
grows RAM without limit and pushes latency up with it, while every
`except queue.Full` handler in the codebase sits there as dead code. Bounding
them turns that failure mode into a bounded, countable drop.

| Queue | Depth | Policy |
| --- | --- | --- |
| Rx (raw buffers, per device) | 8 | Drop new. Blocking here stalls `recv` and turns a processing backlog into a UHD overflow. |
| Save | 64 | Wait briefly, then drop new. Recorded data matters, so a short disk stall is worth absorbing. |
| Display | 16 | Evict oldest. Only the newest chunk is useful; queuing behind stale data only adds latency. |
| Server data output | 32 | Evict oldest. |

`put_or_drop` and `put_drop_oldest` in `bioview_common.utils.queues` implement
the two policies. Both return `False` when an item was dropped, so callers count
drops rather than logging each one.

## Display decimation

Plot buffers are sized from each source's advertised display frequency
(`DataSource.disp_freq`), not from the acquisition rate. Incoming chunks are
stride-decimated to at most one screen's worth of points, which keeps the work
per frame bounded and the displayed rate roughly independent of the much higher
acquisition rate.

For BIOPAC every acquired sample reaches the display, so the display rate *is*
the sample rate; `disp_ds` only sets the chunk size. Reporting a fixed default
instead made the trace scroll at the wrong speed for any other sample rate.

## Pausing and restarting

Stop does not tear anything down. The server pauses each backend's workers, the
data connection stays open for the whole session, and the client's receiver
idles until data resumes. Start resumes the same workers.

Worker processes are only spawned on the first Start, which is why
`START_STREAMING` gets a 90-second budget: on Windows each transmit, receive and
process worker is a full process spawn.
