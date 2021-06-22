# The System On Chip

## Picking a SoC

When it comes to picking an SoC for a new Neotron model, the following are important criteria:

### Mandatory (MUST):

* Use an ARMv6-M compatible core (i.e. any Arm Cortex-M)
* Generate digital RGB video at 25.175 MHz or 40 MHz, or some integer fraction thereof
    * Anything between 1-bit and 24-bits per pixel is acceptable.
    * 40 MHz gives you 800x600 (and 20 MHz is 400x600, etc)
    * 25.175 MHz gives you 640x480 (and 12.588 MHz is 320x480, etc)
    * If you aren't performance sensitive (i.e. playing games), you could have off-chip video support implemented on a second processor, with a fast communications link (e.g. SPI) between them
* Have a four-wire UART, or an ARM Serial Wire Debug interface, for debug logging
* Have at least 256 KiB of memory for BIOS / OS code (in Flash or RAM) 
* Have enough RAM to support the desired video modes (80x25 text mode is 4000 bytes, 800x600 @ 8bpp needs around 470 KiB)
* Have at least 64 KiB of free application RAM (ideally 256 KiB or more)
* Have an SPI Controller interface which can run at least 25 MHz
* Have 7 or more chip-select output pins
	* If you are limited on pins, you can use a `74LS138` 3:8 decoder for the chip-selects
* Have 8 interrupt input pins
	* If you are limited on pins, you can use an `MCP23S17` to mux 16 interrupts inputs down to 1
* Have an I²C Controller interface which can run at 400 kHz
* Have either an I²S audio codec interface, or stereo PWM audio output pins
* Have a ROM bootloader which can boot from UART, USB or SD Card
	* This is so home-made systems do not require an ARM debug probe for initial programming
* Have a footprint that is easy to assemble on a simple 4-layer PCB without micro-vias, i.e. one of
	* 0.8mm pitch ball grid array (BGA) package
	* 0.5mm+ pitch quad flip-chip package (QFP), shrink small-outline package (SSOP) or small-outline IC (SOIC)
* Have 3.3V I/O
* Cost under $20 in one-off quantities
* Be available from mainstream catalog vendors (e.g. Digikey)

### Desirable (SHOULD)

* Use an ARMv7-M compatible core (i.e. Arm Cortex-M3, Cortex-M4, Cortex-M7 or Cortex-M33)
* Have at least one USB 2.0 Host or OTG port (High-Speed or Full-Speed)
* Has a footprint that is easy to hand-assemble at home:
	* 0.65mm pitch QFP or SSOP
	* 1.27mm SOIC package
* Takes a 3.3V power input which is internally regulated down to the desired core voltage
    * Where multiple rails are required, we either need relaxed requirements in sequencing, or an available PMIC chip which brings the rails up in the right order
* Be available from the JLCPCB Parts Catalog
* Supports additional external RAM (either SRAM, SDRAM or QSPI HyperRAM)

We've identified the following parts as meeting some or more of the above criteria:

| Manuf. | Part Number     | Core      | Clock (MHz) | Package   | RAM (K) | Flash (K) | Price (10 off) | Notes                                                                 |
|:-------|:----------------|:----------|:------------|:----------|:--------|:----------|:---------------|:----------------------------------------------------------------------|
| ST     | STM32L552ZET6   | Cortex-M33| 110         | LQFP 144  | 256     | 512       | £6.65          | Supports HyperRAM, and TrustZone, and is low-cost                     |
| ST     | STM32H730ZBT6   | Cortex-M7 | 550         | LQFP 144  | 564     | 128       | £6.97          | Great value due to small Flash. SPI SRAM support.                     |
| NXP    | IMXRT1062DVJ6A  | Cortex-M7 | 600         | BGA 196   | 1024    | 0         | £10.41         | As used on Teensy 4.1. HyperRAM support.                              |
| ST     | STM32H7A3ZIT6   | Cortex-M7 | 280         | LQFP 144  | 1344    | 2048      | £11.00         | Big SRAM - might not need external RAM?                               |
| TI     | TM4C1299KCZADI3 | Cortex-M4 | 120         | VFBGA 212 | 256     | 512       | £11.78         | Poor value, but same family as Neotron-32                             |
| ST     | STM32H743ZGT6   | Cortex-M7 | 480         | LQFP 144  | 1024    | 1024      | £13.75         | Cheapest H7 with 1 MiB SARM. No SPI SRAM support - will require SDRAM |

It is worth also considering the Lattice range of small, low-cost FPGAs, and loading an RISC-V soft-core like the VexRiscv.
