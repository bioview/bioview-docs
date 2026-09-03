# Reference

* [Configuration files](configuration.md) — the JSON schema for experiments and
  device groups.
* [Channel maps and MIMO](channel-map.md) — global channel indexing, layouts,
  source naming.
* [Signal schemes](signal-schemes.md) — CW, FMCW, pulsed Doppler, and the
  calibration overlay.
* [DPIC](dpic.md) — direct-path interference cancellation.
* [Wire protocol](protocol.md) — commands, responses, timeouts, data sources.
* [The `.bvr` format](bvr-format.md) — how recordings are stored.
* [Monitor UI](ui.md) — what each control does.

## Supported devices

* [USRP](usrp.md)
* [BIOPAC](biopac.md)

## Adding a device backend

A backend subclasses `bioview_server.datatypes.Backend` and implements six
methods:

| Method | Contract |
| --- | --- |
| `populate_data_sources()` | Fill `self.data_sources` with a `DataSource` per stream the device produces. |
| `_initialize()` | Open the device. Raise with a real message on failure — a bare falsy return becomes a generic "unable to initialize" upstream. |
| `_start_streaming()` | Start or resume the acquisition workers. Return truthy. |
| `_stop_streaming()` | Pause them. Do not tear them down. |
| `_disconnect()` | Release the device. |
| `_queue_param_update(params)` | Apply a live parameter change. |

Two optional hooks:

* `_post_start_streaming()` — work that must happen once streaming is live but
  must not delay the reply (an auto DPIC balance, for instance).
* `_apply_param_update_local(params)` — mirror parameters that change
  `data_sources` on the parent side, since `get_data_sources()` is answered out
  of the parent process while `_queue_param_update` runs in the child.

Register it in `bioview_server.device.get_device_handler` and add its
configuration class to `bioview_common.datatypes.configuration`. Everything
else — the IPC framing, the display and save workers, the bounded queues — comes
from the base class.
