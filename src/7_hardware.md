# Neotron Hardware

The Netron OS is intended to be binary-portable across any machine with a Neotron BIOS. How that works out in practice remains to be seen, but that's the goal. As the BIOS API, example BIOS implementations and the OS itself are entirely open, anyone can build their own Neotron-compatible system, using any ARMv7-M compatible processor.

To demonstrate the practicality of making a portable OS for ARMv7-M, we are producing a number of Neotron compatible systems.


* [Neotron 32](./61_neotron32.md) - an open-source PCB which plugs into the Texas Instruments TM4C Launchpad (a derivative of the original Montron project).
* [Neotron 340ST](./62_neotron340st.md) - based on an STM32F7-DISCOVERY PCB; not open-source hardware, but easily available from ST and it comes with schematics.
* [Neotron 500](./63_neotron500.md) - an unfinished attempt an an open-source PCB using an STM32H7 and other parts from the JLCPCB catalog so it could be built using their assembly service.
* [Neotron 600](./64_neotron600.md) - an open-source PCB which plugs into the Teensy 4.1.
* [Neotron 1000](./65_neotron1000.md) - an open-source PCB based around the STM32H7, along with a Lattice iCE40 FPGA for hardware accelerated video output.
* [Neotron 9X](./66_neotron9X.md) - an open-source PCB based around the Microchip SAM9X60D5M system-in-package.
