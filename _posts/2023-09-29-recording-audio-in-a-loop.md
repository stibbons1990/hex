---
title:  "Recording audio in a loop"
date:   2023-09-29 23:09:29 +0200
categories: audio recording alsa headphones
---

Sometimes I wish I could hear what someone just said.
Most of the times, me, which is *extra hard* to
recall with precision. We tend to remember what we
*meant* to say, not the actual exact same words.

Normally it takes *only* about an hour, maybe two,
to realize of the desire to *rewind* and listen back.

## Simple audio recording

### Analog mic from gaming headphones

Lets start with the basic: record from an old mic.

**Lexicon** beign an Intel NUC11PAHi3, it has a front
2.5 mm connector for the 4-contact headphone+mic
headset, which is precisely what I have left over
from the old days before I got wireless headphones.

The first question is whether the soundcard for that
connector is supported and detected by the kernel:

```
# dmesg | egrep -i 'audio|sound|snd' | grep -vi hdmi
[    8.525267] snd_hda_intel 0000:00:1f.3: DSP detected with PCI class/subclass/prog-if info 0x040380
[    8.525325] snd_hda_intel 0000:00:1f.3: enabling device (0000 -> 0002)
[    8.527228] snd_hda_intel 0000:00:1f.3: bound 0000:00:02.0 (ops i915_audio_component_bind_ops [i915])
[    8.568725] snd_hda_codec_realtek hdaudioC0D0: autoconfig for ALC256: line_outs=1 (0x21/0x0/0x0/0x0/0x0) type:hp
[    8.568731] snd_hda_codec_realtek hdaudioC0D0:    speaker_outs=0 (0x0/0x0/0x0/0x0/0x0)
[    8.568732] snd_hda_codec_realtek hdaudioC0D0:    hp_outs=0 (0x0/0x0/0x0/0x0/0x0)
[    8.568734] snd_hda_codec_realtek hdaudioC0D0:    mono: mono_out=0x0
[    8.568735] snd_hda_codec_realtek hdaudioC0D0:    inputs:
[    8.568736] snd_hda_codec_realtek hdaudioC0D0:      Internal Mic=0x13
[    8.568737] snd_hda_codec_realtek hdaudioC0D0:      Internal Mic=0x12
[    8.652690] input: HDA Intel PCH Headphone as /devices/pci0000:00/0000:00:1f.3/sound/card0/input4
```

**Note:** without `grep -vi hdmi` there are 12
additional inputs, not the ones we want now.

`/devices/pci0000:00/0000:00:1f.3/sound/card0/input4`
looks like the one we want, but how do we record?

I search for a while how to do this without ALSA or
Pulseadio, but in the end gave in to ALSA:

```
# apt install alsa-utils -y
# arecord -L | grep -C1 'HDA Intel'
hw:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Direct hardware device without any conversions
plughw:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Hardware device with all software conversions
default:CARD=PCH
    HDA Intel PCH, ALC256 Analog
    Default Audio Device
sysdefault:CARD=PCH
    HDA Intel PCH, ALC256 Analog
    Default Audio Device
front:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Front output / input
dsnoop:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Direct sample snooping device

# aplay -L | grep -C1 'HDA Intel.*Analog'
hw:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Direct hardware device without any conversions
--
plughw:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Hardware device with all software conversions
--
default:CARD=PCH
    HDA Intel PCH, ALC256 Analog
    Default Audio Device
sysdefault:CARD=PCH
    HDA Intel PCH, ALC256 Analog
    Default Audio Device
front:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    Front output / input
surround21:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    2.1 Surround output to Front and Subwoofer speakers
surround40:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    4.0 Surround output to Front and Rear speakers
surround41:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    4.1 Surround output to Front, Rear and Subwoofer speakers
surround50:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    5.0 Surround output to Front, Center and Rear speakers
surround51:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    5.1 Surround output to Front, Center, Rear and Subwoofer speakers
surround71:CARD=PCH,DEV=0
    HDA Intel PCH, ALC256 Analog
    7.1 Surround output to Front, Center, Side, Rear and Woofer speakers
--
```

To **list** the available input devices capable of
capturing (recording) audio:

```
# arecord -l
**** List of CAPTURE Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC256 Analog [ALC256 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: S7 [SteelSeries Arctis 7], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

To record from the microphone in the onboard jack:

```
# arecord \
  -D plughw:1,0 \
  -f S16_LE /tmp/test__.wav
Recording WAVE '/tmp/test__.wav' : Signed 16 bit Little Endian, Rate 8000 Hz, Mono
^CAborted by signal Interrupt...
```

To record from the microphone in the USB headphones:

```
# arecord \
  -D plughw:CARD=S7,DEV=1 \
  -f S16_LE /tmp/test_S7_1.wav
Little Endian, Rate 8000 Hz, Mono
^CAborted by signal Interrupt...
```

The problem with this is... well, these are
*headphone* microphones, actually *gaming*
headphones, so they only capture audio from
the mouth a few milimeters from the microphone
and very nearly nothing at all from anything
else. This is very good performance for their
intented use case, useless of today's goal.

### Webcam microphone (much better)

The only other microphone available is an old webcam, never
used otherwise. This begin a common
[Logitec C920 HD Pro](https://www.logitech.com/en-ch/products/webcams/c920-pro-hd-webcam.960-001055.html),
it is well supported:

```
[204236.991023] usb 3-2: new high-speed USB device number 4 using xhci_hcd
[204237.709488] usb 3-2: New USB device found, idVendor=046d, idProduct=0892, bcdDevice= 0.19
[204237.709496] usb 3-2: New USB device strings: Mfr=0, Product=2, SerialNumber=1
[204237.709501] usb 3-2: Product: HD Pro Webcam C920
[204237.709504] usb 3-2: SerialNumber: B9BAB6EF
[204238.583154] videodev: Linux video capture interface: v2.00
[204238.617035] gspca_main: v2.14.0 registered
[204238.621950] gspca_main: vc032x-2.14.0 probing 046d:0892
[204238.622360] gspca_vc032x: reg_r err -32
[204238.622367] vc032x: probe of 3-2:1.0 failed with error -32
[204238.622404] usbcore: registered new interface driver vc032x
[204238.634075] usb 3-2: Found UVC 1.00 device HD Pro Webcam C920 (046d:0892)
[204238.636766] input: HD Pro Webcam C920 as /devices/pci0000:00/0000:00:14.0/usb3/3-2/3-2:1.0/input/input20
[204238.637655] usbcore: registered new interface driver uvcvideo
```

Once again, list capture capable devices and record:

```
# arecord -l
**** List of CAPTURE Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC256 Analog [ALC256 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: C920 [HD Pro Webcam C920], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0

# arecord \
  -D plughw:CARD=C920,DEV=0 \
  -f S16_LE /tmp/test_C920.wav
```

At first this produce *barely audible* sound, need to increase
the capture volume, but not too much (needed to try a few
values to find a good one). More importantly, use `-f cd` to
capture audio in a better format:

```
# amixer -c1 set Mic 70%
Simple mixer control 'Mic',0
  Capabilities: cvolume cvolume-joined cswitch cswitch-joined
  Capture channels: Mono
  Limits: Capture 0 - 60
  Mono: Capture 42 [70%] [41.00dB] [on]

# arecord \
  -D plughw:CARD=C920,DEV=0 \
  -f cd /tmp/test_C920.wav
```

### Clicky noises (~10 Hz)

Audio recording devices are affected by nearby electromagnetic
waves, including those produced by cables. Power cables and
power extensions in particular can be most problematic, so
the webcam (or microphone) must be kept well away from those.

## Loop recording

This example from
[the `arecord(1)` man page](https://linux.die.net/man/1/arecord)
shows how to record indefinitely with a new file per hour:

> **`arecord -f cd -t wav --max-file-time 3600 --use-strftime %Y/%m/%d/listen-%H-%M-%v.wav`**  
> Record in stereo from the default audio source. Create a new file every hour. The files are placed in directories based on their start dates and have names which include their start times and file numbers.

```
# arecord \
  -f cd \
  -t wav \
  -D plughw:CARD=C920,DEV=0 \
  --max-file-time 3600 \
  --use-strftime %Y-%m-%d-%H-%M-%v.wav
```

A 5-minute recording in WAVE format takes about 50 MB, so each
1-hour file should be about 600 MB. Keeping a whole day of such
files would take about 15 GB which is *affordable*.

