# nanoBoot

[![Build](https://github.com/volium/nanoBoot/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/volium/nanoBoot/actions/workflows/build.yml)

This repository contains the source code for the USB HID-based bootloader for ATmegaXXU4 family of devices.

The name *nanoBoot* comes from the fact that the compiled source fits in the smallest available boot size on the ATMegaXXu4 devices, 256 words or 512 bytes. The code is based on Dean Camera's [LUFA](https://github.com/abcminiuser/lufa) USB implementation, but it is **EXTREMELY** streamlined, size-optimized and targeted for the [ATmega16U4](http://www.atmel.com/devices/atmega16u4.aspx) and [ATmega32u4](http://www.atmel.com/devices/atmega32u4.aspx) devices; I had to make quite a few hardware assumptions, mostly to the fuse settings related to clock configuration for things to be as compact as possible, but the code still allows for some flexibility.

It's very likely that a few sections can be rewritten to make it even smaller, and the ultimate goal is to support EEPROM programming as well, although that would require changes to the host code.

The current version (commit #[d0ea26b](https://github.com/volium/nanoBoot/commit/d0ea26bb01e764340dc8ad7b473ad98cefdb52eb)) is supported as-is in the 'hid_bootloader_loader.py' script that ships with [LUFA-151115](https://github.com/abcminiuser/lufa/releases/tag/LUFA-151115), and is exactly 506 bytes long.

## TL;DR WINDOWS

1. in build.zip there are the compiled files. the .hex is what you need
2. download avrdude (software to program the dev boards like arduinos)
3. (you'll have to look elsewhere how to program an arduino using a hardware programmer, I used an Arduino Uno to program a pro micro clone)
4. flash the nanoBoot bootloader   
`avrdude.exe -p atmega32u4 -c stk500v1 -b 19200 -U flash:w:"nanoBoot.hex":i -P COM10 -U efuse:w:0xC3:m -U hfuse:w:0xD9:m -U lock:w:0x3F:m`
5. Since it didn't recognize it on my machine and couldn't use qmk_toolbox , I had to flash the keyboard firmware (that's what i'm using this for) with avrdude and the hardware programmer
6. had to add/replace this on the rules.mk for the keyboard   
`BOOTLOADER = qmk-hid`   
`BOOTLOADER_SIZE = 512`
7. compile the firmware. This is not for this repo, but I'll add it for documentation purposes (my own good) . In QMK_SYS console:   
`make klor:vial`
8. - upload the compiled keyboard firmware using the hardware programmer:   
`avrdude -b 19200 -c arduino_as_isp -p m32u4 -v -e -U efuse:w:0x05:m -U hfuse:w:0xD6:m -U lfuse:w:0xFF:m -P COM10`     
`avrdude -b 19200 -c arduino_as_isp -p m32u4 -v -e -U flash:w:klor_vial.hex -U lock:w:0x0F:m -P COM10`   
The -c parameter is your programmer, arduino uno with ISP sketch, you can find the list with `avrdude -c ?`. The target board is specified with the -p paramenter. You can list the valid parameters with `avrdude -p ?`
   - alternative working with bootloader+firmware HEX:
   `avrdude -b 19200 -c arduino_as_isp -p m32u4 -P COM10 -U flash:w:"klor_vial_production.hex":a -U lfuse:w:0x5E:m -U hfuse:w:0xD9:m -U efuse:w:0xC3:m -U lock:w:0x3F:m ` 

## HW assumptions:

* CLK is 16 MHz Crystal and fuses are setup correctly to support it:
    * Select Clock Source (CKSEL3:CKSEL0) fuses are set to Extenal Crystal, CKSEL=1111 SUT=11
    * Divide clock by 8 fuse (CKDIV8) can be set to either 0 or 1
* Bootloader starts on reset; Hardware Boot Enable fuse is configured, HWBE=0
* Boot Flash Size is set correctly to 256 words (512 bytes), StartAddress=0x3F00, BOOTSZ=11
* Device signature = 0x1E9587

* Fuse Settings:
    * lfuse memory = 0xFF or 0x7F (CKDIV8=1 or 0, 16CK+65ms)
    * hfuse memory = 0xD6 (EESAVE=0, BOOTRST=0)
    * efuse memory = 0xC7 (=0xF7, No BOD)

* Alternatively, BOD can be used to ease CKSEL-SUT setting requirements to
  allow teensy-like FUSE settings:
    * lfuse memory = 0x5F (CKDIV8=0, 16CK + 0ms)
    * hfuse memory = 0xDF (EESAVE=1, BOOTRST=1)
    * efuse memory = 0xF4 (BOD=2.4V)

The documentation is part of the source code itself, and even though some people may find it extremely verbose, I think that's better than lack of documentation; after all, assembly can be hard to read sometimes... ohhh yes, in case that was not expected, this is all written in pure GAS (GNU Assembly), compiled using the [Atmel AVR 8-bit Toolchain](http://www.atmel.com/tools/atmelavrtoolchainforwindows.aspx).

## Toolchain installation

 - ### MacOS
    - Install [Homebrew](https://brew.sh/):

        `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
    
    - Install the avr-gcc toolchain via [osx-cross/avr](https://github.com/osx-cross/homebrew-avr):
        
        - Make sure **Xcode Command Line Tools** are installed:
        
            `xcode-select --install`
        
        - If **Xcode Command Line Tools** are installed, make sure they are up-to-date:
            
            `softwareupdate --all --install --force `
        
        - Install the latest version of avr-gcc:
            
            `brew tap osx-cross/avr`
            
            `brew install avr-gcc`
        
            **NOTE:** I personally experienced an issue with unidentified `___gmpz_X` symbols for `architecture arm64` during the installation on a new M1 MacBook Pro 14 inch; I was able to resolve this by completely removing `gmp` (`brew uninstall --force gmp`) and retrying the installation.

        - Install avrdude to be able to flash the device:
            
            `brew install avrdude`

