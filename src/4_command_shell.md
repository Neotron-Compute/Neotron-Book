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

### DIR

Performs a directory listing, either for the current drive and directory or of the given absolute or relative path.

### MKDIR

Creates a new directory, given either an absolute path or a relative path.

### TYPE

Displays the contents of a file, assuming it is ASCII text.

### HEXDUMP

Displays the contents of a file as hex encoded binary.

### COPY

Copies a file from one location to another.

### RENAME

Changes the name of a file.

### MOVE

Moves a file from one directory to another.

### DEL

Deletes a file.

### RMDIR

Deletes an empty folder.

### DELTREE

Recursively deletes a folder.

### PEEK

Displays the contents of ROM or RAM at the given address.

### POKE

Writes a new value to RAM at the given address. Attempts to write to ROM are ignored.

### SAVE

Copies RAM or ROM to a file on disk.

### LOAD

Copies the contents of a file on disk to RAM.

### EXEC

Performs a LOAD of the given file, followed by a RUN. This command is implied if a command is given which matches the name of a file in the current directory, or in the system PATH.

### RUN

Runs a program in memory.

### SET

Sets an environment variable to the given value.

### ECHO

Prints some text to the console.

### LOADENV

Loads environment variables from a file.

### SAVEENV

Saves environment variables to a file.

### SCRIPT

Runs the shell commands found in the given filename.

### EDIT

Starts a full-screen ASCII file editor.

### DEV

Shows the list of devices reported by the BIOS.

### VOL

Shows the current list of volumes, by drive. Includes their volume names, format and storage capacity.

### SCAN

Re-scans the given device for volumes. Volumes which are no longer present (e.g. because the disk has been removed) are de-allocated their volume ID. New volumes are allocated new volume IDs.

### CD

Change the current directory. May include a volume ID, or be relative to the current volume, or the current folder. Unlike MS-DOS, there is only one current directory for the whole system (instead of one for each drive).

### ATTR

Gets or sets a file's attributes (Read Only, Archive, System and Hidden).
