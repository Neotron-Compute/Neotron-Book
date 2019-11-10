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
* Tracking wall time
* Delaying for defined periods of time
* Hooking scan-line interrupts
* Powering off / rebooting

While you could write an application which uses the BIOS directly, most applications will use the higher-level APIs exposed by the OS.

## Calling a BIOS API

On ARM systems, calling a kernel API is usually done through a `SWI` or `SVC` machine instruction. This effectively triggers an interrupt, putting the CPU into interrupt mode where it starts executing the `SWI` exception handler. Unfortunately, passing arguments to the `SWI` Exception handler can only be done by stuffing pointers into specific registers, which means you can't write it all in pure Rust (or C for that matter) without using unstable in-line assembly.

Monotron instead used a simple alternative - when the application was started by the OS, the OS passed it a pointer to a structure of function pointers. You can think of this as being like an old-fashioned jump table. When the application wanted to get the OS to do something, the application just called the appropriate function through the given pointer.

We're going to take the same approach for the Neotron BIOS. The BIOS is responsible for starting the system and performing hardware initialisation. It will also search for the OS, either in Flash ROM, on disk, or perhaps even loaded over a UART. The OS then has its initialisation function called, to which the structure of BIOS function calls is passed. The initialisation function is specified as a function pointer stored at a specific offset (most likely the first four bytes of the OS image).

## Timeouts

Some functions accept a timeout argument. If this argument is `None`, then the function blocks. Otherwise the function waits for up to the period specified, or the operation is complete, whichever occurs first.

```rust
struct Timeout(u16);

impl Timeout {
	fn frames(frames: u16) -> Timeout;
	fn milliseconds(ms: u16) -> Timeout;
	fn microseconds(us: u16) -> Timeout;
}
```

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
	vendor: String8,
	product: String16,
	revision: String4,
	sector_size: u16,
	num_sectors: u32,
	removable: bool,
	soft_eject: bool,
}

struct DriveSectorAddress(u32);

fn DriveGetInfo(drive: u8) -> Option<DriveInfo>;
```

Returns information about any fixed disks in the system. The OS will call with incrementing values of `drive` until this function returns `None`. The BIOS knows nothing of MBR or GPT partition tables - it only knows how to read, write and verify sectors.

Note that if you have 512 byte sectors, the largest supported block device is `2**32 * 512 bytes = 2 TiB`. Also note that the FAT16/FAT32 file system used by the Neotron OS requires 512 byte sectors, so support for other sector sizes is TBD; the only likely candidate is a CD-ROM where the ISO9660 format uses 2048 byte sectors.

Disks might include:

* SD cards connected to high-speed 4-bit SD Card interfaces
* SD cards connected to SPI peripherals
* An IDE hard disk connected to appropriate controllers
* A LUN on a SCSI device connected to an appropriate controller
* USB flash keys connected to a USB Host port (where the BIOS is operating the USB stack and not the OS)
* A PC-style floppy drive connected to an appropriate controller

Floppy drives with built-in DOS, such as those used with the Commodore 64, are not listed as drives, but instead a CBM Serial interface would appear as a serial port you can send commands down.

### DriveSectorRead

```rust
fn DriveSectorRead(drive: u8, sector: DriveSectorAddress, buffer: &mut [u8]) -> Result<(), Error>;
```

Reads a sector or sectors from a block device. Blocks operation until the read is complete, or an error occurs. The number of sectors to be read is the same as the largest number of whole sectors that fit into `buffer`. For example, if `buffer.len() == 1025` and the sector size is 512 bytes, two sectors will be read, leaving the final 1 byte untouched.

### DriveSectorWrite

```rust
fn DriveSectorWrite(drive: u8, sector: DriveSectorAddress, buffer: &[u8]) -> Result<(), Error>;
```

Writes a sector or sectors to a block device. Blocks operation until the write is complete, or an error occurs. The number of sectors to be written is the same as the largest number of whole sectors that fit into `buffer`. For example, if `buffer.len() == 1025` and the sector size is 512 bytes, two sectors will be written, leaving the final 1 byte untouched.

### DriveSectorVerify

```rust
fn DriveSectorVerify(drive: u8, sector: DriveSectorAddress, buffer: &[u8]) -> Result<(), Error>;
```

Verifies that a block device contains the given sector or sectors at the specified address. Blocks operation until the write is complete, or an error occurs. The number of sectors to be verified is the same as the largest number of whole sectors that fit into `buffer`. For example, if `buffer.len() == 1025` and the sector size is 512 bytes, two sectors will be verified, leaving the final 1 byte untouched.

### DriveCommandSend

```rust
enum DriveCommand {
	Reset,
	TestUnitReady,
	Start,
	Stop,
	Eject,
	Load,
	MediumRemovalAllow,
	MediumRemovalPrevent,
}

fn DriveCommandSend(drive: u8, command: DriveCommand) -> Result<(), Error>;
```

Sends a command to the device. The implementation of these commands is controller-specific, but they have an obvious analogy in the SCSI-2 command set and so should be generally applicable.

### SerialGetInfo

```rust
enum SerialType {
	/// A MIDI interface
	Midi,
	/// A Commodore Serial interface
	Cbm,
	/// An RS-485 bus
	Rs485,
	/// An RS-232 interface
	Rs232,
	/// An RS-232 interface, but at TTL voltages. Typically used with an
	/// FTDI FT232 cable.
	TtlUart,
	/// A USB Device implementing Communications Class Device (also known as
	/// a USB Serial port). The USB Device implementation may be on-chip, or off-chip.
	UsbCdc,
}

struct SerialInfo {
	name: &'static str,
	type: SerialType,
}

fn SerialGetInfo(device: u8) -> Option<SerialInfo>;
```

Get information about the Serial ports in the system. Serial ports are ordered octet-oriented pipes. You can push octets into them using a 'write' call, and pull bytes out of them using a 'read' call. They have options which allow them to be configured at different speeds, or with different transmission settings (parity bits, stop bits, etc) - you set these with a call to `SerialConfigure`. They may physically be a MIDI interface, an RS-232 port or a USB-Serial port. There is no sense of 'open' or 'close' - that is an Operating System level design feature. These APIs just reflect the raw hardware, in a similar manner to the registers exposed by a memory-mapped UART peripheral.

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

fn SerialConfigure(device: u8, config: SerialConfig) -> Result<(), Error>;
```

Set the options for a given serial device. An error is returned if the options are invalid for that serial device.

### SerialWrite

```rust
fn SerialWrite(device: u8, data: &[u8], timeout: Option<Timeout>) -> Result<usize, Error>;
```

Write bytes to a serial port. There is no sense of 'opening' or 'closing' the device - serial devices are always open. If the return value is `Ok(n)`, the value `n` may be less than the size of the given buffer. If so, that means not all of the data could be transmitted - only the first `n` bytes were.

### SerialRead

```rust
fn SerialRead(device: u8, data: &mut [u8], timeout: Option<Timeout>) -> Result<usize, Error>;
```

Read bytes from a serial port. There is no sense of 'opening' or 'closing' the device - serial devices are always open. If the return value is `Ok(n)`, the value `n` may be less than the size of the given buffer. If so, that means not all of the buffer could be filled with received data - only the first `n` bytes were.

### PrinterPortGetInfo

```rust
enum PrinterPortMode {
	/// Standard 'Centronics' uni-directional printer interface.
	Spp,
	/// Nibble-mode bi-directional support
	Bidirectional,
	/// Enhanced Parallel Port
	Epp,
	/// Enhanced Control Port
	Ecp
}

struct PrinterPortInfo {
	name: &'static str,
	mode: PrinterPortMode
}

fn PrinterPortGetInfo(device: u8) -> Option<PrinterPortInfo>;
```

Gets information about any printer ports the system has.

### PrinterPortGetStatus

```rust
struct PrinterPortStatus(u8);

impl PrinterPortStatus {
	/// Read DB25 pin 10
	fn is_ack_asserted(&self) -> bool;
	/// Read DB25 pin 11
	fn is_busy_asserted(&self) -> bool;
	/// Read DB25 pin 12
	fn is_paper_out_asserted(&self) -> bool;
	/// Read DB25 pin 13
	fn is_select_in_asserted(&self) -> bool;
	/// Read DB25 pin 15
	fn is_error_asserted(&self) -> bool;
	/// Get all the bits together
	fn get_byte(&self) -> u8;
}

fn PrinterPortGetStatus(device: u8) -> Result<PrinterPortStatus, Error>;
```

Gets the status of the given printer port.

### PrinterPortSetControl

```rust
struct PrinterPortControl(u8, u8);

impl PrinterPortControl {
	/// Control DB25 pin 1 (enabled is low)
	fn set_strobe(&mut self, enabled: bool);
	/// Control DB25 pin 14 (enabled is low)
	fn set_auto_linefeed(&mut self, enabled: bool);
	/// Control DB25 pin 16 (enabled is high)
	fn set_reset(&mut self, enabled: bool);
	/// Control DB25 pin 17 (enabled is low)
	fn set_select_printer(&mut self, enabled: bool);
	/// Does not appear on the port - controls whether PrinterPortSendBytes blocks or not.
	fn set_blocking(&mut self, enabled: bool);
	/// Set all the bits at once
	fn set_byte(&mut self, byte: u8);
}

fn PrinterPortSetControl(device: u8, control: PrinterPortControl) -> Result<(), Error>;
```

Gets the status of the given printer port. This includes buffer status, and signals from the printer including 'Paper Error'.

### PrinterPortSetData

```rust
fn PrinterPortSetByte(device, byte: u8) -> Result<(), Error>;
```

Sets a byte on the data bus for the parallel port. Does not operate the Strobe line - you have to do that manually using `PrinterPortSetControl`.

### PrinterPortSendBytes

```rust
fn PrinterPortSendBytes(device: u8, data: &[u8], timeout: Option<Timeout>) -> Result<usize, Error>;
```

Sends bytes to the printer. In basic 'SPP' mode the 'strobe' line is raised after each byte is written, and the system waits until the remote device lowers the `busy` pin. The operation in other modes is TBD. This function blocks until all the bytes have been sent, an error occurs or the optional timeout is reached. If the result is `Ok(n)` then `n` indicates how many bytes were successfully sent.

### TimeGet

```rust
struct Time {
	seconds_since_epoch: u32,
	frames_since_second: u8
}

fn TimeGet() -> Time;
```

Get the current wall time. The Neotron BIOS does not understand time zones, leap-seconds or the Gregorian calendar. It simply stores time as an incrementing number of seconds since some epoch, and the number of video frames (at 60 Hz) since that second began. A day is assumed to be exactly 86,400 seconds long. This is a lot like POSIX time, except we have a different epoch. The Neotron epoch is 2000-01-01T00:00:00Z. It is highly recommend that you store UTC in the BIOS and use the OS to handle time-zones.

### TimeSet

```rust
fn TimeSet(time: Time);
```

Set the current wall time to the given value. See [TimeGet](#timeget).

### I2CBusGetInfo

```rust
enum I2CBusSpeed {
	Low10kbps,
	Standard100kbps,
	Fast400kbps,
	FastPlus1Mbps,
	High3M4bps,
	UltraFast5Mbps,
}

struct I2CBusInfo {
	name: &'static str,
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

### SpiBusGetInfo

```rust
struct SpiBusInfo {
	name: &'static str,
	num_chip_selects: u8,
	max_speed_bps: u32,
}

fn SpiBusGetInfo(bus: u8) -> Option<SpiBusInfo>;
```

Gets information about the SPI buses in the system.

### SpiBusSetSpeed

```rust
fn SpiBusSetSpeed(bus: u8, speed_bps: u32) -> Result<u32, Error>;
```

Configure the speed of the SPI bus. Where arbitrary speeds are not supported by the bus, the BIOS will select the closest supported speed which is not greater than the given speed.

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

fn SPIDeviceTransfer(bus: u8, device: u8, buffer: DataBuffer) -> Result<(), Error>;
```

### USBxxx

This is where the Neotron system is a USB Host and the OS wants to access any attached USB devices.

*TODO: Add a bunch of functions here along the lines of the USB OHCI. Then check we can implement them using a Tiva-C 123 and an STM32H7.*

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

### MouseGetPosition

Gets the mouse's relative movement since the last call. The system only supports a single mouse - it may be a Serial mouse, a PS/2 mouse or a USB HID mouse, depending on the hardware and the BIOS implementation. A typical implementation would put a PS/2 mouse into `remote` mode, and then poll it for position data once per frame - as standard a PS/2 mouse is in `stream` mode and would supply an update 100 times per second, which is faster than the video refresh rate.

```rust
struct MouseData {
	/// Relative X movement since last call
	x_movement: i16,
	/// Relative Y movement since last call
	y_movement: i16,
	/// Relative scroll wheel movement since last call
	scroll_movement: u8,
	/// true if currently pressed down
	left_button: bool,
	/// true if currently pressed down
	right_button: bool,
	/// true if currently pressed down
	middle_button: bool,
}

enum MouseBlocking {
	Block,
	NonBlocking
}

fn MouseGetPosition(timeout: Option<Timeout>) -> Option<MouseData>;
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
fn VideoModeSet(mode: u8, buffer: *mut u8, buffer_len: usize) -> Result<(), Error>;
```

Selects a new video mode. A pointer must be given to the area to be used for storing the graphics / text data. This region must be at least as large as the `buffer_size` value given in `VideoModeGetInfo`.

### VideoModeFontGet

```rust
struct FontInfo {
	font: *mut u8,
	font_len: usize,
	glyph_width_pixels: u8,
	glyph_height_rows: u8,
}

fn VideoModeFontGet() -> Result<FontInfo, Error>;
```

Get a pointer to the current text-mode font. See [VideoModeFontSet](#videomodefontset).

### VideoModeFontSet

```rust
fn VideoModeFontSet(font_info: FontInfo) -> Result<(), Error>;
```

Change the current text mode font. An error is returned if the current mode doesn't support soft fonts, or if the width/height parameters are not compatible with the current mode, or if the `font_len` does not equal `int((glyph_width_pixels + 7) / 8) * glyph_height_rows * 256`.

Neotron uses MS-DOS / IBM CodePage 850, so there are only 256 glyphs in a font. Unicode text must be mapped to this set of glyphs for display on-screen. Alternative the OS can implement its own rendering system and simply draw pixels using the BIOS.

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

### AudioGetInfo

Gets information about the audio system - number of tone channels, PCM sample rate, etc. Most Neotron systems will use the 800x600 @ 60 Hz video mode with a 37.878 kHz vertical frequency. The PCM sample rates will therefore likely be some integer fraction of that vertical frequency, as the audio playback needs to be interleaved with the video generation.

```rust
/// Bit-depth for a PCM mode
enum AudioBitsPerSample {
	Eight = 8,
	Sixteen = 16,
}

/// A supported set of settings for the PCM playback engine
struct AudioPcmMode {
	// 1 for mono, 2 for stereo, etc
	num_channels: u8,
	// e.g. 44100 for CD Quality Audio
	samples_per_second: u16,
	// e.g. AudioBitsPerSample::Sixteen for 16-bit PCM audio
	bits_per_sample: AudioBitsPerSample,
}

/// Tones we can get from the synthesiser
enum Waveform {
	Square,
	Sawtooth,
	Sine,
	Noise,
	Custom(&[u8])
}

struct AudioInfo {
	/// How many simultaneous tones can the hardware play?
	num_tone_channels: u8,
	/// The set of supported waveforms on each tone channel
	waveforms: &[Waveform],
	/// Does tone generator support envelope control?
	envelope_supported: bool,
	/// Does tone generator support modulating with a low-frequency oscillator?
	lfo_modulation_supported: bool,
	/// Does tone generator support stereo panning?
	stereo_tones_supported: bool,
	/// Various PCM audio output (playback) modes supported. May be an empty list.
	pcm_output_modes: &[AudioPcmMode],
	/// Various PCM audio input (record) modes supported. May be an empty list.
	pcm_input_modes: &[AudioPcmMode],
}

fn AudioGetInfo() -> Option<AudioInfo>;
```

### AudioPlayTone

Plays an audio tone on a given channel.

```rust
/// Controls the 'shape' of the volume of a tone as it plays
struct Envelope {
	/// Rise time for the signal. 0 => 2ms. 15 => 8s.
	attack: u8,
	/// Decay time for the signal. 0 => 6ms. 15 => 24s.
	decay: u8,
	/// Volume level of the sustained signal. 0 => silence, 15 => maximum volume
	sustain: u8,
	/// Release time for the signal. 0 => 6ms. 15 => 24s.
	release: u8,
}

/// A tone is a combination of a Waveform and a Frequency to play that waveform at.
struct Tone {
	/// The waveform to play
	waveform: Waveform,
	/// The frequency to play it at, in Hertz.
	frequency_hz: u32
}

fn AudioPlayTone(tone_channel: u8, tone: Tone, lfo: Option<Tone>, envelope: Option<Envelope>, peak_volume: u8, pan: i8) -> Result<(), Error>;
```

ADSR envelope filtering controls the volume of the tone as it plays. In general terms, the envelope looks like:

```
    /\
   /  \
  /   ^\________
 /^   |   ^     \
/ |   |   |     ^\
  |   |   |     |
Attack| Sustain |
    Decay     Release
```

### AudioBufferSamples

If supported, queues PCM audio samples to be played. You should use DelayVblank to synchronise playback with the video. The samples are not copied - only the pointer and length is stored. It is therefore important to ensure the samples are allocated in a static buffer, or at least a buffer that lives long enough for them to be played.

In a stereo mode, the left and right samples are interleaved. In a 16-bit mode, each sample is two bytes, in little-endian format. Thus, in a 16-bit stereo mode, there are four bytes per sample:

```
+----------+----------+-----------+-----------+------------+...
|N Left MSB|N Left LSB|N Right MSB|N Right LSB|N+1 Left MSB|
+----------+----------+-----------+-----------+------------+...
```

```rust
/// Configures the PCM playback system
fn AudioConfigureOutput(mode_index: Option<u8>) -> Result<(), Error>;

/// Returns the tag of the most recent queued sample buffer to have been entirely played.
fn AudioSamplesTagReleased() -> Option<u8>;

/// Queues samples for playback. Supplying a unique tag allows the
/// `AudioSamplesTagReleased` function to inform the caller when these samples
/// have been played.
fn AudioQueueSamples(tag: u8, data: *const u8, data_len: usize);
```

### InterruptQuery

```rust
fn InterruptQuery() -> u8;
```

The Neotron BIOS has up to 8 external interrupts, numbered 0 to 7. A bit is set in the returned value for each interrupt that has been triggered. Calling this function automatically clears the interrupts, so the caller must ensure they are actioned. The mapping of interrupts to hardware is configured at the OS level - probably through a configuration file. The primary purpose of these interrupts is to avoid polling external hardware unnecessarily. You should call this function at least once per frame.

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
