# The Neotron Command Shell

## Introduction

The Neotron Shell is the first application started by the OS. It allows the user to give the system commands, including:

* Browsing filesystems on the SD card
* Loading files from SD card into RAM
* Inspecting the contents of RAM and ROM
* Executing applications

The Shell uses the standard OS API and hence is much like a normal application. The only difference is that it uses a special region of memory, allowing it to remaining running whilst it loads another application into the main program area. Once that application is started, it can use the shell's memory region as it's own - the OS will re-initialise the shell when the application exits.

Analogies from the PC world would include MS-DOS's `COMMAND.COM` and CP/M's CCP, but at a functional level it's more like the U-Boot Shell.

## Shell Expansion

Any argument of the form `${VAR}` is replaced with the value stored in the environment variable called `VAR`.

Any argument containing a `*` character is taken to be a glob and is expanded to all the filenames matching that glob. Note that there are a limited number of arguments that can be given to a command, so attempts to glob more files than that number will generate an error.

## Magic Devices

The OS supports a number of magic device filenames. The shell is unaware of this and just asks the OS to perform the specified operation on the given filenames.

## Shell Commands

### `DIR [/S] [<path>]`

Performs a directory listing, either for the current drive and directory or of the given absolute or relative path.

The `/S` option displays directories recursively.

### `MKDIR [/P] <path>`

Creates a new directory, given either an absolute path or a relative path. Can
optionally make all the intermediate paths too.

### `TYPE <path>`

Displays the contents of a file, assuming it is ASCII text.

### `HEXDUMP <path>`

Displays the contents of a file as hex encoded binary.

### `COPY [/S] <source1> [<source2>...] <dest>`

Copies one or more files from one location to another. If one `sourceX` is
specified `dest` can be a file or a directory. If multiple `sourceX` files are
specified, `dest` must be a directory. The argument `/S` makes `COPY` look in
subdirectories recursively.

### `RENAME <old> <new>`

Changes the name of a file. Cannot move files between directories or drives.

### `MOVE <src> <dest>`

Moves a file from one directory to another. The `dest` path can be on a
different drive or in a different directory.

### `DEL <path> [<path2>...]`

Deletes one or more files.

### `RMDIR <path> [<path2>...]`

Deletes one or more empty directories.

### `DELTREE <path> [<path2>...]`

Recursively deletes one or more directories.

### `PEEK <addr>`

Displays the contents of ROM or RAM at the given address.

### `POKE <addr> <value>`

Writes a new value to RAM at the given address. Attempts to write to ROM are ignored.

### `SAVE <addr> <length> <file>`

Copies RAM or ROM to a file on disk.

### `LOAD <path>`

Copies the contents of a file on disk to RAM.

### `EXEC <path>`

Performs a LOAD of the given file, followed by a RUN. This command is implied if a command is given which matches the name of a file in the current directory, or in the system's `PATH` environment variable.

### `RUN`

Runs a program already in memory (e.g. having used `LOAD`)

### `SET <var> <value>`

Sets an environment variable to the given value. The value `var` should not include `${` or `}`.

```console
$ SET PATH 0:/BIN;0:/OS/BIN
```

### `ECHO <text>`

Prints some text to the console.

```console
$ ECHO "Hello, world"
```

### `LOADENV <path>`

Loads environment variables from a file.

```console
$ LOADENV ./FILE.ENV
```

### `SAVEENV <path>`

Saves environment variables to a file.

```console
$ SAVEENV ./FILE.ENV
```

### `SCRIPT <path>`

Runs the shell commands found in the given filename.

```console
$ SCRIPT ./FILE.CMD
```

### `EDIT <path>`

Starts a full-screen text editor.

```console
$ EDIT 1:/README.TXT
```

### `DEV`

Shows the list of devices reported by the BIOS.

### `VOL`

Shows the current list of volumes, by drive. Includes their volume names, format and storage capacity.

### `SCAN`

Re-scans the given device for volumes. Volumes which are no longer present (e.g. because the disk has been removed) are de-allocated their volume ID. New volumes are allocated new volume IDs.

### `CD [<dir>]`

Change the current directory. May include a volume ID, or be relative to the current volume, or the current directory. Unlike MS-DOS, there is only one current directory for the whole system (instead of one for each drive).

### `ATTR [/A] [/A-] [/H] [/H-] [/R] [/R-] [/S] [/S-] <path>`

Gets or sets a file's attributes (Archive, Hidden, Read Only, and System).

The `/X` flag sets a the relevant bit. The `/X-` flag unsets it.

```console
$ # Set System, Hidden and Read Only, but clear Archive
$ ATTR /A- /S /H /R SYSTEM.DAT
```
