---
title:  "Simple QMK firmware for the RoMac macro pad"
date:   2023-05-31 23:05:31 +0200
categories: keyboard macropad qmk firmware
---

QMK firmware for the RoMac macro pad was not *simple enough*
for me, so I had to make my own.

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

The [RoMac Macro Pad](https://github.com/The-Royal/The_Royal_Open-Source-Projects/blob/master/01%20-%20Complete%20Kits/The_RoMac_rev2.1/README.md)
is a wonderful, very useful 12-key macro pad that can be a
custom numpad or, better yet, an *additional* numpad for the
left hand, just not for numbers.
This macro pad, which can be purchased from
[customkbd.com](https://customkbd.com/collections/numpad-and-macropad/products/romac-2-1)
(Australia) or
[mechboards.co.uk](https://mechboards.co.uk/collections/kits/products/romac-macro-pad?variant=40366789034189)
(United Kingdom), the latter also having
[Relegendable Keycaps](https://mechboards.co.uk/collections/keycaps/products/mx-relegendable-keycap)
that are great to custom-label each key.

![RoMac macro pad as seen from above]({{ media }}/romac-macro-pad.png)

**Warning: this is not a keyboard you can *just buy***.
To build this keyboard you need to buy [the kit](https://mechboards.co.uk/collections/kits/products/romac-macro-pad?variant=40366789034189),
*with* a controller (e.g. the
[Pro Micro](https://mechboards.co.uk/collections/controllers/products/pro-micro-5v)),
*plus* **20 switches** *and* **20 keycaps**, *plus* a USB
cable (micro-B or C depending on the controller).
*And then* you have to *actually build* the keyboard,
following the (very detailed)
[RoMac rev2.1 Build Guide](https://imgur.com/a/l24vgvC),
which involves *a lot* of soldering.
There is [another build guide](https://imgur.com/a/rjeD0Pn)
for the [Fauxmac (RoMac+)](https://mechboards.co.uk/collections/kits/products/romac-macro-pad?variant=41666504753357)
variant (with OLED screen and RGB backlight).

*And then* you have to choose, or write, a keymap for it.

Although not included in the upstream
[QMK firmware](https://github.com/qmk/qmk_firmware/tree/master/keyboards/kingly_keys/romac#romac),
my preferred use for this keyboard is adding back the
[F13-F24 keys](https://dev.to/grahamthedev/did-you-know-there-are-f13-f24-keys-368p)
some old keyboards used to have. This is *very* useful to
trigger actions or scripts by pressiong **a single key**
without taking the right hand away from the mouse.

So here is how to create the simplest keymap: F13-F24 keys.

First, clone the
[QMK firmware](https://github.com/qmk/qmk_firmware)
repository:

```
$ python3 -m pip install -U qmk
$ git clone --recurse-submodules \
  https://github.com/qmk/qmk_firmware.git
$ cd qmk_firmware
$ qmk setup -y
```

Then create a new file (and folder)
`keyboards/kingly_keys/romac/keymaps/simple/keymap.c`
with this code:

```c
/* Copyright 2023 Ponder Stibbons
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 2 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program.  If not, see <http://www.gnu.org/licenses/>.
 */

#include QMK_KEYBOARD_H

#define _BASE 0
#define _FN1 1

const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS] = {

        [_BASE] = LAYOUT(
                KC_F13, KC_F14, KC_F15, \
                KC_F16, KC_F17, KC_F18, \
                KC_F19, KC_F20, KC_F21, \
                KC_F22, KC_F23, KC_F24 \
        )
};
```

Finally, compile and *flash* the keymap into the keyboard.

**Note:** `qmk flash` won't work unless the keyboard is
plugged in and you may need to press the reset button
(the tiny black button on the PCB) at the right time:

```
$ qmk compile -kb kingly_keys/romac -km simple
$ qmk flash -kb kingly_keys/romac -km simple
Ψ Compiling keymap with make --jobs=1 kingly_keys/romac:simple:flash


Making kingly_keys/romac with keymap simple and target flash

avr-gcc (GCC) 5.4.0
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

Size before:
   text    data     bss     dec     hex filename
      0   16234       0   16234    3f6a kingly_keys_romac_simple.hex

Copying kingly_keys_romac_simple.hex to qmk_firmware folder                                         [OK]
Checking file size of kingly_keys_romac_simple.hex                                                  [OK]
 * The firmware size is fine - 16234/28672 (56%, 12438 bytes free)
Flashing for bootloader: caterina
Waiting for USB serial port - reset your controller now (Ctrl+C to cancel)......
Device /dev/ttyACM0 has appeared; assuming it is the controller.
Waiting for /dev/ttyACM0 to become writable.

Connecting to programmer: .
Found programmer: Id = "CATERIN"; type = S
    Software Version = 1.0; No Hardware Version given.
Programmer supports auto addr increment.
Programmer supports buffered memory access with buffersize=128 bytes.

Programmer supports the following devices:
    Device code: 0x44

avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.00s

avrdude: Device signature = 0x1e9587 (probably m32u4)
avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
avrdude: erasing chip
avrdude: reading input file ".build/kingly_keys_romac_simple.hex"
avrdude: input file .build/kingly_keys_romac_simple.hex auto detected as Intel Hex
avrdude: writing flash (16234 bytes):

Writing | ################################################## | 100% 1.19s

avrdude: 16234 bytes of flash written
avrdude: verifying flash memory against .build/kingly_keys_romac_simple.hex:
avrdude: load data flash data from input file .build/kingly_keys_romac_simple.hex:
avrdude: input file .build/kingly_keys_romac_simple.hex auto detected as Intel Hex
avrdude: input file .build/kingly_keys_romac_simple.hex contains 16234 bytes
avrdude: reading on-chip flash data:

Reading | ################################################## | 100% 0.13s

avrdude: verifying ...
avrdude: 16234 bytes of flash verified

avrdude: safemode: Fuses OK (E:CB, H:D8, L:FF)

avrdude done.  Thank you.
```

Now the keyboard is ready to emit the keycodes for the
function keys from F13 to F24, but those keycodes may be
already assigned by your OS to something else.

In Linux systems, Xorg assigns a few of these keycodes to
vendor-specific keys found in laptops (e.g. mute mic).
To override these, add the following lines to your
`~/.Xmodmap` file:

```
keycode 191 = F13 F13 F13
keycode 192 = F14 F14 F14
keycode 193 = F15 F15 F15
keycode 194 = F16 F16 F16
keycode 195 = F17 F17 F17
keycode 196 = F18 F18 F18
keycode 197 = F19 F19 F19
keycode 198 = F20 F20 F20
keycode 199 = F21 F21 F21
keycode 200 = F22 F22 F22
keycode 201 = F23 F23 F23
keycode 202 = F24 F24 F24
```

What you do with these is now up to you, and depends on your
OS and desktop environments. For instance, in KDE Plasma you
can assign these to standard
[Keyboard Shortcuts](https://docs.kde.org/stable5/en/khelpcenter/fundamentals/shortcuts.html)
or create more flexible (powerful)
[Custom Shortcuts](https://docs.kde.org/stable5/en/khotkeys/kcontrol/khotkeys/khotkeys.pdf).