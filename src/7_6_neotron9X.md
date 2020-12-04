# Neotron 9X

If the Neotron 32 is like a BBC Micro, the Neotron 9X is like an Acorn RISC PC. It uses the Microchip SAM9X system-in-package, which bundles DDR SDRAM alongside the SoC in the same package. This hugely simplifies PCB routing as compared to the Neotron 500.

As it runs an ARM9 core and has 64 MiB of DDR2 SDRAM, it would run RISC OS, or even Linux quite happily. In future, we may develop a RISC OS or Windows 95 like shell which sits above the Neotron OS. Performance wise, this system is broadly similar to a mid-1990s Pentium 200 MMX.

Because the ARM9 will not run Cortex-M3/4/7 (i.e. ARMv7-M or ARMv7E-M) machine code, it is incompatible with standard Neotron applications, and they will need to be recompiled for the ARM9 (i.e. ARMv5TE architecture). You could make an argument, therefore, that the Neotron 9X isn't a Neotron at all.

## Features

* **CPU:** Microchip SAM9X60D5M System-in-Package
  * **Processor Core**: 600 MHz ARM926EJ-S
    * 32 KiB Instruction Cache
    * 32 KiB Data Cache
    * Memory Management Unit
  * **RAM:** 64 MiB on-package DDR2
    * Also has 64 KiB on-die SRAM
  * **ROM:** 256 KiB Serial Flash
    * Can load BIOS from "boot.bin" on SD-Card, like an Amiga 1000 loads Kickstart from floppy disk
    * Can also load BIOS into RAM from Serial Flash
  * **GPU:** Microchip SAM9 LCD Controller
    * 800x600 resolution (chip supports up to 1024x768 resolution maximum)
    * Typically 16 or 256 colour pallete mode to save Video RAM
    * Optional 16-bit mode for improved colour depth
    * Three hardware overlays with transparency
    * Hardware scaling
  * **GPU RAM:** Shared with main SDRAM
* **Video Output:** SVGA
  * Nominally 800x600 at 60 Hz
  * Other resolutions TBD (pixel clock is limited to integer divisors of system clock)
  * Texas Instruments THS7316 Video Amplifier used to drive 75 ohm VGA output
  * Triple 6-bit 0.1% R2R DACs, to give max 262,144 (2^18) colours
* **Storage:** SD Card slot, supports FAT16/FAT32 and MS-DOS partition tables
* **USB:** 2x external USB 2.0 Full-speed Type-A Host ports (with a header for one more)
* **Audio:** 16-bit 48 kHz stereo
  * Wolfson WM8731 Audio CODEC
  * 3.5mm stereo Line Out
  * 3.5mm stereo Amplified headphone output
  * Digital volume control
* **Keyboard/Mouse:** 2x PS/2 Ports (2x 6-pin mini-DIN)
* **Serial**: RS-232 on 10-pin 2.54mm header suitable for DE-9 plug on IDC ribbon
* **Ethernet**: Standard 100base-TX RJ45 port.
* **Expansion**: 4x internal 2.54mm 2x6 headers carrying power, SPI, I2C and a dedicated IRQ line
* **RTC**: Coin-cell battery backup for the main CPU's RTC.
* **Mechanical Form Factor:** microATX
