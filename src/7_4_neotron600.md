# Neotron 600

The Neotron 600 is an open-source PCB which plugs into the Teensy 4.1. Using the Teensy 4.1 gives it excellent performance, an Ethernet PHY and a support for an easy-to-fit external QuadSPI SRAM chip. The only downside is that the pinout limits the colour output to 7 bits (128 colours).

## Features

* **Processor Module**: Teensy 4.1
  * **CPU:** NXP i.MX RT1062 SoC
    * **Core**: 600 MHz Cortex-M7
    * **RAM:** 1024 KiB FlexRAM
    * **ROM:** 128 KiB Boot ROM (factory programmed mask ROM)
    * **GPU:** NXP i.MXRT LCD Controller
      * 1366x768 maximum resolution
      * Supports bitblit, rotation, alpha, chroma-key
  * 8 MiB Flash
  * 8 MiB QuadSPI SRAM (optional)
  * 10/100 Ethernet PHY
* **Video Output:** SVGA
  * Nominally 800x600 at 60 Hz
  * Other resolutions TBD (depends on pixel clock)
  * Texas Instruments THS7316 Video Amplifier used to drive 75 ohm VGA output
  * 2x 2-bit + 1x 3-bit 0.1% R2R DACs, to give max 128 (2^7) colours
* **Storage:** SD Card slot, supports FAT16/FAT32 and MS-DOS partition tables
* **USB:** 2x external USB 2.0 Full-speed Type-A Host ports (with a header for two more)
* **Audio:** 16-bit 48 kHz stereo
  * Wolfson WM8731 Audio CODEC
  * 3.5mm stereo Line Out
  * 3.5mm stereo Amplified headphone output
  * Digital volume control
* **Keyboard/Mouse:** 2x PS/2 Ports (2x 6-pin mini-DIN)
* **Serial**: RS-232 on 10-pin 2.54mm header suitable for DE-9 plug on IDC ribbon, MAX3232 transceiver
* **Ethernet**: Standard 100base-TX RJ45 port (driven from PHY on Teensy 4.1)
* **Expansion**: 4x internal 2.54mm 2x6 headers carrying power, SPI, I2C and a dedicated IRQ line
* **RTC**: MCP7940N RTC and coin-cell battery backup
* **Mechanical Form Factor:** microATX
