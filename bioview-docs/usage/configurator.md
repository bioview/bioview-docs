# Using the Configurator

```bash
bioview-configurator
```

The Configurator lists every device attached to the server and lets per-device
properties be edited — the role NI's USRP Configuration Utility plays for
LabVIEW. Like the Monitor, it opens with a localhost server of its own, or
reuses one that is already running.

Unlike the Monitor it needs **no configuration file**: it asks the server what
is plugged in right now (`LIST_DEVICES`), rather than resolving device groups
declared in a config.

## Flow

1. **Discover** lists what is attached. This runs every backend's discovery on
   the server — a `uhd.find` plus BIOPAC's WMI walk — so it takes a few seconds.
2. Selecting a device enables **Edit**, if that backend declares any editable
   properties. Backends with none leave the button greyed out.
3. **Edit** opens a modal with those fields, plus Save and Cancel. The modal
   stays up while the change is applied, showing a busy indicator and then
   Completed or Failed.
4. The list is re-read afterwards, so a renamed device shows its new name
   straight away.

## What is listed

Each row shows the device's name, what it actually reports about itself, and
whether the OS considers it usable. A device whose driver failed to load is
still discovered and still listed — without the usability check it would look
perfectly fine right up until acquisition failed to open it.

Not every device reports a serial number (a BIOPAC MP36 supplies none over USB),
so the identifying line shows only what the device actually gives.

Backends that failed to load are reported too, with the reason. A missing driver
or Python dependency shows up here rather than the hardware silently never
appearing.

Virtual (dummy) devices are excluded: this window lists hardware that is
actually attached, and a simulated device sitting among the real ones is
misleading.

## Naming a USRP

The only editable property today is a USRP's `device_name`. Names are stored by
BioView, keyed on the radio's serial, and applied during discovery — nothing is
written to the radio's EEPROM, so renaming is safe and reversible. Configuration
files, the channel map and the serial cache all see the chosen name.

One caveat the dialog warns about: **a renamed USRP keeps announcing its old
name until it is power cycled.** BioView's alias is already in effect, so
configurations referring to the new name work immediately; it is the radio that
has not caught up.

See [USRP](../reference/usrp.md) for the details.
