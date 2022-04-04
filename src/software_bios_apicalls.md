# The Neotron BIOS - API Calls

The BIOS exports its functionality to the OS through a structure of function pointers. This structure exists in BIOS memory (probably in Flash, but it could be in RAM) and a pointer to it is passed to the Operating System (OS) when the OS is started by the BIOS.

The OS should only access the hardware through the function pointers contained within the given structure. The only assumption the OS can make is about the location of the memory regions defined in its Linker Script.

The BIOS API is documented at <https://docs.rs/neotron-common-bios/latest/neotron_common_bios/>.

