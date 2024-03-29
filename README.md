# pi2co-usb

This project is based on [i2c-tiny-usb](https://github.com/harbaum/I2C-Tiny-USB/) by Till Harbaum and the [V-USB stack](https://www.obdev.at/products/vusb/index.html) by Objective Development. It uses the fact that you can implement a low-speed USB device with the IO pins on an Atmel AVR chip to create a very simple I2C bridge.

![](photos/header.jpg)

## Hardware

In the original `i2c-tiny-usb` implementation, an Atmel AVR ATtiny45 was used with an external oscillator. Unfortunately this also meant that you needed to use all six pins on the chip – including the RESET pin. Should anything ever go wrong you would need a high-voltage serial programmer (HVSP).

On more recent Digispark devices, which use an ATtiny85, a modified version of the V-USB stack is used. They use only the internal PLL oscillator calibrated to 16.5 MHz. No external crystal is required, ISP remains available and you can even connect an indicator LED on the remaining pin!

In 2020 I soldered [a protype with a DIP ATtiny85](prototype/README.md) and a small leftover piece of protoboard – enough to build myself an I2C bridge! This year I updated the design and created a custom PCB in KiCad. It brings an additional voltage regulator for 3.3V, a selection slider between the two available voltages, a [bidirectional level shifter based on two N-channel MOSFETs](https://cdn-shop.adafruit.com/datasheets/AN10441.pdf) and a standard JST-SH connector; the latter is used by a number of different maker vendors under the names [Qwiic (SparkFun)](https://www.sparkfun.com/qwiic) or [STEMMA QT (Adafruit)](https://learn.adafruit.com/introducing-adafruit-stemma-qt/what-is-stemma-qt). *Note: for interoperability with Qwiic devices **always use 3.3V**!*

The [schematic](hardware/schematic.pdf) and the KiCad project can be found in [`hardware/`](hardware/).

![](photos/side-by-side-top.jpg)
![](photos/side-by-side-bottom.jpg)

Since the ATtiny85 is basically the same, you can use the exact same bootloader and firmware as I used for the prototype; the rest of this README thus still applies.

## Software

The firmware is almost fully plucked from Till Harbaum's `i2c-tiny-usb` project, so most credit goes to him. I converted the directory to a PlatformIO project, updated the vendored V-USB stack and modified the configuration header to use a different pinout.

It is meant to work together with a current [micronucleus](https://github.com/micronucleus/micronucleus/) bootloader, which allows loading new firmware over the same USB port that is used in the userspace firmware. This implementation also relies on the oscillator calibration included in the bootloader to achieve a stable 16.5 MHz clock.

### Flash the Bootloader

At first I used a default release for the ATtiny85 that I flashed with an ISP programmer and later changed a few options to enable the indicator LED, enter the bootloader only after shorting the RESET pin and disable the timeout. The configuration can be found in `bootloader/t85_pi2co` and can be linked or copied to a micronucleus project under `firmware/configuration/`.

For example, to compile the firmware with the config profile above and flash it with an [Adafruit FT232H breakout](https://wiki.semjonov.de/tips/arduino.html#adafruit-ftdi-ft232h-breakout-board):

```sh
make clean
make CONFIG=t85_pi2co
make CONFIG=t85_pi2co PROGRAMMER="-c ft232h" flash
make CONFIG=t85_pi2co PROGRAMMER="-c ft232h" fuse
```

After that you should be able to speak to the chip with the `micronucleus` commandline application if you connect a USB breakout appropriately. **To enter the bootloader later**, you'll need to reset the chip by briefly shorting the RESET pin to ground.

To allow normal users to speak with the bootloader, apply the
udev rules found in [49-micronucleus.rules](https://github.com/micronucleus/micronucleus/blob/e74ce6f064e0bcbe1c52459a0988187c76834222/commandline/49-micronucleus.rules).

### Flash the Firmware

Now that the bootloader works, install [PlatformIO](https://platformio.org/) and go to the `firmware/` directory. You can now compile the firmware simply by running `pio run`. PlatformIO should handle all framework and platform dependencies for you.

Afterwards flash the firmware by running:

```sh
pio run -t upload
```

**Note:** PlatformIO currently ships an outdated version of the `micronucleus` commandline tool, which won't speak to the current bootloader releases. Either update the binary in `~/.platformio/packages/tool-micronucleus/` or use a current binary with the compiled firmware directly:

```sh
micronucleus .pio/build/tiny85/firmware.hex
```

### Kernel driver

The `i2c-tiny-usb` project has been around for a while, so it already has a driver in the mainline Linux kernel. I only needed to `modprobe i2c-dev` for the device to appear. Install the `i2c-tools` in your Linux distribution of choice and start talking to I2C devices. You may need to run the tools as `root`.

As an example I am reading the date and time from a DS3231 RTC clock:

```sh
$ sudo i2cdetect -l
[sudo] password for ansemjo: 
/* ... */
i2c-10	i2c       	i2c-tiny-usb at bus 001 device 028	I2C adapter
/* ... */

$ sudo i2cdetect -y 10
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- 57 -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- --  << this is the rtc
70: -- -- -- -- -- -- -- --                         

$ i=0; for value in seconds minutes hours weekday day month year; do \
> printf '%7s: %s\n' $value "$(sudo i2cget -y 10 0x68 $i)"; : $((i++)); done
seconds: 0x39
minutes: 0x07
  hours: 0x19
weekday: 0x07
    day: 0x15
  month: 0x03
   year: 0x20
```

The bytes need BCD decoding but you can see the date was `2020-03-15 19:07:39`.
