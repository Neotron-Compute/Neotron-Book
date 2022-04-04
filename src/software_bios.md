# The Neotron BIOS

## Introduction

The Neotron BIOS is the hardware-abstraction layer for the Neotron Operating
System. It allows the OS to run on different hardware platforms with the minimum
number of changes.

```
+-------+-------------+
|       |             |
| Shell | Application |
|       |             |
+-------+-------------+
|                     |
|  Operating System   |
|                     |
++===================++
||                   ||
||       BIOS        ||
||                   ||
++===================++
|                     |
|    Raw Hardware     |
|                     |
+---------------------+
```

## What goes in the BIOS and what goes in the OS?

As a general rule, the BIOS should contain all the drivers for the main
system-on-chip (SoC) in that particular Neotron system. It should also
understand which pins on the SoC have been assigned to which functions. It is
therefore a function of the netlist for the main PCB, and the components fitted
to it.

If a particular function might apply to SoCs from different manufacturers, or to
an expansion card that could be inserted into multiple different models of
Neotron system, it is probably better placed in the Neotron OS.

## Interfaces

A BIOS should offer an interface upwards to the OS so that the OS can:

* Discover the hardware in the system
* Select and use various video modes:
	* Text modes
	* Bitmap graphics modes
	* Split modes
	* Layered Modes
	* Hooking video scan-line interrupts
* Read any keyboard key events
* Access (read from, write to and/or configure) any Serial/UART interfaces
* Access any I²C buses, and any devices attached to its bus
* Access any SPI buses, and any devices attached to its chip selects
* Access any USB Host Controllers, and any USB devices attached
* Access any block devices, via whatever interface they are connected to
* Read from / write from any audio FIFOs
* Configure a periodic 'system tick' timer
* Obtain / set the current calendar time
* Measure elapsed time since boot-up (e.g. in microseconds)
* Hook interrupts for the external IRQ lines
* Access any OS-specific non-volatile configuration

On top of these APIs sits the Neotron OS, which should be portable with almost
zero changes to any device with a Neotron BIOS. While you could write an
application which uses the BIOS directly, most applications will use the
higher-level APIs exposed by the Neotron OS.

## Configuration

Many Neotron sytems will have an element of user configuration to them - e.g.
which cards are fitted to which expansion slots, or whether an optional item has
been fitted to the PCB. To handle this, it is expected that the BIOS have some
kind of configuration application, and some non-volatile storage that will hold
the configuration across a power cycle. Typically this will be in the
battery-backed real-time clock, but it could equally be in an EEPROM.

## Calling a BIOS API

On ARM systems, calling a kernel API is usually done through a `SWI` or `SVC`
machine instruction. This effectively triggers an interrupt, putting the CPU
into interrupt mode where it starts executing the `SWI` exception handler.
Unfortunately, calling the `SWI` instruction and passing arguments via registers
can only be performed from assembly language, and is not supported by the
`cortex-m` crate at this time.

Monotron instead used a simple alternative - when the application was started by
the OS, the OS passed it a pointer to a structure of function pointers. You can
think of this as being like an old-fashioned jump table. When the application
wanted to get the OS to do something, the application just called the
appropriate function through the given pointer. The downside was that the
Monotron OS functions used the same stack as the application, and it was
impossible for an application to exit back to the OS unless it returned from
`main` (i.e. there was no `exit()` function you could call). The Neotron BIOS
accepts these downsides in the same of simplicity, and takes the Monotron
approach - at least for the BIOS-OS ABI.

The BIOS is responsible for starting the system and performing hardware
initialisation. It will also search for the OS, either in Flash memory, on disk, or
perhaps even loaded over a UART. The OS then has its initialisation function
called, to which the structure of BIOS function calls is passed. The
initialisation function is specified as a function pointer stored at a
specific offset (most likely the first four bytes of the OS image). To avoid
undefined behaviour, that initialisation function should memset the `.bss`
section to all-zeroes and, if necessary, initialise the `.data` section using
values stored in Flash memory.

Most of the BIOS APIs will be 'blocking' - that is, the function call will not
return until the BIOS has completed the operation (e.g. read the sector from
disk, or written the bytes to the UART). It is possible in the future that BIOS
APIs will be added which allows operations to be *asynchronous* - that is, to
return immediately once the operation has been *queued* and then either call
some function or set some marker value in memory once the operation has been
completed. Such an API is more complex to develop, and has interesting
challenges around ensuring the memory passed to the function remains available
(i.e. it isn't just on the stack of the calling function). However, these APIs
are generally more efficient in terms of CPU time, as the CPU does not have to
waste cycles spinning in the main thread whilst it waits for the operation to
complete - for example, you could be writing some sectors to disk and performing
an I²C read from the HID controller asynchronously, whilst also calculating new
display graphics and audio in the main thread.

For now however, the blocking APIs were sufficient for MS-DOS, and they will be
sufficient here.

## Why split the OS and the BIOS?

We could take the Linux / Windows NT approach, with a boot-loader that performs
the bare minimum required to get the kernel (or, in our case, what we call the
'OS') into RAM and start it running. The boot-loader for a PC is usually a UEFI
BIOS but on an older system might be an IBM PC-compatible 'legacy' BIOS. The
operations performed typically include DRAM setup, rudimentary console setup
(either frame-buffer or over UART) and accessing the kernel from some kind of
block device or non-volatile memory (e.g. SD Card). The kernel then replaces
the boot-loader, which is never used again.

Because our Neotron systems generally have Flash memory, we don't actually
need a boot-loader in that sense. Instead, we implement something more like
the interface provided by an IBM PC-compatible 'legacy' BIOS for running
MS-DOS, which itself was modeled on the design of CP/M's BIOS and BDOS
components. In that case, the BIOS provides a fixed application binary
interface (ABI) which is unchanging. This allowed development of MS-DOS (and
CP/M) to be performed against a fixed target (or rather, multiple
implementations of a fixed ABI), allowing the same copy of MS-DOS or CP/M to
run on a variety of PCs from a variety of manufacturers. The alternative would
be to require all of the drivers and board support packages to be added into
OS kernel (as usually happens with Linux), or for the drivers to be loaded
from disk at boot-up (as with Windows). We have seen with Linux that, for
whatever reason, not all boards and peripherals get their driver support
upstreamed, and because Linux does not maintain a binary ABI for drivers,
anything that isn't upstream quickly becomes obsolete. As a case in point, try
to update the version of Linux used on circa-2015 Android tablet - you'll
quickly find yourself stuck on an old Kernel version with bespoke patches (or
worse, binary blobs) that weren't upstreamed or open-sourced. Whilst Windows
maintains a binary ABI for drivers, they are usually closed-source and of
varying quality, leading to system instabilities that are almost impossible to
debug.

The Neotron approach allows for:

* Less centralised development: the Neotron developers do not need to look
  after (or even know about) your BIOS implementation, as long as it meets the
  ABI
* Easier upgrades: any system with a Neotron BIOS should be updateable to a
  newer version of the Neotron OS
* A smaller OS: the amount of code in the OS (e.g. for drivers) is reduced, which
  means the OS is easier to understand and review
* Simpler builds: everyone can run basically the same build of the OS (subject
  to some constraints about the specific Arm architecture being targeted) as
  everything board specific is in the BIOS
* Better portability: because anything CPU specific is in the BIOS, it should
  - in theory - be possible to port the Neotron OS to another CPU architecture
  with just a re-compile.

The downsides are:

* It's slower: there can be no optimisation across the OS / BIOS boundary.
* Limited support: it's harder to add new classes of peripheral bus (e.g. if
  we ever wanted to add support for I3C), as the old BIOSes won't know about
  them.
* Harder to design: a binary-stable ABI is harder to design in Rust than a
  plain function-call API, as you have to use `extern "C"` and `#[repr(C)]`,
  and therefore can't use significant portions of the Core Library in your API
  definition.

## Testing

For BIOS testing, there will be a special 'cut-down' version of the OS which
offers only a very basic console, and direct access to all the BIOS APIs via
that console.

## Character Sets

The video display engines on a Neotron system only support monospaced 8-bit
fonts (that is, fonts with only 256 glyphs and where each glyph occupies the
same horizontal space on screen), and so the mapped text buffer memory works
exclusively in the character set defined by the font. The rest of the BIOS API
works exclusively in UTF-8 encoded text, and mapping functions are provided by
the BIOS to convert from a Unicode codepoint to the font-specific character set.
Generally the first 128 glyphs will match basic ASCII (and hence the first 128
Unicode code points) for performance reasons, although that isn't necessarily
required.

It is possible that Neotron could be adapted to support multi-byte character
sets (e.g. for Chinese, Japanese or Korean text display where more than 256
glyphs are required), but there is no support for that planned at this time.
