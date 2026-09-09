# Feature requests

Open an issue on the repository that would carry the change — see [the table in
Reporting a bug](bug-report.md#which-repository). For anything that spans the
protocol, a proposal in `bioview-common` is the right place to start, since both
the server and the client have to agree.

## What makes a proposal actionable

* **The measurement you are trying to make.** Feature requests phrased as an
  experiment are easier to answer than ones phrased as an API — often the rig
  can already do it, and if it cannot, the constraint is usually physical rather
  than about interface design.
* **Where it sits.** A new device is a backend; a new modulation is a signal
  scheme; a new view is a client component. [Code
  style](code-style.md) sets out what belongs where, and the [backend
  contract](../reference/index.md#adding-a-device-backend) is six methods long.
* **What it costs on the streaming path.** Anything per-chunk competes with
  real-time acquisition. If a proposal adds work between `recv` and the display
  queue, say roughly how much.
* **What it would break.** Configuration keys, the wire protocol and the `.bvr`
  layout are all consumed by something that already exists.

## Adding a device

This is the most common request, and the cheapest one. A backend subclasses
`bioview_server.datatypes.Backend`, implements six methods, and is registered in
`bioview_server.device.get_device_handler` with a configuration class in
`bioview_common.datatypes.configuration`. The IPC framing, the display and save
workers and the bounded queues all come from the base class. See [Adding a
device backend](../reference/index.md#adding-a-device-backend); the microphone
backend is the shortest worked example in the tree.

## Discussions

For an idea that is not yet a proposal — an approach you want a second opinion
on, or a question about whether something is already possible — open a GitHub
Discussion rather than an issue.
