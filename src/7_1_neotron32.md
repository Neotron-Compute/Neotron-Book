# Neotron 32

The Neotron 32 is the baby of the Neotron range. As the name suggests, it has just 32 KiB of SRAM (along with 256 KiB of Flash ROM). It is based around a Texas Instruments Launchpad board, and uses the same basic design as the earlier Monotron project.

## Links

* Hardware: <https://github.com/Neotron-Compute/Neotron-32-Hardware>
* BIOS: <https://github.com/Neotron-Compute/Neotron-32-BIOS>
* Main CPU Board: <https://www.ti.com/tool/EK-TM4C123GXL>

## Features

The full specifications are:

* **CPU:** Texas Instruments TM4C123 SoC, on a Texas Instruments Tiva-C Launchpad PCB
  * **Processor Core**: 80 MHz ARM Cortex-M4F
  * **RAM:** 32 KiB on-die SRAM
  * **ROM:** 256 KiB on-die Flash
  * **GPU:** None - software VGA using 3x SPI interfaces
    * 800x600 nominal resolution
    * 400x300 effective resolution
    * 48-column colour text mode and 80-column mono text mode
    * Maximum 8 colours
* **Video Output:** SVGA
  * Nominally 800x600 at 60 Hz
  * Simple 330 ohm terminated video output
* **Storage:** SD Card slot, supports FAT16/FAT32 and MS-DOS partition tables
* **USB:** 1x external USB 2.0 Full-speed micro-AB port
* **Audio:** 37 kHz PWM audio with three channel wavetable synthesiser
  * 3.5mm stereo Line Out
* **Keyboard/Mouse:** 2x PS/2 Ports (2x 6-pin mini-DIN)
* **MIDI**: MIDI In and MIDI Out (2x 5-pin 180-degree DIN)
* **Serial**: RS-232 on 10-pin 2.54mm header suitable for DE-9 plug on IDC ribbon
* **Parallel**: 3.3v PC-style Parallel Port with DB-25 port
  * Also functions as 3.3v GPIO with 12 outputs and 4 inputs.
* **Joystick**: 2x 9-pin game ports
  * supports Atari-compatible two-button joysticks, or SEGA MegaDrive/Genesis controllers
* **Expansion**: 2x internal 2.54mm 2x6 headers carrying power, SPI, I2C and a dedicated IRQ line
* **RTC**: MCP7940N and coin-cell battery backup
* **Mechanical Form Factor:** Fits Hammond [1598D] case

The Cortex-M4 is responsible for generating the video pixels, at a rate of 40 million mono pixels/second (in high-res text mode) or 20 million colour pixels/second (in all other modes). This leaves only around 4.5% of the CPU available (during the vertical blanking interval) to run your application, which brings the performance down to around the same as a 2 MHz 6502. This is enough to run simple games or a BASIC interpreter. The challenge is to see how much you can squeeze out of such limited resources!

The Neotron 32 uses a Texas Instruments Launchpad board to provide the processing power. This is fitted to a custom PCB which breaks out all of the ports, and adds an AtMega AVR CPU to control the PS/2 and Joystick interfaces. The PCB design is [registered open-source hardware], and [available on Github]. The [ROM BIOS] and [Neotron OS] are also fully open-source and available on Github. The ROM BIOS and OS are loaded over the Launchpad's USB debug connection using standard TI Tiva-C flashing tools (such as OpenOCD, or lm4flash). Once flashed, applications can be loaded from a FAT32 formatted SD card.

[registered open-source hardware]: https://certification.oshwa.org/uk000007.html
[available on Github]: https://github.com/neotron-compute/neotron-32-hardware
[ROM BIOS]: https://github.com/neotron-compute/neotron-32-bios
[Neotron OS]: https://github.com/neotron-compute/neotron-os

## Architecture

In a typical 8-bit home computer, the main processor was just a CPU, and all the other peripherals (GPIO ports, RAM, video, etc) were external devices that sat on the processor's address and data bus. On the Neotron 32, the Texas Instruments TM4C123 SoC integrates all of those features (well, not video, but we work around that by misusing some of the serial peripherals) into a single package. The main external interfaces from the SoC are narrow serial buses such as SPI, I²C and UART rather than the wide 8-bit data bus and 16-bit address bus of an 8-bit home computer. The I²C bus is ideal for low-bandwidth peripherals, such as real-time clocks, whilst the SPI bus can be driven at up to 20 MHz, giving 20 Mbit/sec (or around 2400 KiB/sec) making it better for mass storage and networking.

On current revisions of the Neotron 32, the I/O controller (responsible for the PS/2 interfaces and the two Joystick ports) is connected to the main SoC via a UART link. It is likely, however, that this will change to be an I²C based device with an interrupt line in future, following the standard Microsoft HID over I²C Protocol.

```
+-----------------+
|                 |
|     TM4C123     |
|                 |
+-----------------+
   |   |  | | | |
   |   |  | | | +---<UART> Neotron I/O Controller
   |   |  | | +---<UART> Wi-Fi Modem
   |   |  | +---<UART> RS-232 Interface
   |   |  +---<UART> MIDI In/Out
   |   |
   | <I2C>
   |   |
   |   +-- Real-Time Clock
   |   +-- Neotron I/O Controller (future possibility)
   |   +-- Expansion Port A
   |   +-- Expansion Port B
   |
 <SPI>
   |
   +--- SD Card
   +--- GPIO Port / Parallel Port
   +--- Expansion Port A
   +--- Expansion Port B
```
