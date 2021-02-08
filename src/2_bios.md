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

## Supported API calls

Note that these systems calls are listed here for documentation and discussion
purposes. The canonical reference is the BIOS source code. Note also that these
API must be `extern "C"`, which means we can't use references, str-slices,
`Option` or `Result`. Instead we provide our own `extern "C"` alternatives.

### ApiVersionGet

```rust
fn ApiVersionGet() -> u32;
```

Gets the version number of the BIOS API. You need this value to determine which
of the following API calls are valid in this particular version.

### BiosVersionGet

```rust
fn BiosVersionGet() -> ApiStrRef;
```

Returns a pointer to a static string slice. This string contains the version
number and build string of the BIOS.

### MemoryInfoGet

```rust
enum MemoryType {
	/// An on-chip memory optimised for code
	InstructionTightlyCoupled,
	/// An on-chip memory optimised for data
	DataTightlyCoupled,
	/// An on-chip Static RAM that can be used for code or data
	InternalStatic,
	/// An on-chip Dynamic RAM that can be used for code or data
	InternalDynamic,
	/// An off-chip Static RAM that can be used for code or data
	ExternalStatic,
	/// An off-chip Dynamic RAM that can be used for code or data
	ExternalDynamic,
}

struct MemoryInfo {
	/// A human-readable label for this region
	name: ApiStrRef,
	/// The first address in the region (e.g. 0x0000)
	start_addr: usize,
	/// The length of the region
	length: usize,
	/// The memory type
	memory_type: MemoryType,
}

fn MemoryInfoGet(index: u8) -> Option<MemoryInfo>
```

Gets information about the regions of memory in the system. An OS can use this
to work out where it can store the heap, and load any applications to. Regions
should be ordered according to their relative performance, with the fastest
region listed first. The fastest regions will be used for the most commonly used
data structures (e.g. the system stack). Regions labelled as
`InstructionTightlyCoupled` will ony be used for code (e.g code loaded from
disk) and not for data (e.g. variables).

Neotron OS has no support for memory refresh. If this is required, it must be
arranged by the BIOS in the background.

### SerialGetInfo

```rust
enum SerialType {
	/// An RS-232 interface, but at TTL voltages. Typically used with an
	/// FTDI FT232 cable.
	TtlUart,
	/// An RS-232 interface
	Rs232,
	/// A USB Device implementing Communications Class Device (also known as
	/// a USB Serial port). The USB Device implementation may be on-chip
	/// (handled by the BIOS), or off-chip.
	UsbCdc,
	/// A MIDI interface
	Midi,
	/// A Commodore Serial interface
	Cbm,
	/// An RS-485 bus
	Rs485,
}

struct SerialInfo {
	name: ApiStrRef,
	type: SerialType,
	data_rate_bps: u32,
}

fn SerialGetInfo(device: u8) -> Option<SerialInfo>;
```

Get information about the Serial ports in the system. Serial ports are ordered
octet-oriented pipes. You can push octets into them using a 'write' call, and
pull bytes out of them using a 'read' call. They have options which allow them
to be configured at different speeds, or with different transmission settings
(parity bits, stop bits, etc) - you set these with a call to `SerialConfigure`.
They may physically be a MIDI interface, an RS-232 port or a USB-Serial port.
There is no sense of 'open' or 'close' - that is an Operating System level
design feature. These APIs just reflect the raw hardware, in a similar manner to
the registers exposed by a memory-mapped UART peripheral.

### SerialConfigure

```rust
enum SerialParity {
	Odd,
	Even,
	None
}

enum SerialHandshaking {
	None,
	RtsCts,
}

enum SerialStopBits {
	One,
	Two
}

enum SerialDataBits {
	Seven,
	Eight,
}

struct SerialConfig {
	data_rate_bps: u32,
	data_bits: SerialDataBits,
	stop_bits: SerialStopBits,
	parity: SerialParity,
	handshaking: SerialHandshaking,
}

fn SerialConfigure(device: u8, config: Option<SerialConfig>) -> Result<(), Error>;
```

Set the options for a given serial device. An error is returned if the options
are invalid for that serial device (e.g. you have picked a data rate that isn't
supported).

Passing `None` will power-down and/or deactivate the peripheral to the extent
supported by the peripheral.

### SerialWrite

```rust
fn SerialWrite(device: u8, data: &[u8], timeout: Option<Timeout>) -> Result<usize, Error>;
```

Write bytes to a serial port. There is no sense of 'opening' or 'closing' the
device - serial devices are always open. If the return value is `Ok(n)`, the
value `n` may be less than `data.len()`. If so, that means not all of the data
could be transmitted - only the first `n` bytes were.

### SerialWriteAsync

```rust
struct AsyncBuffer(*mut u8, usize);
struct AsyncHandle(u16);

fn SerialWriteAsync(device: u8, data: AsyncBuffer, timeout: Option<Timeout>) -> Result<AsyncHandle, Error>;
```

Write bytes to a serial port asynchronously - that is, the call will return when
the write is queued, not when the write has completed. The `AsyncBuffer` must
remain valid until the call has complete. An implementation that does not have
asynchronous support may just implement this function by calling `SerialWrite`.

You should register an interrupt with `SerialRegisterInterrupt` and poll all of
your `AsyncHandle` objects on interrupt.

Each implementation will have a limit on the number of `AsyncBuffer` objects it
can queue before returning an error.

### SerialRead

```rust
fn SerialRead(device: u8, data: &mut [u8], timeout: Option<Timeout>) -> Result<usize, Error>;
```

Read bytes from a serial port. There is no sense of 'opening' or 'closing' the
device - serial devices are always open. If the return value is `Ok(n)`, the
value `n` may be less than `data.len()`. If so, that means not all of the buffer
could be filled with received data - only the first `n` bytes were.

### SerialFlush

```rust
fn SerialFlush(device: u8, timeout: Option<Timeout>) -> Result<(), Error>;
```

Wait until any bytes previously accepted by `SerialRead` have actually left the
UART device (as opposed to just sitting in a hardware buffer).

### SerialRegisterInterrupt

```rust
enum SerialInterruptFlag {
	/// SerialRead will return a non-zero value.
	RxReady = 1,
	/// The last SerialWrite has completed.
	TxComplete = 2,
	Break = 4
}

struct SerialInterruptFlagSet(u32);

type SerialCallback = Fn(device: u8, flags: SerialInterruptFlagSet);

fn SerialRegisterInterrupt(device: u8, function: SerialCallback, flags: SerialInterruptFlagSet) -> Result<(), Error>;
```

When any of the properties specified in `flags` are true, the given `function`
will be executed from the hardware interrupt handler. Note that calling Neotron
BIOS functions from interrupt handlers is not, in general, supported, and so the
callback should set some state and to inform the main thread that something has
occurred.

### TimestampGet

```rust
fn TimestampGet() -> u64
```

Returns a value indicating how long the system has been running. This is typically the number of video lines generated since the system was powered on. This value is guaranteed to always be greater than (or equal to) the last time this function was called - unless the system is rebooted.

### TimestampRate

```rust
fn TimestampRate() -> u16
```

Returns the number of timestamp ticks in a second. You can use this to convert the difference between two `TimestampGet` values into a duration in seconds.

### TimeGet

```rust
struct Time {
	seconds_since_epoch: u32,
	ticks_since_second: u16
}

fn TimeGet() -> Time;
```

Get the current wall time. The Neotron BIOS does not understand time zones,
leap-seconds or the Gregorian calendar. Nor does it promise that time is
monotonic - users can (and will) move the clock backwards and forwards in time.
It simply stores time as an incrementing number of seconds since some epoch, and
the number of ticks (typically 60 Hz, but see `TimestampRate`) since that second
began. A day is assumed to be exactly 86,400 seconds long. This is a lot like
POSIX time, except we have a different epoch - the Neotron epoch is
2000-01-01T00:00:00Z. It is highly recommend that you store UTC in the BIOS and
use the OS to handle time-zones.

### TimeSet

```rust
fn TimeSet(time: Time);
```

Set the current wall time to the given value. See [TimeGet](#timeget).

### I2CBusGetInfo

```rust
enum I2CBusSpeed {
	// 10kbps
	Low,
	/// 100kbps
	Standard,
	/// 400kbps
	Fast,
	/// 1Mbps
	FastPlus,
	/// 3.4Mbps
	High,
	/// 5Mbps
	UltraFast,
}

struct I2CBusInfo {
	name: ApiStrRef,
	supported_speeds: &[I2CBusSpeed],
}

fn I2CBusGetInfo(bus: u8) -> Option<I2CBusInfo>;
```

Gets information about the I2C buses in the system..

### I2CBusSetSpeed

```rust
fn I2CBusSetSpeed(bus: u8, speed: I2CBusSpeed) -> Result<(), Error>;
```

Set the I2C bus speed.

### I2CBusWriteRead

```rust
fn I2CBusWriteRead(bus: u8, address: u8, out_buffer: &[u8], in_buffer: &mut [u8]) -> Result<(), Error>;
```

Writes data to the I2C bus, then reads data. Performs the following operations (courtesy of the Embedded HAL documentation):

```
		  +---+-----+---+--+---+--+---+---+--+---+--+-----+---+--+---+--+---+---+--+----+--+
	 Main | ST|SAD+W|   |O0|   |O1|   |...|OM|   |SR|SAD+R|   |  |MAK|  |MAK|...|  |NMAK|SP|
		  +---+-----+---+--+---+--+---+---+--+---+--+-----+---+--+---+--+---+---+--+----+--|
Secondary |   |     |SAK|  |SAK|  |SAK|...|  |SAK|  |     |SAK|I0|   |I1|   |...|IN|    |  |
		  +---+-----+---+--+---+--+---+---+--+---+--+-----+---+--+---+--+---+---+--+----+--+
```

Where:

	* ST = start condition
	* SAD+W = secondary address followed by bit 0 to indicate writing
	* SAK = secondary acknowledge
	* Oi = ith outgoing byte of data
	* SR = repeated start condition
	* SAD+R = secondary address followed by bit 1 to indicate reading
	* Ii = ith incoming byte of data
	* MAK = main acknowledge
	* NMAK = main no acknowledge
	* SP = stop condition

The given address must in the range `1..127`. The BIOS does not handle device detection or even know what devices are fitted - that is all handled in the operating system.

### I2CBusWriteReadAsync

```rust
fn I2CBusWriteRead(bus: u8, address: u8, out_buffer: AsyncBuffer, in_buffer: AsyncBuffer) -> Result<AsyncHandle, Error>;
```

### SpiBusGetInfo

```rust
struct SpiBusInfo {
	/// A name for this SPI bus
	name: ApiStrRef,
	/// The number of unique chip select signals associated with this SPI bus
	num_chip_selects: u8,
	/// The maximum SPI bus speed supported
	max_bus_speed_bps: u32,
	/// Will be 1, 2 or 4
	max_bus_width_bits: u8,
}

fn SpiBusGetInfo(bus: u8) -> Option<SpiBusInfo>;
```

Gets information about the SPI buses in the system.

### SpiBusConfigure

```rust
enum SpiMode {
	Mode0,
	Mode1,
	Mode2,
	Mode3
}

struct SpiBusConfig {
	bus_speed_bps: u32,
	bus_width_bits: u8,
	mode: SpiMode
}

fn SpiBusConfigure(bus: u8, config: Option<SpiBusConfig>) -> Result<u32, Error>;
```

Configure the SPI bus. Where arbitrary speeds are not supported by the bus, the
BIOS will select the closest supported speed which is not greater than the given
speed.

### SpiDeviceTransfer

Write to and read from an SPI device.

```rust
enum MutableDataBuffer<'a> {
	U8(&'a mut [u8]),
	U16(&'a mut [u16]),
	U32(&'a mut [u32]),
}

fn SPIDeviceTransfer(bus: u8, device: u8, word_size: u8, buffer: MutableDataBuffer) -> Result<(), Error>;
```

Writes data to an SPI device and reads data from the same SPI simultaneously. Data is both read from and written to the same buffer. When writing with a word size of 8 or less, each word should be stored in the least-significant bits of each `u8` in a `MutableDataBuffer::U8`. When writing with a word size of `8 < x <= 16`, each word should be stored in the least-significant bits of each `u16` in a `MutableDataBuffer::U16`. When writing with a word size of `16 < x <= 32`, each word should be stored in the least-significant bits of each `u32` in a `MutableDataBuffer::U32`.

### SpiDeviceWrite

Write to an SPI device, discarding any read data.

```rust
enum DataBuffer<'a> {
	U8(&'a [u8]),
	U16(&'a [u16]),
	U32(&'a [u32]),
}

fn SpiDeviceWrite(bus: u8, device: u8, buffer: DataBuffer) -> Result<(), Error>;
```

### USBxxx

This is where the Neotron system is a USB Host and the OS wants to access any attached USB devices.

*TODO: Add a bunch of functions here along the lines of the USB OHCI. Then check we can implement them using a Tiva-C 123 and an STM32H7.*

### GpioPortGetInfo

```rust
struct GpioPortInfo {
	name: ApiStrRef,
	num_pins: u8,
}

fn GpioPortGetInfo(port: u8) -> Option<GpioPortInfo>;
```

Gets information about the GPIO ports on this system. Note that each GPIO port can only have up to 32 pins.

### GpioPinGetInfo

Gets information about the GPIO pins on a specific port.

```rust
enum GpioPinMode {
	InputFloating,
	InputPullUp,
	InputPullDown,
	Output,
	OutputOpenCollector,
}

struct GpioPinInfo {
	name: ApiStrRef,
	state: GpioPinMode
}

fn GpioPinGetInfo(port: u8, pin: u8) -> Option<GpioPinInfo>;
```

### GpioPinSetMode

Sets the mode for a GPIO pin.

```rust
fn GpioPinSetMode(port: u8, pin: u8, mode: GpioPinMode) -> Result<(), Error>;
```

### GpioPortSetMode

Sets the mode for a number of pins in a GPIO port. The length of `mode` must be in the range 1 to `num_pins`. You can pass `None` to leave a pin's mode unchanged.

```rust
fn GpioPortSetMode(port: u8, mode: &[Option<GpioPinMode>]) -> Result<(), Error>;
```

### GpioPinSetLevel

Sets the output level for an Output GPIO pin. The behaviour if this pin is not currently configured as an output is undefined.

```rust
fn GpioPinSetLevel(port: u8, pin: u8, level: bool) -> Result<(), Error>;
```
### GpioPortSetLevel

Sets the level for a number of pins in a GPIO port. Only pins with a corresponding `1` bit in `mask` get set to the corresponding bit in `level`.

```rust
fn GpioPortSetLevels(port: u8, level: u32, mask: u32) -> Result<(), Error>;
```

### GpioPinGetLevel

Gets the input level for an Input GPIO pin. The behaviour if this pin is not currently configured as an input is undefined.

```rust
fn GpioPinGetLevel(port: u8, pin: u8) -> Result<bool, Error>;
```
### GpioPortGetLevel

Gets the level for a number of pins in a GPIO port. Only pins with a corresponding `1` bit in `mask` get a bit set in the returned value.

```rust
fn GpioPortGetLevels(port: u8, mask: u32) -> Result<u32, Error>;
```

### DelayMs

```rust
fn DelayMs(period_ms: u16);
```

Delays for at least a given number of milliseconds. Because some systems have lengthy interrupt routines (e.g. CPU powered video rendering), the application may be delayed for considerably more than the requested period (but never less than the requested period).

### DelayUs

```rust
fn DelayUs(period_us: u16);
```

Delays for at least a given number of microseconds. Because some systems have lengthy interrupt routines (e.g. CPU powered video rendering), the application may be delayed for considerably more than the requested period (but never less than the requested period).

### DelayVblank

```rust
fn DelayVblank();
```

Delays until the next vertical blanking interval. That is, it waits until the screen has finished drawing and the raster beam is in the off-screen portion. During this time the video memory can be safely updated without causing tearing on the screen.

### VideoModeGetInfo

```rust
enum AttrFormat {
	// No attribute bytes
	None,
	// Suports the given number of foreground and background colours
	IndexColour(u8),
}

struct VideoTextModeInfo {
	/// Number of columns of text on screen
	width: u8,
	/// Number of rows of text on screen
	height: u8,
	/// Number of bytes per row (must be >= cols * 2)
	row_size: usize,
	/// Do we support colour or other attributes?
	attr_format: AttrFormat,
	/// How many pages of text are available
	num_pages: u8,
	/// Can the user supply a new bitmap font? If so, see `VideoModeFontSet`.
	support_soft_font: bool,
}

enum ChunkyPixelFormat {
	/// 1 bit per pixel monochrome.
	Grey1,
	/// 8-bit greyscale (256 shades of grey), packed as bytes.
	Grey8,
	/// 8-bit colour (256 colours), packed as bytes (with optional pallette lookup).
	Colour8,
	/// 16-bit colour (65536 colours), packed as 16-bit words.
	Colour16,
	/// 24-bit colour, packed in 32-bit words.
	Colour32
}

struct VideoChunkyBitmapModeInfo {
	/// Number of pixels across the screen
	width: u16,
	/// Number of pixels down the screen
	height: u16,
	/// Number of bytes per row (must be >= width * bits-per-pixel / 8)
	row_size: usize,
	/// How many pages of video are available (e.g. 2 for double-buffering)
	num_pages: u8,
	/// What format are the pixels in? Determines how many bits are required for each pixel.
	pixel_format: ChunkyPixelFormat,
	/// How big each pallette entry is. '4' would mean each pallette entry gives a 4-bit (16-colour) value
	/// (like CGA), whilst 18 would mean each pallette entry gave an 18-bit
	/// (262,144-colour) value (like VGA)
	pallete_entry_size_bits: u8,
}

enum PlanarPixelFormat {
	/// 2 planes, 1-bpp each, for 4 colours (with pallette look-up)
	Colour2,
	/// 3 planes, 1-bpp each, for 8 colours (with optional pallette look-up)
	Colour3,
	/// 4 planes, 1-bpp each, for 16 colours (with optional pallette look-up)
	Colour4,
	/// 3 planes, 8-bpp (i.e. one byte per pixel) each (24-bit true-colour)
	Colour3x8,
}

struct VideoPlanarBitmapModeInfo {
	/// Number of pixels across the screen
	width: u16,
	/// Number of pixels down the screen
	height: u16,
	/// Number of bytes per row (must be >= width * bits-per-pixel / 8)
	row_size: usize,
	/// How many pages of video are available (e.g. 2 for double-buffering)
	num_pages: u8,
	/// What format are the pixels in? Determines how many bitplanes are required for each pixel.
	pixel_format: PlanarPixelFormat,
	/// How big each pallette entry is. '4' would mean each pallette entry gives a 4-bit (16-colour) value
	/// (like CGA), whilst 18 would mean each pallette entry gave an 18-bit
	/// (262,144-colour) value (like VGA)
	pallete_entry_size_bits: u8,
}

enum VideoModeInfo {
	Disabled,
	Text(VideoTextModeInfo),
	ChunkyBitmap(VideoChunkyBitmapModeInfo),
}

fn VideoModeGetInfo(mode: u8) -> Option<VideoModeInfo>;
```

Gets information about a specific video mode. Returns `None` if that video mode is not supported. Video Mode 0 is always 'no video' - i.e. the video output is disabled or not available.

### VideoModeGet

```rust
fn VideoModeGet() -> u8;
```

Gets which video mode the system is currently in. A value of zero means video is currently disabled.

### VideoModeSet

```rust
fn VideoModeSet(mode: u8) -> Result<(), Error>;
```

Selects a new video mode.

### VideoModeFontGet

```rust
struct FontInfo {
	font: *const u8,
	font_len: usize,
	glyph_width_pixels: u8,
	glyph_height_rows: u8,
}

fn VideoModeFontGet() -> Result<FontInfo, Error>;
```

Get a information about the currently loaded bitmap font. See [VideoModeFontSet](#videomodefontset).

### VideoModeFontSet

```rust
fn VideoModeFontSet(font_info: FontInfo) -> Result<(), Error>;
```

Change the current bitmap font. Pixel data is laid out row-wise, with each row of each glyph packed into an integer number of bytes. For example, a 7-pixel wide glyph is packed into bytes (leaving the LSB unused), whilst a 9-pixel glyph is packed into two bytes as a big-endian 16-bit value, leaving the 7 LSBs of the second (low) byte unused.

An error is returned if the current mode doesn't support soft fonts, or if the width/height parameters are not compatible with the current mode, or if the `font_len` does not equal `int((glyph_width_pixels + 7) / 8) * glyph_height_rows * 256`.

Neotron uses 8-bit fonts, so there are only 256 glyphs in a font. Unicode text must be mapped to this set of glyphs for display on-screen. Alternative the OS can implement its own rendering system and simply draw pixels using the BIOS.

The standard Neotron fonts match MS-DOS / IBM Code Page 850, but user-supplied fonts can do anything (they don't even have to be characters - they could be background tiles for a game, for example).

### VideoModePageSet

```rust
fn VideoModePageSet(page: u8) -> Result<(), Error>;
```

Changes the currently displayed video page (if the current video mode supports multiple pages).

### VideoModeTextScroll

```rust
fn VideoModeTextScroll(line: u8) -> Result<(), Error>;
```

Changes which 'line' of the text buffer is displayed at the top of the screen. When the text renderer gets the bottom of the text buffer, it starts taking lines from the top of the buffer.

The default start line is 0, giving:

```
+----------+
| Line 0   |
| Line 1   |
| Line 2   |
:          :
| Line N-3 |
| Line N-2 |
| Line N-1 |
+----------+
```

Setting the start line to 2, will give:

```
+----------+
| Line 2   |
| Line 3   |
| Line 4   |
:          :
| Line N-1 |
| Line 0   |
| Line 1   |
+----------+
```

### VideoModeGetBuffer

```rust
fn VideoModeGetBuffer() -> *mut u8;
```

Gets a pointer to a specific page in the video buffer (an array of text characters and attributes, or an array of pixels, depending on the mode). Writing to this buffer during the rendering period (i.e. without waiting using `DelayVblank`) may cause temporarily graphical glitches, but won't crash the system.

Writing outside the bounds of the buffer may crash the system.

### VideoHookScanline

```rust
type VideoHookFunction = fn(u32, u16);
fn VideoHookScanline(context: u32, scan_line: u16, fn: VideoScanlineFunction);
```

Supply a function to be called just before the given scan-line is rendered. The function is called in Interrupt Context, and so may pre-empt other functions. The function is automatically un-hooked before it is called, and it is safe for the called function to re-hook itself, or to hook some other function, using `VideoHookScanline` but it is not safe to call any other BIOS function.

### InterruptQuery

```rust
fn InterruptQuery() -> u8;
```

The Neotron BIOS has up to 8 external interrupts, numbered 0 to 7. A bit is set in the returned value for each interrupt that has been triggered. Calling this function automatically clears the interrupts, so the caller must ensure they are actioned. The mapping of interrupts to hardware is configured at the OS level - probably through a configuration file. The primary purpose of these interrupts is to avoid polling external hardware unnecessarily. An OS should call this function at least once per frame, and action those interrupts as soon as possible.

### SystemReboot

```rust
fn SystemReboot() -> !;
```

Reboots the system.

### SystemHalt

```rust
fn SystemHalt() -> !;
```

Halts the system. The OS should display some 'This system is now safe to power off' message first, in case the hardware can't actually power itself off.

### SystemControl

```rust
fn SystemControl(command: u32, data: *const u8, data_len: usize) -> Result<u32, Error>;
```

Sends a BIOS-specific command. See your BIOS user guide for details.
