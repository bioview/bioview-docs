# DPIC: direct-path interference cancellation

In a continuous-wave bistatic measurement the transmitter's direct path into the
receiver dominates everything the target contributes. DPIC nulls it by radiating
an anti-phase copy from a second transmitter.

## The model

One Tx carries the measurement signal. Its direct path leaks into the Rx as a
complex term `d`. A second ("inject") Tx radiates a copy of the **same** IF tone
through a coupling `h`, scaled by a digital weight `w = a·exp(jφ)` and by the
inject Tx's analog gain `g`. The residual at the receiver is

```
r(w) = d + h(g)·w
```

which is **affine in w**. Everything below follows from that one fact.

* `|r|` is sinusoidal in `φ` with a single minimum at `angle(-d/h)`, independent
  of `a`; and V-shaped in `a` once `φ` is fixed. Both axes are unimodal, so a
  coarse-to-fine bracket is safe.
* Better: being affine, `h` and `d` are **identifiable from two probes**, after
  which the cancelling weight follows in closed form.

## Two-probe identification

```
r0 = r(0)   = d
r1 = r(a0)  = d + h·a0     =>   h  = (r1 - r0) / a0
                                w* = -d / h = -r0·a0 / (r1 - r0)
```

Two measurements and a division replace thousands of grid points.

Because the model is affine, a residual measured at any `w` also gives an exact
Newton correction `w <- w - r/h`. One step is exact for the model; it is
repeated a few times only to absorb what the real hardware does that the model
does not — DAC nonlinearity, slow drift in `d`.

## Analog gain stepping

The digital weight only spans `|w| <= 1`. When the required `|w*|` falls outside
a comfortable band the **analog** gain of the inject Tx is stepped so that
`|w*|` lands near mid-scale, and identification is repeated. That is what makes
the full range of direct-path strengths reachable, instead of clipping at
`|w| = 1` for a strong path or squeezing the injection into a handful of DAC
codes for a weak one.

## Grid fallback

When the measurement path cannot supply a complex phasor — only a magnitude —
or when the closed-form solve does not actually beat the starting point, a
coarse-to-fine grid search over `(φ, a)` runs instead. If no usable metric is
read at all, the pair's settings are restored to their pre-search values; the
hardware is never left at an arbitrary point, and in particular never at
amplitude 0 just because the measurement path was silent.

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
  "gain_settle_time_s": 0.05,
  "time_budget_s": 25.0,
  "probe_amplitude": 0.5,
  "target_weight": 0.5,
  "min_weight": 0.15,
  "refine_iterations": 3,
  "phase_step_deg": 0.1,
  "amp_step": 0.05
}
```

Indices are **global** Tx/Rx indices across the whole virtual device group.
`measure_rx` is a *receive* index and must be given explicitly unless it happens
to equal `measure_tx`, which is only true for a 1x1 layout.

A balance can be triggered from the Monitor at any time, or run automatically at
Start with `auto_on_start`. An auto balance runs *after* `START_STREAMING` has
been answered — it can take a minute or more, and must never hold up the reply.
