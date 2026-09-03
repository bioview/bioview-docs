# Quick start

## 1. Open the Monitor

```bash
bioview
```

That is the whole command. The launcher starts a hidden localhost server for you
and opens the Monitor against it. If a BioView server is already running on this
machine — because a Configurator is open, say — it is reused.

You should see the status bar report a connected server within a second or two.

## 2. Load a configuration

If no configuration file was given, the Monitor prompts for one at startup. Or
pass it directly:

```bash
bioview --config-file my_experiment.json
```

A minimal file with a single USRP:

```json
{
  "Experiment": {
    "type": "EXPERIMENT",
    "enable_save": true,
    "save_dir": "./recordings",
    "file_name": "session.bvr"
  },
  "USRP": {
    "type": "USRP",
    "signal_scheme": "cw",
    "samp_rate": 1000000,
    "carrier_freq": 1000000000,
    "hardware": {
      "MyB210": {
        "tx_channels": [0],
        "rx_channels": [0],
        "if_freq": [100000],
        "tx_gain": [40],
        "rx_gain": [40],
        "if_filter_bw": 5000
      }
    }
  }
}
```

`MyB210` must be a device name the server can resolve. Use the
[Configurator](configurator.md) to see what is attached and to assign names.

No hardware to hand? Swap the device block for a `DUMMY` one and everything
below still works.

## 3. Initialize

Press **Initialize**. The device indicators in the status bar go yellow, then
green. A USRP can take a while — `uhd.find` alone takes seconds.

If a device goes red instead, the log says which one and why. Recognised causes
carry a suggested remedy.

## 4. Choose what to plot

Open the **Experiment** settings tab and tick sources in **Plot Sources**. Each
is named `Device: Source`, e.g. `USRP: Tx1Rx1`. Set the grid with **Plot
Layout** and the visible window with **Display Time**.

## 5. Record

Set **File Name** and **Save Path**, tick **Save**, and press **Start**. Press
**Mark Event** to drop a timestamped annotation into the recording.

**Stop** ends the stream. The recording is finalized on stop — the file's
metadata trailer is written then. Everything, including annotations and any
parameter changes you made along the way, is in that one `.bvr` file.

## 6. Read the data back

`bioview-client/tests/bvr_reader.py` is a minimal reference reader. See
[the `.bvr` format](../reference/bvr-format.md).

## Where things are

| | |
| --- | --- |
| Server log | `server.log` in the BioView cache directory. This is where a GUI-spawned server's output goes, and the first place to look when a device will not initialize. |
| Recordings | Wherever **Save Path** points. |
