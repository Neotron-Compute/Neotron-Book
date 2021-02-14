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

A BIOS should offer interfaces for:

* Discovering the hardware available to the OS
	* UART controllers
	* I²C controllers
	* SPI controllers (with chip selects)
	* USB Host Controllers
	* SD/MMC controllers
	* Internal Real-Time Clock
* Selecting and using various video modes:
	* Text modes
		* A rectangular grid of *C* columns and *R* rows, giving *R x C* cells
		  in total.
		* Each cell takes two bytes of RAM
		* Each cell accepts one 8-bit character
		* Each cell also has a 4-bit foreground and 4-bit background colour
		  attribute
		* The BIOS provides a Unicode to 8-bit Glyph mapping function for the
		  current font
	* Bitmap graphics modes
		* 1-bpp monochrome mode
		* 1-bpp block-colour mode (with the colour taken from each cell in the
		  corresponding text mode)
		* 1-, 4- or 8-bpp planar colour modes (three greyscale bitplanes - one
		  for each of R, G and B)
		* 4- or 8-bit chunky colour modes (each N-bit value is one pixel) using
		  a pallette lookup
		* 16-bit, 24-bit and 32-bit high-colour modes (each N-bit value is one
		  pixel)
		* RAM region used as frame buffer must be supplied by OS and can be
		  moved
		* Adjustable row stride length - choosing a value greater than *number
		  of pixels per row * number of bytes per pixel* allows for smooth
		  scrolling effects.
	* Split modes
		* A scan-line can be specified to switch from text mode to video mode
		  (or vice-versa), provided they have the same native resolution. This
		  can reduce RAM requirements for video modes and improve text rendering
		  performance.
	* Layered Modes
		* Some video systems allow multiple layers, each in a different mode
		  (but at the same resolution).
	* Hooking video scan-line interrupts
* Accessing Serial/UART Controllers
* Accessing any I²C Controllers, and any devices attached to its bus
* Accessing any SPI Controllers, and any devices attached to its chip selects
* Accessing any USB Host Controllers, and any USB devices attached
* Accessing any SD/MMC Controllers, and the SD card attached to the controller
* Reading/writing an internal RTC
* Playing/recording audio samples at specific sample rates
* Tracking wall time (in microseconds)
* Setting timer interrupts
* Hooking interrupts for the external IRQ lines

On top of these APIs sits the Neotron OS, which should be portable with almost
zero changes to any device with a Neotron BIOS. While you could write an
application which uses the BIOS directly, most applications will use the
higher-level APIs exposed by the Neotron OS.

Note that the BIOS does not currently offer support for reading a Keyboard,
unlike an IBM PC BIOS. It is expected that the Keyboard, and other Human Input
Devices (HID) will be handled over I²C from an I²C HID Controller peripheral,
over USB from a USB HID Device, or via an I²C or SPI GPIO interface connected to
a keyboard matrix.

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
approach.

The BIOS is responsible for starting the system and performing hardware
initialisation. It will also search for the OS, either in Flash ROM, on disk, or
perhaps even loaded over a UART. The OS then has its initialisation function
called, to which the structure of BIOS function calls is passed. The
initialisation function is specified as a function pointer stored at a specific
offset (most likely the first four bytes of the OS image).

Most of these APIs will be 'blocking' - that is, the function call will not
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

## Why split the OS and the BIOS?

We could take the Linux / Windows NT approach, where the bootloader performs the
bare minimum required to get the kernel (or, in our case, what we call the 'OS')
into RAM and start it running. The bootloader for a PC is usually a UEFI BIOS
but on an older system might be an IBM PC-compatible 'legacy' BIOS. The
operations performed typically include DRAM setup, rudimentary console setup
(either framebuffer or over UART) and accessing the kernel from some kind of
block device or non-volatile memory (e.g. SD Card). The kernel then replaces the
bootloader, which is never used again.

Because our Neotron systems generally have Flash ROM, we don't actually need a
bootloader in that sense. Instead, we implement something more like the
interface provided by an IBM PC-compatible 'legacy' BIOS for running MS-DOS. In
that case, the BIOS provides a fixed application binary interface (ABI) which is
unchanging. This allowed development of MS-DOS to be performed against a fixed
target (or rather, multiple implementations of a fixed ABI), allowing the same
copy of MS-DOS to run on a variety of PCs from a variety of manufacturers. The
alternative would be to require all of the drivers and board support packages to
added into OS kernel (as usually happens with Linux), or for the drivers to be
loaded from disk at boot-up (as with Windows). We have seen with Linux that, for
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

* Less centralised development - the Neotron developers do not need to look
  after (or even know about) your BIOS implementation, as long as it meets the
  ABI
* Easier upgrades - any system with a Neotron BIOS should be updateable to a
  newer version of the Neotron OS
* Smaller OS - the amount of code in the OS (e.g. for drivers) is reduced, which
  means the OS is easier to understand and review
* Simpler builds - everyone can run basically the same build of the OS (subject
  to some constraints about the specific Arm architecture being targeted) as
  everything board specific is in the BIOS

The downsides are:

* It's slower - there can be no optimisation across the OS / BIOS boundary
* It's harder to add new classes of peripheral bus (e.g. if we ever wanted to
  add support for I3C), as the old BIOSes won't know about them
* It's harder to design a binary-stable ABI in Rust than a plain function-call
  API, as you have to use `extern "C"` and avoid significant portions of the
  Standard Library in your API definition.

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

## Timeouts

Some functions accept a timeout argument. If this argument is `None`, then the
function blocks. Otherwise the function waits for up to the period specified, or
the operation is complete, whichever occurs first.

```rust
struct Timeout(u16);

impl Timeout {
	fn frames(frames: u16) -> Timeout;
	fn milliseconds(ms: u16) -> Timeout;
	fn microseconds(us: u16) -> Timeout;
}
```

All video modes on Neotron are 60 Hz and so 1 frame is approximately 16.667ms.
