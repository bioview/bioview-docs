# Overview

BioView has been architected as a server-client application in order to make it flexible enough to support use-cases wherein two separate devices might handle data acquisition and data display, predominantly useful when acquisition devices are connected to a micro-PC such as a Raspberry PI and control is done using another PC with a display. 

* **Server:** BioView servers can be run independently on any device connected to acquisition hardware. *This may require driver installation as instructed by the device manufacturer.*
* **Client:** BioView clients include a Monitor app for real-time data display and device control along with a Configurator app which allows tweaking configuration parameters of hardware. *Clients do not require hardware-specific drivers to be installed and can seamlessly switch between different servers.*

By default, BioView installers install both a server and all clients. Clients launched by themselves (such as through GUI) will simply spawn a backend server on their own. *This server is shared across all clients and is only terminated once all clients connected to it are closed.*

## Features

There are a fair number of QoL features that we wanted to provide with BioView, some of which are listed below -

* **Data Synchronization:** All devices start data acquisition simultaneously. Any device that experiences failure inhibiting it from acquiring data will simply not have any data collected from it.
* **Process Isolation:** In order to ensure robustness and reliability, BioView is specifically architected to have separate processes for UI and backend, as well as separate threads for each device group. This ensures that, even if a subset of the hardware causes failure *(such as from misconfigured drivers or sudden disconnects)*, the remainder of session does not get impacted.
* **Easy-to-Edit File Structure:** BioView declares a special ```.bvi``` file extension which allows the user to simply double-click on such a file to launch a BioView session. Under the hood, these files follow a simple JSON schema and allow for a large amount of control, not just over device configurations but also over UI layout.
* **Real-Time Device Control:** Each device exposes parameters that may be edited in real-time by the user, even when a session is actively running. This allows for a large amount of control for experimentation, especially when it comes to calibration.
* **Time-Stamped Annotations:** BioView natively supports annotation of events during data acquisition. These annotations are then stored alongside acquired data in a custom ```.bvr``` format which neatly packs in useful metadata for ease of analysis. 

## Supported Hardware

* [Ettus Universal Software Radio Peripheral (USRP)](/reference/usrp): Real-time transmit and receive through ```uhd```, including MIMO configurations, are supported out of the box.
* [BIOPAC Devices](/reference/biopac): Support for BIOPAC MP36, MP150 and MP160 devices is currently implemented. If you would like other hardware to be supported, feel free to [contribute](/contributing/feature-request). *A copy of the BIOPAC Hardware API purchased from BIOPAC Systems Inc. is required and support exists for Windows only.*
* [Microphone](/reference/microphone): Audio input devices recognized by OSes may be used for data collection by BioView as well.

## Performance Considerations

BioView has been implemented in Python. While some may gasp at the performance implications of it, we have been careful to use NumPy wherever possible to ensure performance is as close to C++ as can get. However, there are still some things to keep in mind -

* **Real-time acquisition needs real resources.** A USRP at 1 MSps with several channels will use a core or two for demodulation alone. It is recommended to use as fast a CPU as required and highly recommended to use SSDs for recordings.
* **Memory consumption is bounded.** Data queues implemented in BioView are finite to prevent out-of-memory (OOM) errors. However, this does imply that if your hardware is bottlenecked, data drops will happen and there is no way around it.
* **Defaults are chosen for performance.** While device configuration parameters can be chosen to your liking, defaults have been chosen for optimal performance. Please ensure you do not needlessly override defaults since that may lead to performance regressions.