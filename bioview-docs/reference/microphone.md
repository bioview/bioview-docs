# Microphone

Host audio input, through PortAudio (`sounddevice`). One row per captured
channel at the negotiated sample rate, emitted through the same display path as
every other device — which is also the save path — so speech lands in the
`.bvr` sample-aligned with the RF and physiological rows recorded beside it.

There is no sidecar `.wav` and nothing to line up afterwards. That is the point:
an audio file written on its own clock has to be aligned against the rest of the
session, and the alignment is exactly what a speech experiment needs to be sure
of.

## Configuration

```json
"MIC": {
  "type": "MICROPHONE",
  "samp_rate": 16000,
  "channels": 1,
  "device": "default",
  "gain": 1.0,
  "labels": ["Audio"]
}
```

| Key | Default | Meaning |
| --- | --- | --- |
| `samp_rate` | `16000` | Requested rate, in Hz. Also the recorded rate — see below. |
| `channels` | `1` | A **count**, not the enable mask BIOPAC uses. A sound card's inputs are not individually selectable. |
| `device` | `"default"` | Which host input to open. See below. |
| `blocksize` | `0` | Frames per callback. `0` picks a tenth of a second. Capped at 100 ms either way. |
| `gain` | `1.0` | Applied to each chunk on the way out. Adjustable while streaming. |
| `labels` | — | Per-channel names; defaults to `Audio`, or `Audio1`…`AudioN` for more than one. |
| `disp_ds`, `save_ds` | `1` | Present for parity. Audio is emitted at full rate. |

A `channels` value written as a BIOPAC-style mask (`[1, 1, 0, 0]`) is accepted
and read as its number of set entries, so a block copied from a BIOPAC config
still loads.

## Naming the input

`device` accepts, in this order:

* `"default"` or omitted — the host's default input.
* an integer — a PortAudio device index.
* a discovery key — the sanitized name BioView lists (`Headset_Microphone_USB`).
* any case-insensitive substring of the host's own device name (`"headset"`).

Host names carry punctuation and vendor strings ("Microphone (Realtek(R)
Audio)"), so discovery reduces them to alphanumerics, underscores and hyphens.
Two host APIs exposing one physical input produce the same name twice; the
second is suffixed with its index so every key stays addressable.

### Loopbacks are never chosen automatically

Windows enumerates loopback inputs — "Stereo Mix", "What U Hear" — alongside
real microphones. A loopback records what the machine is *playing*, which during
a routine is the instruction audio and not the participant, and nothing about
the recording says so until it is opened.

When there is no host default, the fallback deliberately skips them and warns
which input it took instead. A config that names one by hand is honoured: the
backend has no way to know that was not deliberate.

### An input that will not open is not chosen

Enumerating is not the same as working. A Realtek front-panel jack appears in
the device list whether or not anything is plugged into it, and an unpopulated
one refuses every open with `Invalid device`. Taking it because it came first in
enumeration order failed the whole device group at Connect while a working input
sat further down the list.

When there is no host default, each non-loopback candidate is probed by opening
it before it is chosen. If none of them opens, the error names the ones that
were rejected and why, and lists every input that could be named instead --
including the loopbacks, which are still never chosen automatically but can be
asked for by name.

A device named explicitly is *not* second-guessed. That is the operator's
decision, and the error PortAudio raises at Connect names it.

## Silence is reported

A muted, unplugged or zero-gain input opens cleanly, streams chunks on time and
plots a perfectly flat line -- which is also what a working input in a quiet
room looks like. After five seconds of digital silence the backend says so, once
per streaming run, and says explicitly that the stream itself is healthy so the
search goes to the operating system's sound settings rather than to BioView. The
warning is withdrawn as soon as any signal arrives.

## Sample rate is negotiated, not demanded

Windows' MME host API commonly offers a device only at its native rate, so a
perfectly reasonable 16 kHz request is refused outright. Rather than fail the
session, the backend probes for a rate the input does support — the device's own
native rate first, then 48/44.1/32/22.05/16/11.025/8 kHz — warns which it took,
and captures at that.

**The probe opens the stream; it does not ask.** `sd.check_input_settings` asks
the host API about a *format*, and under WDM-KS that answer says nothing about
the *device*. Measured on a Realtek machine:

| Input | Settings | `check_input_settings` | `Pa_OpenStream` |
| --- | --- | --- | --- |
| FrontMic | 1ch @ 44.1 kHz | OK | **Invalid device** |
| Stereo Mix | 1ch @ 44.1 kHz | refused | **opens** |

Wrong in both directions, so the negotiated rate was one the device would refuse
and the chosen input was one that could never be opened. Each candidate rate is
now tried by opening a stream and closing it again -- a few milliseconds, run a
handful of times at configuration time.

**Resampling was considered and rejected.** A per-chunk resample rings at every
chunk boundary, and carrying polyphase state across chunks is a lot of machinery
whose only purpose is to make the recording *less* faithful than the one the
hardware would have given.

The negotiated rate is carried on each `DataSource` as `disp_freq`, which is
what the `.bvr` header records, so a file captured at 44.1 kHz after a 16 kHz
request still reads back on the right timebase.

Negotiation runs in the **parent** process, during construction. `GET_DATA_SOURCES`
is answered out of the parent, so a rate settled on inside the device subprocess
would leave the advertised `disp_freq` — the recording's timebase — stale.

## Capture

Capture is PortAudio's callback, not its blocking read: the blocking API is not
implemented on every Windows host API (WDM-KS refuses it outright), and the
callback path is the one that works everywhere.

The callback runs on PortAudio's own high-priority thread, so it does nothing
but copy the frames into a bounded queue. All conversion, gain and queueing
happens on the acquisition worker, where a slow consumer costs a dropped chunk
rather than stalling the audio device itself. Both loss counters — host
overflows and chunks dropped before conversion — are reported together, rate
limited, as a warning: either means the recording is short by that much and is
no longer sample-aligned with the rest of the session.

The PortAudio stream is opened at Connect and **stopped**, not closed, at Stop.
Start/Stop is pressed many times in a session and reopening the device each time
is both slow and a chance for another application to take it in between. What
the stream captured before Start is discarded, so the recording is not offset
against the devices starting alongside it.

## Cost on the plots

Every captured sample is emitted: saving is fed from the display stream, so
decimating for display would decimate the recording too. A 44.1 kHz input
therefore feeds the plot at 44.1 kHz, which is two orders of magnitude faster
than BIOPAC. It is fine on its own and worth watching if it shares a window with
sixteen RF sources — plot fewer of them, not fewer audio samples.

## Availability

The backend registry probes PortAudio at import and reports the reason if it
cannot be reached, so a missing `sounddevice` reads as "backend not available"
rather than failing at Connect inside a device subprocess.

A group is available as soon as the machine has any input at all. Like BIOPAC
and unlike the USRP, the `hardware` keys here are user-chosen labels rather than
the names discovery reports — a host input is named by the operating system, and
a configuration is usually written before anyone has seen that name.
