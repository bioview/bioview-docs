# The server

`bioview_server.server.Server` owns the hardware. It listens on two TCP ports —
control (8998) and data (8999) — and forwards client commands to per-device
backend processes.

```bash
python -m bioview_server.server --local --control-port 8998 --data-port 8999
python -m bioview_server.server --exit-when-idle 20
```

`--local` refuses connections from anywhere but this machine and its LAN.
`--exit-when-idle N` retires the server once it has had no client for N seconds.

## Client sessions

Several clients may be connected at once — typically a Monitor and a
Configurator sharing the one server. Each is a `ClientSession` holding its own
control connection, data connection and client info, served by its own command
thread. The accept loop hands a newly authenticated client to its thread and
goes straight back to accepting, so a second window can connect while the first
is mid-operation.

Every reply is addressed to the client that asked. The session a thread is
currently serving is held in a `threading.local`, which routes replies correctly
without threading a session argument through every handler.

Because the hardware is shared, **device configuration, device status and
streaming state are server-wide**. A device operation started by one client is
visible to the others through `GET_DEVICE_STATUS`.

## Connecting

1. Client sends `CONNECT_SERVER` with its info.
2. Server replies `SERVER_CHALLENGE` with a nonce.
3. Client answers `AUTHENTICATE_CLIENT` with an HMAC over the nonce using the
   shared secret (`BIOVIEW_SHARED_SECRET`, defaulted for localhost use).
4. On success the server accepts the client's data connection and starts its
   command thread.

## Device backends

Each device group in the configuration gets one `Backend` subprocess:

| Group type | Backend |
| --- | --- |
| `USRP` | `bioview_server.device.usrp.USRPBackend` |
| `BIOPAC` | `bioview_server.device.biopac.BIOPACBackend` |
| `DUMMY` | `bioview_server.device.dummy.DummyBackend` |

A backend that cannot be imported (missing driver, missing Python dependency) is
recorded in `UNAVAILABLE_BACKENDS` with the reason and reported to the
Configurator alongside the device list, rather than silently not appearing.

### Backend IPC

The parent talks to each backend over a `multiprocessing.Queue` pair:

* `command_queue` — parent to child, carrying `{command, args, request_id}`.
* `response_queue` — child to parent, carrying `{type, result | message,
  request_id}`.

**Every backend has its own response queue.** A reply carries no sender, so a
queue shared between backends lets one device consume another's answer — which
is exactly what broke multi-device streaming: with a USRP and a BIOPAC in the
same session, a slow USRP start left the parent reading a reply that was never
going to be its own, and a bare `queue.Empty` surfaced as a
`Failed to start streaming:` with nothing after the colon.

Replies are matched by `request_id`, so a late answer to a request that has
already timed out is discarded instead of being handed to the next caller.

Timeouts are per operation, because the operations are not comparable:

| Operation | Timeout |
| --- | --- |
| `CONNECT_DEVICES` | 150 s (a `uhd.find` plus device init) |
| `START_STREAMING` / `STOP_STREAMING` | 90 s (worker processes are spawned) |
| `DISCONNECT_DEVICES` | 15 s |
| everything else | 10 s |

`UPDATE_RUNNING_PARAMETER` is fire-and-forget: applying it can restart the
device stream, and the server must not block its command thread on that. The
child's reply is discarded by request-id matching on the next request.

### Starting a stream

`START_STREAMING` starts every device in turn. If any device fails, the ones
that already started are stopped again and the whole command fails — a partially
started session records data that cannot be aligned across devices. The error
names each device that failed and why:

```
Failed to start streaming -- USRP: USRP did not answer START_STREAMING within 90s
```

`STOP_STREAMING` is the mirror image: every device is asked to stop even if one
of them raises, so a single stubborn device cannot leave the rest running.

## Data fan-out

One thread serves the whole server. Backends push
`{"data": ndarray, "sources": [source dicts]}` onto a shared output queue; the
data thread drains it and writes each chunk to every live session. A client
whose socket has gone is dropped and the rest keep streaming. The queue can only
be drained once, which is why this is one thread rather than one per client.
