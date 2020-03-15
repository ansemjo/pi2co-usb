These release files were compiled on 2020-03-15 using the steps
detailed in the project's README.md.


01. Flash the Bootloader
========================

First, flash the bootloader with your favourite ISP programming
method with avrdude. Personally, I like using an Adafruit FT232H
breakout for this:

  avrdude -c UM232H -p t85 -U flash:w:bootloader.hex:i

Don't forget to set the fuses if this is the first time flashing
micronucleus:

  avrdude -c UM232H -p t85 \
    -U lfuse:w:0xe1:m -U hfuse:w:0xdd:m -U efuse:w:0xfe:m


02. Verify USB connection
=========================

Connect a USB breakout to the ATtiny85 to verify that the command-
line tool can speak to the micronucleus bootloader. The pull-up
resistor on D- is required but depending on your computer you may
be fine without the zener diodes. This is not to specification though!

  VCC -->  VCC (pin 8)
  D-  -->  PB3 (pin 2)
  D+  -->  PB4 (pin 3)
  GND -->  GND (pin 4)

You should see a device appear on the bus and micronucleus should
be able to program the chip:

  micronucleus --erase-only


03. Upload the Firmware
=======================

Now use micronucleus to upload the firmware:

  micronucleus ./firmware.hex

After you have uploaded the firmware and replugged the USB cable, the
chip should execute the firmware immediately and the i2c-tiny-usb
should appear on the bus:

  ID 0403:c631 Future Technology Devices International, Ltd i2c-tiny-usb interface

To re-enter the bootloader, briefly short the RESET pin (pin 1) to
GND with some tweezers, for example.
