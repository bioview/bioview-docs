# The `.bvr` recording format

Recordings are written by the **server**, in a self-describing binary format
called `bioview-raw-v3`. One session is one file — there are no sidecar files.

Writing happens server-side because the file holds each device's **save-rate**
stream, and that stream never crosses the wire. Only the decimated *display*
stream is sent to the client. A device's saved rate is `samp_rate / save_ds`;
`disp_ds` decimates the display path again on top of that and has no effect on
what is recorded.

## Layout

```
[magic "BVR3"    (4 bytes)]
[Header Length   (4 bytes, big-endian uint32)]
[JSON header]
[record][record]...
[JSON trailer]
[Trailer Length  (8 bytes, big-endian uint64)]
[magic "BVRMETA1" (8 bytes)]
```

A reader finds the trailer from the magic and length at EOF; the record region
is everything between the header and the trailer. That ordering is what allows
the file to be appended to for the whole recording and finalized once, at close.

Because each batch is flushed to the OS as it is written, a recording that
crashed is still readable up to its last complete record — it simply has no
trailer, and therefore no end time, annotations or parameter changes.

## Why records, not one matrix

Devices run at independent rates and each emits chunks containing only its own
rows: a USRP might produce 5 rows at 10 kHz while a BIOPAC produces 2 rows at
1 kHz. There is no single matrix that can hold both without resampling, so the
file stores what actually arrived, tagged with which device it came from.

*(This replaces `bioview-raw-v2`, which declared one width for the union of all
devices' channels and concatenated every chunk into it. That was only correct
for single-device sessions; with two devices the columns did not correspond to
the header's source list at all.)*

## Header

Written when the recording opens:

* `t0_unix` / `t0_utc` — the recording start. **The only absolute time in the
  file**; every other time is an offset from it.
* `devices` — the ordered device table. A record's `device_idx` indexes into it.
  Each entry carries `device_id`, `fs` (its save rate), `n_rows`, `dtype` and
  its ordered `sources`.
* `device_config` — a snapshot of the device configuration at start.

## Records

Each record is a fixed 24-byte header, little-endian so the sample block needs
no byteswap on x86, followed by its samples:

| field | type | meaning |
|---|---|---|
| `device_idx` | uint16 | index into `header["devices"]` |
| `flags` | uint16 | bit0: float64 samples (else float32); bit1: complex/IQ |
| `n_samples` | uint32 | samples **per row** in this record |
| `t_offset_us` | uint64 | µs since `t0_unix`, wall clock at emit |
| `sample_idx` | uint64 | device sample counter of the first sample |

then `n_rows * n_samples` samples, row-major. For a complex record the rows are
all real rows followed by all imaginary rows, so it carries `2 * n_rows`.

The two time fields answer different questions and both matter:

* **`sample_idx` is exact.** A record is contiguous with the previous one from
  the same device iff `sample_idx == prev_sample_idx + prev_n_samples`. Any
  other value says precisely how many samples were lost upstream — which was
  previously only a log line.
* **`t_offset_us` is wall clock.** It aligns devices against each other, and
  comparing it against `sample_idx / fs` catches a device that cannot achieve
  its configured rate.

## Trailer

Written when the recording closes:

* `t_end_offset_us` — session length, as an offset from `t0`.
* `devices` — per-device tallies: `records`, `samples`, `gaps`,
  `dropped_samples`, `achieved_fs`.
* `param_changes` — every device-parameter change made while recording, each at
  an `offset_us`.
* `Annotations` — every event annotation ("Mark Event"), each at an `offset_us`.

Annotations reach the file over the wire: the client sends `MARK_EVENT`, since
the recording lives on the server. Parameter changes need no message — the
server already sees them when it applies them.

## Reading one

`bvr_tools.py` in the repository root is a standalone reader and quick-analysis
CLI (it imports nothing from the BioView packages). It groups records by device
and hands back one `(n_rows, n_samples)` array per device, each with its own
time vector, plus gap and achieved-rate reporting:

```
python bvr_tools.py recordings/session.bvr
python bvr_tools.py recordings/
python bvr_tools.py recordings/session.bvr --plot out.png --start 10 --stop 20
```

`explore_recording.ipynb` walks through the same API interactively.
