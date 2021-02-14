# The Neotron BIOS - API Calls

Note that these systems calls are listed here for documentation and discussion
purposes. The canonical reference is the BIOS source code. Note also that these
API must be `extern "C"`, which means we can't use references, str-slices,
`Option` or `Result`. Instead we provide our own `extern "C"` alternatives.

## Metadata and Versioning

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

## System Functions

### SystemMemoryInfoGet

```rust
enum SystemMemoryType {
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

struct SystemMemoryInfo {
	/// A human-readable label for this region
	name: ApiStrRef,
	/// The first address in the region (e.g. 0x0000)
	start_addr: usize,
	/// The length of the region
	length: usize,
	/// The memory type
	memory_type: SystemMemoryType,
}

fn SystemMemoryInfoGet(index: u8) -> Option<SystemMemoryInfo>
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

### SystemInterruptQuery

```rust
fn SystemInterruptQuery() -> u8;
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

## Serial Ports (UARTs)

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

## Current Time

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

## I2C Bus

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

## SPI Bus

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

## Universal Serial Bus

### USBxxx

This is where the Neotron system is a USB Host and the OS wants to access any attached USB devices.

*TODO: Add a bunch of functions here along the lines of the USB OHCI. Then check we can implement them using a Tiva-C 123 and an STM32H7.*

## General-Purpose Input/Output

### GpioPortGetInfo

```rust
struct GpioPortInfo {
	name: ApiStrRef,
	num_pins: u8,
}

#[derive(Copy, Clone)]
struct GpioPort(pub u8);

#[derive(Copy, Clone)]
struct GpioPin(pub u8);

#[derive(Copy, Clone)]
struct GpioPinMask(pub u32);

fn GpioPortGetInfo(port: GpioPort) -> Option<GpioPortInfo>;
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

fn GpioPinGetInfo(port: GpioPort, pin: GpioPin) -> Option<GpioPinInfo>;
```

### GpioPinSetMode

Sets the mode for a GPIO pin.

```rust
fn GpioPinSetMode(port: GpioPort, pin: GpioPin, mode: GpioPinMode) -> Result<(), Error>;
```

### GpioPortSetMode

Sets the mode for a number of pins in a GPIO port. The length of `mode` must be in the range 1 to `num_pins`. You can pass `None` to leave a pin's mode unchanged.

```rust
fn GpioPortSetMode(port: GpioPort, mode: &[Option<GpioPinMode>]) -> Result<(), Error>;
```

### GpioPinSetLevel

Sets the output level for an Output GPIO pin. The behaviour if this pin is not currently configured as an output is undefined.

```rust
fn GpioPinSetLevel(port: GpioPort, pin: GpioPin, level: bool) -> Result<(), Error>;
```
### GpioPortSetLevel

Sets the level for a number of pins in a GPIO port. Only pins with a corresponding `1` bit in `mask` get set to the corresponding bit in `level`.

```rust
fn GpioPortSetLevels(port: GpioPort, mask: GpioPinMask, levels: u32) -> Result<(), Error>;
```

### GpioPinGetLevel

Gets the input level for an Input GPIO pin. The behaviour if this pin is not currently configured as an input is undefined.

```rust
fn GpioPinGetLevel(port: GpioPort, pin: GpioPin) -> Result<bool, Error>;
```
### GpioPortGetLevel

Gets the level for a number of pins in a GPIO port. Only pins with a corresponding `1` bit in `mask` get a bit set in the returned value.

```rust
fn GpioPortGetLevels(port: GpioPort, mask: GpioPinMask) -> Result<u32, Error>;
```

## Delay functions

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

Delays until the next vertical blanking interval. That is, it waits until the screen has finished drawing and the raster beam is in the off-screen portion. During this time the video memory can be safely updated without causing tearing on the screen. Only supported if you have built-in video.

## Video functions

### VideoModeGetInfo

```rust
#[derive(Copy, Clone)]
struct VideoMode(pub u8);

enum VideoAttrFormat {
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
	attr_format: VideoAttrFormat,
	/// How many pages of text are available
	num_pages: u8,
	/// Can the user supply a new bitmap font? If so, see `VideoModeFontSet`.
	support_soft_font: bool,
}

enum ChunkyPixelFormat {
	/// 1 bit per pixel monochrome.
	Grey1,
	/// 8-bit greyscale (256 shades of grey), packed as one pixel per byte.
	Grey8,
	/// 4-bit colour (16 colours), packed two pixels per byte, with pallette lookup.
	Indexed8,
	/// 8-bit colour (256 colours), packed as one pixel per byte, with pallette lookup.
	Indexed8,
	/// 16-bit colour (65536 colours), using two bytes per pixel.
	Colour16,
	/// 24-bit colour, packed in four-byte words.
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

fn VideoModeGetInfo(mode: VideoMode) -> Option<VideoModeInfo>;
```

Gets information about a specific video mode. Returns `None` if that video mode is not supported. Video Mode 0 is always 'no video' - i.e. the video output is disabled or not available.

### VideoModeGet

```rust
fn VideoModeGet() -> VideoMode;
```

Gets which video mode the system is currently in. A value of zero means video is currently disabled.

### VideoModeSet

```rust
fn VideoModeSet(mode: VideoMode) -> Result<(), Error>;
```

Selects a new video mode. Selecting any mode where `VideoModeGetInfo(mode)` returns `None` will give an error. 

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

### VideoSetText

```rust
fn VideoSetText(
	top_left: VideoCoord, bottom_right: VideoCoord,
	text: &[u8],
	foreground: u8, background: u8);
```

Write text to the display, using the given indexed foreground and background
colours. The written text will wrap inside the video rectangle, but does not
understand carriage return characters, new-line characters or any other screen
formatting - each value simply selects one of 256 glyphs to be displayed at
that position on the screen.

### VideoSetPixel

```rust
enum VideoColour {
	Binary(bool),
	Indexed(u8),
	High(u16),
	True(u8, u8, u8)
}

struct VideoCoord {
	x: u16,
	y: u16
}

fn VideoSetPixel(pos: VideoCoord, colour: VideoColour);
```

Sets a single pixel at the given location to the given colour.

This function is not supported if video is currently disabled, or in a text mode.

### VideoSetPixels

```rust
fn VideoSetPixels(
	top_left: VideoCoord,
	bottom_right: VideoCoord,
	colour: VideoColour);
```

Sets multiple pixels to the same value, as specified by the rectangle given by
`top_left` to `bottom_right`.

This function is not supported if video is currently disabled, or in a text mode.

### VideoBlitRaw

```rust
unsafe fn VideoBlitRaw(
	top_left: VideoCoord,
	bottom_right: VideoCoord,
	source: *const u8);
```

Draws pixel data onto the screen, in the region specified by the rectangle
given by `top_left` to `bottom_right`. The supplied pixel data must be in the
same format as the screen.

The buffer pointed to by `source` must be at least `(1 + bottom_right.x -
top_left.x) * (1 + bottom_right.y - top_left.y)` pixels in length where the
length of a pixel in bytes depends on the current video mode.

This function is not supported if video is currently disabled, or in a text mode.

### VideoBlitBinary

```rust
fn VideoBlitBinary(
	top_left: VideoCoord,
	bottom_right: VideoCoord,
	foreground: VideoColour,
	background: VideoColour,
	source: *const u8);
```

Draws monochrome greyscale pixel data to the screen - where a bit is set in
the source data the `foreground` colour is used, otherwise the `background`
colour is used. You might use this to draw text to a bitmap display, for
example.

The buffer pointed to by `source` must be at least `(1 + bottom_right.x -
top_left.x) * (1 + bottom_right.y - top_left.y)` pixels in length.

This function is not supported if video is currently disabled, or in a text mode.

### VideoBlit

```rust
fn VideoBlitRaw(
	top_left: VideoCoord,
	bottom_right: VideoCoord,
	source: &[VideoColour]);
```

Draws pixel data onto the screen, in the region specified by the rectangle
given by `top_left` to `bottom_right`. The supplied pixel data will be
converted to screen format as required.

The slice `source` must be at least `(1 + bottom_right.x - top_left.x) * (1 +
bottom_right.y - top_left.y)` elements in length.

This function is not supported if video is currently disabled, or in a text mode.

### VideoDrawLine

```rust
fn VideoDrawLine(
	start: VideoCoord,
	end: VideoCoord,
	colour: VideoColour);
```

Draws a line from `start` to `end`, applying the colour given by `colour` (see `VideoSetPixel`).

This function is not supported if video is currently disabled, or in a text mode.

### VideoDrawRect

```rust
fn VideoDrawRect(
	top_left: VideoCoord,
	bottom_right: VideoCoord,
	colour: VideoColour);
```

Sets multiple pixels to the same value, as specified by the rectangle given by
`top_left` to `bottom_right`. Only the outer edge is filled - the interior of
the rectangle is unchanged.

This function is not supported if video is currently disabled, or in a text mode.

### VideoHookScanline

```rust
type VideoHookFunction = fn(usize, u16);
fn VideoHookScanline(context: usize, scan_line: u16, fn: VideoScanlineFunction) -> Result<(), Error>;
```

Supply a function to be called just before the given scan-line is rendered. The function is called in Interrupt Context, and so may pre-empt other functions. The function is automatically un-hooked before it is called, and it is safe for the called function to re-hook itself, or to hook some other function, using `VideoHookScanline` but it is not safe to call any other BIOS function.

This function is not supported if video is currently disabled. Passing a value
of `scan_line` which is out of bounds for the current video mode will return
an error.

