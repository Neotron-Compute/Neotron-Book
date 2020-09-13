# PS/2

The Neotron I/O controller is responsible for the PS/2 interfaces and the two Joystick ports. It consists of an AtMega 328 microcontroller with custom firmware, and speaks a bespoke *HID over UART* protocol. The Joystick ports can be either standard Atari joysticks, or SEGA MegaDrive / Genesis 3-button pads. With some software work, 6-button pads could possibly also be supported.

In the future, we may change this to speak *HID over I²C* to reduce the number of UARTs required.

Currently the AtMega firmware is written in C++. As Rust on AVR matures we may look to a Rust implementation to allow us to share code with the BIOS.
