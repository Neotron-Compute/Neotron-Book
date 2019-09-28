# The Neotron Shell

The Neotron Shell is the first application started by the OS. It allows the user to give the system commands, including:

* Browsing filesystems on the SD card
* Loading files from SD card into RAM
* Inspecting the contents of RAM and ROM
* Executing applications

The Shell uses the standard OS API and hence is much like a normal application. The only difference is that it uses a special region of memory, allowing it to remaining running whilst it loads another application into the main program area. Once that application is started, it can use the shell's memory region as it's own - the OS will re-initialise the shell when the application exits.
