# The Neotron BIOS

The Neotron BIOS is the hardware-abstraction layer for the Neotron Operating System. It allows the OS to run on different hardware platforms with the minimum number of changes.

```
+-------+-------------+
|       |             |
| Shell | Application |
|       |             |
+-------+-------------+
|                     |
|  Operating System   |
|                     |
+=====================+
|                     |
|        BIOS         |
|                     |
+=====================+
|                     |
|    Raw Hardware     |
|                     |
+---------------------+
```

A BIOS should offer interfaces for:

* Block devices (e.g. SD cards)
* Selecting and using various video modes:
	* Text modes which accept an 8-bit Code Page 850 characters
	* Bitmap graphics modes
* Accessing Serial/UART devices
* Accessing any I2C buses
* Accessing any SPI buses
* Playing an audio sample
* System timers
* Hooking interrupts (including scan-line interrupts)

While you could write an application which uses the BIOS directly, most applications will use the higher-level APIs exposed by the OS.
