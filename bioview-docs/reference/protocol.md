# Wire protocol

Two TCP connections per client.

* **Control (8998)** — request/response, JSON, framed as
  `[Length (4 bytes, big-endian)][JSON payload]`. Framing rather than a raw
  write, so messages survive TCP coalescing and fragmentation and are not
  capped by a buffer size.
* **Data (8999)** — one-way, server to client, framed as
  `[Total Length (4)][Header Length (4)][JSON header][raw data]`.

## Commands

Only the enum *name* travels on the wire.

| Command | Purpose |
| --- | --- |
| `PING_SERVER` | Liveness check. |
| `DISCOVER_SERVERS` | Server self-description: hostname, ip, version, live client count, advertised data sources. Also the handshake a client uses to tell a BioView server apart from any other listener on the port. |
| `CONNECT_SERVER` | Begin the handshake; answered with `SERVER_CHALLENGE`. |
| `AUTHENTICATE_CLIENT` | Answer the challenge. |
| `DISCONNECT_SERVER` | Leave. |
| `DISCOVER_DEVICES` | Resolve the device *groups* declared in a loaded configuration against what is attached. |
| `LIST_DEVICES` | Config-free enumeration of everything attached right now, plus each backend's editable-property schema. What the Configurator asks. |
| `SET_DEVICE_CONFIG` | Apply a Configurator edit to one device. |
| `INITIALIZE_DEVICES` | Bring the configured device groups up. |
| `DISCONNECT_DEVICES` | Release them. |
| `START_STREAMING` / `STOP_STREAMING` | Start and pause acquisition across every initialized device. |
| `GET_DEVICE_STATUS` | Poll the server-wide device state. Used while a long device operation runs, and while a DPIC balance runs -- `dpic_balance` carries `{pending, ok, message, results, device_id}` for the most recent one. |
| `UPDATE_RUNNING_PARAMETER` | Change one parameter on a live device. The reply carries the new source list when the change added or removed streams. |
| `RUN_DPIC_BALANCE` | Start a direct-path cancellation search on one device group. Acknowledged at once; the outcome is read from the `dpic_balance` field of `GET_DEVICE_STATUS`, the same acknowledge-then-poll shape the device operations use. |

## Responses

`SUCCESS`, `ERROR`, `WARNING`, `INFO`, `DEBUG`, plus the connection responses
(`CONNECTION_ACCEPTED`, `CONNECTION_REFUSED`, `SERVER_CHALLENGE`,
`AUTHENTICATION_SUCCESS`, `AUTHENTICATION_FAILURE`) and the device responses
(`DEVICE_STATUS_CHANGED`, `DEVICE_DISCOVERY_COMPLETED`, `DEVICE_CONNECTING`,
`DEVICE_CONNECTED`, `DEVICE_DISCONNECTED`, `DEVICE_LIST`,
`DEVICE_CONFIG_UPDATED`).

## Device status

`DeviceStatus` is one of `Not Initialized`, `Available`, `Unavailable`,
`Connecting`, `Connected`, `Streaming`, `Disconnected`.

`ClientStatus` is ordered by connectivity so checks can compare levels:

```
DEFAULT (-1) < SERVER_DISCONNECTED < SCANNING < SERVER_CONNECTED
             < DEVICES_DISCOVERED < DEVICES_CONNECTED < STREAMING
```

## Timeouts

All of these live in `bioview_common.constants.network`.

| Constant | Value | Used for |
| --- | --- | --- |
| `AUTH_TIMEOUT` | 5 s | The handshake. |
| `RESPONSE_TIMEOUT` | 30 s | Ordinary control exchanges. |
| `DEVICE_OP_COMMAND_TIMEOUT` | 30 s | `LIST_DEVICES` and other backend walks. |
| `DEVICE_OP_POLL_INTERVAL` | 2 s | How often `GET_DEVICE_STATUS` is polled during a long device operation. |
| `STREAMING_COMMAND_TIMEOUT` | 45 s | `START_STREAMING` / `STOP_STREAMING`. |
| `DISCOVER_TIMEOUT` | 120 s | Device discovery. |
| `DISCOVERY_CACHE_TTL` | 60 s | How long a discovery result is reused. |
| `INIT_TIMEOUT_USRP` | 300 s | USRP initialization. |
| `INIT_TIMEOUT_DEFAULT` | 120 s | Other backends. |
| `DPIC_BALANCE_TIMEOUT` | 900 s | Ceiling on how long the client polls for a balance outcome. |
| `DPIC_BALANCE_POLL_INTERVAL` | 0.5 s | Balance poll rate — also the refresh rate of the live values in the settings panel. |

`STREAMING_COMMAND_TIMEOUT` is a backstop for a server that has stopped
answering, not a budget for a slow device. The server starts devices in sequence
and bounds each one itself — 5 s for a start, 15 s for a stop — so it answers
with an error naming the device long before the wire timeout fires. Those
per-device budgets are tight on purpose: the acquisition workers are threads
that already exist, so a start is a resume rather than a bring-up, and a device
that has not answered in a few seconds is wedged rather than slow.

Opening the device is where the time goes, and `INIT_TIMEOUT_USRP` is sized for
it: USB enumeration, FPGA and CODEC bring-up and clock locking.

## Authentication

A challenge/response HMAC over a shared secret, taken from
`BIOVIEW_SHARED_SECRET` on both machines and defaulted so localhost and
trusted-LAN use work out of the box. It is a guard against accidentally driving
someone else's rig, not a security boundary — run a server with `--local` unless
the network is trusted.

## Data sources

A `DataSource` addresses one physical stream:

```json
{"group_id": "BIOPAC", "channel": 0, "label": "Ch1", "disp_freq": 1000.0}
```

Identity is `(group_id, channel)`. `label` is a display name that can change
freely, so it is excluded from equality and hashing — sources are used as dict
keys for routing. The UI name is `"<group_id>: <label>"`.
