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
* Co-operative multi-tasking through an event-loop / task-scheduler

Selected non-goals (i.e. things we aren't going to support) include:

* Multi-threading
* Running multiple processes
* Virtual Memory (although we might support a special API for paging to and from disk)
* Run-time filesystem mounting/unmounting

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
FOLDER       <DIR> 2019-09-13 20:20:11

0:/> dir 1:/FOLDER
Listing files in 1:/FOLDER
BAR.TXT      1234  2019-10-18 10:11:13 ---

0:/> cd 1:/FOLDER

1:/FOLDER> dir
Listing files in 1:/FOLDER
BAR.TXT      1234  2019-10-18 10:11:13 ---

1:/>
```

## Special Devices

There are special devices which look like files, but are not. They have names which are like volumes, but contain a '$' character. It is an error to try and use a regular volume with `$` in the name.

* `PRN0$:` - The printer. You can copy files here to print them.
* `CON$:` - The console. You can read from here to get text input and write here to put text on the screen.
* `KBD$:` - The raw keyboard. You can read from here to get raw keyboard events that you can't get through the `CON$:` device (such as Up Arrow key, or Page Down).
* `SER0$:` - The first RS-232 serial device.
* `SER1$:` - The second RS-232 serial device.
* `TONE0$:` - The first tone generator device.
* `PCM0$:` - The first PCM device (write for playback, read for record).

Additional parameters may be specified after the `$`, separated by `;`. For example:

* `SER0$bps=9600;parity=N:` - The first RS-232 serial device, at 9600 bps.
* `PCM0$channels=2;bit=8;samplerate=8000:` - A PCM interface configured for stereo 8kHz 8-bit.

Special devices do not support filenames or paths. To access something like a network volume, the volume must be mounted and given a normal volume ID. The API for this is TBD.

## File Permissions

Files can be opened as read-only (write-none), write-truncate or write-append. Because Neotron is single-tasking and non-re-entrant, there is no support for lock files or exclusive file creation, but files are locked when they are opened so they can only be opened once. Files on disk support the standard FAT16/FAT32 attributes - Archive, Read-Only, Hidden and System. Seeking uses 32-bit byte offsets, giving a maximum file size of 4 GiB.