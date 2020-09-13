# Neotron 32

## Features

The Neotron 32 is the baby of the Neotron range. As the name suggests, it has just 32 KiB of SRAM (along with 256 KiB of Flash ROM). The full specifications are:

* Texas Instruments TM4C123GXL Launchpad Microcontroller Board
	* Texas Instruments TM4C123 Microcontroller
		* ARM Cortex-M4F Core @ 80 MHz
		* 32 KiB On-Chip SRAM
		* 256 KiB On-Chip Flash
	* USB Debug/Programming interface (via USB micro-B socket)
	* USB micro-AB On-The-Go Interface
* 800x600 @ 60Hz 8 colour VGA output (via standard 15-pin VGA connector)
* One or Two Channel PWM Audio Output, with three tone generators and four waveforms (via 3.5mm stereo jack)
* 80x36 green/black hi-res text mode
* 48x36 eight-colour text mode
* Two built-in 8x8 fonts: MS-DOS Code Page 850 and Teletext
* 388x288 cell-colour graphics mode
* 192x144 eight-colour graphics mode
* SD card slot
* IBM PC Parallel Port (via 2x13 pin 0.1" header, or optional DB25-F connector)
* RS232 Serial Port (via 2x5 pin 0.1" header, or optional DE9-M connector)
* Optional ESP01 WiFi module (via 2x3 0.1" socket)
* MIDI In and Out (via two 6-pin DIN sockets)
* 2x Neotron Expansion Slots (two 2x6 pin 0.1" headers)
* 2x Atari/SEGA 9-pin Joystick Ports (via two DE9-M sockets)
* PS/2 Keyboard and Mouse (via two 6-pin mini-DIN sockets)

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
