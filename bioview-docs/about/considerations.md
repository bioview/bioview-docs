# Performance considerations

* Real-time acquisition needs real resources. A USRP at 1 MSps with several
  channels will use a core or two for demodulation alone.
* **Use an SSD for recordings.** The save queue is the deepest of the bounded
  queues precisely because disk writes are bursty, but it is still finite.
* Memory use scales with the bounded queue depths and the plot grid, both of
  which are fixed and small. Nothing on the streaming path grows without limit.
* Only plotted sources are drawn, and each chunk is decimated to at most one
  screen's worth of points, so the display cost is bounded by the grid size and
  the monitor refresh rate rather than by the acquisition rate.
* **Keep the receive buffer reasonably large.** A small one produces spikes in
  the data from filtering edge effects.
* **B210s behave poorly with the default frame size.** BioView's default receive
  frame size is 1024 for this reason.
* Data goes over the wire as float32 — no plot resolves more, and it halves the
  streamed volume. The server-side save path stays float64.
* The first Start of a session is slow: each device's transmit, receive and
  process workers are OS process spawns, and on Windows that is seconds per
  worker. Later Start/Stop cycles just pause and resume them.
