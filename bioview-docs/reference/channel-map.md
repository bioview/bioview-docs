# Channel maps and MIMO

Several radios in one USRP group form a **virtual device**. Their channels are
flattened into flat global Tx and Rx index spaces, in the order the devices
appear in `hardware` and the order channels appear within each device.

```json
"hardware": {
  "MyB210_4": { "tx_channels": [0, 1], "rx_channels": [0, 1], "...": "..." },
  "MyB210_7": { "tx_channels": [0],    "rx_channels": [0, 1], "...": "..." }
}
```

gives global Tx `0, 1` on `MyB210_4` and global Tx `2` on `MyB210_7`; global Rx
`0, 1` then `2, 3`. **Everything in `channel_map` is written in these global
indices.**

## Layouts

### `full_nxn` (default)

Every measurement Tx crossed with every measurement Rx. One source per pair,
labelled `Tx<i>Rx<j>` where `i` and `j` are 1-based *positions within the
measurement set*, not global indices — so labels stay stable and readable when
some global channels are reserved for injection.

### `hybrid_mimo`

Names the measurement channels explicitly, leaving the rest free for other
duties (typically DPIC injection):

```json
"channel_map": {
  "layout": "hybrid_mimo",
  "mimo": { "tx_global": [0, 1], "rx_global": [0, 1] },
  "dpic": [ { "inject_tx": 2, "measure_tx": 0, "measure_rx": 0 } ]
}
```

Here global Tx 2 radiates the cancellation copy and never appears as a source.

### `custom`

An explicit list of Tx/Rx pairs, each optionally named:

```json
"channel_map": {
  "layout": "custom",
  "pairs": [
    { "tx": 0, "rx": 0, "label": "Chest" },
    { "tx": 1, "rx": 2, "label": "Wrist" }
  ]
}
```

## Injection channels are not sources

Any Tx named as an `inject_tx` in `dpic` is excluded from the measurement set.
It radiates, but it produces no data source and no plot.

## Per-channel IF

`if_freq` is per Tx channel, in the device's local channel order. The receive
chain band-passes around each measurement Tx's IF (`if_filter_bw`), which is how
simultaneous Tx channels are separated at a shared receiver. It is also why a
DPIC inject Tx must sit on the *same* IF as the Tx it is cancelling — see
[DPIC](dpic.md).

## Calibration reference rows

When calibration is enabled with `record_reference`, the backend advertises an
extra `CalRef_*` source per injecting Tx. Those rows go to the display as well
as to disk, so the reference envelope can be watched live.
