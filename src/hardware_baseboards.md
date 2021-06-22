# Baseboards

## Form Factor

The original Monotron didn't have a particular form-factor - the PCB was simply made large enough to accept all the desired connectors. The Neotron 32 was upsized slightly to fit the Hammond 1598C series case, however, there were so many connectors packed into a tight space that there was nowhere left fit a full-size SD card slot. The work-around was to use a microSD card, but all available examples proved difficult to hand-solder. An upgrade to the larger Hammond 1598D was investigated, but expansion options for the system still remained limited - there were two small expansion headers, but no real space for any extra connectors.

One of the nice features of the IBM PC (and its clones), and also the Apple II series, was the provision of expansion slots. These allowed extra functionality to be added at a later date - often well beyond what the original designer had envisaged at the time the main system was developed. Whilst the Hammond cases work well as a small desktop chassis, they do lack the space for expansion cards - or at least for expansion cards with their own external connectors.

The alternative would be to produce a board based around the ATX standard - either full-size ATX (305x244mm), micro-ATX (244x244mm), or micro-ITX (170x170mm). These boards would include some crucial components (e.g. audio codec, VGA DAC, power supply), but move much of the other functionality off onto expansion cards (MIDI, Parallel Port, Joystick Ports, etc). Rather than use ISA-style card-edge connectors (which require the expansion card to have a harder gold finish which can survive the plugging-unplugging), we're considering using standard IDC headers. These would, however, be located such that the expansion cards would line up with standard ATX case expansion slot holes.

## Putting the processor on a separate card

Unfortunately, being quite a large board means it can be quite expensive to produce. We also note that many of our target CPUs are only supplied in BGA package, which means four layers at a minimum. With a BGA SoC the decoupling capacitors would usually be fitted to the reverse of the board, meaning a simple hot-plate cannot be used for re-flow. This seems to place a high-bar on self-assembly at home - requiring basically a full surface-mount rework station. Instead, it seemed to make sense to locate the complex components on a carrier card (CPU, SDRAM, etc) and leave the main board to carry the through-hole connectors and simpler, lower pin-count, parts like the audio codec.

There are a number of standards we could use for the interface to the processor card:

* [Feather](https://learn.adafruit.com/adafruit-feather/feather-specification): Lots of GPIO (including I²C and SPI) but no defined interface for video, and not specific set of pins assigned for I²S.
* [MikroBus](https://www.mikroe.com/mikrobus): Designed for expansion cards, rather than processor cards. But our expansion connectors will carry broadly similar sets of pins.
* [Mikro MCU Card](https://download.mikroe.com/flyers/mcu-card-flyer-web.pdf): Carries almost all of the connections we'd like to get off a SoC (including the all-important RGB LCD interface we need for VGA output). The only downside is the use of two high density 168-pin headers which look pretty tricky to solder, and cost about £4.50 each. 
* [Raspberry Pi CM1/CM3](https://www.raspberrypi.org/products/compute-module-3/): This pinout is quite specific to the BCM2835 and BCM2837 SoCs used, and includes HDMI, Display Serial Interface and Camera Serial Interface. It does support USB and parallel RGB though. 
* [Raspberry Pi CM4](https://www.raspberrypi.org/products/compute-module-4/): This pinout is quite specific to the BCM2711, and if parallel RGB is used, there are no other GPIO pins available. Uses two 100-pin high-density headers.
* [SparkFun MicroMod](https://www.sparkfun.com/micromod): This is specifically designed to allow different SoCs to be inserted into a baseboard, but the specification does not include pins dedicated to video support. It also uses an M.2 header which is surface-mount and has a 0.5mm pitch.

In the end, it's probably easier to come up with our own standard to suit our specific requirements.

![Obligatory XKCD reference - 'Standards'](https://imgs.xkcd.com/comics/standards.png)

<https://xkcd.com/927/>

The signals we need include:

* Parallel RGB video (8 to 29 pins)
	* 3-bit, 6-bit, 9-bit, 12-bit, 15-bit, 18-bit or 24-bit RGB digital video
	* H-Sync
	* V-Sync
	* DDC SDA
	* DDC SCL
	* Pixel Clock
* 2x USB 2.0 Host @ 480 Mbps (4 pins)
	* Port 0 D+
	* Port 0 D-
	* Port 1 D+
	* Port 1 D-
* Digital Audio Interface (5)
	* DAC Data
	* DAC Left/Right Clock
	* ADC Data
	* ADC Left/Right Clock
	* Bit Clock
* I²C @ 400 kHz (2)
    * SDA
    * SCL
* Control Signals (10)
	* /Reset (in)
	* /Enable (in)
    * 8x External Interrupt (in)
* Power (8)
	* 2x 5.0V @ 2A max, combined
	* 2x 3.3V @ 200mA max, combined
	* 4x GND
* SPI Bus @ 25 MHz (11)
    * COPI (out)
    * CIPO (in)
    * CLK (out)
    * 8x Chip Select (out)

| Interface     | Pins  | Running Total |
|:--------------|:------|:--------------|
| Video         | 23    | 23            |
| USB           | 4     | 27            |
| Console UART  | 4     | 31            |
| PS/2          | 4     | 35            |
| Digital Audio | 5     | 40            |
| I²C           | 2     | 42            |
| Control       | 10    | 52            |
| Power         | 8     | 60            |
| SPI           | 11    | 71            |
|               | TOTAL | 71 pins       |

This requires at least 71 pins, ideally more as we probably want to include more Ground pins around the high-speed signals. There is a risk that despite this, sending high speed signals (USB 2.0 High-Speed, 40MHz+ parallel RGB) will cause significant EMC issues.

We could include the VGA DAC on the MCU card, reducing the VGA pins from 22 (18x digital RGB + 4) to 7 (3x analog video + 4), but that would then preclude the use of a parallel-RGB to DVI/HDMI encoder at a later date.

If we stick with simple 0.1" pitch pins (easy to solder), one option is two 2xN headers, like the Launchpad uses.

```
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
```

This has 80 pins, and occupies 51mm by 31mm. As you can see it would produce quite a long CPU card, and seems to waste the space between the two headers. It is, however, the approach taken by the [Waveshare CoreH743I board] used by the [Colour Maximite 2].

[Waveshare CoreH743I board]: https://www.waveshare.com/coreh743i.htm
[Colour Maximite 2]: https://geoffg.net/CMM2_Description.html

We could also try a double-square arrangement:

```
o o o o o o o o o o o o
o o o o o o o o o o o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o                 o o
o o o o o o o o o o o o
o o o o o o o o o o o o
```

This has (11 + 9) * 4 = 80 pins, and occupies around 31mm x 31mm, with a central area of 20mm x 20mm to fit the SoC and all the decoupling capacitors. Of course, the general shape can be scaled up if required - the Intel 80386 had a 132-pin PGA package laid out as follows:


```
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o                 o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
o o o o o o o o o o o o o o
```

The problem is an LQFP-144 part is 20mm x 20mm by itself, never mind the decoupling caps and the space needed for routing. All things considered, the 2x20 x 2 design may be the best compromise.