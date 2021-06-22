# Video from a System On Chip

Video support from your SoC can vary greatly - from SoCs which basically have no video support but can be 'tricked' into generating a signal that's sort-of VGA compatible, to SoCs which have on-board graphics accelerators that can render multiple semi-transparent planes together.

## Video Modes

First, we should discuss the concept of a _video mode_. This is a collection of parameters which describe the picture that is being generated.

In this section, we generally assume that the video is VGA-like - that is, each frame is drawn line-by-line, starting with the top line and moving downwards. A line comprises pixels, which are drawn from left to right. Each pixel has a colour, which can be anywhere between 'on or off' or 'one of 16.8 million colours'. As the pixels are drawn left to right, there will be some extra non-visible 'off-screen' pixels, which are designed to give an old-fashioned cathode-ray tube (CRT) monitor time to move the scanning 'gun' back over to the left hand side of the image ready to draw the next line. In the same fashion, there will be some non-visible 'off-screen' lines which give a CRT's gun time to move back to the top of the frame. Although we almost universally use LCD monitors now, the standards still include these 'blanking periods'.

The monitor is able to find the visible picture within the signal, thanks to the provision of two extra signals - *Horizontal Sync* and *Vertical Sync*. These pulse high (or low) during the middle of the relevant blanking period.

The relevant parameters are:

* Horizontal Visible Pixels - the number of visible pixels on each line
* Horizontal Front Porch - the number of non-visible pixels before the sync pulse
* Horizontal Sync Width - the number of non-visible pixels during the sync pulse
* Horizontal Back Porch - the number of non-visible pixels after the sync pulse
* Horizontal Pixels - the total number of pixels on each line
* Horizontal Frequency - the number of lines (visible or non-visible) drawn per second
* Vertical Visible Lines - the number of visible lines in each frame
* Vertical Front Porch - the number of non-visible lines before the sync pulse
* Vertical Sync Width - the number of non-visible lines during the sync pulse
* Vertical Back Porch - the number of non-visible lines after the sync pulse
* Vertical Lines - the total number of lines in each frame
* Frame rate - the number of frames drawn per second
* Pixel or Dot Clock - the number of pixels (visible or non-visible - we'll get to that shortly) produced per second

Given this, we can say that:

```
Horizontal Pixels = Horizontal Visible Pixels + Horizontal Front Porch + Horizontal Sync Width 
+ Horizontal Back Porch
Vertical Lines = Vertical Visible Lines + Vertical Front Porch + Vertical Sync Width 
+ Vertical Back Porch
Horizontal Frequency = Vertical Lines x Frame rate (Hz)
Pixel Clock = Horizontal Frequency x Horizontal Pixels
```

The monitor generally doesn't care about the number of colours each pixel can take, as the pixels are transmitted as analog red, green and blue signals (0.7V peak). However, the SoC will care as the pixels must be stored digital and the more bits per pixel, the more bytes of framebuffer RAM we require!

The original IBM standard for the _Video Graphics Array_ included the following video modes:

| H. Visible | V. Visible | H. Total | V. Total | H. Freq    | Frame Rate | Pixel Clock |
|:-----------|:-----------|:---------|:---------|:-----------|:-----------|:------------|
| 640        | 480        | 800      | 525      | 31.469 kHz | 60 Hz      | 25.175 MHz  |
| 720        | 400        | 900      | 450      | 31.469 kHz | 70 Hz      | 28.322 MHz  |
| 320        | 200 (2x)   | 400      | 450      | 31.469 kHz | 70 Hz      | 25.175 MHz  |
| 320        | 240 (2x)   | 400      | 525      | 31.469 kHz | 60 Hz      | 25.175 MHz  |

The rates marked _(2x)_ simply have each line drawn twice by the VGA card, in order to keep the Horizontal Frequency at the standard value.

We see from this table that there are two values for lines - 480 line and 400 line - corresponding to frame rates of 60 Hz and 70 Hz respectively. There is also the option to increase the pixel clock up from 25.175 MHz to 28.322 MHz, increasing the number of pixels per line from 800 to 900. This, in turn, increases the number of pixels per character on an 80-column text mode display from 8 to 9 - this is used for the classic MS-DOS text mode (which uses a 9 pixel x 16 pixel bitmap for each character on the screen).

During the 1990s, the resolutions and colour depths supported by video cards increased greatly, and monitors began to support a wider range of horizontal frequencies. These various modes were standardised by the Video Electronics Standards Association (_VESA_), and there are too many to list here. However, some modes of note include:

| H. Visible | V. Visible | H. Total | V. Total | H. Freq  | Frame Rate | Pixel Clock |
|:-----------|:-----------|:---------|:---------|:---------|:-----------|:------------|
| 800        | 600        | 1056     | 628      | 37.9 kHz | 60 Hz      | 40.0 MHz    |
| 1024       | 768        | 1344     | 806      | 48.4 kHz | 60 Hz      | 65.0 MHz    |
| 1280       | 720        | 1664     | 748      | 44.8 kHz | 60 Hz      | 74.5 MHz    |
| 1920       | 1080       | 2576     | 1120     | 67.2 kHz | 60 Hz      | 173.000 MHz |

We don't really think much of 1080p (1920x1080) video these days - often preferring QuadHD (2560x1440) or 4K (3840x2880) for our desktop PC monitors. These tables show, however, that even 1080p is a staggering amount of data for an SoC to generate - especially without any built-in hardware acceleration. The higher resolution modes also consume vast amounts of framebuffer RAM - 1080p in 24-bit True Colour (usually stored as 32-bits per pixel) needs 8100 KiB of RAM, just for a single frame!

## Video Hardware

If we want to generate a VGA compatible signal, we have several options:

### Misuse the wrong peripheral

We can generate pixels on basically any SoC peripheral which has a synchronous (clock-driven) output. Even a basic SPI peripheral can be used, which will generate a 1 bit-per-pixel (black and white) video signal. The Monotron project used three SPI periperals on a TM4C to produce a 3-bit-per-pixel (8 colour) signal, and with a 20 MHz pixel clock to generate a 400x600 image (like 800x600 but with half as many pixels across, and each pixel twice as wide). Various projects for the Espressif ESP32 have used the I²S interface to push out up to 14 pixels per clock cycle (16,384 colour).

The Raspberry Pi Pico takes peripheral abuse even further, by driving a DC-balanced 251.75 MHz TMDS (transition minimised differential signallling) signal out of some standard GPIO pins using a high-speed FIFO, some pre-calculation and some serious over-clocking. This signal is almost exactly like DVI-D video, which thanks to backwards compatibility, can be sent over an HDMI cable to any HDMI or DVI-D compatible display.

### Use a TFT-LCD Controller Peripheral

Many SoCs include hardware for driving an RGB TFT LCD panel. The good news is that these RGB panels are designed to look just like a VGA monitor (but with digital bits for each pixel rather than three analog signals), and these controllers can often be programmed to generate VGA compatible signals.

Look for the LTDC (LCD TFT Display Controller) on the STM32 line, for example.

### Use an MIPI DSI Controller Peripheral

MIPI DSI (Mobile Industry Processor Interface Display Serial Interface) is a standard for pushing pixels into a display using a high-speed serial interface. This reduces pin-count and electronic magnetic interference, compared to waggling the 26 individual signal lines required to send 24-bit-per-pixel digital video. DSI is often sent over flexible ribbon cable, you can see a DSI port on most Raspberry Pi boards.

You can get chips which will take in a DSI signal and output HDMI, such as the [Analog Devices ADV7533](https://www.analog.com/en/products/adv7533.html) but implementing an proper 'HDMI' output will require licensing and for this reason the chips are often only available under NDA.

### Use an off-chip video controller

If there really is nothing suitable, you can always get a second SoC or dedicated video controller to drive your display. Such chips are popular for driving LCD panels (e.g. the ILI9341 or the ST7920) but can sometimes be configured to generate VGA compatible signals.

## Text vs Graphics

Neotron's video support is based broadly on that of the IBM PC _Video Graphics Array_. In the previous section, we discussed the physical attributes of the analog video signal generated by a _VGA_ card. What we didn't cover, is how we generate that signal based on the contents of Video RAM (VRAM).

Broadly, the IBM PC has two kinds of video modes - text mode and graphics mode. Text mode is what MS-DOS would boot up in, and Graphics mode (also known in Linux as _framebuffer mode_) is what Windows 3.1 or a game like Doom would switch to. Whilst some systems (in particular 1990s RISC workstations, and modern ARM systems) have no concept of a 'text mode' and instead draw text to a graphical framebuffer, we retain the two distinct types of mode in Neotron in order to save on VRAM.

### Text Modes

Text modes divide the screen into a rectangular grid of _characters_. Each _character_ is represented by a image, known as a _glyph_. All of the glyphs in a given _font_ have the same width and height in pixels, and they each use precisely two colours - _foreground_ and _background_. The specific value of _foreground_ and _background_ can be set for each character cell on the screen, meaning that each cell takes up two bytes: one for the character, and one for the colour pair. As the raster beam moves across the display, the video card will fetch the character for that cell from VRAM, then load the corresponding glyph from the font (which may be in VRAM, or it may be in the card's ROM). The appropriate horizontal slice is taken from the glyph, and then converted to a sequence of coloured pixels by applying the foreground and background colour for that cell.

Each character is an 8 bit value, and so there can only be 256 glyphs in a font. Merging multiple characters into a single glyph (as you might do in Unicode with a `U+0041 LATIN CAPITAL LETTER A` followed by a `U+02CA MODIFIER LETTER ACUTE ACCENT` to produce an `Á`) is generally not supported, although such canonicalisation could be performed by the OS when the string was first written to the VRAM.

The two main text mode resolutions for VGA are 80 columns by 25 rows, and 80 columns by 50 rows. The video output resolutions these correspond to depend on the glyph size, as follows:

| Columns | Rows | Glyph Width | Glyph Height | Horizontal Pixels | Vertical Pixels | Standard      |
|:--------|:-----|:------------|:-------------|:------------------|:----------------|:--------------|
| 80      | 25   | 9           | 14           | 720               | 350             | MDA, Hercules |
| 40      | 25   | 8           | 8            | 320               | 200             | CGA, EGA      |
| 80      | 25   | 8           | 8            | 640               | 200             | CGA, EGA      |
| 80      | 25   | 8           | 14           | 640               | 350             | EGA           |
| 80      | 43   | 8           | 8            | 640               | 350             | EGA           |
| 80      | 25   | 9           | 16           | 720               | 400             | VGA           |
| 80      | 50   | 9           | 8            | 720               | 400             | VGA           |

Because a Neotron system might support other video resolutions (e.g. the native 400x300 of a Neotron 32), there is no prescriptive list of Neotron text modes. Instead, there is a BIOS API to query which modes are supported, and the width in columns and height in rows of each mode. It is assumed that each mode supports 16 foreground colours and 8 background colours, just like VGA, and that memory is arranged in a linear array of 16-bit values, where the first (lower) 8-bits identify the character and the second (higher) 8-bits identify the foreground and background colour.

```
+-----+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|             Attribute           |           Character           |
+-----+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|  7  | 6 | 5 | 4 | 3 | 2 | 1 | 0 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
+-----+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|Blink| Backgr'nd |  Foreground   |           Code Point          |
+-----+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
```

(Diagram courtesy <https://en.wikipedia.org/wiki/VGA_text_mode>)

The `Blink` bit causes the text to alternate between being drawn normally, and being drawn entirely in the background colour (thus rendering it invisible).

Where the graphics are drawn by an off-chip GPU, the Neotron BIOS will need to arrange for the on-chip VRAM to be copied to the off-chip GPU during the vertical blanking interval of each frame. A text mode with 80 x 50 characters will require 4000 bytes of VRAM, which at 70 Hz needs a link of just over 41 Mbit/sec in order for the copy to complete during the blanking interval. If you are prepared to 'chase the beam' you can run a little slower. 80x25 video modes will require half that, and using run-length encoding will likely reduce it further (especially as consecutive cells are usually the same colour).

## Support in Neotron
