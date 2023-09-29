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
