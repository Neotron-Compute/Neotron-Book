# SDMMC

The SD Card Slot takes microSD cards (frustratingly, full size SD card slots don't really fit on the front edge with the Joystick ports, because of the placement of the PCB/case screw holes). These are connected up using the *SPI Compatibility* mode built into all SD and SDHC memory cards. No interrupt line is required - the SD card responds only when polled and gives out repeated `0xFF` bytes when busy completing an operation.
