# Neotron Hardware

The Netron OS is intended to be binary-portable across any machine with a Neotron BIOS. How that works out in practice remains to be seen, but that's the goal. As the BIOS API, example BIOS implementations and the OS itself are entirely open, anyone can build their own Neotron-compatible system, using any ARMv7-M compatible processor. However, there will naturally be value in having some commonality between Neotron systems, in order to reduce development effort.

Another important factor in the Neotron Hardware is that, like the software, it should be fully Open. That is, anyone should be able to pick up a Neotron System and understand it right down to the component parts (and why those component parts were chosen). It isn't sufficient to simply post some Gerbers on GitHub and call it done - any more than it would be to post some compiled firwmware binaries. Making an Open System means making a system where people are invited and encouraged to learn everything there is to know about that system.

## Components

A Neotron System will naturally be comprised of:

* A microcontroller, also known as a System On Chip (SoC) - this will contain one or more Processor Cores, RAM, possibly Flash, and some internal peripherals
	* The processor core will be compatible with the [ARMv6-M ISA]\*.
* External peripherals - such as UART level shifters, audio amplifiers, SD cards
* A baseboard (tradtionally known as a _motherboard_) - a PCB which holds the SoC, the external peripherals, and any connectors

\* _The ARMv6-M ISA (as implemented by the Cortex-M0/M0+ processore cores) has fewer instructions, and so is easier to understand. However, ARMv6-M has no Compare-and-Swap Atomic instructions, meaning that sharing data between the main thread and interrupts can be more complicated in some situations. It is TBD as to whether this is too much of a burden for the OS to carry and we may need to raise the minimum to [ARMv7-M ISA], i.e. a Cortex-M3 or higher._

[ARMv6-M ISA]: https://static.docs.arm.com/ddi0419/d/DDI0419D_armv6m_arm.pdf
[ARMv7-M ISA]: https://static.docs.arm.com/ddi0403/e/DDI0403E_B_armv7m_arm.pdf

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
* Has an I²S audio codec interface suitable for 16-bit stereo at 22,050 Hz or higher, or can generate analog audio
* Has a ROM bootloader which can boot from UART, USB or SD Card
    * This is so home-made systems do not require an ARM debug probe for initial programming
* Is possible to manually solder to a simple 4-layer PCB without micro-vias, i.e. one of
	* 0.8mm+ pitch ball grid array (BGA) package
	* 0.65mm+ pitch quad flip-chip (QFP) package
* Has 3.3V I/O
* Ideally takes a single 5.0V or 3.3V power rail which is internally regulated as required
    * Where multiple rails are required, we either need relaxed requirements in sequencing, or an available PMIC chip which brings the rails up in the right order
* Costs under $20 in one-off quantities
* Available from mainstream catalog vendors (e.g. Digikey, Mouser, RS, Farnell)
    * Ideally also available from the JLCPCB Parts Catalog

We've identified the following parts as meeting some or more of the above criteria:

| Manuf.       | Part Number     | Core          | Clock   | Package   | RAM (KiB) | Flash (KiB) | Price (10 off) | Notes                                                              |
| :----------- | :-------------- | :------------ | :------ | :-------- | :-------- | :---------- | :------------- | :----------------------------------------------------------------- |
| ST           | STM32H730ZBT6   | Cortex-M7     | 550 MHz | LQFP 144  | 564       | 128         | £6.97          | Great value due to small Flash. SPI SRAM support.                  |
| NXP          | IMXRT1062DVJ6A  | Cortex-M7     | 600 MHz | BGA 196   | 1024      | 0           | £10.41         | As used on Teensy 4.1. SPI SRAM support.                           |
| ST           | STM32H7A3ZIT6   | Cortex-M7     | 280 MHz | LQFP 144  | 1344      | 2048        | £11.00         | Big SRAM - might not need external RAM?                            |
| TI           | TM4C1299KCZADI3 | Cortex-M4     | 120 MHz | VFBGA 212 | 256       | 512         | £11.78         | Poor value, but same family as Neotron-32                          |
| ST           | STM32H743ZGT6   | Cortex-M7     | 480 MHz | LQFP 144  | 1024      | 1024        | £13.75         | Cheapest 1 MiB H7. No SPI SRAM support - will require SDRAM        |
| Raspberry Pi | RP2040          | 2x Cortex-M0+ | 133 MHz | QFN 56    | 256       | External    | £0.85          | Incredibly cheap. Cortex-M0 only supports ARMv6-M. No RAM upgrades |

Prices are as of late 2020, before the great "chip shortage" of 2021.

It is worth also considering the Lattice range of small, low-cost FPGAs, and loading an RISC-V soft-core like the VexRiscv.

