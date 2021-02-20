# Parallel Port

The MCP23S17 I/O Expander sits on the SPI bus, with a dedicated IRQ line. This can be used to drive a PC-style 25-pin Parallel Printer Port, or it can be used as a general-purpose 16-bit GPIO interface. Running at up to 10 MHz, all 16-bits can be written to in 4 bytes, which takes 32 microseconds or 256 clock cycles. As the Parallel Port strobe pin is driven separately from an SoC GPIO line, and with the MCP23S17 able to generate an Interrupt when the remote device raises the ACK line, this interface should be sufficient for around 150 KiB/sec.
