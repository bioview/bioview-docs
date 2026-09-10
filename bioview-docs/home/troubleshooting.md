# Troubleshooting

If you encounter errors while using BioView clients, your best first bet is to look at  ```server.log``` in the BioView cache directory, where all records are logged. If you can debug using this log, great. If not, [raise an issue](/contributing/bug-report). 

Below are some common errors that you may encounter, along with potential fixes.

## Networking Issues

### Client status bar says "Server Disconnected"

Whenever a client window is launched, a localhost server is automatically spawned in the background. If you are seeing this error, it means that either this server died or could never get connected at the specified ports. This may happen if another process in the OS is using the server ports. By default, BioView spawns servers and clients to connect using port 8998 and 8999. If these are being used by another process, consider launching BioView with a different port.

## Driver Issues

### BIOPAC device fails to initialize

This is a fairly common error which occurs because Windows Memory Integrity refuses to allow ```mpdev.dll```, the BIOPAC driver, to load. This can be fixed by going to Windows Security, turning Memory Integrity off, and rebooting.

### uhd was not found

This error occurs if you have not installed the UHD drivers from Ettus' official website, or if there is a version mismatch among the driver version installed and the driver version expected by BioView. Follow [install instructions](/setup/installation) to fix this.

### USRP device not found

There are 2 cases for this issue to arise -

1. **The device is not connected.** *I suppose the fix for this is self-explanatory.*
2. **The device name was recently changed.** Often when we rename a USRP using Configurator, it only properly kicks in with unplugging and replugging the device back in. *[This may be relevant](https://www.youtube.com/watch?v=nn2FB1P_Mn8).*

## Performance Issues

### The plot scrolls too slowly

The x-axis is drawn from the *nominal* sample rate, so a plot that scrolls at half speed means the acquisition path is only achieving half the requested rate.

For BIOPAC this is almost always the per-sample polling fallback: if the DLL cannot start the acquisition daemon behind `receiveMPData`, a Python loop plus a driver round-trip per sample cannot keep up at 1 kHz. BioView measures the achieved rate in that path and warns when it drifts. See [BIOPAC](../reference/biopac.md).

### Data drops

Since all streaming queues (saving and display) are bounded, constrained hardware may experience data drops. If drops are being sustained on the save queues, it implies disk I/O bottlenecks so... just use an SSD. If drops are being sustained on the display queue, it implies the networking stack is constrained (if using multiple devices) or the CPU is unable to keep up.

### High CPU utilization

If you notice CPU utilization to be too high, reduce the plot grid size and/or the display duration and/or lower the display rate. Since we only continuously process data being displayed, CPU usage scales with the number of open display plots.

## Data Integrity Issues

### Incorrect microphone recording

Windows enumerates loopback inputs ("Stereo Mix", "What U Hear") alongside real microphones. During a routine, using a loopback device as microphone input may cause the instruction audio being recorded rather than the participant. While BioView tries to skip loopbacks by name and inform the user about what inputs are being taken, specifying the audio input device by name in ```.bvi``` configuration is a much more concrete fix. See [microphone reference](../reference/microphone.md) for more details.

### Spikes in the displayed data

Filtering edge effects can arise if the receive buffer is small and show up as spikes. Increase the receiver frame buffer fixes this.