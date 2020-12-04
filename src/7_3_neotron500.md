# Neotron 500

The Neotron 500 is an unfinished attempt an an open-source PCB using an STM32H7. The idea was to only use parts from the JLCPCB catalog so that it could be built using their low-cost surface-mount PCB Assembly service. Unfortunately, it transpires that the STM32H7's QuadSPI controller only supports Flash ROM, not SRAM, meaning to get the targeted 8 MiB of RAM, a DDR SDRAM chip would be required. Currently that appears to be more difficult to lay out, and so the project was abandoned in favour of the Neotron 600.

It should also be noted that almsot all STM32 F7 and H7 series parts currently do not include a High-Speed USB2.0 PHY, instead requiring an external ULPI PHY. There are some very new parts which do include a USB2.0 High-Speed PHY, but they don't include a TFT controller.

## Features

* 480 MHz 32-bit ARM Cortex-M7 CPU core
* 1024 KiB of internal ECC SRAM
* 8 MiB external DRAM
* Super-VGA output
    * 640x480
    * 800x600
    * 256 colours from a pallette of 262,144
* 16-bit 48 kHz audio input and output (line-in, mic-in, line-out and headphone-out)
* Four high-speed (480 Mbps) USB 2.0 ports
    * Two USB A ports on-board
    * Header for additional two ports
* IEEE-1284 Parallel Port
* RS232 Port (five-wire)
* SD/MMC Slot
* PS/2 Keyboard and Mouse ports
* 2x Atari/Sega joystick ports
* IDE Interface for hard drive, Zip drive, LS-120 or CD-ROM
* Battery-backed Real-time Clock and CMOS RAM
* SPI and I2C based expansion bus
