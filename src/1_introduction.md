# Introduction

## What is Neotron?

Neotron is an attempt to make computers simple again, whilst also taking advantage of the very latest in programming language development. It is based around three simple concepts:

* The ARM Thumb-v7M instruction set (as supported by the ARM Cortex-M3, -M4, -M7 and -M33 processor cores).
* A standardised OS interface, which uses a standardised BIOS interface (just like both CP/M and the IBM PC)
* Use of the Rust Programming Language to write as much of the software as possible (we avoid raw assembler as much as possible, but we're happy to port existing applications that are written in C even if we avoid that language in the system software).

Neotron systems don't aim to be super cheap, although we tend to target commodity microcontrollers that only cost circa $10 or less so they shouldn't be outrageously expensive. They do, however, aim to be usable computers that can do interesting things, while being simple enough to understand in their entirety and open enough to allow you to gain that understanding. Looking back at classic home computers of the 1980s and early 1990s though, we see systems that were (and still are) simple enough to understand, or even - with time - learn to master. Systems such as the ZX Spectrum, Commodore 64 or Acorn Archimedes. That is where we take our inspiration.

## But what about ${EXISTING_PRODUCT}?

### Raspberry Pi

Yes, you can buy a Raspberry Pi for a little as $5, but you get a big, complicated piece of silicon, a proprietary GPU subsystem, 25 million lines of kernel source code, and even more in a user-land which has parts dating back over 30 years. No one person could hope to understand all of that!

### x86 based systems, like the original IBM PC

You could build yourself a clone of the original IBM PC (kits do exist), but it involves a lot of components that are very difficult to obtain nearly 40 years since that design first came out. Rust is also designed around a flat memory model, and that means using at least an 80386.

### 6502 based systems (like the PE6502 and Commander X16)

You could build a 6502 based system, and many people do, but the Rust programming language doesn't (and probably never will) support an 8-bit CPU with only 256 bytes of stack.

### Atmel AVR based systems

Many people have built Atmel AVR based systems, but those CPUs are fairly limited in terms of performance and tend to need to be programmed in raw assembler. There is a Rust AVR port, but it's only in alpha state at the moment.

## What is this book?

This book describes the Neotron system - from the hardware schematics, all the way up to how to write applications for the system. It is intended to be the canonical reference, although where the source code differs from what is described in this book, we reserve the right to be pragmatic and change either to match the other (or leave them in disagreement...). This book also represents our latest thoughts on a particular topic, and the source code may lag behind while we get around to the implementation. We endeavour to note where this is the case.

## Versioning

Each component has a semantic version number - `major.minor.patch`. The assocation is:

* The BIOS will require a specific hardware version.
* The OS will require a specific BIOS version.
* The Shell and Application will require a specific OS version.

## Open Source

Neotron is designed to be open - the user must have free reign to inspect the source code and the schematics, and change them to suit their needs. To this end, the main BIOS and OS implementations are licensed under the [GNU Public Licence v3](https://www.gnu.org/licenses/gpl-3.0.en.html) or any later version. To encourage the adoption of Rust for embedded development, most of the library crates developed for this project are licensed under both the [MIT] and [Apache 2.0] licences, just like the Rust compiler itself.

This book is licensed under Creative Commons [CC-BY-SA 4.0]. Any source examples in this book may also be used under the [MIT] or [Apache 2.0] licences.

[MIT]:https://opensource.org/licenses/MIT
[Apache 2.0]:https://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA 4.0]:https://creativecommons.org/licenses/by-sa/4.0/legalcode

## Credits

Neotron started as a project called 'Monotron', by [@thejpster](https://github.com/thejpster). This was based on the work of the [Rust Embedded Working Group](https://github.com/rust-embedded), and inspiration from the following (in no particular order):

* [Colour Maximite](http://geoffg.net/maximite.html) - A PIC32 based system.
* [PE6502](http://putnamelectronics.com/products.html) - A 6502 based system.
* [Gigatron](https://gigatron.io/) - A system with a custom CPU built from TTL logic chips.
* [Commander X16](http://commanderx16.com/) - A 6502/65816 based system with an FPGA based video chip.
* [Craft](http://www.linusakesson.net/scene/craft/) - VGA bitbashed from an AtMega88
* [AVR-ISA](http://tinyvga.com/avr-isa-vga) - An ISA bus driven from an AtMega128

After meeting [@IGBC](https://github.com/IGBC) at Oxidize 2019, discussions started around a next-generation system with higher performance. This led to the Neotron 1000, and then to the whole Neotron concept.
