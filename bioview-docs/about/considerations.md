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
  streamed volume.
* **Choose `save_ds` and `disp_ds` together.** Recordings are written by the
  client from the display stream, so the recorded rate is
  `samp_rate / (save_ds * disp_ds)`. Lowering `disp_ds` to spend less on
  plotting also raises the volume going to disk and over the wire, and raising
  it thins the recording. See [the streaming
  path](../architecture/streaming.md).
* **Opening a device is the slow step, not starting the stream.** USB
  enumeration, FPGA and CODEC bring-up and clock locking put Initialize in the
  tens of seconds for a USRP; that is why it gets a 150-second budget. Start is
  near-instant by comparison — measured at about 0.4 s for a two-channel USRP
  and 0.02 s for BIOPAC — because the workers are threads that already exist and
  are merely resumed. A device that has not answered Start within a few seconds
  is wedged rather than slow, which is what the 5-second budget encodes.
* **A DPIC balance is a minute of wall clock, by design.** 241 measurements per
  pair, and the per-point dwells are the reference VI's — without them the sweep
  finishes in seconds and the characteristic phase curve never appears on the
  plot. It runs asynchronously, so the rest of the UI stays live; budget for it
  rather than trying to shorten it. See [DPIC](../reference/dpic.md).
