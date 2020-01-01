# The Neotron OS

The Neotron OS is the hardware-agnosting implementation of the Neotron API. It uses the BIOS to interact with the hardware, allowing it to run on different hardware platforms with the minimum number of changes.

```
+-------+-------------+
|       |             |
| Shell | Application |
|       |             |
+=======+=============+
|                     |
|  Operating System   |
|                     |
+=====================+
|                     |
|        BIOS         |
|                     |
+---------------------+
|                     |
|    Raw Hardware     |
|                     |
+---------------------+
```

It should implement:

* File/Device handling (open, read, write, seek, etc)
* A FAT16/32 compatible filesystem with MS-DOS MBR partitions (max 2 TiB disk size - 2<sup>32</sup> sectors of 512 bytes)
* A text-mode console, with cursor
* Basic line-based input and character based raw input
* Simple bitmap graphics primitives (lines, rectangles, fill, etc)
* An audio synthesiser
* Input/output stream handling
* Digital (Atari) joystick support
* A TCP/IP networking stack
* MIDI support
* Jumping to applications located in RAM or ROM, giving them access to the OS
* Supporting special 'shell' applications which can chain-load small programs (e.g. command-line utilities) without being unloaded
* Memory allocation/deallocation routines

Selected non-goals (i.e. things we aren't going to support) include:

* Co-operative multi-tasking
* Pre-emptive multi-threading
* Virtual Memory (although we might support a special API for paging to and from disk)
* Processes / process isolation
* Run-time filesystem mounting/unmounting (although by time-limiting any cached data, we can support the changing of media at run-time)

Our ultimate goal is that one day we could offer a version of the Rust Standard Library (`std`) which uses the Neotron OS API - albeit without the sub-process support.

Programming on the Neotron usually involves first loading the specific interpreter for that language - unless you've swapped out the standard shell for something like a BASIC interpreter.

## Filename Convention

Files live in volumes. Volumes on fixed or removable disks must be formatted with the FAT12, FAT16 or FAT32 filesystem and live either live in a disk partition identified by a PC Master Boot Record (with a suitable Filesystem Type), or live at the start of the drive. Volumes are allocated a volume number, starting with `0:`, then `1:` etc. File paths are relative to a volume number. Paths are separated with a `/` character. The root directory is called `/`.

When a volume has a name, you can refer to it as `<name>:` instead of `<id>:`. For example `work:/Documents/hello.txt` refers to a file called `hello.txt` inside a folder called `Documents` which lives on a volume called `work`.

It is an error to try and create a file containing a `:` character. Any files or directories which do contain invalid characters are as if the character was replaced with `_`. When a directory contains multiple files with the same name, any attempt to read the file will only be able to access the one that is found first. Any attempt to create a new file will cause any existing files of the same name in the same folder to be deleted.

For example:

```console
0:/> dir 1:
Listing files in 1:/
FOO.TXT      1234  2019-10-17 23:20:01 A--
COMMANDS.SH    34  2019-10-17 23:20:11 AR-
FOLDER       <DIR> 2019-09-13 20:20:11 D--

0:/> dir 1:/FOLDER
Listing files in 1:/FOLDER
BAR.TXT      1234  2019-10-18 10:11:13 ---

0:/> cd 1:/FOLDER

1:/FOLDER> dir
Listing files in 1:/FOLDER
BAR.TXT      1234  2019-10-18 10:11:13 ---

1:/>
```

Unlike MS-DOS, which has a separate *current directory* for each volume (i.e. drive letter), Neotron maintains a UNIX-like single *current directory* which includes the name of the current volume. Where a volume is given without a directory path, the root directory (`/`) is presumed.

## File Permissions

Files can be opened as read-only (write-none), write-truncate or write-append. Because Neotron is single-tasking and non-re-entrant, there is no support for lock files or exclusive file creation, but files are locked when they are opened so they can only be opened once at any one time. Files on disk support the standard FAT16/FAT32 attributes - Archive, Read-Only, Hidden and System. Seeking uses 64-bit byte offsets, although note that FAT32 limits files to 4 GiB in size.

## File Operations

A file handle, whether pointing to a file on a volume, or to a [Special Device](#special-devices), generally has the following functions available. These functions broadly follow the POSIX model.

* `read` - obtain some bytes from this handle. A `&mut [u8]` is supplied, and either an integer is returned which reports how many bytes in that buffer were filled by the function call, or an error is returned. The call will wait until the buffer is full, or the timeout has been met - the timeout may be zero, some finite value, or infinite. Some files (e.g. block devices) only support reading fixed size blocks and attempts to read any other size block will give an error.

* `write` - sent some bytes to this handle. A `&[u8]` is supplied, and either an integer is returned which reports how many bytes from that buffer were sent by the function call, or an error is returned. The call will wait until all the bytes have been sent, or the timeout has been met - the timeout may be zero, some finite value, or infinite.  Some files (e.g. block devices) only support writing fixed size blocks and attempts to write any other size block will give an error.

* `seek` - adjust the current pointer in the file. Files opened for reading start with the pointer at the beginning. Files opened for writing start with the pointer at the end. Some special devices are not seekable and this function will return an error. The pointer can be adjusted with a number of bytes relative to a) the current position, b) the start of the file, or c) the end of the file. Seeking beyond the end of a file on disk is not supported.  Some files (e.g. block devices) only support seeking to particular offsets (e.g. a multiple of the block size) and attempting to seek to any other offset will give an error.

## Special Devices

There are special devices which look like files, but are not. They have names which are like volumes, but contain a '$' character. It is an error to try and use a regular volume with `$` in the name.

* `PRN0$:` - The printer. Write here to send data to the printer, read to get print status.
* `CON$:` - The console. You can read from here to get line-buffered text input and write here to put UTF-8 text on the screen (with ANSI code sequence support).
* `KBD$:` - The raw keyboard. You can read from here to get raw keyboard events (such as Up Arrow key, or Page Down).
* `SER0$:` - The first RS-232 serial device (read/write).
* `SER1$:` - The second RS-232 serial device (read/write).
* `TONE0$:` - The first tone generator device (write-only).
* `PCM0$:` - The first PCM device (write for playback, read for record).
* `JS0$:` - The first Joystick (read-only).
* `JS1$:` - The second Joystick (read-only).
* `TIME$:` - Gets/sets the system time (read/write).
* `GFX$:` - A bitmap framebuffer device.

Additional parameters may be specified after the `$`, separated by `;`. For example:

* `SER0$bps=9600;parity=N;timeout=100:` - The first RS-232 serial device, at 9600 bps, with a 100 frame read/write timeout.
* `PCM0$channels=2;bit=8;samplerate=8000:` - A PCM interface configured for stereo 8kHz 8-bit.

Special devices do not support filenames or paths. To access something like a network volume, the volume must be mounted and given a normal volume ID. The API for this is TBD.

### Writing to `CON$:`

The Console device `CON$:`, like every other file, is a bi-direction octet pipe. Unfortunately, one octet is not enough to represent the full set of characters a modern computer needs to support. Rust handles this by insisting that text (`&str` string slices and `String` owned strings) is in Unicode Translation Format 8 (UTF-8). This is a mechanism by which Unicode characters (which are around 21 bits in size) can be encoded as between one and six octets. The useful property is that the first 127 Unicode characters map to a single octet, which makes it interchangeable with standard ASCII.

In text mode, most Neotron systems will only have one octet avilable per text cell to record which particular character being displayed in that cell, therefore placing a limit of 256 different glyphs in any given font. The font must therefore also provide a translation function which can convert from a 21-bit Unicode character to an octet for storage in the screen buffer. Typically the fonts will follow some standard 8-bit *code page*, such as [Code Page 850](https://en.wikipedia.org/wiki/Code_page_850), but this is not enforced and aside from some characters failing to render correctly on the screen, is invisible from an application point of view.

Ordinarily the OS will maintain a cursor position and any text written to the Console will be added to the screen at that position. The screen has a nominal width and height, and when the cursor gets to the end of a line it moves down to the start of the next line. The console will also handle the following special characters in the usual way:

| ASCII Code | C Escape Sequence | Keypress | Name            | Function                                         |
|------------|-------------------|----------|-----------------|--------------------------------------------------|
| 0x07       | `\a`              | Ctrl+H   | Bell            | Produces a beep from the speaker                 |
| 0x08       | `\b`              | Ctrl+H   | Backspace       | Move to one character to the left                |
| 0x09       | `\t`              | Ctrl+I   | Tab             | Move to next column which is a multiple of 8     |
| 0x0A       | `\n`              | Ctrl+J   | Line Feed       | Move to start of next line                       |
| 0x0C       | `\f`              | Ctrl+L   | Form Feed       | Clear the screen and move to start of first line |
| 0x0D       | `\r`              | Ctrl+M   | Carriage Return | Move to start of current line                    |

For some text mode applications, more precise control is required over the position of the cursor, the colour of the text, and so on. This is acheived by inserting *ANSI Escape Sequences* into the octet stream. The sequences supported are a subset of those defined in [ECMA-48], and broadly align with those commonly used on Linux/UNIX systems, although support for the specific colours, etc, depends on the video support available in the BIOS. In the following table:

* `ESC` means *Escape*. It is represented by the single Unicode character `U+001B` (`0x1B` in UTF-8).
* `CSI` means *Control Sequence Introducer*. It is represented by the the single Unicode character `U+009B` (or `0xC2 0x9B` in UTF-8), or alternatively the two character sequence `ESC [` (`0x1B 0x5B` in UTF-8).
* The lowercase characters *n* and *m* represent optional integer parameters, rendered as decimal in ASCII using the characters `0` to `9`.
* Rows and Columns are 1 indexed, with `1,1` being the top left of the screen.

[ECMA-48]: https://www.ecma-international.org/publications/files/ECMA-ST/Ecma-048.pdf

| Sequence            | Function                                                          |
|---------------------|-------------------------------------------------------------------|
| ESC c               | Reset to Intial State                                             |
| CSI *n* A           | Cursor Up *n* (default 1) rows                                    |
| CSI *n* B           | Cursor Down *n* (default 1) rows                                  |
| CSI *n* C           | Cursor Forward *n* (default 1) columns                            |
| CSI *n* D           | Cursor Back *n* (default 1) columns                               |
| CSI *n* ; *m* H     | Cursor Position to row *n* (default 1) and column *m* (default 1) |
| CSI *n* [ ; *m* ] m | *Select Graphic Rendition* (see below for valid parameters)       |

The *Select Graphic Rendition* codes are as follows. Note that multiple codes can be specified in one escape sequence, separated by semicolons. Most of the codes only affect text which is subsequently printed to the console - the only exception is the font select commands 10 to 13 which change the font for the entire screen. The loading of custom fonts is performed through a separate OS call.

| SGR Code | Function                                        |
|----------|-------------------------------------------------|
| 0        | All attributes off                              |
| 1        | Subsequent text is bold                         |
| 8        | Subsequent text is concealed (replaced with *)  |
| 4        | Subsequent text is underlined                   |
| 7        | Swap foreground/background colours              |
| 10       | Select default font (usually VGA Code Page 850) |
| 11       | Select alternative font 1 (usually Teletext)    |
| 12       | Select alternative font 2 (user defined)        |
| 13       | Select alternative font 3 (user defined)        |
| 28       | Turns off conceal mode                          |
| 30       | Set foreground colour to Black                  |
| 31       | Set foreground colour to Red                    |
| 32       | Set foreground colour to Green                  |
| 33       | Set foreground colour to Yellow                 |
| 34       | Set foreground colour to Blue                   |
| 35       | Set foreground colour to Magenta                |
| 36       | Set foreground colour to Cyan                   |
| 37       | Set foreground colour to White                  |
| 40       | Set background colour to Black                  |
| 41       | Set background colour to Red                    |
| 42       | Set background colour to Green                  |
| 43       | Set background colour to Yellow                 |
| 44       | Set background colour to Blue                   |
| 45       | Set background colour to Magenta                |
| 46       | Set background colour to Cyan                   |
| 47       | Set background colour to White                  |

The Neotron application library allows access to the console via two means:

* Implicitly, through the use of the `println!` macro.
* Explicitly, through the use of the Standard Output file handle, obtained with `neotron::io::stdout()` or by opening the `CON$:` device for writing.

The Neotron OS doesn't have a concept of 'Standard Error' as distinct from 'Standard Output'. Application libraries may wish to emulate this feature by, for example, printing any text output by `eprintln!` in a different colour.

### Reading from `CON$:`

Performing a read on the console device will block the application until either the Enter key is pressed on the keyboard, or `Control + C` is pressed on the keyboard. If the Enter key is pressed, the UTF-8 encoding of the characters entered by the user are copied to the given buffer (up to as many as will fit). If too many characters are entered to fit in to the given buffer, the Console will beep and the user must use Backspace to remove some characters and free up some buffer space. Characters being entered are echoed to the console (although Conceal mode will help if the user is entering a password).

In the background during this function, the OS is polling the keyboard for key events, handling special keys (like Backspace, or Shift) and when a valid key is pressed, the matching Unicode character is UTF-8 encoded and added to the buffer.

On some Neotron systems, pressing the 'Up' arrow key will restore the previous command line contents (like when using the `readline` library on GNU/UNIX systems).

```rust
use neotron::fs::File;
let mut buffer = [0u8; 16];
/// The standard Neotron file read functions can be used to read from the console.
let mut console = File::open("CON$:").expect("Failed to open console for reading");
match console.read(&mut buffer) {
	Ok(0) => {
		println!("You entered an empty string");
	}
	Ok(n) => {
		// The Neotron OS guarantees this will be valid UTF-8
		let read_string = unsafe { core::str::from_utf8_unchecked(&buffer[0..n]) };
		println!("You entered {:?}", read_string);
	}
	Err(e) => {
		eprintln!("Oh, I got read error {:?} reading from the console", e);
	}
}
/// There is also a Neotron Application Library helper function which reads to a buffer (unlike the normal Rust Standard Library function which reads to a `String`).
let bytes_read = neotron::io::stdin().read_line(&mut buffer).expect("Failed to read from console");
```

### Reading from `KBD$:`

For some applications, such as games, the application will need raw key up/down events rather than the Unicode characters those key events correspond to. For this, there is a device which returns raw key events. Reading from this device is non-blocking. Mapping is performed if the current keyboard layout is non-QWERTY (e.g. the top left letter key on an AZERTY keyboard is `Key::LetterA` not `Key::LetterQ`), but you must perform your own handling of shifted characters (for example how Shift + `.` gives you a `>` character when using a United Kingdom keyboard layout). The easiest way to handle this is to allow users to map their own keys (e.g. "Press the key you want to use for 'Jump': ").

```rust
enum Key {
	LetterA,
	LetterB,
	LetterC,
	LetterD,
	...
	Digit0,
	Digit1,
	...
	Dot,
	Comma,
	PrintScreen,
	Up,
	Down,
	Left,
	Right,
	Home,
	End,
	Delete,
	...
}

struct KeyEvent(u8);

impl KeyEvent {
	unsafe fn from_octet(u8) -> KeyEvent;
	fn is_keydown(&self) -> bool;
	fn is_keyup(&self) -> bool;
	fn get_key(&self) -> Key;
}

use neotron::fs::File;
let mut kb = File::open("KBD$:").expect("Failed to open raw keyboard");
let buffer = [0u8; 16];
/// The standard file read function is used to read from the keyboard
match kb.read(&mut buffer) {
	Ok(0) => {
		println!("No raw key events available");
	}
	Ok(n) => {
		for octet in buffer[0..n].iter() {
			let ev = unsafe { KeyEvent::from_octet(octet) };
			println!("Received event {:?}", ev);
		}
	}
	Err(e) => {
		eprintln!("Oh, I got read error {:?} reading from the keyboard", e);
	}
}

```

### Writing to `KBD$:`

The least significant (1) bit of any byte written to KBD$ sets the *Num Lock* light. The next (2) bit of any byte written to KBD$ sets the *Caps Lock* light. The next (4) bit of any byte sets the *Scroll Lock* light. This is useful if you are using raw keyboard mode to handle key events and want to perform your own Num Lock, Caps Lock and Scroll Lock handling.

### Writing to `PRNx$:`

Every byte written to this device is sent to the Parallel Port, followed by high pulse on the `!STROBE` pin, and a busy-wait for the `!BUSY` pin to go low. If any error signals are activated by the printer, the write is ended early.

### Reading from `PRNx$:`

A read will return one byte which is a bitmask of the status bits.

| Bit Number | Description                       |
|------------|-----------------------------------|
|          0 | 1 when ERROR is active (low)      |
|          1 | 1 when SELECT_IN is active (high) |
|          2 | 1 when PAPER_OUT is active (high) |
|          3 | 1 when ACK is active (low)        |
|          4 | 1 when BUSY is active (low)       |

### Writing to `SERx$:`

A write will block until all of the octets have been transmitted to the remote device at the specified bit rate, or a timeout occurs (if specified).

### Reading from `SERx$:`

A read will block until the given number of octets have been received from the remote device at the specified bit rate, or a timeout occurs (if specified).

### Writing to `TONEx$:`

A tone is specified with three parameters:

* A frequency, in Hertz.
* A waveform (e.g. Square, Sine, Triangle, White Noise, etc)
* A volume (between 1 and 16)

These three parameters are packed into four bytes:

```
 Byte 0        Byte 1        Byte 2        Byte 3
+-------------+-------------+-------------+-------------+
|Frequency LSB|Frequency MSB|Waveform     |Volume       |
+-------------+-------------+-------------+-------------+
```

The buffer written must be a multiple of four octets in length.

### Reading from `TONEx$:`

Not supported.

### Writing to `PCMx$:`

When the PCM device is opened, the number of channels (mono/stereo), sample rate (e.g. 22,050 Hz), bit depth (e.g. 8 bits) are specified. The bytes written are the samples to be played, as little-endian integers. If there are multiple channels, the samples for each channel are supplied in turn before moving on to the next sample (e.g. *Left Sample N*, *Right Sample N*, *Left Sample N+1*, *Right Sample N+1*). You should write sufficiently often to avoid underflowing the internal PCM buffer.

### Reading from `PCMx$:`

When the PCM device is opened, the number of channels (mono/stereo), sample rate (e.g. 22,050 Hz), bit depth (e.g. 8 bits) are specified. The bytes read are the samples which have been recorded, as little-endian integers. If there are multiple channels, the samples for each channel are supplied in turn before moving on to the next sample (e.g. *Left Sample N*, *Right Sample N*, *Left Sample N+1*, *Right Sample N+1*). You should read sufficiently often to avoid overflowing the internal PCM buffer.

### Writing to `JSx$:`

Not supported

### Reading from `JSx$:`

Returns a bitmask of inputs from that joystick:

| Bit Number | Description   |
|------------|---------------|
|          0 | Up            |
|          1 | Down          |
|          2 | Left          |
|          3 | Right         |
|          4 | Fire 1 (B)    |
|          5 | Fire 2 (C)    |
|          6 | Fire 3 (A)    |
|          7 | Fire 4 (Start)|

Yes, the fire buttons are in that order - mainly because on a Master System pad, you only have Fire Button 1 and Fire Button 2, but on a Mega Drive pad, those same pins on the interface correspond to Fire Buttons B and C. If you have a standard Atari/Commodore joystick, you will probably only have Fire Button 1.

The OS will poll the joystick once per video frame, so attempting to read more often than that will given repeated results. A fire button is very likely to be held for multiple frames, so you will need to store the previous reading and check which bits have flipped since last time.

## Threads and Processes

The Neotron OS has no support for running multiple processes, nor for multi-threading, nor for multi-core systems. It is very much like MS-DOS and CP/M in this regard. It does, however, use locks to ensure that should the use of interrupts cause a function to be 're-entered', the situation is caught gracefully rather than leading to system instability. A user is, therefore, free to implement multi-threading within their application if they so wish.

## Memory Allocation

The Neotron BIOS initialise all of the RAM, configures a stack, reserves a further portion of RAM for its own use, and passes the OS a structure which describes the start and end address of each contiguous block of remaining memory (and there may be several if your CPU has multiple separate SRAMs). From this remainder, the Neotron OS allocates what it needs for its own purposes.

The Neotron OS has a built-in heap memory allocation routine (like `malloc`) and matching deallocation routine (like `free`) and it offers these to the currently running application. This saves the application having to include its own memory allocation routines. When an application is loaded, the heap is automatically set to use all of the remaining un-used RAM (which varies depending on the size of the application). Applications are usually (but not always) given the same stack as the OS and the BIOS, and so it is very possible for a badly behaved application to corrupt and/or crash the entire system.

An application can always assume its own RAM for code and global variables starts at address `0x2000_0000`. The memory allocation routines may return an address from some other range (e.g. the AXI SRAM on an STM32H7 is at 0x2400_0000).
