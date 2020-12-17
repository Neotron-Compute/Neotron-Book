# Neotron Hardware

The Netron OS is intended to be binary-portable across any machine with a Neotron BIOS. How that works out in practice remains to be seen, but that's the goal. As the BIOS API, example BIOS implementations and the OS itself are entirely open, anyone can build their own Neotron-compatible system, using any ARMv7-M compatible processor.

## Example systems

To demonstrate the practicality of making a portable OS for ARMv7-M, we are in the process of producing a number of Neotron compatible systems. Note that some (or all) of these may never be completed, but all in-progress designs remain available under an open-licence.

* [Neotron 32](./7_1_neotron32.md) - an open-source PCB which plugs into the Texas Instruments TM4C Launchpad (a derivative of the original Montron project).
* [Neotron 340ST](./7_2_neotron340st.md) - based on an STM32F7-DISCOVERY PCB; not open-source hardware, but easily available from ST and it comes with schematics.
* [Neotron 500](./7_3_neotron500.md) - an unfinished attempt an an open-source PCB using an STM32H7 and other parts from the JLCPCB catalog so it could be built using their assembly service.
* [Neotron 600](./7_4_neotron600.md) - an open-source PCB which plugs into the Teensy 4.1.
* [Neotron 1000](./7_5_neotron1000.md) - an open-source PCB based around the STM32H7, along with a Lattice iCE40 FPGA for hardware accelerated video output.
* [Neotron 9X](./7_6_neotron9X.md) - an open-source PCB based around the Microchip SAM9X60D5M system-in-package. Possibly not a "Neotron" as not a Cortex-M chip.

## Form Factor

The original Monotron didn't have a particular form-factor - the PCB was simply made large enough to accept all the desired connectors. The Neotron 32 was upsized slightly to fit the Hammond 1598C series case, however, there were issues that meant a full-size SD card did not fit. The work-around was to use a microSD card, but all available examples proved difficult to hand-solder. An upgrade to the larger Hammond 1598D was investigated, but expansion options for the system still remained limited.

One of the nice features of the IBM PC (and its clones), and also the Apple II series, was the provision of expansion slots. These allowed extra functionality to be added at a later date - often well beyond what the original designer had envisaged at the time the main system was developed. Whilst the Hammond cases work well as a small desktop, they do lack the space for expansion cards - at least, expansion cards with their own connectors.

The alternative would be to produce a board based around the ATX standard - either full-size ATX (305x244mm), micro-ATX (244x244mm), or micro-ITX (170x170mm). These boards would include some crucial components (e.g. audio codec, VGA DAC, power supply), but move much of the other functionality off onto expansion cards (MIDI, Parallel Port, Joystick Ports, etc). Rather than use ISA-style card-edge connectors (which require the expansion card to have a harder gold finish which can survive the plugging-unplugging), we're considering using standard IDC headers. These would, however, be located such that the expansion cards would line up with standard ATX case expansion slot holes.

Unfortunately, being quite a large board means it can be quite expensive to produce. We also note that many of our target CPUs are only supplied in BGA package, which means four layers at a minimum, and the decoupling capacitors have to be fitted to the reverse of the board. This seems to place a high-bar on self-assembly at home - requiring basically a full surface-mount rework station. Instead, it seemed to make sense to locate the complex components on a carrier card (CPU, SDRAM, etc) and leave the main board to carry the through-hole connectors and simpler, lower pin-count, parts like the audio codec.

There are a number of standards we could use for the interface to the processor card:

* [Feather](https://learn.adafruit.com/adafruit-feather/feather-specification): Lots of GPIO (including I²C and SPI) but no defined interface for video, and not specific set of pins assigned for I²S.
* [MikroBus](https://www.mikroe.com/mikrobus): Designed for expansion cards, rather than processor cards. But our expansion connectors will carry broadly similar sets of pins.
* [Mikro MCU Card](https://download.mikroe.com/flyers/mcu-card-flyer-web.pdf): Carries almost all of the connections we'd like to get off a SoC (including the all-important RGB LCD interface we need for VGA output). The only downside is the use of two high density 168-pin headers which look pretty tricky to solder, and cost about £4.50 each. 
* [Raspberry Pi CM1/CM3]: This pinout is quite specific to the BCM2835 and BCM2837 SoCs used, and includes HDMI, Display Serial Interface and Camera Serial Interface. It does support USB and parallel RGB though. 
* [Raspberry Pi CM4]: This pinout is quite specific to the BCM2711, and if parallel RGB is used, there are no other GPIO pins available. Uses two 100-pin high-density headers.

In the end, it's probably easier to come up with our own standard to suit our specific requirements. The signals we need include:

* Parallel RGB video (22)
	* 18-bit RGB
	* H-Sync
	* V-Sync
	* DDC SDA
	* DDC SCL
* 2x USB 2.0 Host @ 480 Mbps (4)
	* Port 0 D+
	* Port 0 D-
	* Port 1 D+
	* Port 1 D-
* Console UART @ 115,200 bps (4)
	* TX
	* RX
	* RTS
	* CTS
* 2x PS/2 Interface (4)
	* Port 1 DAT
	* Port 1 CLK
	* Port 2 DAT
	* Port 2 CLK
* I²S Codec (5)
	* DAC Data
	* DAC Left/Right Clock
	* ADC Data
	* ADC Left/Right Clock
	* Bit Clock
* I²C @ 400 kHz (2)
    * SDA
    * SCL
* Control (10)
	* /Reset
	* /Enable
    * IRQ x8
* Power (6)
	* 2x 5.0V @ 2A max, combined
	* 2x 3.3V @ 200mA max, combined
	* 4x GND
* SPI @ 25 MHz (11)
    * COPI
    * CIPO
    * CLK
    * CS x8
* SD/MMC (6)
	* DAT0
	* DAT1
	* DAT2
	* DAT3
	* CMD
	* CLK

This requires at least 74 pins, ideally more as we probably want to include more Ground pins around the high-speed signals. There is a risk that despite this, sending high speed signals (USB 2.0 High-Speed, 40MHz+ parallel RGB) will cause significant EMC issues.

We could include the VGA DAC on the MCU card, reducing the VGA pins from 22 (18x digital RGB + 4) to 7 (3x analog video + 4), but that would then preclude the use of a parallel-RGB to DVI/HDMI encoder at a later date.

If we stick with simple 0.1" pitch pins (easy to solder), one option is two 2xN headers, like the Launchpad uses.

```
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
```

This has 80 pins, and occupies 51mm by 31mm. As you can see it would produce quite a long CPU card, and seems to waste the space between the two headers. It is, however, the approach taken by the [Waveshare CoreH743I board] used by the [Colour Maximite 2].

[Waveshare CoreH743I board]: https://www.waveshare.com/coreh743i.htm
[Colour Maximite 2]: https://geoffg.net/CMM2_Description.html

We could also try a double-square arrangement:

```
o o o o o o o o o o o o
o o o o o o o o o o o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o o o o o o o o o o o
o o o o o o o o o o o o
```

This has (11 + 9) * 4 = 80 pins, and occupies around 31mm x 31mm, with a central area of 20mm x 20mm to fit the SoC and all the decoupling capacitors. Of course, the general shape can be scaled up if required - the Intel 80386 had a 132-pin PGA package laid out as follows:


```
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
```

## Future Models

When it comes to picking an SoC for a new Neotron model, the following are important criteria:

* Uses an ARMv7E-M compatible core (i.e. Cortex-M4, Cortex-M7 or Cortex-M33)
* Can generate digital RGB video at 40 MHz (800x600) or some integer fraction
    * Anything between 3-bits and 24-bits per pixel is acceptable.
    * Ideally would also support the VGA standard 640x480@60 and 640x400@70 25.175 MHz modes as well
* Has a four-wire UART for a backup console
* Has at least 256 KiB of memory for BIOS / OS (Flash or RAM) 
* Has enough RAM to support the desired video modes (800x600, 8bpp needs around 470 KiB)
* Has at least 64 KiB of free application RAM (ideally 256 KiB or more)
* Has at least one USB 2.0 Host port
    * Ideally supports High-Speed without an external PHY
* Has an SPI Controller interface which can run at 25 MHz, along with 8 chip-select and 8 interrupt GPIO lines
* Has an I²C Controller interface which can run at 400 kHz
* Has an I²S audio codec interface, or can generate analog audio
* Has a ROM bootloader which can boot from UART, USB or SD Card
    * This is so home-made systems do not require an ARM debug probe for initial programming
* Is possible to manually solder to a simple 4-layer PCB without micro-vias, i.e. one of
	* 0.8mm+ pitch ball grid array (BGA) package
	* 0.5mm+ pitch quad flip-chip (QFP) package
* Has 3.3V I/O
* Ideally takes a 3.3V power rail which is internally regulated
    * Where multiple rails are required, we either need relaxed requirements in sequencing, or an available PMIC chip which brings the rails up in the right order
* Cost under $20 in one-off quantities
* Available from mainstream catalog vendors (e.g. Digikey)
    * Ideally also available from the JLCPCB Parts Catalog

We've identified the following parts as meeting some or more of the above criteria:

| Manuf. | Part Number     | Core      | Clock   | Package   | RAM (KiB) | Flash (KiB) | Price  | Notes                                     |
|:-------|:----------------|:----------|:--------|:----------|:----------|:------------|:-------|:------------------------------------------|
| ST     | STM32H730ZBT6   | Cortex-M7 | 550 MHz | LQFP 144  | 564       | 128         | £5.87  | Great value due to small Flash.           |
| NXP    | IMXRT1062DVJ6A  | Cortex-M7 | 600 MHz | BGA 196   | 1024      | 0           | £8.70  | As used on Teensy 4.1                     |
| ST     | STM32H743ZGT6   | Cortex-M7 | 480 MHz | LQFP 144  | 1024      | 1024        | £9.38  | Cheapest 1 MiB H7                         |
| ST     | STM32H7A3ZIT6   | Cortex-M7 | 280 MHz | LQFP 144  | 1344      | 2048        | £10.35 | Big SRAM - might not need external RAM?   |
| TI     | TM4C1299KCZADI3 | Cortex-M4 | 120 MHz | VFBGA 212 | 256       | 512         | £12.78 | Poor value, but same family as Neotron-32 |

It is worth also considering the Lattice range of small, low-cost FPGAs, and loading an RISC-V soft-core like the VexRiscv.
