# The `.bvr` recording format

Recordings are written by the client, on its own thread, in a self-describing
binary format called `bioview-raw-v2`. Everything about a session lives in this
one file — there are no sidecar files.

## Layout

```
[Header Length   (4 bytes, big-endian)]
[JSON header]
[float32 samples, time-major: each chunk as (num_samples, num_sources)]
[JSON trailer]
[Trailer Length  (8 bytes, big-endian)]
[8-byte magic "BVRMETA1"]
```

A reader finds the trailer from the magic and length at EOF; the sample region
is everything between the header and the trailer. That ordering is what allows
the file to be appended to for the whole recording and finalized once, at close.

The corollary is that **a recording that did not close cleanly has no trailer**.
A crashed session, or a file inspected while it is still being written, leaves a
valid header and valid samples with no end time, no annotations and no parameter
history. A reader should treat that as normal and fall back to the sample region
running to EOF.

## Header

Written when the recording opens:

* `dtype` and layout.
* The ordered source descriptors, one per column — `group_id`, `channel`,
  `label`, `disp_freq`.
* The recording start time.
* A snapshot of the device configuration constants in force at start.

`disp_freq` is the timebase: it carries the rate that actually reached disk,
after both `save_ds` and `disp_ds` (see [the streaming
path](../architecture/streaming.md)), and for the microphone the rate PortAudio
negotiated rather than the one the configuration asked for. Reading sample
timing off the configured `samp_rate` instead will be wrong.

!!! warning "The source list is snapshotted at start"

    The header's column list is written once, when the recording opens. A live
    change that adds or removes streams mid-recording — enabling a BIOPAC
    channel, editing the channel map — alters the emitted row set without
    updating the header, and the rest of that file reshapes wrong. Make source
    changes between recordings, not during one.

## Trailer

Written when the recording closes:

* The recording end time.
* Every device-parameter change made while recording, timestamped.
* Every event annotation ("Mark Event"), timestamped, under `Annotations`.

## Reading one

`bioview-client/tests/bvr_reader.py` is a minimal reference reader used by the
test suite; it shows how to seek the trailer and reshape the sample region.

Because samples are stored time-major per chunk, a whole file is read as
`(total_samples, num_sources)` and transposed once, rather than being stitched
chunk by chunk.
