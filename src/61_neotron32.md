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
   |    |  | | |
   |    |  | | +---<UART> Neotron I/O Controller
   |    |  | +---<UART> Wi-Fi Modem
   |    |  +---<UART> RS-232 Interface
   |  <I2C>
   |    |
 <SPI>  +-- Real-Time Clock
   |
   +--- SD Card
   +--- GPIO Port / Parallel Port
```

### Video

Video is generated on the Neotron-32 by driving three SPI interfaces in a synchronised fashion - one generating a stream of red pixels, one a stream of green pixels and one a stream of green pixels. There is also a Horizontal Sync line driven from a Timer, which is synchronised to the pixel outputs, and a Vertical Sync line which is a GPIO line driven in the Horizontal Scan-line Interrupt Handler.

With a CPU clock of 80 MHz, the SPI ports can be run at either 40 MHz or 20 MHz. 40 MHz generates an 800x600 pixel image with a 60 Hz refresh rate, and 20 MHz generates a 400x600 pixel image with a 60 Hz refresh rate. To counter the 'stretch' effect of the latter mode, we only generate 300 lines of video and clock each out one twice, giving an effective 400x300 resolution.

The hardware on the motherboard simply consists of some resistors which, in conjunction with the 75 ohm input impedance of a VGA monitor, drops the 3.3V signal from the SoC down to the 0.7V maximum given in the VGA standard. With the three channels (R, G and B) being either on or off, we get 8 familiar colour combinations:

* Black
* Red
* Green
* Yellow
* Blue
* Magenta
* Cyan
* White

As the SoC can only just push out the 8 mA required to drive a VGA signal into 75 ohms, later Neotron systems are likely to use a small active VGA/RGB video filter to act as a line driver.

### Audio

The Neotron 32 generates mono pulse-width-modulated (PWM) audio with 8-bit centre-aligned pulses at a frequency of four times the horizontal video scan rate (giving roughly 151,500 samples per second). At the end of each scan-line, a new 8-bit PCM sample is generated by summing the outputs of three basic wavetable synthesisers. That PCM sample is fed to the PWM hardware at the start of the next scan line (to reduce jitter) and it gets played out four times during that scan line, giving an effective sample rate of just under 38 kHz. The hardware then consists of a simple low-pass filter to remove the PWM carrier frequency, and a headphone jack. As the SoC can't drive a large amount of current, amplified speakers are required.

Each of the three channels on the wavetable synthesiser can produce a square, sawtooth or sine waveforms, or random noise, at a selected frequency. There is also a rudimentary volume control for each channel.

### I²C Devices

#### Real-Time Clock

An MCP7940N battery-backed real-time clock chip keeps track of calendar time when the system is powered off, and also offers 64 bytes of battery backed SRAM for system settings. Like on an IBM PC-compatible, removing the battery will "reset" the system settings (on a PC this is known as *clearing the CMOS*, for various historical reasons).

### SPI Devices

#### SD Card Slot

The SD Card Slot takes microSD cards (frustratingly, full size SD card slots don't really fit on the front edge with the Joystick ports, because of the placement of the PCB/case screw holes). These are connected up using the *SPI Compatibility* mode built into all SD and SDHC memory cards. No interrupt line is required - the SD card responds only when polled and gives out repeated `0xFF` bytes when busy completing an operation.

#### GPIO I/O Expander / Printer Port driver

The MCP23S17 I/O Expander sits on the SPI bus, with a dedicated IRQ line. This can be used to drive a PC-style 25-pin Parallel Printer Port, or it can be used as a general-purpose 16-bit GPIO interface. Running at up to 10 MHz, all 16-bits can be written to in 4 bytes, which takes 32 microseconds or 256 clock cycles. As the Parallel Port strobe pin is driven separately from an SoC GPIO line, and with the MCP23S17 able to generate an Interrupt when the remote device raises the ACK line, this interface should be sufficient for around 150 KiB/sec.

### UART Devices

#### ESP-01 Wi-Fi Modem

The Espressif ESP-01 is a Wi-Fi Modem based on the Espressif ESP8266 Wi-Fi chipset. It has an on-board TCP/IP stack, and speaks AT commands. To the Neotron 32 SoC, it basically appears as a classic RS-232 Hayes-compatible AT modem.

#### Neotron I/O Controller

Responsible for the PS/2 interfaces and the two Joystick ports. It consists of an AtMega 328 microcontroller with custom firmware, and speaks a bespoke *HID over UART* protocol. The Joystick ports can be either standard Atari joysticks, or SEGA MegaDrive / Genesis 3-button pads. With some software work, 6-button pads could possibly also be supported.

#### MIDI

MIDI is basically an opto-isolated UART based protocol running at a fixed 31,250 bps. The MIDI In port is connected to UART RX via an opto-isolator, and the MIDI Out port is driven from UART TX via a 5V level shifter (two back-to-back inverters from a hex-inverter).

#### RS-232 Interface

A four-wire (`TX`, `RX`, `RTS`, `CTS`) RS-232 interface is provided via a 10-pin 0.1" boxed header. These headers are used on old ISA Serial Port expansion cards for when the DE-9 header didn't fit on the edge of the card and had to be moved to a second bracket, one slot over.

The `DTR`, `DSR`, `DCD` and `RI` pins are not connected.
