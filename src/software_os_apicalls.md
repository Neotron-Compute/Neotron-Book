# The Neotron OS - API Calls

The OS exports its functionality to each Application through a structure of function pointers. This structure exists in OS memory (probably in Flash, but it could be in RAM) and a pointer to it is passed to the Application when it is started by the OS. This is the same process as used when the BIOS starts the OS.

The Application should only access the hardware and the operating system state through the function pointers contained within the given structure. The only assumption the Application can make is about the location of the memory regions defined in its Linker Script.

The OS API is (or will be) documented at <https://github.com/Neotron-Compute/Neotron-API>.

