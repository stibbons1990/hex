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

```
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

```
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
