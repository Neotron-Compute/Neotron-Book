# Neotron Hardware

The Netron OS is intended to be binary-portable across any machine with a Neotron BIOS. How that works out in practice remains to be seen, but that's the goal. As the BIOS API, example BIOS implementations and the OS itself are entirely open, anyone can build their own Neotron-compatible system, using any ARMv7-M compatible processor. However, there will naturally be value in having some commonality between Neotron systems, in order to reduce development effort.

Another important factor in the Neotron Hardware is that, like the software, it should be fully Open. That is, anyone should be able to pick up a Neotron System and understand it right down to the component parts (and why those component parts were chosen). It isn't sufficient to simply post some Gerbers on GitHub and call it done - any more than it would be to post some compiled firwmware binaries. Making an Open System means making a system where people are invited and encouraged to learn everything there is to know about that system.

## Components

A Neotron System will naturally be comprised of:

* A microcontroller, also known as a System On Chip (SoC) - this will contain one or more Processor Cores, RAM, possibly Flash, and some internal peripherals
	* The processor core will be compatible with the [ARMv6-M ISA]\*.
* External peripherals - such as UART level shifters, audio amplifiers, SD cards
* A baseboard (tradtionally known as a _motherboard_) - a PCB which holds the SoC, the external peripherals, and any connectors

\* _The ARMv6-M ISA (as implemented by the Cortex-M0/M0+ processore cores) has fewer instructions, and so is easier to understand. However, ARMv6-M has no Compare-and-Swap Atomic instructions, meaning that sharing data between the main thread and interrupts can be more complicated in some situations. It is TBD as to whether this is too much of a burden for the OS to carry and we may need to raise the minimum to [ARMv7-M ISA], i.e. a Cortex-M3 or higher._

[ARMv6-M ISA]: https://static.docs.arm.com/ddi0419/d/DDI0419D_armv6m_arm.pdf
[ARMv7-M ISA]: https://static.docs.arm.com/ddi0403/e/DDI0403E_B_armv7m_arm.pdf
