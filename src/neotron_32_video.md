# Video

Video is generated on the Neotron-32 by driving three SPI interfaces in a synchronised fashion - one generating a stream of red pixels, one a stream of green pixels and one a stream of green pixels. There is also a Horizontal Sync line driven from a Timer, which is synchronised to the pixel outputs, and a Vertical Sync line which is a GPIO line driven in the Horizontal Scan-line Interrupt Handler.

With a CPU clock of 80 MHz, the SPI ports can be run at either 40 MHz or 20
MHz. 40 MHz generates an 800x600 pixel image with a 60 Hz refresh rate, and
20 MHz generates a 400x600 pixel image with a 60 Hz refresh rate. To counter
the 'stretch' effect of the latter mode, we optionally drop to 300 lines of
video but clocking each out one twice, giving an effective 400x300 resolution
on a nominal 800x600 signal.

The hardware on the motherboard simply consists of some resistors which, in conjunction with the 75 ohm input impedance of a VGA monitor, drops the 3.3V signal from the SoC down to the 0.7V maximum given in the VGA standard. With the three channels (R, G and B) being either on or off, we get 8 familiar colour combinations:

* Black
* Red
* Green
* Yellow
* Blue
* Magenta
* Cyan
* White

As the SoC can only just push out the 8 mA required to drive a VGA signal into 75 ohms, later Neotron systems are likely to use a small active VGA/RGB video filter to act as a line driver.

## Supported video modes:

The following [video modes](./hardware_soc_video.md) are supported:

* Mode 0x28 - 50x37 text mode at 400x600 
* Mode 0xAF - 400x300 2-colour
