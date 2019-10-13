# The Neotron BIOS

## Introduction

The Neotron BIOS is the hardware-abstraction layer for the Neotron Operating System. It allows the OS to run on different hardware platforms with the minimum number of changes.

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

* Block devices (e.g. SD cards)
* Selecting and using various video modes:
	* Text modes which accept an 8-bit Code Page 850 characters
	* Bitmap graphics modes
* Accessing Serial/UART devices
* Accessing any I2C buses
* Accessing any SPI buses
* Playing an audio sample
* System timers
* Hooking interrupts (including scan-line interrupts)

While you could write an application which uses the BIOS directly, most applications will use the higher-level APIs exposed by the OS.

## Calling a BIOS API

On ARM systems, calling a kernel API is usually done through a `SWI` or `SVC` machine instruction. This effectively triggers an interrupt, putting the CPU into interrupt mode where it starts executing the `SWI` exception handler. Unfortunately, passing arguments to the `SWI` Exception handler can only be done by stuffing pointers into specific registers, which means you can't write it all in pure Rust (or C for that matter) without using unstable in-line assembly.

Monotron instead used a simple alternative - when the application was started by the OS, the OS passed it a pointer to a structure of function pointers. You can think of this as being like an old-fashioned jump table. When the application wanted to get the OS to do something, the application just called the appropriate function through the given pointer.

We're going to take the same approach for the Neotron BIOS.

## Supported API calls

### ApiVersionGet

```rust
fn ApiVersionGet() -> u32;
```

Gets the version number of the BIOS API. You need this value to determine which of the following API calls are valid in this particular version.

### BiosVersionGet

```rust
fn BiosVersionGet() -> &'static str;
```

Returns a pointer to a static string slice. This string contains the version number and build string of the BIOS.

### DriveGetInfo

```rust
struct DriveInfo {
	name: &'static str,
	sector_size: u16,
	num_sectors: u32,
	removable: bool,
	soft_eject: bool,
}

struct DriveSectorAddress(u32);

fn DriveGetInfo(drive: u8) -> Option<DriveInfo>;
```

Returns information about any fixed disks in the system. The OS will call with incrementing values of `drive` until this function returns `None`. The BIOS knows nothing of MBR or GPT partition tables - it only knows how to read and write sectors.

Note that if you have 512 byte sectors, the largest supported block device is `2**32 * 512 bytes = 2 TiB`. Large hard disks solve this by using 4 KiB sectors.

Disks might include:

* SD cards connected to high-speed 4-bit SD Card interfaces
* SD cards connected to SPI peripherals
* IDE or SCSI hard disks connected to appropriate controllers
* USB flash keys connected to a USB Host port
* PC-style Floppy drives connected to an appropriate controller

Floppy drives with built-in DOS, such as those used with the Commodore 64, are not listed as drives, but instead the CBM Serial interface would appear as a serial port you can send commands down.

### DriveReadSector

```rust
fn DriveReadSector(drive: u8, sector: DriveSectorAddress, buffer: &mut [u8]) -> Result<(), Error>;
```

Reads a sector or sectors from a block device. Blocks operation until the read is complete, or an error occurs. The number of sectors to be read is the same as the largest number of whole sectors that fit into `buffer`. For example, if `buffer.len() == 1025` and the sector size is 512 bytes, two sectors will be read, leaving the final 1 byte untouched.

### DriveWriteSector

```rust
fn DriveWriteSector(drive: u8, sector: DriveSectorAddress, buffer: &[u8]) -> Result<(), Error>;
```

Writes a sector or sectors to a block device. Blocks operation until the write is complete, or an error occurs. The number of sectors to be written is the same as the largest number of whole sectors that fit into `buffer`. For example, if `buffer.len() == 1025` and the sector size is 512 bytes, two sectors will be written, leaving the final 1 byte untouched.

### SerialGetInfo

```rust
enum SerialType {
	/// A MIDI interface
	Midi,
	/// An RS-232 interface
	Rs232,
	/// An RS-232 interface, but at TTL voltages. Typically used with an
	/// FTDI FT232 cable.
	TtlUart,
	/// A USB Device implementing Communications Class Device (also known as
	/// a USB Serial port).
	UsbCdc,
}

struct SerialParity {
	Odd,
	Even,
	None
}

struct SerialHandshaking {
	None,
	RtsCts,
}

struct SerialInfo {
	name: &'static str,
	type: SerialType,
	data_rate_bps: u32,
	data_bits: u8,
	stop_bits: u8,
	parity: SerialParity,
	handshaking: SerialHandshaking,
	blocking: bool
}

fn SerialGetInfo(device: u8) -> Option<SerialInfo>;
```

Get information about the Serial ports in the system. Serial ports are ordered octet-oriented pipes. You can push octets into them using a 'write' call, and pull bytes out of them using a 'read' call. They may have options which allow them to be configured at different speeds, or with different transmission settings (parity bits, stop bits, etc). They may physically be a MIDI interface, an RS-232 port or a USB-Serial port.

### SerialWrite

```rust
fn SerialWrite(device: u8, data: &[u8]) -> Result<usize, Error>;
```

Write bytes to a serial port. There is no sense of 'opening' or 'closing' the device - serial devices are always open. This function may block if `blocking` is true. If the return value is `Ok(n)`, the value `n` may be less than the size of the given buffer. If so, that means not all of the data could be transmitted - only the first `n` bytes were.

### SerialRead

```rust
fn SerialRead(device: u8, data: &mut [u8]) -> Result<usize, Error>;
```

Read bytes from a serial port. There is no sense of 'opening' or 'closing' the device - serial devices are always open. This function may block if `blocking` is true. If the return value is `Ok(n)`, the value `n` may be less than the size of the given buffer. If so, that means not all of the buffer could be filled with received data - only the first `n` bytes were.

### SerialConfigure

Configure a serial port (parity, data bits, blocking, handshaking, etc).

### PrinterPortGetInfo

```rust
enum PrinterPortMode {
	/// Standard 'Centronics' uni-directional printer interface
	Spp,
}

struct PrinterPortInfo {
	name: &'static str,
	mode: PrinterPortMode
}

fn PrinterPortGetInfo(device: u8) -> Option<PrinterPortInfo>;
```

Gets information about any printer ports the system has.

### PrinterPortGetState

```rust
struct PrinterPortStatus(u8);

fn PrinterPortGetStatus(device: u8) -> PrinterPortStatus;
```

Gets the status of the given printer port. This includes buffer status, and signals from the printer including 'Paper Error'.

### PrinterPortSendBytes

```rust
fn PrinterPortSendBytes(device: u8, data: &[u8]) -> Result<usize, Error>;
```

Sends bytes to the printer. The 'strobe' line is raised after each byte is written, and the system waits until the remote device lowers the `busy` pin. This function blocks until all the bytes have been sent, an error occurs or some timeout is reached. If the result is `Ok(n)` then `n` indicates how many bytes were successfully sent.

### TimeGet

```rust
fn TimeGet() -> (u32, u8)
```

Get the current wall time. The Neotron BIOS does not understand time zones, leap-seconds or the Gregorian calendar. It simply stores time as an incrementing number of seconds since some epoch, and the number of video frames (at 60 Hz) since that second began. A day is assumed to be exactly 86,400 seconds long. This is a lot like POSIX time, except we have a different epoch. The Neotron epoch is 2000-01-01T00:00:00Z.

### TimeSet

```rust
fn TimeSet(seconds_since_epoch: u32, frames_since_second: u8);
```

Set the current calendar time.

### I2CBusGetInfo

Gets information about the I2C buses in the system.

### I2CBusWriteRead

Writes data to the I2C bus, then reads data.

```rust
fn I2CBusWriteRead(bus: u8, address: u8, out_buffer: &[u8], in_buffer: &mut [u8]) -> Result<(), Error>;
```

Performs the following operations (courtesy of the Embedded HAL documentation):

```
Master: ST SAD+W     O0     O1     ... OM     SR SAD+R        MAK    MAK ...    NMAK SP
Slave:           SAK    SAK    SAK ...    SAK          SAK I0     I1     ... IN
```

Where:

	* ST = start condition
	* SAD+W = slave address followed by bit 0 to indicate writing
	* SAK = slave acknowledge
	* Oi = ith outgoing byte of data
	* SR = repeated start condition
	* SAD+R = slave address followed by bit 1 to indicate reading
	* Ii = ith incoming byte of data
	* MAK = master acknowledge
	* NMAK = master no acknowledge
	* SP = stop condition

### SPIBusGetInfo

Returns information about each SPI bus.

### SPIDeviceGetInfo

Returns information about each SPI device (specified by a unique Chip Select) on a given SPI bus. Each device has its own word length, which may not be 8 bits.

### SPIDeviceTransfer

Write to and read from an SPI device.

```rust
enum MutableDataBuffer<'a> {
	U8(&'a mut [u8]),
	U16(&'a mut [u16]),
	U32(&'a mut [u32]),
}

fn SPIDeviceTransfer(bus: u8, device: u8, buffer: MutableDataBuffer) -> Result<(), Error>;
```

### SPIDeviceWrite

Write to an SPI device, discarding any read data.

```rust
enum DataBuffer<'a> {
	U8(&'a [u8]),
	U16(&'a [u16]),
	U32(&'a [u32]),
}

fn SPIDeviceTransfer(bus: u8, device: u8, buffer: DataBuffer) -> Result<(), Error>;
```

### USBDevicexxx

This is where the computer is a USB Host and we want to access any attached USB devices.

Add a bunch of functions here along the lines of the USB OHCI. Then check we can implement them using a Tiva-C 123 and an STM32H7.

### GPIOxxx

These functions are for finding all of the GPIO pins on the system. We assume the pins have a numeric Arduino-style numbering. There's a function which takes a slice of pins - if possible the BIOS should condense those into a singla atomic port write.

### GpioPortGetInfo

Gets information about the GPIO ports on this system.

```rust
struct GpioPortInfo {
	name: &'static str,
	num_pins: u8,
}

fn GpioPortGetInfo(port: u8) -> Option<GpioPortInfo>;
```

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
	name: &'static str,
	state: GpioPinMode
}

fn GpioPinGetInfo(port: u8, pin: u8) -> Option<GpioPinInfo>;
```

### GpioPinSetMode

Sets the mode for a GPIO pin.

```rust
fn GpioPinSetMode(port: u8, pin: u8, mode: GpioPinMode) -> Result<(), Error>;
```

### GpioPinSetLevel

Sets the output level for an Output GPIO pin. The behaviour if this pin is not currently configured as an output is undefined.

```rust
fn GpioPinSetLevel(port: u8, pin: u8, high: true) -> Result<(), Error>;
```

### KeyboardRead

Reads a scan-code from the attached keyboard. The BIOS decodes the scan-codes and we just get make/break and a key number. Mapping the key number to a particular character based on a particular keyboard layout is a job for the OS. This routine does not block. The system only supports a single keyboard - it may be a PS/2 device, or the BIOS may implement basic USB Host support for USB HID devices.

```rust
struct Key(u8);

enum KeyboardEvent {
	Make(Key),
	Break(Key),
}

fn KeyboardRead() -> Option<KeyboardEvent>;
```

### KeyboardSetLed

```rust
fn KeyboardSetLed(num_lock: bool, caps_lock: bool, scroll_lock: bool);
```

Sets the keyboard status LEDs. Handling Num Lock, Caps Lock and Scroll Lock is up to the Operating System.

### DelayMs

```rust
fn DelayMs(ms: u16);
```

Delays a given number of milliseconds.

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
	rows: u8,
	cols: u8,
	buffer_size: usize,
	attr_format: AttrFormat,
	num_pages: u8,
	support_soft_font: bool,
}

enum ChunkyPixelFormat {
	/// 1 bit per pixel monochrome
	Grey1,
	/// Up to 8-bit greyscale
	Grey8,
	/// Up to 8-bit colour, packed as bytes.
	Colour8,
	/// Up to 16-bit colour, packed as 16-bit words.
	Colour16,
	/// Up to 24-bit colour, packed in 32-bit words
	Colour32
}

struct VideoChunkyBitmapModeInfo {
	width: u16,
	height: u16,
	buffer_size: usize,
	num_pages: u8,
	attr_format: ChunkyPixelFormat,
}

enum PlanarPixelFormat {
	/// 3 planes, 1-bpp each
	Colour3,
	/// 3 planes, 8-bpp (i.e. one byte per pixel) each
	Colour24,
}

struct VideoPlanarBitmapModeInfo {
	width: u16,
	height: u16,
	buffer_size: usize,
	num_pages: u8,
	attr_format: PlanarPixelFormat,
}

enum VideoModeInfo {
	Text(VideoTextModeInfo),
	ChunkyBitmap(VideoChunkyBitmapModeInfo),
}

fn VideoModeGetInfo(mode: u8) -> Option<VideoModeInfo>;
```

Gets information about a specific video mode.

### VideoModeGet

```rust
fn VideoModeGet() -> u8;
```

Gets which video mode we're currently in.

### VideoModeSet

```rust
fn VideoModeSet(mode: u8, buffer: *mut u8, buffer_len: usize) -> Result<(), Error>;
```

Selects a new video mode. A pointer must be given to the area to be used for storing the graphics / text data. This region must be at least as large as the `buffer_size` value given in `VideoModeGetInfo`.

### VideoModeFontGet

Get a pointer to the current text-mode font. The specific layout of this font will depend on the current video mode.

### VideoModeFontSet

```rust
fn VideoModeFontSet(font: *mut u8, font_len: usize, glyph_width_pixels: u8, glyph_height_rows: u8) -> Result<(), Error>;
```

Change the current text mode font. An error is returned if the current mode doesn't support soft fonts, or if the width/height parameters are not compatible with the current mode, or if the `font_len` does not equal `int((glyph_width_pixels + 7) / 8) * glyph_height_rows * 256`.

Neotron uses MS-DOS / IBM CodePage 850, so there are only 256 glyphs in a font. Unicode text must be mapped to this set of glyphs for display on-screen. Alternative the OS can implement its own rendering system and simply draw pixels using the BIOS.

### VideoModePageSet

```rust
fn VideoModePageSet(page: u8) -> Result<(), Error>;
```

Changes the currently displayed video page (if the current video mode supports multiple pages).

### VideoModeGetBuffer

```rust
fn VideoModeGetBuffer() -> *mut u8;
```

Gets a pointer to a specific page in the video buffer (an array of text characters and attributes, or an array of pixels, depending on the mode). Writing to this buffer during the rendering period (i.e. without waiting using `DelayVblank`) may cause temporarily graphical glitches, but won't crash the system.

Writing outside the bounds of the buffer may crash the system.

### VideoHookScanline

```rust
fn VideoHookScanline(scan_line: u16, fn: impl FnOnce());
```

Supply a function to be called just before the given scan-line is rendered. The function is automatically un-hooked before it is called.

### AudioGetInfo

Gets information about the audio system - number of tone channels, PCM sample rate, etc.

### AudioPlayTone

Plays an audio tone on a given channel.

### AudioBufferSamples

If supported, buffers PCM audio samples to be played. You should use DelayVblank to synchronise playback with the video.
