# USB

The TM4C Launchpad used in the Neotron 32 has a full-speed (11 Mbps) USB On-The-Go peripheral routed to a USB micro-AB port. This means it is normally a USB Device (e.g. a USB Serial Adaptor, or a USB Human Interface Device), but it can be placed into a mode where it acts as a USB Host. To make this work, we need two things:

1. A USB micro-AB to USB Type-A Receptacle adaptor.
  * These adaptors are commonly used with the USB micro-AB ports on a Raspberry Pi.
  * We could optionally plug in a USB Hub if we wanted to connect more than one USB Device.
2. A USB Host stack.

The part latter is going to be difficult - the USB protocol intentionally places the burden of work on the Host developer, because that end of the link generally costs more, and has more power available. We have a couple of options available:

1. Use the Texas Instruments Tiva-C *usblib* stack (https://github.com/yuvadm/stellaris/tree/master/usblib).
2. Use the USB Host stack from Das U-Boot, or some similar project.
3. Write a USB Host stack from scratch in Rust.

All of these options are out-of-scope for the time being.
