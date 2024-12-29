---
date: 2022-10-12
categories:
 - linux
 - hardware
 - ubuntu
 - server
 - crucial
 - ssd
title: Crucial MX500 SSD found problematic
---

**Crucial MX500 SSD disks are problematic**, 
in strange ways they should not be.

<!-- more --> 

## Prologue

Encouraged by a sudden price fall (by 15% down to $300), and
spurred by the recent
[failure of 6TB HDD RAID](https://stibbons1990.github.io/hex/2022/09/27/undead-yes-unraid-no.html),
I added a
[Crucial MX500 4TB 3D NAND SATA SSD](https://www.crucial.com/ssd/mx500/ct4000mx500ssd1) to serve as an backup to some of my
precious files in that cursed RAID.

This *should* have been all too easy and trouble-free, but alas
these disks have something (in their firmware?) that makes these
disks *crash* when writing to them *too fast*.

Before diving into the details, it should be noted that this was
an entirely new problem, never encountered with Intel or Samsung
SATA SSDs, *at least* those used in this environment since 2010:

*  [Intel X25-V G2 40GB SATA-II](https://www.storagereview.com/review/intel-x25-v-ssd-review-40gb) (2010)
*  [Samsung SSD 840 Basic 120GB SATA](https://www.samsung.com/uk/support/model/MZ-7TD120BW/) (2013)
*  [Samsung 850 EVO Basic 250GB SATA](https://www.samsung.com/uk/support/model/MZ-7TD250BW/) (2016)
*  [Samsung 850 EVO Basic 1000GB SATA](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-850-evo-2-5-sata-iii-1tb-mz-75e1t0b-am/) (2017)
*  [Samsung 860 EVO Basic 2000GB SATA](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-860-evo-2-5--sata-iii-2tb-mz-76e2t0b-am/) (2019)
*  [Samsung SSD 860 EVO M.2 1TB SATA](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-860-evo-m-2-sata-1tb-mz-n6e1t0bw/) (2019)
*  [Samsung 870 EVO 4000GB SATA](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/870-evo-sata-2-5-ssd-4tb-mz-77e4t0b-am/) (2020)
*  [Samsung 860 EVO 500GB m.2 SATA](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-860-evo-m-2-sata-500gb-mz-n6e500bw/) (2021)
*  [Samsung 970 EVO Plus 1000 GB M.2 NVMe PCIe 3.0](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-970-evo-plus-nvme-m-2-1-tb-mz-v7s1t0b-am/) (2021)
*  **3** [Samsung 970 EVO Plus 2000 GB M.2 NVMe PCIe 3.0](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-970-evo-plus-nvme-m-2-2-tb-mz-v7s2t0b-am/) (2022)

Of all these, the only one ever to fail was the
**Samsung 860 EVO Basic 2000GB** from 2019, failing in 2021.
When it failed, it started with the `WRITE FPDMA QUEUED` errors
that appeared this time around too. Other than that, Samsung SSD
disks have proven reliable and trouble-free for the last 12 years
which is more than can be said for nearly any other product.

## What Happened

Since the previous files (my Audible library and select Podcasts)
take up nearly 500 MB, the natural move would be to copy the
files directly.

First, formatted the (entire) drive as BTRFS and mounted it at
`/home/ssd` (there is only room for one SSD).

Then the problem: when trying to do this, possibly because
the files were originally in the NVMe SSD that is much faster,
the Crucial SSD failed after copying less than 8 GB:

``` dmesg
[  614.810936] BTRFS info (device sda): flagging fs with big metadata feature
[  614.810947] BTRFS info (device sda): using free space tree
[  614.810951] BTRFS info (device sda): has skinny extents
[  614.814588] BTRFS info (device sda): enabling ssd optimizations
[  614.814892] BTRFS info (device sda): checking UUID tree
[  750.641105] ata1.00: exception Emask 0x0 SAct 0x80e00003 SErr 0x0 action 0x6 frozen
[  750.641124] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641128] ata1.00: cmd 61/00:00:d0:3f:6c/0a:00:00:00:00/40 tag 0 ncq dma 1310720 ou
                        res 40/00:00:00:00:00/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641144] ata1.00: status: { DRDY }
[  750.641148] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641152] ata1.00: cmd 61/00:08:d0:49:6c/0a:00:00:00:00/40 tag 1 ncq dma 1310720 ou
                        res 40/00:00:00:00:00/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641164] ata1.00: status: { DRDY }
[  750.641168] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641171] ata1.00: cmd 61/70:a8:60:21:6c/00:00:00:00:00/40 tag 21 ncq dma 57344 out
                        res 40/00:00:00:4f:c2/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641182] ata1.00: status: { DRDY }
[  750.641186] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641189] ata1.00: cmd 61/00:b0:d0:21:6c/0a:00:00:00:00/40 tag 22 ncq dma 1310720 ou
                        res 40/00:ff:00:00:00/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641202] ata1.00: status: { DRDY }
[  750.641206] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641209] ata1.00: cmd 61/00:b8:d0:2b:6c/0a:00:00:00:00/40 tag 23 ncq dma 1310720 ou
                        res 40/00:01:00:00:00/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641220] ata1.00: status: { DRDY }
[  750.641224] ata1.00: failed command: WRITE FPDMA QUEUED
[  750.641227] ata1.00: cmd 61/00:f8:d0:35:6c/0a:00:00:00:00/40 tag 31 ncq dma 1310720 ou
                        res 40/00:00:00:00:00/00:00:00:00:00/00 Emask 0x4 (timeout)
[  750.641238] ata1.00: status: { DRDY }
[  750.641244] ata1: hard resetting link
[  756.009469] ata1: link is slow to respond, please be patient (ready=0)
[  760.677688] ata1: COMRESET failed (errno=-16)
[  760.677734] ata1: hard resetting link
[  766.017924] ata1: link is slow to respond, please be patient (ready=0)
[  770.690140] ata1: COMRESET failed (errno=-16)
[  770.690185] ata1: hard resetting link
[  776.018382] ata1: link is slow to respond, please be patient (ready=0)
[  805.739547] ata1: COMRESET failed (errno=-16)
[  805.739590] ata1: limiting SATA link speed to 3.0 Gbps
[  805.739594] ata1: hard resetting link
[  810.743856] ata1: COMRESET failed (errno=-16)
[  810.743904] ata1: reset failed, giving up
[  810.743910] ata1.00: disabled
[  810.745057] ata1: EH complete
[  810.745101] sd 0:0:0:0: [sda] tag#2 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.745109] sd 0:0:0:0: [sda] tag#2 CDB: Write(16) 8a 00 00 00 00 00 00 6c 35 d0 00 00 0a 00 00 00
[  810.745111] blk_update_request: I/O error, dev sda, sector 7091664 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.745135] sd 0:0:0:0: [sda] tag#3 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.745138] sd 0:0:0:0: [sda] tag#3 CDB: Write(16) 8a 00 00 00 00 00 00 6c 2b d0 00 00 0a 00 00 00
[  810.745140] blk_update_request: I/O error, dev sda, sector 7089104 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.745151] sd 0:0:0:0: [sda] tag#4 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.745154] sd 0:0:0:0: [sda] tag#4 CDB: Write(16) 8a 00 00 00 00 00 00 6c 21 d0 00 00 0a 00 00 00
[  810.745156] blk_update_request: I/O error, dev sda, sector 7086544 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.745165] sd 0:0:0:0: [sda] tag#5 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.745168] sd 0:0:0:0: [sda] tag#5 CDB: Write(16) 8a 00 00 00 00 00 00 6c 21 60 00 00 00 70 00 00
[  810.745170] blk_update_request: I/O error, dev sda, sector 7086432 op 0x1:(WRITE) flags 0x100000 phys_seg 1 prio class 0
[  810.745180] BTRFS error (device sda): bdev /dev/sda errs: wr 1, rd 0, flush 0, corrupt 0, gen 0
[  810.747865] sd 0:0:0:0: [sda] tag#6 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.747872] sd 0:0:0:0: [sda] tag#6 CDB: Write(16) 8a 00 00 00 00 00 00 6c 49 d0 00 00 0a 00 00 00
[  810.747875] blk_update_request: I/O error, dev sda, sector 7096784 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.747894] sd 0:0:0:0: [sda] tag#7 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=91s
[  810.747899] sd 0:0:0:0: [sda] tag#7 CDB: Write(16) 8a 00 00 00 00 00 00 6c 3f d0 00 00 0a 00 00 00
[  810.747902] blk_update_request: I/O error, dev sda, sector 7094224 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.748109] sd 0:0:0:0: [sda] tag#2 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=0s
[  810.748120] sd 0:0:0:0: [sda] tag#2 CDB: Write(16) 8a 00 00 00 00 00 00 6c 53 d0 00 00 0a 00 00 00
[  810.748125] blk_update_request: I/O error, dev sda, sector 7099344 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.748156] sd 0:0:0:0: [sda] tag#3 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=0s
[  810.748161] sd 0:0:0:0: [sda] tag#3 CDB: Write(16) 8a 00 00 00 00 00 00 6c 5d d0 00 00 0a 00 00 00
[  810.748164] blk_update_request: I/O error, dev sda, sector 7101904 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.748181] sd 0:0:0:0: [sda] tag#4 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=0s
[  810.748186] sd 0:0:0:0: [sda] tag#4 CDB: Write(16) 8a 00 00 00 00 00 00 6c 67 d0 00 00 0a 00 00 00
[  810.748189] blk_update_request: I/O error, dev sda, sector 7104464 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.748205] sd 0:0:0:0: [sda] tag#5 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=0s
[  810.748210] sd 0:0:0:0: [sda] tag#5 CDB: Write(16) 8a 00 00 00 00 00 00 6c 71 d0 00 00 0a 00 00 00
[  810.748213] blk_update_request: I/O error, dev sda, sector 7107024 op 0x1:(WRITE) flags 0x104000 phys_seg 20 prio class 0
[  810.748327] BTRFS error (device sda): bdev /dev/sda errs: wr 2, rd 0, flush 0, corrupt 0, gen 0
[  810.748350] BTRFS error (device sda): bdev /dev/sda errs: wr 3, rd 0, flush 0, corrupt 0, gen 0
[  810.750196] BTRFS error (device sda): bdev /dev/sda errs: wr 4, rd 0, flush 0, corrupt 0, gen 0
[  810.750211] BTRFS error (device sda): bdev /dev/sda errs: wr 5, rd 0, flush 0, corrupt 0, gen 0
[  810.750218] BTRFS error (device sda): bdev /dev/sda errs: wr 6, rd 0, flush 0, corrupt 0, gen 0
[  810.750224] BTRFS error (device sda): bdev /dev/sda errs: wr 7, rd 0, flush 0, corrupt 0, gen 0
[  810.750231] BTRFS error (device sda): bdev /dev/sda errs: wr 8, rd 0, flush 0, corrupt 0, gen 0
[  810.750237] BTRFS error (device sda): bdev /dev/sda errs: wr 9, rd 0, flush 0, corrupt 0, gen 0
[  810.750244] BTRFS error (device sda): bdev /dev/sda errs: wr 10, rd 0, flush 0, corrupt 0, gen 0
[  810.752702] BTRFS: error (device sda) in btrfs_commit_transaction:2438: errno=-5 IO failure (Error while writing out transaction)
[  810.752722] BTRFS info (device sda): forced readonly
[  810.752728] BTRFS warning (device sda): Skipping commit of aborted transaction.
[  810.752732] BTRFS: error (device sda) in cleanup_transaction:2011: errno=-5 IO failure
[ 1229.016176] scsi_io_completion_action: 5045 callbacks suppressed
[ 1229.016188] sd 0:0:0:0: [sda] tag#23 FAILED Result: hostbyte=DID_BAD_TARGET driverbyte=DRIVER_OK cmd_age=0s
[ 1229.016197] sd 0:0:0:0: [sda] tag#23 CDB: ATA command pass through(16) 85 06 20 00 00 00 00 00 00 00 00 00 00 00 e5 00
```

The disk did not *actually* write any data; although `df`
reported 1.4G used at this point, it reported only a few MB
after reboot:

``` console
# /home/depot # df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda        3.7T  1.4G  3.7T   1% /home/ssd
```

After this failure, the disk was not *just* read only,
it was entirely disabled and unavailable:

``` console
# du -sh /home/ssd/*
du: cannot access '/home/ssd/*': Input/output error
```

After reboot:

``` console
# /home/depot # df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda        3.7T  3.6M  3.7T   1% /home/ssd
```

## Get S.M.A.R.T.

Based on the experience with the recently failed
**Samsung 860 EVO Basic 2000GB**, this looked like it could be
a defective disk that would need an RMA. In search of more
definitive signs of hardware failure, I run S.M.A.R.T. long test:
`smartctl -t long /dev/sda`

After some time the results were as follows:

``` console
# smartctl -a /dev/sda
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.15.0-48-generic] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     CT4000MX500SSD1
Serial Number:    2221E632FD50
LU WWN Device Id: 5 00a075 1e632fd50
Firmware Version: M3CR044
User Capacity:    4,000,787,030,016 bytes [4.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    Solid State Device
Form Factor:      2.5 inches
TRIM Command:     Available
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-3 T13/2161-D revision 5
SATA Version is:  SATA 3.3, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Wed Oct 12 21:31:31 2022 CEST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x03) Offline data collection activity
                                        is in progress.
                                        Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0) The previous self-test routine completed
                                        without error or no self-test has ever 
                                        been run.
Total time to complete Offline 
data collection:                ( 1402) seconds.
Offline data collection
capabilities:                    (0x7b) SMART execute Offline immediate.
                                        Auto Offline data collection on/off support.
                                        Suspend Offline collection upon new
                                        command.
                                        Offline surface scan supported.
                                        Self-test supported.
                                        Conveyance Self-test supported.
                                        Selective Self-test supported.
SMART capabilities:            (0x0003) Saves SMART data before entering
                                        power-saving mode.
                                        Supports SMART auto save timer.
Error logging capability:        (0x01) Error logging supported.
                                        General Purpose Logging supported.
Short self-test routine 
recommended polling time:        (   2) minutes.
Extended self-test routine
recommended polling time:        (  30) minutes.
Conveyance self-test routine
recommended polling time:        (   2) minutes.
SCT capabilities:              (0x0031) SCT Status supported.
                                        SCT Feature Control supported.
                                        SCT Data Table supported.

SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x002f   100   100   000    Pre-fail  Always       -       0
  5 Reallocated_Sector_Ct   0x0032   100   100   010    Old_age   Always       -       0
  9 Power_On_Hours          0x0032   100   100   000    Old_age   Always       -       0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       6
171 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
172 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
173 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
174 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       1
180 Unused_Rsvd_Blk_Cnt_Tot 0x0033   000   000   000    Pre-fail  Always       -       217
183 Runtime_Bad_Block       0x0032   100   100   000    Old_age   Always       -       0
184 End-to-End_Error        0x0032   100   100   000    Old_age   Always       -       0
187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always       -       0
194 Temperature_Celsius     0x0022   070   068   000    Old_age   Always       -       30 (Min/Max 25/32)
196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       0
197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x0032   100   100   000    Old_age   Always       -       0
202 Unknown_SSD_Attribute   0x0030   100   100   001    Old_age   Offline      -       0
206 Unknown_SSD_Attribute   0x000e   100   100   000    Old_age   Always       -       0
210 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
246 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       2843016
247 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       22592
248 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       11555

SMART Error Log Version: 1
Warning: ATA error count 0 inconsistent with error log pointer 1

ATA Error Count: 0
        CR = Command Register [HEX]
        FR = Features Register [HEX]
        SC = Sector Count Register [HEX]
        SN = Sector Number Register [HEX]
        CL = Cylinder Low Register [HEX]
        CH = Cylinder High Register [HEX]
        DH = Device/Head Register [HEX]
        DC = Device Command Register [HEX]
        ER = Error register [HEX]
        ST = Status register [HEX]
Powered_Up_Time is measured from power on, and printed as
DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
SS=sec, and sss=millisec. It "wraps" after 49.710 days.

Error 0 occurred at disk power-on lifetime: 0 hours (0 days + 0 hours)
  When the command that caused the error occurred, the device was in an unknown state.

  After command completion occurred, registers were:
  ER ST SC SN CL CH DH
  -- -- -- -- -- -- --
  00 ec 00 00 00 00 00

  Commands leading to the command that caused the error were:
  CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
  -- -- -- -- -- -- -- --  ----------------  --------------------
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  c8 00 00 00 00 00 00 00      00:00:00.000  READ DMA

SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed without error       00%         0         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x10):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```

The long test *completed without error*, but there are other
signs that this disk may be problematic.

## What's Wrong #1

First, the disk seems to think it can go as fast as 6.0 TB/s which is *definitely* not true; that would be 12 times fater
than the 560/530 (read/write) MB/s it is rated as, and confirmed
by `hdparm`:

``` console
# hdparm -t -T /dev/sda

/dev/sda:
 Timing cached reads:   34646 MB in  2.00 seconds = 17350.03 MB/sec
 Timing buffered disk reads: 1596 MB in  3.00 seconds = 531.82 MB/sec
```

Note this line from `smartctl`, in the `INFORMATION SECTION`
at the top:

``` dmesg
SATA Version is:  SATA 3.3, 6.0 Gb/s (current: 6.0 Gb/s)
```

It seems the kernel is expecting *up to* **6.0 Gbps**

``` console
[    1.625648] ahci 0000:00:17.0: AHCI 0001.0301 32 slots 2 ports 6 Gbps 0x3 impl SATA mode
[    1.646123] ata1: SATA max UDMA/133 abar m2048@0x6a602000 port 0x6a602100 irq 131
[    1.646129] ata2: SATA max UDMA/133 abar m2048@0x6a602000 port 0x6a602180 irq 131
[    1.963187] ata2: SATA link down (SStatus 4 SControl 300)
[    1.963249] ata1: SATA link up 6.0 Gbps (SStatus 133 SControl 300)
```

A transfer rate of 6.0 Gbps would be 750 MB/s, which is about
50% more than this disk is rated for, but that *shouldn't* be a
problem. It *certainly* is not a problem with the
**Samsung 870 EVO 4000GB SATA** disk, which does show the same
*up to 6.0 Gbps* line in `dmesg`:

``` console
[    1.178462] ata6: SATA max UDMA/133 abar m2048@0xfc600000 port 0xfc600180 irq 66
[    1.651823] ata6: SATA link up 6.0 Gbps (SStatus 133 SControl 300)
[    1.654633] ata6.00: supports DRM functions and may not be fully accessible
[    1.655166] ata6.00: ATA-11: Samsung SSD 870 EVO 4TB, SVT01B6Q, max UDMA/133
```

Somehow, the Crucial SSD is unable to *self-regulate* when
given data to write at any rate higher than what it's rated for,
while (all) Samsung disks have no such problem.

## What's Wrong #2

Second, back to `smartctl` output, this entire block is new
and unique to the Crucial SSD:

```
Warning: ATA error count 0 inconsistent with error log pointer 1

ATA Error Count: 0
        CR = Command Register [HEX]
        FR = Features Register [HEX]
        SC = Sector Count Register [HEX]
        SN = Sector Number Register [HEX]
        CL = Cylinder Low Register [HEX]
        CH = Cylinder High Register [HEX]
        DH = Device/Head Register [HEX]
        DC = Device Command Register [HEX]
        ER = Error register [HEX]
        ST = Status register [HEX]
Powered_Up_Time is measured from power on, and printed as
DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
SS=sec, and sss=millisec. It "wraps" after 49.710 days.

Error 0 occurred at disk power-on lifetime: 0 hours (0 days + 0 hours)
  When the command that caused the error occurred, the device was in an unknown state.

  After command completion occurred, registers were:
  ER ST SC SN CL CH DH
  -- -- -- -- -- -- --
  00 ec 00 00 00 00 00

  Commands leading to the command that caused the error were:
  CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
  -- -- -- -- -- -- -- --  ----------------  --------------------
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  ec 00 00 00 00 00 00 00      00:00:00.000  IDENTIFY DEVICE
  c8 00 00 00 00 00 00 00      00:00:00.000  READ DMA
```

This a long way of saying, the READ DMA command lead to an error
state, reflected as value `0xEC` in the `Status register`.

## Workaround

Manually limiting the transfer rate when writing files seems to
be the only way to get large write operations to succeed.
Luckly, this is rather easy when using `rsync`:

``` console hl_lines="2"
$ rsync -turva \
  --bwlimit=500000 \
  /home/depot/audio/Audiobooks \
  /home/ssd/audio/
```

Attempting the same with a higher `--bwlimit` value (700 MB/s)
reproduces the error almost immediately:

``` dmesg
[10745.786389] BTRFS error (device sda): bdev /dev/sda errs: wr 4, rd 0, flush 0, corrupt 0, gen 0
[10745.786405] BTRFS error (device sda): bdev /dev/sda errs: wr 5, rd 0, flush 0, corrupt 0, gen 0
[10745.786418] BTRFS error (device sda): bdev /dev/sda errs: wr 6, rd 0, flush 0, corrupt 0, gen 0
[10745.786429] BTRFS error (device sda): bdev /dev/sda errs: wr 7, rd 0, flush 0, corrupt 0, gen 0
[10745.786532] BTRFS error (device sda): bdev /dev/sda errs: wr 8, rd 0, flush 0, corrupt 0, gen 0
[10745.788424] BTRFS error (device sda): bdev /dev/sda errs: wr 9, rd 0, flush 0, corrupt 0, gen 0
[10745.788687] BTRFS error (device sda): bdev /dev/sda errs: wr 10, rd 0, flush 0, corrupt 0, gen 0
[10745.811622] BTRFS: error (device sda) in btrfs_commit_transaction:2438: errno=-5 IO failure (Error while writing out transaction)
[10745.811633] BTRFS info (device sda): forced readonly
[10745.811635] BTRFS warning (device sda): Skipping commit of aborted transaction.
[10745.811637] BTRFS: error (device sda) in cleanup_transaction:2011: errno=-5 IO failure
```

## Root Cause Confirmed

Shortly after dealing with this problem in
[Lexicon](2022-07-03-low-effort-homelab-server-with-ubuntu-server-on-intel-nuc.md),
the very same problem reproduced with *another* Crucial MX500 SSD
on a different system (**Rapture**) with a
[ASUS PRIME X370-PRO](https://www.linuxlookup.com/review/asus_prime_x370_pro_motherboard_review) motherboard, which is
the one where the **Samsung 870 EVO 4000GB SATA** disk
exhibits no such problem.

**Crucial MX500 SSD are no longer welcome or recommended**,
cheap as they may be.
