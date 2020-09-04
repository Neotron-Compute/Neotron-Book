# Introduction

## What is Neotron?

Neotron is an attempt to make computers simple again, whilst also taking advantage of the very latest in programming language development. We are saddened by chat clients that require multi-Gigabyte installs, and systems with hundreds of millions of lines of source code that no one person could ever hope to understand. We want to build a machine that is sized for an individual to comprehend, not a trillion-dollar corporation.

If you want a pithy sound-bite, it's like CP/M for tiny ARM microcontrollers, but written in Rust.

## Tell me more...

Neotron is based around four fundamental components.

* A standardised OS interface, for portable Applications to call. This provides APIs for reading/writing files, accessing devices, writing to the screen, playing audio, etc.
* A standardised BIOS interface, for the Operating System to call. The BIOS abstracts the specific hardware implementation of the Video, Audio, UART, SPI, I2C, GPIO, Disk Drive, Parallel Printer, Keyboard and Mouse interfaces. By using the BIOS, we should be able to run the *exact same* Neotron OS on a variety of different microcontrollers.
* Use of the Rust Programming Language to write as much of the software as possible (we avoid raw assembler as much as possible, but we're happy to port existing applications that are written in C even if we avoid that language in the system software).
* The ARM Thumb-v7M instruction set (as supported by ARM Cortex-M based microcontrollers from the M3 and up). This is what allow us to run the same programs on microcontrollers from different vendors.

Looking back at classic home computers of the 1980s and early 1990s though, we see systems that were (and still are) simple enough to understand, or even - with time - to learn to master. Here's a rough comparison with just a few of the classic systems we have taken as inspiration:

| Feature          | Neotron 32    | Neotron 320ST | IBM PC 5150  | BBC Micro Model-B | Commodore 64 | Macintosh 128K | Amstrad PCW8256 | Amiga 1000   | Acorn Archimedes A305 |
|:-----------------|:--------------|:--------------|:-------------|:------------------|:-------------|:---------------|:----------------|:-------------|:----------------------|
| Launch Year      | 2020          | 2020          | 1981         | 1981              | 1982         | 1984           | 1985            | 1985         | 1987                  |
| Instruction Set  | ARMv7E-M      | ARMv7E-M      | Intel x86    | MOS 6502          | MOS 6502     | Motorola 68k   | Zilog Z80       | Motorola 68k | ARM v2                |
| CPU              | Cortex-M4     | Cortex-M7     | 8088         | 6502A             | 6510         | 68000          | Z80A            | 68000        | ARM2                  |
| Clock Speed      | 80 MHz        | 216 MHz       | 4.77 MHz     | 2 MHz             | 1 MHz        | 8 MHz          | 4 MHz           | 8 MHz        | 8 MHz                 |
| Low Level OS     | Neotron BIOS  | Neotron BIOS  | IBM BIOS     | Acorn MOS         | KERNAL       | Toolbox        | XBIOS           | Kickstart    | RISC OS               |
| High Level OS    | Neotron OS    | Neotron OS    | PC-DOS 1.0   | Acorn MOS         | N/A          | System         | CP/M Plus       | AmigaDOS     | RISC OS               |
| Shell            | Neotron Shell | Neotron Shell | COMMAND.COM  | BBC BASIC         | BASIC v2     | Finder         | CP/M CCP        | Workbench    | RISC OS Desktop       |
| Primary Language | Rust          | Rust          | Assembly / C | Assembly          | Assembly     | Object Pascal  | Assembly        | BCPL / C     | Assembly / BBC BASIC  |
| ROM              | 256K          | 1024K         | 8K           | 32K               | 16K          | 64K            | 256 bytes       | 256K         | 512K                  |
| RAM              | 32K           | 8512K*        | 16K to 256K  | 32K               | 64K          | 128K           | 64K             | 256K         | 512K                  |
| Self-hosting     | No            | No            | Yes          | Yes               | No           | Yes            | Yes             | Yes          | Yes                   |

\* Made up of 320K of internal SRAM and 8192K of external SDRAM

The [IBM PC](https://en.wikipedia.org/wiki/IBM_Personal_Computer) BIOS was stored in a ROM chip on the motherboard. It provided a certain level of hardware abstraction, with APIs for writing to the screen, setting the video mode and reading/writing from block devices such as floppy drives. The BIOS initialised the hardware, loaded the first sector of a chosen block device into RAM and then executed the code contained within. This was the Boot Sector and contained enough code to load the rest of the Operating System. PC-DOS made use of BIOS APIs, but often games would bypass both MS-DOS and the BIOS and access hardware directly. Famously, Microsoft was able to sell copies of PC-DOS (relabelled as MS-DOS) to manufacturers of 'PC compatibles', provided they had a BIOS ROM which offered the same (reverse-engineered) API as the IBM BIOS.

The [BBC Micro](https://en.wikipedia.org/wiki/BBC_Micro) had a very advanced Operating System for the time, known as the Acorn Machine Operating System (MOS). You could add extra ROMs to the system, such as the Disk Filing System (DFS). BBC BASIC was also very advanced for the time, including a built-in 6502 assembler.

The [Commodore 64](https://en.wikipedia.org/wiki/Commodore_64) contained two 8 KiB ROM chips - one contained the OS (called the KERNAL) and one contained a rebadged Microsoft 6502 BASIC. The KERNAL was very low level and Microsoft BASIC didn't include any commands to produce graphics or sound - developers were expected to interact with memory mapped hardware directly. The C64 KERNAL could be assembled using PET RESIDENT ASSEMBLER on a Commodore PET, but the [Microsoft BASIC source code](https://www.pagetable.com/?p=774) was written on a PDP-10 using the MACRO-10 assembler.

The original [Apple Macintosh](https://en.wikipedia.org/wiki/Macintosh_128K) had a ROM on the motherboard which initialised the hardware and drew a graphical image on the screen indicating that the user should insert the System disk. The ROM then loaded the OS (known as *System* up to version 7, and *Mac OS* for versions 8 and 9) into RAM, along with a graphical desktop called Finder. Applications on the Macintosh (including the Finder) made use of a number of graphical routines stored in the ROM, known as the *Macintosh Toolbox*.

The [Amstrad PCW8256](https://en.wikipedia.org/wiki/Amstrad_PCW) was similar to the Amiga 1000 in that the main BIOS was loaded from floppy disk. Going further than the Amiga though, the PCW didn't include a ROM chip at all. Instead the 256 bytes of bootstrap code were loaded from the microcontroller responsible for managing the printer interface. The bootstrap loaded the so-called *XBIOS* from floppy disk into RAM. XBIOS then initialised the hardware and loaded CP/M Plus (also known as CP/M 3.0). The shell was the familiar CP/M Command Console Processor (CCP), and both the OS and the CCP were the same across a wide range of CP/M machines from many different vendors, with the BIOS providing the hardware abstraction layer.

The [Amiga 1000](https://en.wikipedia.org/wiki/Amiga_1000) hardware didn't include much in the way of a ROM at all. A basic bootstrap program stored in ROM was able to load most of the low-level OS (known as *Kickstart*) into a special write-only area of RAM. Kickstart could then load either the particular game being played, or AmigaDOS and a graphical shell known as Workbench. On later machines (such as the A500), the Kickstart was stored in ROM.

The [Acorn Archimedes](https://en.wikipedia.org/wiki/Acorn_Archimedes) was first shipped with an OS in ROM called Arthur, but this was soon replaced with the more familiar RISC OS 2 (and later RISC OS 3). The entire OS - bootstrap, HAL, filesystem and GUI - was one on ROM chip, meaning it booted up very quickly, without needing to read from any sort of disc.

The key feature for many these systems (apart from the Commodore 64), was portability across other machines in the family (or, indeed, third-party clones). The PC, the Amiga, the Macintosh and CP/M all provided a *platform*, and if you respected certain limits when writing your software, that software was then portable across all the systems which provided that platform. For Neotron, this is a key concept - because microcontrollers vary so wildly, we have a BIOS that implements hardware abstraction, just like with the PC or CP/M machines. We then run a standard OS (indeed, the same OS image should work on every Neotron machine - more-or-less), either from ROM or loaded into RAM by the BIOS. On top of the OS is the text-mode Neotron Shell (like CP/M's CCP, or MS-DOS's COMMAND.COM). Booting to a graphical desktop would be nice, but given the Neotron 32 only has 32 KiB of RAM (barely enough for a low resolution frame-buffer), text mode is the default. Ultimately, a Neotron 32 should be able to 'compete' on a functional level with an un-expanded IBM PC 5150 or a BBC Microcomputer Model-B. The Neotron 320ST meanwhile, should be more like a Macintosh, Archimedes or Amiga in terms of functionality and in time we do hope to develop a GUI for this system.

As with the CP/M and MS-DOS machines, we hope that in the future there will be a wide range of machines in the Neotron family. These Neotron systems won't generally aim to be super cheap, although we will tend to target commodity microcontrollers that only cost circa $10 or less, so they shouldn't be outrageously expensive. They will, however, aim to be usable computers that can do interesting things, while being simple enough to understand in their entirety and open enough to allow you to gain that understanding.

## What can I do with it?

You can do what you can do with most 1980s home computers:

* Type things on the keyboard
* Use a joystick or a mouse
* Manage files on disk
* Type in your own programs
* Load programs from external media (although we have SD Cards rather than Floppy Discs)
* Put text and graphics on the screen
* Make various beepy noises
* Connect to external hardware, such as Printers, Ethernet interfaces, or WiFi modules.

But most importantly, you can learn the fundamentals of what it takes for a computer to be a *computer*. And you can study the source code and hardware schematics required to make that happen. You can even take the designs and go off and produce your own version - maybe to add a specific interface for a particular peripheral that interests you - all with no licences fees to pay.

## Is this a good idea?

It's certainly not a profitable idea. It's not even likely to be an efficient use of our time and resources. But it is proving to be a fun and educational project to work on, and we hope others find it useful and/or enjoyable too.

## But what about ${EXISTING_PRODUCT}?

### Raspberry Pi

Yes, you can buy a Raspberry Pi for a little as $5 - and it's a wonderful product that has done arguably more than anything since the BBC Microcomputer in terms of putting hardware you can actually program into the hands of young people. But, the downside is that you get a big, complicated piece of silicon, a proprietary GPU subsystem, 25 million lines of kernel source code, and even more in a user-land which has parts dating back over 30 years. No one person could hope to understand all of that! While we could probably port the Neotron OS to the Raspberry Pi, we'd have to treat the video sub-system as a black-box (like RISC OS does) and the complexity and lack of documentation of that subsystem presents a big barrier to fully understanding the system. Not to mention the the fact that only partial schematics are available, there are no PCB designs available, and there's no general availability on the main processor. It does what it needs to do for its target market quite admirably - but it's not a Neotron.

### x86 based systems, like the original IBM PC

You could build yourself a clone of the original IBM PC ([kits do exist](https://monotech.fwscart.com/)), but it involves a lot of components that are very difficult to obtain nearly 40 years since that design first came out. Rust is also designed around a flat memory model (rather than the segmentation:offset model used by the 16-bit 8086), and that means using at least a 32-bit [Intel 80386](https://hackaday.com/2015/11/17/a-modern-386-development-board/). And, of course, these are plain CPUs rather than microcontrollers, so you'll need to find a 'chipset' containing interrupt controller, DRAM memory controller and timers; as well as separate video, sound and IO sub-systems. Our System-on-Chip based design simplifies the motherboard design and reduces cost, and unlike with modern x86 SoCs, we don't have to deal with complicated legacy subsystems such as UEFI, ACPI or PCI.

### 6502 based systems (like the PE6502 and Commander X16)

You could build a 6502 based system, and many people do, but the Rust programming language doesn't (and probably never will) support an 8-bit CPU with only 256 bytes of stack.

### 68000 based systems

At the present time, there is no 68k backend in LLVM. If there was, a 68000 system would certainly make an interesting alternative to Neotron.

### PowerPC based systems

LLVM does have a PowerPC backend, but unfortunately the only PowerPC CPU or SoC available in a hand-solderable package (i.e. not a BGA) is an aging 200 MHz 603e from Microchip.

### RISC-V based systems

There is an increasing number of RISC-V based microcontrollers with very small SRAMs, matched by an increasing number of multi-core 64-bit SoCs designed to run Linux from a huge bank of DDR3. At the current time, however, there isn't much in the middle.

A Neotron system built around an open-source RISC-V code run from an FPGA would be an interesting way to increase the openness of the system and remains a future possibility.

### SPARC based systems

The Gaisler LEON3 is available in VHDL form under the GPL, along with a number of other components required to make a full SoC. The LEON3 implements the SPARC V8 ISA, support for which is available in the Rust compiler. However, this would require an FPGA based system and it's unknown whether the full GRLIB SoC would fit into any of the FPGAs supported by the various open-source FPGA synthesis tools (such as the Lattice iCE40).

### Atmel AVR based systems

Many people have built Atmel AVR based systems, but those CPUs are fairly limited in terms of performance and tend to need to be programmed in raw assembler. There is a Rust AVR port, but it's only in alpha state at the moment.

## What is this book?

This book describes the Neotron system - from the hardware schematics, all the way up to how to write applications for the system. It is intended to be the canonical reference, although where the source code differs from what is described in this book, we reserve the right to be pragmatic and change either to match the other (or leave them in disagreement...). This book also represents our latest thoughts on a particular topic, and the source code may lag behind while we get around to the implementation. We endeavour to note where this is the case.

## Versioning

Each component has a semantic version number - `major.minor.patch`. The assocation is:

* The BIOS will require a specific hardware version.
* The OS will require a specific BIOS version.
* The Shell and Applications will require a specific OS version.

## Open Source and Commercial Sales

Neotron is designed to be open - the user must have free reign to inspect the source code and the schematics, and change them to suit their needs. To this end, the main BIOS and OS implementations are licensed under the [GNU Public Licence v3](https://www.gnu.org/licenses/gpl-3.0.en.html) or any later version. To encourage the adoption of Rust for embedded development, most of the library crates developed for this project are licensed under both the [MIT] and [Apache 2.0] licences, just like the Rust compiler itself. We do intend to sell a range of kits and pre-built PCBs, but you can (and should!) take our designs as inspiration and put your own spin on them.

This book is licensed under Creative Commons [CC-BY-SA 4.0]. Any source examples in this book may also be used under the [MIT] or [Apache 2.0] licences.

[MIT]:https://opensource.org/licenses/MIT
[Apache 2.0]:https://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA 4.0]:https://creativecommons.org/licenses/by-sa/4.0/legalcode

## Credits

Neotron started as a project called 'Monotron', by [@thejpster]. This was based on the work of the [Rust Embedded Working Group](https://github.com/rust-embedded), and inspiration from the following (in no particular order):

[@thejpster]:https://github.com/thejpster

* [Colour Maximite](http://geoffg.net/maximite.html) - A modern PIC32 based single-board home computer.
* [PE6502](http://putnamelectronics.com/products.html) - A modern 6502 based single-board home computer.
* [Gigatron](https://gigatron.io/) - A single-board home computer, with a custom CPU built from TTL logic chips.
* [Commander X16](http://commanderx16.com/) - A single-board home computer, with a 6502 and an FPGA based video chip.
* [Craft](http://www.linusakesson.net/scene/craft/) - Demonstrates bit-bashing VGA from an AtMega88
* [AVR-ISA](http://tinyvga.com/avr-isa-vga) - Demonstrates driving an ISA bus (and an ISA VGA card) driven from an AtMega128
* The Commodore 64C - [@thejpster]'s first home computer
* The BBC Microcomputer range (the Model B through to the Archimedes A3000) - as used in UK schools in the 1980s
* The Amstrad range of CP/M machines, particularly the PCW9512

After meeting [@IGBC](https://github.com/IGBC) at Oxidize 2019, discussions started around a next-generation system with higher performance. This led to the Neotron 1000, and then to the whole Neotron concept.
