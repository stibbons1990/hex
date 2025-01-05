---
date: 2023-08-27
categories:
 - linux
 - xhci
 - usb
 - hardware
 - failure
 - glitch
 - workaround
 - timeout
title: xHCI host controller not responding, assume dead
---

Today, a number of USB devices were unavailable for no apparent reason.
 
Having a terminal always visible running `dmesg -w` the following showed up:

``` dmesg
xhci_hcd 0000:0b:00.3: Abort failed to stop command ring: -110
xhci_hcd 0000:0b:00.3: xHCI host controller not responding, assume dead
xhci_hcd 0000:0b:00.3: HC died; cleaning up
```

Hmm, *assume dead...* that doesn’t sounds good.

<!-- more --> 

Searching for posts about these error messages,
the best match that showed up was 
[techoverflow.net/2021/09/16/how-i-fixed-xhci-host-controller-not-responding-assume-dead](https://techoverflow.net/2021/09/16/how-i-fixed-xhci-host-controller-not-responding-assume-dead/)
which summarizes the discussion and findings from this thread in the
ArchLinux forums:
[xHCI host controller not responding, assume dead](https://bbs.archlinux.org/viewtopic.php?pid=1855486#p1855486).

The solution provided there worked in this case too, without even waiting for 5 seconds between the unbind and bind commands:

``` console
echo -n 0000:0b:00.3 > /sys/bus/pci/drivers/xhci_hcd/unbind
echo -n 0000:0b:00.3 > /sys/bus/pci/drivers/xhci_hcd/bind
```

After restarting this controller a few times, it was never again problematic.

Created script for future use and reference:

``` bash linenums="1" title="restart-usb.sh"
#!/bin/bash
#
# Workaround for
#
# xhci_hcd 0000:0b:00.3: Abort failed to stop command ring: -110
# xhci_hcd 0000:0b:00.3: xHCI host controller not responding, assume dead
# xhci_hcd 0000:0b:00.3: HC died; cleaning up
#
# Run with 0000:0b:00.3 (id).
#
# Sources:
# https://techoverflow.net/2021/09/16/how-i-fixed-xhci-host-controller-not-responding-assume-dead/
 
id=$1
echo -n "$id" > /sys/bus/pci/drivers/xhci_hcd/unbind
sleep 1
echo -n "$id" > /sys/bus/pci/drivers/xhci_hcd/bind
```

#### Update (2023-12-09)

The same, or similar, problem happened again when leaving
the computer running overnight while most devices powered
off. What is most strange is that the problem seems to
have started when (be triggered by?) powering most USB
devices *off*:

``` dmesg
usb 5-2: USB disconnect, device number 3
usb 5-2.7: USB disconnect, device number 7
retire_capture_urb: 2 callbacks suppressed
usb 6-2: USB disconnect, device number 2
r8152-cfgselector 6-2.6: USB disconnect, device number 3
usb 5-3.2.3.1: USB disconnect, device number 24
usb 5-3.2.3.1.6: USB disconnect, device number 25
usb 5-4.1: USB disconnect, device number 8
usb 5-4.1.6: USB disconnect, device number 10
xhci_hcd 0000:0a:00.3: Abort failed to stop command ring: -110
xhci_hcd 0000:0a:00.3: xHCI host controller not responding, assume dead
xhci_hcd 0000:0a:00.3: HC died; cleaning up
xhci_hcd 0000:0a:00.3: Timeout while waiting for configure endpoint command
xhci_hcd 0000:0a:00.3: Unsuccessful disable slot 23 command, status 25
usb 5-1: USB disconnect, device number 2
usb 5-3.3: Not enough bandwidth for altsetting 0
usb 5-3: USB disconnect, device number 4
usb 5-3.1: USB disconnect, device number 6
usb 5-3.2: USB disconnect, device number 9
usb 5-3.2.1: USB disconnect, device number 13
usb 5-3.4.4.1: Not enough bandwidth for altsetting 0
usb 5-3.4.4.1: 1:0: usb_set_interface failed (-19)
```

#### Update (2025-01-05)

Again, lots of errors in `dmesg` when startint the PC after a 2-week break:

```dmesg
[    6.794420] usb 5-1.1: new high-speed USB device number 17 using xhci_hcd
[    6.806582] nvidia-nvlink: Nvlink Core is being initialized, major device number 236
[    6.807921] nvidia 0000:09:00.0: vgaarb: VGA decodes changed: olddecodes=io+mem,decodes=none:owns=none
[    6.852458] NVRM: loading NVIDIA UNIX x86_64 Kernel Module  550.120  Fri Sep 13 10:10:01 UTC 2024
[    6.863691] nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver for UNIX platforms  550.120  Fri Sep 13 10:01:25 UTC 2024
[    6.881884] [drm] [nvidia-drm] [GPU ID 0x00000900] Loading driver
[    6.905141] input: HDA NVidia HDMI/DP,pcm=7 as /devices/pci0000:00/0000:00:03.1/0000:09:00.1/sound/card0/input23
[    6.905812] input: HDA NVidia HDMI/DP,pcm=8 as /devices/pci0000:00/0000:00:03.1/0000:09:00.1/sound/card0/input24
[    6.905950] input: HDA NVidia HDMI/DP,pcm=9 as /devices/pci0000:00/0000:00:03.1/0000:09:00.1/sound/card0/input25
[    6.908061] input: HD-Audio Generic Rear Mic as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input27
[    6.908214] input: HD-Audio Generic Line as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input28
[    6.908300] input: HD-Audio Generic Line Out Front as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input29
[    6.908403] input: HD-Audio Generic Line Out Surround as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input30
[    6.908502] input: HD-Audio Generic Line Out CLFE as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input31
[    6.908616] input: HD-Audio Generic Front Headphone as /devices/pci0000:00/0000:00:08.1/0000:0b:00.4/sound/card1/input32
[    7.634589] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[    7.802425] usb 5-1.1: new high-speed USB device number 18 using xhci_hcd
[    7.876154] BTRFS: device fsid e290b1ae-192c-46b9-9ddb-3680d3cefd73 devid 1 transid 606 /dev/nvme0n1p4 scanned by mount (1754)
[    7.876234] BTRFS: device fsid 6b809fc0-0b85-4041-ac25-47ec4682f5f5 devid 1 transid 28482 /dev/sdb scanned by mount (1755)
[    7.876483] BTRFS info (device nvme0n1p4): first mount of filesystem e290b1ae-192c-46b9-9ddb-3680d3cefd73
[    7.876498] BTRFS info (device nvme0n1p4): using crc32c (crc32c-intel) checksum algorithm
[    7.876502] BTRFS info (device nvme0n1p4): using free-space-tree
[    7.880380] BTRFS info (device sda): first mount of filesystem a4ee872d-b985-445f-94a2-15232e93dcd5
[    7.880386] BTRFS info (device sda): using crc32c (crc32c-intel) checksum algorithm
[    7.880390] BTRFS info (device sda): using free-space-tree
[    7.880737] BTRFS: device fsid 5cf65a95-4ae5-41ed-9a14-7d7fbeee1951 devid 1 transid 574916 /dev/sdc scanned by mount (1757)
[    7.880970] BTRFS info (device sdc): first mount of filesystem 5cf65a95-4ae5-41ed-9a14-7d7fbeee1951
[    7.880975] BTRFS info (device sdc): using crc32c (crc32c-intel) checksum algorithm
[    7.880979] BTRFS info (device sdc): disk space caching is enabled
[    7.884039] BTRFS info (device sdb): first mount of filesystem 6b809fc0-0b85-4041-ac25-47ec4682f5f5
[    7.884045] BTRFS info (device sdb): using crc32c (crc32c-intel) checksum algorithm
[    7.884048] BTRFS info (device sdb): using free-space-tree
[    7.891874] BTRFS info (device sdc): bdev /dev/sdc errs: wr 0, rd 0, flush 0, corrupt 1, gen 0
[    7.926088] [drm] Initialized nvidia-drm 0.0.0 20160202 for 0000:09:00.0 on minor 1
[    8.040524] nvidia_uvm: module uses symbols nvUvmInterfaceDisableAccessCntr from proprietary module nvidia, inheriting taint.
[    8.062807] nvidia-uvm: Loaded the UVM driver, major device number 234.
[    8.170951] usb 5-1.1: device descriptor read/64, error -71
[    8.466340] usb 5-1.1: New USB device found, idVendor=534d, idProduct=2109, bcdDevice=21.00
[    8.466348] usb 5-1.1: New USB device strings: Mfr=1, Product=0, SerialNumber=0
[    8.466351] usb 5-1.1: Manufacturer: MACROSILICON
[    8.506594] hid-generic 0003:534D:2109.0011: hiddev1,hidraw2: USB HID v1.10 Device [MACROSILICON] on usb-0000:0b:00.3-1.1/input4
[    8.529034] usb 5-1.4.4.2: new full-speed USB device number 19 using xhci_hcd
[    8.529417] mc: Linux media interface: v0.10
[    8.543738] videodev: Linux video capture interface: v2.00
[    8.552875] usb 5-1.1: Found UVC 1.00 device <unnamed> (534d:2109)
[    8.570883] usbcore: registered new interface driver uvcvideo
[    8.575010] Bluetooth: Core ver 2.22
[    8.575027] NET: Registered PF_BLUETOOTH protocol family
[    8.575028] Bluetooth: HCI device and connection manager initialized
[    8.575032] Bluetooth: HCI socket layer initialized
[    8.575034] Bluetooth: L2CAP socket layer initialized
[    8.575037] Bluetooth: SCO socket layer initialized
[    8.633644] usbcore: registered new interface driver btusb
[    8.639658] usb 5-1.4.4.2: New USB device found, idVendor=1038, idProduct=12c2, bcdDevice= 0.74
[    8.639664] usb 5-1.4.4.2: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[    8.639667] usb 5-1.4.4.2: Product: SteelSeries Arctis 9
[    8.639670] usb 5-1.4.4.2: Manufacturer: SteelSeries
[    8.708836] Bluetooth: hci0: BCM: chip id 63
[    8.709836] Bluetooth: hci0: BCM: features 0x07
[    8.725868] Bluetooth: hci0: BCM20702A
[    8.725873] Bluetooth: hci0: BCM20702A1 (001.002.014) build 0000
[    8.727715] Bluetooth: hci0: BCM: firmware Patch file not found, tried:
[    8.728070] Bluetooth: hci0: BCM: 'brcm/BCM20702A1-0b05-17cb.hcd'
[    8.728316] Bluetooth: hci0: BCM: 'brcm/BCM-0b05-17cb.hcd'
[    8.825867] hid-generic 0003:1038:12C2.0012: hiddev4,hidraw16: USB HID v1.11 Device [SteelSeries SteelSeries Arctis 9] on usb-0000:0b:00.3-1.4.4.2/input0
[    8.828010] input: SteelSeries SteelSeries Arctis 9 as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.4/5-1.4.4.2/5-1.4.4.2:1.1/0003:1038:12C2.0013/input/input33
[    8.879513] hid-generic 0003:1038:12C2.0013: input,hidraw17: USB HID v1.11 Device [SteelSeries SteelSeries Arctis 9] on usb-0000:0b:00.3-1.4.4.2/input1
[    8.889781] hid-generic 0003:1038:12C2.0014: hiddev5,hidraw18: USB HID v1.11 Device [SteelSeries SteelSeries Arctis 9] on usb-0000:0b:00.3-1.4.4.2/input2
[    9.080415] usb 5-1.4.4.4: new full-speed USB device number 20 using xhci_hcd
[    9.139554] usbcore: registered new interface driver snd-usb-audio
[    9.921744] usb 5-1.4.4.1: new full-speed USB device number 21 using xhci_hcd
[   10.009656] usb 5-1.4.4.1: New USB device found, idVendor=1038, idProduct=12c4, bcdDevice= 0.03
[   10.009663] usb 5-1.4.4.1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   10.009665] usb 5-1.4.4.1: Product: SteelSeries Arctis 9
[   10.009667] usb 5-1.4.4.1: Manufacturer: SteelSeries
[   10.009669] usb 5-1.4.4.1: SerialNumber: 000000000000
[   10.330714] hid-generic 0003:1038:12C4.0015: No inputs registered, leaving
[   10.331716] hid-generic 0003:1038:12C4.0015: hidraw19: USB HID v1.11 Device [SteelSeries SteelSeries Arctis 9] on usb-0000:0b:00.3-1.4.4.1/input5
[   16.544125] audit: type=1400 audit(1736080991.270:2): apparmor="STATUS" operation="profile_load" profile="unconfined" name="ch-run" pid=2463 comm="apparmor_parser"
[   16.544131] audit: type=1400 audit(1736080991.270:3): apparmor="STATUS" operation="profile_load" profile="unconfined" name="QtWebEngineProcess" pid=2456 comm="apparmor_parser"
[   16.544134] audit: type=1400 audit(1736080991.270:4): apparmor="STATUS" operation="profile_load" profile="unconfined" name="buildah" pid=2459 comm="apparmor_parser"
[   16.544136] audit: type=1400 audit(1736080991.270:5): apparmor="STATUS" operation="profile_load" profile="unconfined" name="element-desktop" pid=2468 comm="apparmor_parser"
[   16.544139] audit: type=1400 audit(1736080991.270:6): apparmor="STATUS" operation="profile_load" profile="unconfined" name=4D6F6E676F444220436F6D70617373 pid=2455 comm="apparmor_parser"
[   16.544141] audit: type=1400 audit(1736080991.270:7): apparmor="STATUS" operation="profile_load" profile="unconfined" name="chrome" pid=2464 comm="apparmor_parser"
[   16.544144] audit: type=1400 audit(1736080991.270:8): apparmor="STATUS" operation="profile_load" profile="unconfined" name="crun" pid=2466 comm="apparmor_parser"
[   16.544146] audit: type=1400 audit(1736080991.270:9): apparmor="STATUS" operation="profile_load" profile="unconfined" name="devhelp" pid=2467 comm="apparmor_parser"
[   16.544148] audit: type=1400 audit(1736080991.270:10): apparmor="STATUS" operation="profile_load" profile="unconfined" name="busybox" pid=2460 comm="apparmor_parser"
[   16.544151] audit: type=1400 audit(1736080991.270:11): apparmor="STATUS" operation="profile_load" profile="unconfined" name="balena-etcher" pid=2457 comm="apparmor_parser"
[   16.596655] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[   16.596768] Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[   16.596857] Loaded X.509 cert 'wens: 61c038651aabdcf94bd0ac7ff06c7248db18c600'
[   16.800779] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[   16.800782] Bluetooth: BNEP filters: protocol multicast
[   16.800789] Bluetooth: BNEP socket layer initialized
[   16.802639] Bluetooth: MGMT ver 1.22
[   16.806138] NET: Registered PF_ALG protocol family
[   16.956495] loop29: detected capacity change from 0 to 8
[   16.959123] block nvme0n1: No UUID available providing old NGUID
[   16.971925] NET: Registered PF_QIPCRTR protocol family
[   19.452853] igc 0000:05:00.0 enp5s0: NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX/TX
[   19.634474] systemd-journald[875]: Time jumped backwards, rotating.
[   20.856614] usb 5-1.1: reset high-speed USB device number 18 using xhci_hcd
[   21.706730] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   22.562724] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   22.730662] usb 5-1.1: reset high-speed USB device number 18 using xhci_hcd
[   23.146428] usb 5-1.1: device not accepting address 18, error -22
[   24.002722] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   24.003710] usb 5-1.1: USB disconnect, device number 18
[   24.914594] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   25.090417] usb 5-1.1: new high-speed USB device number 23 using xhci_hcd
[   25.805129] Bluetooth: RFCOMM TTY layer initialized
[   25.805136] Bluetooth: RFCOMM socket layer initialized
[   25.805140] Bluetooth: RFCOMM ver 1.11
[   25.970585] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   25.970751] usb 5-1-port1: attempt power cycle
[   27.122585] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   27.290416] usb 5-1.1: new high-speed USB device number 25 using xhci_hcd
[   27.290457] usb 5-1.1: Device not responding to setup address.
[   27.498459] usb 5-1.1: Device not responding to setup address.
[   27.706423] usb 5-1.1: device not accepting address 25, error -71
[   27.706502] usb 5-1-port1: unable to enumerate USB device
[   28.666719] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   28.834418] usb 5-1.1: new high-speed USB device number 27 using xhci_hcd
[   29.674720] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   29.675017] usb 5-1-port1: attempt power cycle
[   30.842707] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   31.030612] loop29: detected capacity change from 0 to 360192
[   31.698707] usb 5-1-port1: Cannot enable. Maybe the USB cable is bad?
[   31.698999] usb 5-1-port1: unable to enumerate USB device
[   75.560962] usb 5-1.4.3: new full-speed USB device number 30 using xhci_hcd
[   75.710609] usb 5-1.4.3: New USB device found, idVendor=046d, idProduct=c52b, bcdDevice=24.11
[   75.710616] usb 5-1.4.3: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[   75.710618] usb 5-1.4.3: Product: USB Receiver
[   75.710620] usb 5-1.4.3: Manufacturer: Logitech
[   75.840778] input: Logitech USB Receiver as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.0/0003:046D:C52B.0016/input/input35
[   75.937910] hid-generic 0003:046D:C52B.0016: input,hidraw2: USB HID v1.11 Keyboard [Logitech USB Receiver] on usb-0000:0b:00.3-1.4.3/input0
[   75.941846] input: Logitech USB Receiver Mouse as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.1/0003:046D:C52B.0017/input/input36
[   75.941972] input: Logitech USB Receiver Consumer Control as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.1/0003:046D:C52B.0017/input/input37
[   75.994233] input: Logitech USB Receiver System Control as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.1/0003:046D:C52B.0017/input/input38
[   75.994571] hid-generic 0003:046D:C52B.0017: input,hiddev1,hidraw20: USB HID v1.11 Mouse [Logitech USB Receiver] on usb-0000:0b:00.3-1.4.3/input1
[   75.998911] hid-generic 0003:046D:C52B.0018: hiddev6,hidraw21: USB HID v1.11 Device [Logitech USB Receiver] on usb-0000:0b:00.3-1.4.3/input2
[   76.295219] logitech-djreceiver 0003:046D:C52B.0018: hiddev1,hidraw2: USB HID v1.11 Device [Logitech USB Receiver] on usb-0000:0b:00.3-1.4.3/input2
[   76.405080] input: Logitech Wireless Device PID:406a Keyboard as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.2/0003:046D:C52B.0018/0003:046D:406A.0019/input/input40
[   76.464687] input: Logitech Wireless Device PID:406a Mouse as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.2/0003:046D:C52B.0018/0003:046D:406A.0019/input/input41
[   76.464808] hid-generic 0003:046D:406A.0019: input,hidraw20: USB HID v1.11 Keyboard [Logitech Wireless Device PID:406a] on usb-0000:0b:00.3-1.4.3/input2:1
[   76.646991] input: Logitech MX Anywhere 2S as /devices/pci0000:00/0000:00:08.1/0000:0b:00.3/usb5/5-1/5-1.4/5-1.4.3/5-1.4.3:1.2/0003:046D:C52B.0018/0003:046D:406A.0019/input/input45
[   76.703921] logitech-hidpp-device 0003:046D:406A.0019: input,hidraw20: USB HID v1.11 Keyboard [Logitech MX Anywhere 2S] on usb-0000:0b:00.3-1.4.3/input2:1
[   80.767209] logitech-hidpp-device 0003:046D:406A.0019: HID++ 4.5 device connected.
[   92.328643] show_signal: 123 callbacks suppressed
```

Yet eventually all USB devices seem to be available:

```console
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 1d6b:0104 Linux Foundation Multifunction Composite Gadget
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 002: ID 0b05:1939 ASUSTek Computer, Inc. AURA LED Controller
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 005 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 005 Device 002: ID 1a40:0101 Terminus Technology Inc. Hub
Bus 005 Device 003: ID 04a9:1912 Canon, Inc. LiDE 400
Bus 005 Device 005: ID 05e3:0610 Genesys Logic, Inc. Hub
Bus 005 Device 006: ID 041e:324d Creative Technology, Ltd Sound Blaster Play! 3
Bus 005 Device 007: ID 09ea:0130 Generic Virtual HUB
Bus 005 Device 008: ID 05e3:0610 Genesys Logic, Inc. Hub
Bus 005 Device 009: ID 09ea:0131 Generic Virtual HID
Bus 005 Device 010: ID 2109:2817 VIA Labs, Inc. USB2.0 Hub             
Bus 005 Device 011: ID 4b4b:0001 TheRoyalSweatshirt romac
Bus 005 Device 012: ID 3233:1311 Ducky Ducky One 3 RGB
Bus 005 Device 013: ID 046d:c545 Logitech, Inc. USB Receiver
Bus 005 Device 014: ID 0b05:17cb ASUSTek Computer, Inc. Broadcom BCM20702A0 Bluetooth
Bus 005 Device 015: ID 058f:9254 Alcor Micro Corp. Hub
Bus 005 Device 019: ID 1038:12c2 SteelSeries ApS SteelSeries Arctis 9
Bus 005 Device 021: ID 1038:12c4 SteelSeries ApS SteelSeries Arctis 9
Bus 005 Device 030: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 006 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```

After turning it off and on again a few hours later, nearly no
errors:

```dmesg
[   15.659476] usb 5-1.4.4.1: device descriptor read/64, error -32
[   15.855391] usb 5-1.4.4.1: New USB device found, idVendor=1038, idProduct=12c4, bcdDevice= 0.03
[   15.855397] usb 5-1.4.4.1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   15.855400] usb 5-1.4.4.1: Product: SteelSeries Arctis 9
[   15.855402] usb 5-1.4.4.1: Manufacturer: SteelSeries
[   15.855405] usb 5-1.4.4.1: SerialNumber: 000000000000
```

The output from `lsusb` is the same as above except for the order
of devices in `Bus 005`