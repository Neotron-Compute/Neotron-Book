# RTC

The TM4C SoC has a battery-backed real-time clock function, but the TM4C Launchpad doesn't break out the battery backup pin and so it will always forget the time when the system is powered off.

Instead, an MCP7940N battery-backed real-time clock chip keeps track of calendar time when the system is powered off, and also offers 64 bytes of battery backed SRAM for system settings. Like on an IBM PC-compatible, removing the battery will "reset" the system settings (on a PC this is known as *clearing the CMOS*, for various historical reasons).

