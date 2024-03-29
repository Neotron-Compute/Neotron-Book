# Video

Video is generated by the BIOS, through the PIO0 peripheral of the RP2040. Two of its state machines (SM0 and SM1) are used to generate a VGA signal:
 * **SM0** - "Timing Program" - generates HSYNC and VSYNC signals
 * **SM1** - "Pixel Program" - generates the actual 16-bit RGB pixels (in pairs)

The two state machines are synchronized through the IRQ0 line. Data is carried from the timing and pixel buffers to the PIO0 timing and pixel FIFOs using DMA channels 0 and 1 respectively.


```
                                                     ┌─┬────┐
                                               ┌────►│H│  0 │
                                               │     ├─┼────┤
                                               ├────►│V│  1 │
                                               │     ├─┼────┤
┌────────────────────────┐                     │     │ │  2 │
│                        │                     │     │ ├────┤
│   ┌────────────────┐   │                     │     │ │  3 │
│   │                │   │                     │  ┌─►│R├────┤
│   │                │   │       ┌───────────┐ │  │  │ │  4 │
│   │     Timing     │   │       │           │ │  │  │ ├────┤
│   │     Buffer     │   │DMA CH0│ ┌──┬────┐ │ │  │  │ │  5 │
│   │                ├───┼───────┼─► F│ SM0├─┼─┘  │  ├─┼────┤
│   └────────────────┘   │       │ └──┴─┬──┘ │    │  │ │  6 │
│                        │       │  IRQ0│    │    │  │ ├────┤
│   ┌────────────────┐   │DMA CH1│ ┌──┬─▼──┐ │    │  │ │  7 │
│   │                ├───┼───────┼─► F│ SM1├─┼────┼─►│G├────┤
│   │                │   │       │ └──┴────┘ │    │  │ │  8 │
│   │     Pixel      │   │       │           │    │  │ ├────┤
│   │     Buffer     │   │       └───────────┘    │  │ │  9 │
│   │                │   │            PIO0        │  ├─┼────┤
│   └────────────────┘   │                        │  │ │ 10 │
│                        │                        │  │ ├────┤
│                        │                        │  │ │ 11 │
└────────────────────────┘                        └─►│B├────┤
           RAM                                       │ │ 12 │
                                                     │ ├────┤
                                                     │ │ 13 │
                                                     └─┴────┘
```
<!-- https://asciiflow.com/#/share/eJy9lktuwjAQhq9izZqFHQqF7KCIwiIqTauuvGERqkghCwRSEOIWVQ6DehpOUpMqIo%2FxI07TyI4c2f5nPv92khPE620AbnyIoh5E62OwAxdOHBIO7njo9DgcRcsZDURrHyR78cCB2FzXr2%2B0cB63VEov4r4QldCs00rxP3L8uMVh3eUo67Iruiyy6uQ0sgmFORotlEcBrM3elKFfZkCmGegVffbVNDp1U8I87p3kASd5D7dh%2FCmLVcmkpqojmR42m2BXVp95E%2FK0oLhCmaBOMTDxw7SkFzK%2FKbx5tLZDisZ1uv9M1oFUVmHY6GRlraX%2FSotSxj5a8%2F36zLAIqYbvsRuXmXqYuD%2Br18LgdDb2coSzrsIkiORRaj4buplNwU5ls3WtUYzt3pOr5QstDZAF7O6NXyVh1P6rZehDG03C2D2%2FPyyK2Nf8dExRpsJgf%2BLJdBCkjMax%2B%2BMpqShzaqxGWL9lTvgOhjOcfwAyLhr9) -->


## Hardware

```
┌───────────────────────────────────────────────────────────────┐
│                          RP2040 GPIO                          │
├─────┬─────┬───┬───┬───┬───┬───┬───┬───┬───┬────┬────┬────┬────┤
│  0  │  1  │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │ 10 │ 11 │ 12 │ 13 │
├─────┼─────┼───┴───┴───┴───┼───┴───┴───┴───┼────┴────┴────┴────┤
│ Hs  │ Vs  │      RED      │     GREEN     │       BLUE        │
└─────┴─────┴───────────────┴───────────────┴───────────────────┘
```
<!-- https://asciiflow.com/#/share/eJyrVspLzE1VssorzcnRUcpJrEwtUrJSqo5RqohRsrI0M9OJUaoEsowsQKyS1IoSICdG6dGUPUMdxcTkAUkFnCAowMjAxEDBPcDTH7ciuDlDHcFCwwDsJwUFQwhtBCaNwaQJmDQFk2Zg0hxMWoBJSzBpaAChDCEURLuh8bALJ49iSPiEQWlIinF1gacKMO0e5OrqhyKioODkE%2Bo6%2FFKPUq1SLQDfo6E1) -->

The first 14 GPIO pins of the RP2040 are dedicated to generating the VGA signal. The 4-bits of each color are fed through an R-2R resistor ladder DAC. The analog signals are then fed through a video buffer ([THS7316](https://www.ti.com/product/THS7316) or equivalent). The I2C bus of the BMC is also connected to the [DDC](https://en.wikipedia.org/wiki/Display_Data_Channel) pins of the VGA port and may be used in the future for display identification purposes.

Physical connection to the screen is done through a [TPD7S019](https://www.ti.com/product/TPD7S019) ESD protection circuit and a female DE-15 connector.


## Supported video modes:

The following [video modes](./hardware_soc_video.md) are supported:

* Mode 0x00 - 80x30 text mode at 640x480
* Mode 0x01 - 80x60 text mode at 640x480
* Mode 0x10 - 80x25 text mode at 640x400
* Mode 0x11 - 80x50 text mode at 640x400
* Mode 0x91 - 80x25 text mode at 640x200
* Mode 0x05 - 640x480 in 16 colours
* Mode 0x06 - 640x480 in 4 colours
* Mode 0x07 - 640x480 in 2 colours
* Mode 0x15 - 640x400 in 16 colours
* Mode 0x16 - 640x400 in 4 colours
* Mode 0x17 - 640x400 in 2 colours
* Mode 0x94 - 640x200 in 256 colours
* Mode 0x95 - 640x200 in 16 colours
* Mode 0x96 - 640x200 in 4 colours
* Mode 0x97 - 640x200 in 2 colours
* Mode 0x8C - 320x240 in 256 colours
* Mode 0x8D - 320x240 in 16 colours
* Mode 0x8E - 320x240 in 4 colours
* Mode 0x8F - 320x240 in 2 colours
* Mode 0x9C - 320x200 in 256 colours
* Mode 0x9D - 320x200 in 16 colours
* Mode 0x9E - 320x200 in 4 colours
* Mode 0x9F - 320x200 in 2 colours
