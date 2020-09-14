# Neotron 600

The Neotron 600 is an open-source PCB which plugs into the Teensy 4.1. Using the Teensy 4.1 gives it excellent performance, an Ethernet PHY and a support for an easy-to-fit external QuadSPI SRAM chip. The only downside is that the pinout limits the colour output to 7 bits (128 colours).

## Features

* 600 MHz 32-bit ARM Cortex-M7 CPU core
* 1024 KiB of internal SRAM (used as VRAM and for OS buffers)
* 8 MiB external QuadSPI Flash
* 8 MiB external QuadSPI SRAM (optional)
* Super-VGA output
    * 640x480, 720x400 and 800x600 resolution
    * Maybe 320x200 and 320x240 (if we can re-map the video buffer on each scan-line to do line doubling)
    * Fixed RRGGGBB palette of 128 colours (sadly the Teensy 4.1 only breaks out 7 eLCDIF data pins)
* 16-bit 48 kHz audio input and output (line-in, mic-in, line-out and headphone-out)
* Four USB ports
    * Two USB A ports on-board
    * Header for additional two ports
* IEEE-1284 Parallel Port
* RS232 Port (five-wire)
* 2x SD Card Slots (one micro-SD internal, one full-size SD external)
* 2x PS/2 ports (Keyboard and Mouse)
* 2x Atari/Sega joystick ports
* Battery-backed Real-time Clock and CMOS RAM
* SPI and I2C based expansion bus (Neotron-32 Compatible)
