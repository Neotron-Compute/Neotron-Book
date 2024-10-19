# The Neotron Command Shell

## Introduction

The Neotron Shell is a command-line interface OS. It allows the user to give the system commands, including:

* Browsing filesystems on disk drives, such as SD Cards
* Loading and executing applications from a disk drive or ROM Filing System
* Inspecting the contents of RAM and ROM
* Configuring system settings, such as the current video mode and the default video mode

The Shell is built into the OS, which allows it to use a special region of memory. This means it can remain running whilst it loads another application into the main program area. Once that application is started, it can use the shell's memory region as it's own - the OS will re-initialise the shell when the application exits.

Analogies from the PC world would include MS-DOS's `COMMAND.COM` and CP/M's CCP, but at a functional level it's more like the U-Boot Shell.

## Shell Expansion

Any argument containing a substring of the form `${VAR}` has that substring replaced with the value stored in the environment variable called `VAR`.

Any argument containing a `*` character is taken to be a glob and is expanded to all the filenames matching that glob. Note that there are a limited number of arguments that can be given to a command, so attempts to glob more files than that number will generate an error.

## Magic Devices

The OS supports a number of magic device filenames. The shell is unaware of this and just asks the OS to perform the specified operation on the given filenames.

## Shell Commands

Numeric arguments like `length` and `addr` are parsed as decimal unless specified
as hex with an `0x` prefix. A `_` can be used as a digit separator.

Arguments named `path` must be a valid relative or absolute Neotron OS path.

### `DIR [/S] [/P] [ <path>]`

Performs a directory listing, either for the current drive and directory or of the given absolute or relative path.

* The `/S` option displays directories recursively.
* The `/P` option pauses after each screen-full of output.

### `MKDIR <dir>`

Creates a new directory. Creates any intermediate directories as required.

### `TYPE <file> [ /P ]`

Displays the contents of a file, assuming it is UTF-8 encoded text. An error is
given if the file is not UTF-8 encoded.

* The `/P` option pauses after each screen-full of output.

### `HEXDUMP <file> [ <offset> [ <length> ] ] /P`

Displays the contents of a file as both hex and as ISO-8859-1. A maximum of 16
bytes are shown per line. An optional offset and length can be given if you only
wish to dump part of a file.

* The `/P` option pauses after each screen-full of output.

```console
> HEXDUMP test.dat
0000_0000: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
0000_0010: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
0000_0030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
```

### `COPY [/S] [/Y] <source> [ <source> ... ] <dest>`

Copies one or more files from one location to another.

Each `source` can be:

* a single `<file>`,
* a single `<dir>`, or
* a `<path>` containing `*` and `?` wildcards, like `1:/*.TXT`.

If one `source` is specified `dest` can be a file or a directory. If multiple
`source` files are specified, `dest` must be a directory. 

* The argument `/S` makes `COPY` look in subdirectories recursively.
* The argument `/Y` makes `COPY` overwrite files without asking.

```console
> COPY *.TXT 1:/TEXT
Copying A.TXT to 1:/TEXT...
Copying B.TXT to 1:/TEXT...
Copying C.TXT to 1:/TEXT...
```

### `RENAME <old> <new>`

Changes the name of a file. Cannot move files between directories or drives.

### `MOVE <src> <dest>`

Moves a file from one directory to another. The `dest` path can be on a
different drive or in a different directory.

### `DEL <path> [ <path2> ... ]`

Deletes one or more files.

### `RMDIR <path> [ <path2> ... ]`

Deletes one or more empty directories.

### `DELTREE <path> [ <path2> ... ]`

Recursively deletes one or more directories.

### `PEEK <addr> [ <length> ] [ /P ]`

Displays the contents of ROM or RAM at the given address, as both hex and as
ISO-8859-1. Displays `length` bytes, or 4 bytes if `length` is not given. A
maximum of 16 bytes are shown per line.

* The `/P` option pauses after each screen-full of output.

```console
> PEEK 0x2000_1000 48
2000_1000: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
2000_1010: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
2000_1030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00 |................|
```

### `POKE <addr> <value>`

Writes a new value to RAM at the given address. No testing of the address is
performed, so it is very easy to cause a hard fault with this function.

```console
> POKE 0x2000_1000 0x39
> PEEK 0x2000_1000
0x20000000: 39 00 00 00                                      |9...            |
```

### `SAVE <addr> <length> <path>`

Copies RAM or ROM to a file on disk.

### `LOAD <path> [ <addr> ]`

Copies the contents of a file on disk to RAM. Loads into the given address, or
into the transient program area by default.

### `EXEC`

Execute a program which 

### `RUN`

Runs a program already in memory (e.g. having used `LOAD`)

### `SET [ <var> [ <value> ] ]`

* With no arguments, prints all the environment variables.
* With one arguments, prints the environment variable with that name.
* With two arguments, sets an environment variable to the given value.

Environment variables are case sensitive. To set a variable containing whitespace, wrap it in double-quote (`"`) marks.

```console
> SET
PATH=0:/BIN;1:/APPS
NAME=jgp
> SET NAME
NAME=jgp
> SET NAME steve
> SET NAME
NAME=steve
```

### `ECHO [ <args> ... ]`

Prints some text to the console.

```console
> SET NAME steve
> ECHO "Hello, ${NAME}"
Hello, steve
```

### `DEV [ /P ]`

Shows the list of devices reported by the BIOS.

* The `/P` option pauses after each screen-full of output.

### `VOL [ <volume> ]`

Shows the volume name, drive, format and storage capacity for the given volume,
or all volumes.

### `SCAN`

Re-scans the given device for volumes. Volumes which are no longer present (e.g.
because the disk has been removed) have their volume ID de-allocated. New
volumes are allocated new volume IDs.

### `CD [ <dir> ]`

Change the current directory. May include a volume ID, or be relative to the
current volume, or the current directory. Unlike MS-DOS, there is only one
current directory for the whole system (instead of one for each drive).

### `ATTR <path> [/A] [/A-] [/H] [/H-] [/R] [/R-] [/S] [/S-]`

Gets or sets a directory or file's attributes (Archive, Hidden, Read Only, and System).

The `/X` flag sets a the relevant bit. The `/X-` flag unsets it.

```console
> # Set System, Hidden and Read Only, but clear Archive
> ATTR /A- /S /H /R SYSTEM.DAT
```
