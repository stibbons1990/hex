---
title:  "xHCI host controller not responding, assume dead"
date:   2023-08-27 20:08:27 +0200
categories: linux xhci usb hardware failure glitch workaround timeout
---

Today, a number of USB devices were unavailable for no apparent reason.
 
Having a terminal always visible running `dmesg -w` the following showed up:

```
xhci_hcd 0000:0b:00.3: Abort failed to stop command ring: -110
xhci_hcd 0000:0b:00.3: xHCI host controller not responding, assume dead
xhci_hcd 0000:0b:00.3: HC died; cleaning up
```

Hmm, *assume dead...* that doesn’t sounds good.

Searching for posts about these error messages,
the best match that showed up was 
[techoverflow.net/2021/09/16/how-i-fixed-xhci-host-controller-not-responding-assume-dead](https://techoverflow.net/2021/09/16/how-i-fixed-xhci-host-controller-not-responding-assume-dead/)
which summarizes the discussion and findings from this thread in the
ArchLinux forums:
[xHCI host controller not responding, assume dead](https://bbs.archlinux.org/viewtopic.php?pid=1855486#p1855486).

The solution provided there worked in this case too, without even waiting for 5 seconds between the unbind and bind commands:

```
echo -n 0000:0b:00.3 > /sys/bus/pci/drivers/xhci_hcd/unbind
echo -n 0000:0b:00.3 > /sys/bus/pci/drivers/xhci_hcd/bind
```

After restarting this controller a few times, it was never again problematic.

Created script for future use and reference:

```bash
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