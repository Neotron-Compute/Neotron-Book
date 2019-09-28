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
* A FAT16/32 compatible filesystem
* A text-mode console, with cursor
* Simple bitmap graphics primitives (lines, rectangles, fill, etc)
* An audio synthesiser
* Input/output stream handling
* Digital (Atari) joystick support
* A TCP/IP networking stack
* MIDI support
* Memory allocation/deallocation routines
* Co-operative multi-tasking through an event-loop / task-scheduler

Selected non-goals (i.e. things we aren't going to support) include:

* Multi-threading
* Running multiple processes
* Virtual Memory (although we might support a special API for paging to and from disk)
* Run-time filesystem mounting/unmounting

Our ultimate goal is that one day we could offer a version of the Rust Standard Library (`std`) which uses the Neotron OS API - albeit without the sub-process support.

Programming the Neotron usually involves first loading the specific interpreter for that language - unless you've swapped out the standard shell for something like a BASIC interpreter.
