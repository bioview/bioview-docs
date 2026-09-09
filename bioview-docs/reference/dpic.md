# DPIC: direct-path interference cancellation

In a continuous-wave bistatic measurement the transmitter's direct path into the
receiver dominates everything the target contributes. DPIC nulls it by radiating
an anti-phase copy from a second transmitter.

## The model

One Tx carries the measurement signal. Its direct path leaks into the Rx as a
complex term `d`. A second ("inject") Tx radiates a copy of the **same** IF tone
through a coupling `h`, scaled by a digital weight `w = a*exp(j*phi)`. The
residual at the receiver is

```
r(w) = d + h*w
```

`|r|` is sinusoidal in `phi` with a single minimum at `angle(-d/h)`, largely
independent of `a`, and V-shaped in `a` once `phi` is fixed. Both axes are
unimodal, which is what makes a coarse-to-fine bracket safe.

## The search

A direct port of the `Pig_2Ch_NCS_BIOPAC_BalanceSignal` LabVIEW VI. Four
sweeps, in the VI's order:

| # | Sweep | Range | Step | Points |
|---|-------|-------|------|--------|
| 1 | Coarse phase, at `coarse_probe_amplitude` (0.1) | 0..360 deg | 6 deg | 60 |
| 2 | Coarse amplitude, at the phase from 1 | 0..1 | 0.05 | 20 |
| 3 | Fine phase, around the phase from 1 | +/- 6 deg | 0.2 deg | 60 |
| 4 | Fine amplitude, around the amplitude from 2 | +/- 0.05 | 0.001 | 100 |

Each fine sweep spans exactly +/- one coarse step, so the coarse winner's
bracket is covered whichever side the true minimum falls on. Every sweep takes
the argmin of everything it measured -- it is not required to beat the seed.

Per-point dwell is the VI's too: **200 ms** on the coarse sweeps, **100 ms** on
the fine ones, and **500 ms** after each sweep's winner is applied. These sit
*on top of* the wait for fresh Rx chunks, not instead of it. Without them the
whole search runs in a couple of seconds, each point on screen for a frame or
two, and the characteristic rise-and-fall of the phase sweep never appears on
the plot.

241 measurements per pair, ~53 s of them dwell and chunk waits. `time_budget_s`
caps the wall clock and is split across the pairs *on one radio* (see
[Radios are balanced together](#radios-are-balanced-together)); the shipped
configuration default is 120 s (the balancer class itself defaults to 300 s if
no configuration says otherwise). A sweep that runs out of budget stops where it
is and keeps the best point found so far, and says so — the result is then
marked `truncated` and logged as a warning rather than passed off as a completed
search.

The VI's Rx-gain step ("tune Rx gain to a DC value of ~0.5") runs on either
side of the search, through the channel's `auto_gain_rx` callback: before,
because a null search is meaningless if the direct path sits in the noise
floor; and again after, because the null leaves the Rx far below its operating
point. `min_metric` is recorded before the second call, so the reported null
depth compares two measurements at one gain setting.

Two deliberate departures from the VI:

* The VI waits a fixed 100-200 ms per point for the Rx to catch up. Here every
  measurement instead blocks for chunks captured *after* the change (see
  "Stale reads" below), which is both faster and exact.
* The VI adjusts the measure Tx's analog gain in lockstep with the Rx gain.
  `_auto_gain_rx` moves the Rx gain only, by a proportional
  `20*log10(target/level)` correction rather than the VI's +/-1 dB ladder.

## Gain stage

Before the search and again once the path is nulled, the balancer walks the
level into range the way the VI does: **the measure Tx's analog gain and the
measure Rx's gain move together, 1 dB at a time**, with a 250 ms settle, until
the measured amplitude sits inside `amp_target ± amp_tolerance`. Raising Rx
gain alone would lift the noise floor with the signal; raising the measurement
Tx as well lifts the direct path that is about to be cancelled.

The VI's loop is unbounded. `max_gain_steps` bounds it here, and the ladder
also stops when both gains are already against the end of their range, so a
level that can never be reached (a dead path, a disconnected antenna) fails
instead of spinning.

A backend with no analog gain control -- the simulator -- leaves the gain
accessors unset and the stage is skipped.

## Watching it run

Every measurement reports the stage, the point index, the value applied, the
metric and both gains. The server attaches the newest report to
`GET_DEVICE_STATUS`, which the client is already polling, and the settings
panel writes the values into the matching spin boxes: IF phase and amplitude
for the inject Tx, Tx gain for the measure Tx, Rx gain for the measure Rx. The
Balance button carries the stage, e.g. *coarse phase 12/60*.

The panel does not echo these back to the server -- they are values the server
just set, and sending them back would have the UI fighting the search.

## Running it

A balance is asynchronous at every layer, because it drives hardware for a
minute or more and nothing else may wait on it. The backend child runs the
search on its own thread so its command loop keeps answering Stop and Shutdown;
the server acknowledges the command and publishes the outcome for polling; the
client dispatches it to a worker thread and polls once a second, leaving the
GUI thread and the control socket free. Only one balance runs at a time -- a
second is refused, not queued.

Stopping the stream (or disconnecting, or shutting down) **aborts** a running
balance. `DpicBalancer.should_abort` is consulted at every sweep point, so the
search unwinds immediately instead of measuring a radio that is no longer
transmitting, once per point, until its time budget expires.

Editing the channel map applies live, including the DPIC loop list: the
backend rebuilds its sources and pairs from the new map, so a loop added in the
settings panel is balanceable without re-initializing the device. The map is
frozen while streaming, since it decides how many rows the pipeline emits.

## Inject frequency

Balance retunes each pair's inject Tx onto its measure Tx's IF before the
search. The receive chain band-passes around the measure Tx's IF, so an
injection at any other frequency is rejected by that filter and cannot cancel
the direct path whatever weight the search picks. With `if_freq` of
`[100e3, 110e3]` and a pair injecting from Tx2 to measure Tx1, both run at
100 kHz for the balance and afterwards.

The change reaches the transmit workers, the processing worker's band-pass, and
the stored configuration together, so the settings panel shows the IF the radio
is actually driven at.

## Reporting

Every outcome carries a reason, including "it did not run". `_run_dpic_balance`
returns `{"ok", "message", "results"}`; the IPC layer replies `ERROR` when `ok`
is false and the server forwards the backend's own message. Returning `None` on
the early-outs -- no pairs configured, processing worker not started -- used to
be answered as `SUCCESS`, so the Monitor reported "DPIC balance complete" in a
couple of milliseconds while nothing had been driven.

Each sweep records a `DpicStage`: points planned, visited, and measured, plus
its winner and elapsed time. These are logged at debug level per pair, so a
short balance names the sweep that was cut short instead of just finishing
early. `truncated` is set when the time budget stopped a sweep mid-way; the
result is then the best point seen, not a completed search, and it is logged as
a warning.

The Rx-gain step runs *before* the deadline is set. It is a prerequisite of the
search rather than part of it, and on a slow measurement path it could
otherwise consume the whole budget and leave every sweep to break on its first
point.

## Retired channels

Adding a DPIC pair retires **both halves** of the inject channel: the Tx is
radiating the cancellation tone rather than a measurement signal, and the Rx
sharing that physical port has nothing to receive, so every `TxNRxM` row
against it is noise by construction. `resolve_channel_map` drops both, matching
the Rx on `(device, channel)` from the registry rather than on index, and the
Configurator's channel-map matrix resizes as pairs are added and removed.

`custom` layouts are left alone -- the pairs are written out one by one, so the
author has said exactly what they want.

## Same IF, always

**The inject Tx must be driven at the same IF as the measure Tx.** The receive
chain band-pass filters around the measure Tx's IF, so an inject Tx placed on a
different IF is rejected by that filter and can never cancel the direct path,
whatever weight the solver picks.

## Stale reads

The Rx path buffers deeply — roughly 20 packets plus whatever is queued — so a
value read immediately after a phase or amplitude change still describes the
*old* setting. Each measurement carries a sequence number, and the balancer
waits for a measurement taken strictly after its last change. Skipping that
biases every step of the search.

The metric is always the normalized mean baseband magnitude. It must not be
derived from the first stored component: under `save_iq` that is `mean(Re{·})`,
a signed quantity whose minimum is the most negative excursion rather than a
null.

## Radios are balanced together

Every cancellation loop in a device group used to be balanced one after
another, with the time budget divided between all of them: a four-radio group
waited four times as long *and* gave each search a quarter of the budget.

The loops on different radios are physically independent — separate Tx chains,
separate Rx chains — so `balance_all` groups them into one lane per radio,
runs the lanes concurrently, and runs each lane's loops in series. Two loops on
one radio still contend for its channels, so they stay in one lane. A lane
divides the budget between its own loops only, so a one-loop lane gets the
whole of it.

Which radio a loop belongs to is `DpicChannel.device`, taken from the inject
Tx: that is the channel the search actually drives.

Set `"parallel_devices": false` for a rig where the radios are *not* in fact
independent — a shared LO, a shared antenna — in which case the old serial
behaviour is what is wanted.

Results are re-ordered into the configuration's pair order on the way out,
whichever lane finished first, so the log, the report and the saved
`last_results` are unaffected by the timing.

## Configuration

```json
"channel_map": {
  "layout": "hybrid_mimo",
  "mimo": { "tx_global": [0, 1], "rx_global": [0, 1] },
  "dpic": [ { "inject_tx": 2, "measure_tx": 0, "measure_rx": 0 } ]
},
"dpic_balance": {
  "auto_on_start": false,
  "amp_target": 0.5,
  "settle_time_s": 0.02,
  "time_budget_s": 120.0,
  "coarse_phase_step_deg": 6.0,
  "coarse_amp_step": 0.05,
  "coarse_probe_amplitude": 0.1,
  "phase_step_deg": 0.2,
  "amp_step": 0.001,
  "parallel_devices": true
}
```

Indices are **global** Tx/Rx indices across the whole virtual device group.
`measure_rx` is a *receive* index and must be given explicitly unless it happens
to equal `measure_tx`, which is only true for a 1x1 layout.

A balance can be triggered from the Monitor at any time, or run automatically at
Start with `auto_on_start`. An auto balance runs *after* `START_STREAMING` has
been answered — it can take a minute or more, and must never hold up the reply.
