---
title:  "Undead Yes ─ UnRAID No"
date:   2022-09-27 22:09:27 +0200
categories: linux hardware failure raid btrfs
---

*My only NAS is my PC*. At least, what people would usually do with,
or build a NAS for, I just do it with my PC.

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

Most of my disk storage space is a
[BTRFS RAID 1](https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.html#mkfs-section-profiles)
using two
[6TB WD BLACK 3.5″ HDD](https://www.westerndigital.com/products/internal-drives/wd-black-desktop-sata-hdd#WD6003FZBX).
This setup offers
[block-level redundancy which is better](https://www.reddit.com/r/btrfs/comments/xosn2p/ask_better_do_you_really_need_raid1/)
than the classic device-level redundancy offered by
[Linux Software RAID](https://raid.wiki.kernel.org/index.php/Linux_Raid)
or hardware RAID.
To keep BTRFS file systems healthy, it is strongly recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, I run the script from
crontab every Saturday night (it usually ends around noon the next day).

One Sunday morning, after many successful scrubs, I woke up to both
disks failing, each in a different way. But this was not the end of
it. And the end of this adventure, disks emerged victorious.

Keeping reading to find out how the disks came back from the dead.

![Illustration by Paul Kidby: Zombie leads a small parade of undead citizens with a wooden sign that reads UNDEAD YES - UNPERSON NO]({{ media }}/fresh_start_club.jpg)

## Meet My Disks

The disks configured in this RAID1 array are:
`/dev/sdb` and `/dev/sdc`

*  `/dev/sdb` is a
   [WD6002FZWX](https://documents.westerndigital.com/content/dam/doc-library/es_mx/assets/public/western-digital/product/internal-drives/wd-black-hdd/data-sheet-wd-black-pc-hard-drives-2879-771434.pdf)
   6TB (128 MB cache) purchased in 2017.
*  `/dev/sdc` is a
   [WD6003FZBX](https://www.westerndigital.com/products/internal-drives/wd-black-desktop-sata-hdd?sku=WD6003FZBX#WD6003FZBX)
   6TB (256 MB cache) purchased in 2020.

They are both *rated* as 227 MB/s but have significantly different performance:

```
# btrfs filesystem show
Label: 'HomeDepot6TB'  uuid: a4ee872d-b985-445f-94a2-15232e93dcd5
        Total devices 2 FS bytes used 4.59TiB
        devid    1 size 5.46TiB used 4.59TiB path /dev/sdb
        devid    2 size 5.46TiB used 4.59TiB path /dev/sdc

# hdparm -tT /dev/sd[bc]

/dev/sdb:
 Timing cached reads:   58408 MB in  2.00 seconds = 29275.83 MB/sec
 Timing buffered disk reads: 612 MB in  3.00 seconds = 203.77 MB/sec

/dev/sdc:
 Timing cached reads:   55672 MB in  2.00 seconds = 27901.25 MB/sec
 Timing buffered disk reads: 754 MB in  3.01 seconds = 250.89 MB/sec
```

This will be noticeable in the I/O charts shown below.

## What Happened

When the scrub starts, both disks are start reading at about 300 MB/s
which is the normal for these disks. The problem starts when, after
about one hour, `sdb` becomes inactive:

![Disk I/O chart shows disk sdb becomes inactive around 2am]({{ media }}/1.png)

Before dropping down to zero, transfer (read) rate on `sdb` drops
sharply on both disks, and that that time RAM usage drops sharply
too, down to about half of its previous value:

![RAM usage chart shows used RAM drops by about 50%]({{ media }}/2.png)

At that time, which is about 3900 seconds after scrub starts on that
disk, btrfs errors start showing up in dmesg warning of tasks being
blocked waiting for I/O:

```
[21158.812097] BTRFS info (device sdb): balance: start -musage=0 -susage=0
[21158.819155] BTRFS info (device sdb): balance: ended with status: 0
[21159.105609] BTRFS info (device sdb): balance: start -musage=20 -susage=20
[21159.107671] BTRFS info (device sdb): relocating block group 16994275426304 flags system|raid1
[21159.661753] BTRFS info (device sdb): found 53 extents, stage: move data extents
[21160.134413] BTRFS info (device sdb): balance: ended with status: 0
[21160.357813] BTRFS info (device sdb): balance: start -dusage=0
[21160.363849] BTRFS info (device sdb): balance: ended with status: 0
[21160.608367] BTRFS info (device sdb): balance: start -dusage=20
[21160.616713] BTRFS info (device sdb): balance: ended with status: 0
[21160.730133] BTRFS info (device sdb): scrub: started on devid 1
[21160.732650] BTRFS info (device sdb): scrub: started on devid 2

… 3,910 seconds later …

[25068.665995] INFO: task btrfs-transacti:1291 blocked for more than 122 seconds.
[25068.666007]       Not tainted 5.15.0-47-lowlatency #53-Ubuntu
[25068.666009] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[25068.666011] task:btrfs-transacti state:D stack:    0 pid: 1291 ppid:     2 flags:0x00004000
[25068.666015] Call Trace:
[25068.666018]  <TASK>
[25068.666023]  __schedule+0x240/0x5c0
[25068.666032]  schedule+0x67/0xe0
[25068.666035]  btrfs_scrub_pause+0x9a/0x100 [btrfs]
[25068.666095]  ? wait_woken+0x70/0x70
[25068.666101]  btrfs_commit_transaction+0x277/0xb50 [btrfs]
[25068.666135]  ? start_transaction+0xd1/0x5f0 [btrfs]
[25068.666169]  ? __bpf_trace_timer_class+0x10/0x10
[25068.666173]  transaction_kthread+0x137/0x1b0 [btrfs]
[25068.666209]  ? btrfs_cleanup_transaction.isra.0+0x550/0x550 [btrfs]
[25068.666243]  kthread+0x13b/0x160
[25068.666246]  ? set_kthread_struct+0x50/0x50
[25068.666249]  ret_from_fork+0x22/0x30
[25068.666254]  </TASK>
[25068.666450] INFO: task btrfs:3834821 blocked for more than 122 seconds.
[25068.666453]       Not tainted 5.15.0-47-lowlatency #53-Ubuntu
[25068.666455] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[25068.666456] task:btrfs           state:D stack:    0 pid:3834821 ppid:970896 flags:0x00004002
[25068.666517] Call Trace:
[25068.666518]  <TASK>
[25068.666519]  __schedule+0x240/0x5c0
[25068.666522]  ? autoremove_wake_function+0x12/0x40
[25068.666525]  schedule+0x67/0xe0
[25068.666526]  __scrub_blocked_if_needed+0x77/0xd0 [btrfs]
[25068.666570]  ? wait_woken+0x70/0x70
[25068.666573]  scrub_pause_off+0x26/0x60 [btrfs]
[25068.666614]  scrub_enumerate_chunks+0x376/0x7a0 [btrfs]
[25068.666658]  ? wait_woken+0x70/0x70
[25068.666660]  btrfs_scrub_dev+0x1d1/0x510 [btrfs]
[25068.666714]  ? __check_object_size.part.0+0x134/0x150
[25068.666719]  ? _copy_from_user+0x2e/0x70
[25068.666724]  btrfs_ioctl+0x651/0x1250 [btrfs]
[25068.666764]  ? get_task_io_context+0x54/0x90
[25068.666768]  ? set_task_ioprio+0xa1/0xb0
[25068.666771]  ? __fget_light+0xa7/0x130
[25068.666775]  __x64_sys_ioctl+0x95/0xd0
[25068.666778]  do_syscall_64+0x5c/0xc0
[25068.666783]  ? exit_to_user_mode_prepare+0x37/0xb0
[25068.666787]  ? syscall_exit_to_user_mode+0x27/0x50
[25068.666789]  ? do_syscall_64+0x69/0xc0
[25068.666792]  ? irqentry_exit_to_user_mode+0x9/0x20
[25068.666794]  ? irqentry_exit+0x3b/0x50
[25068.666795]  ? exc_page_fault+0x89/0x190
[25068.666797]  entry_SYSCALL_64_after_hwframe+0x61/0xcb
[25068.666802] RIP: 0033:0x7f57d6944aff
[25068.666804] RSP: 002b:00007f57d6825c40 EFLAGS: 00000246 ORIG_RAX: 0000000000000010
[25068.666807] RAX: ffffffffffffffda RBX: 000055b4db0cb4f0 RCX: 00007f57d6944aff
[25068.666809] RDX: 000055b4db0cb4f0 RSI: 00000000c400941b RDI: 0000000000000003
[25068.666810] RBP: 0000000000000000 R08: 00007fff102d3e9f R09: 0000000000000000
[25068.666811] R10: 0000000000000000 R11: 0000000000000246 R12: 00007f57d6826640
[25068.666812] R13: 000000000000006f R14: 00007f57d68be850 R15: 00007fff102d3ee0
[25068.666815]  </TASK>

… 4,032 seconds later …

[25191.545405] INFO: task btrfs-transacti:1291 blocked for more than 245 seconds.

… 4,155 seconds later …

[25314.425912] INFO: task btrfs-transacti:1291 blocked for more than 368 seconds.

… 4,278 seconds later …

[25437.306444] INFO: task btrfs-transacti:1291 blocked for more than 491 seconds.

… 4,401 seconds later …

[25560.188030] INFO: task btrfs-transacti:1291 blocked for more than 614 seconds.
```

These timings indicate that, from the time scrub starts:

*  3,788 s. (01:03:08 after 01:20:50 → **02:23:58**) disk gets stuck
*  4,279 s. (01:11:19 after 01:20:50 → **02:32:09**) disk was still stuck

### What happened to `sdb`

Comparing these timings to read transfer rate on sdb:

*  Scrub starts at 01:20:50 (320-350 MB/s)
*  Slows down around 01:54:40 to 270-300 MB/s
*  Drops down to nearly zero at 02:04:00
*  Resumes reading at only 25 MB/s at 02:05:45
*  Stops reading at **02:21:30**
*  There is a small burst of read activity at 03:20:00 peaking at 25 MB/s
*  Then another burst of activity from 04:30 to 04:45 peaking at 62 MB/s
*  Last, attempts of read activity every 10 minutes, tiny spikes of 500 **B/s**

![Disk I/O chart shows disk sdb becomes slower in steps (1 of 3)]({{ media }}/3.png)
![Disk I/O chart shows disk sdb becomes slower in steps (2 of 3)]({{ media }}/4.png)
![Disk I/O chart shows disk sdb becomes slower in steps (3 of 3)]({{ media }}/5.png)

At this point attempting to read `sdb` just doesn’t work at all;
it times out without even a message on `dmesg` (it was from `btrfs`),
and can’t be interrupted:

```
root@rapture:~# hdparm -tT /dev/sdb

/dev/sdb:
^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C^C
```

This attempt to read sdb produced just one spike shy of 10 kB/s,
and then nothing more than the spikes of 512 B/s every 10 minutes,
as above.

### What happened to sdc

Meanwhile, read activity on sdc started off pretty good at
350-400 MB/s but then, after about an hour, it gradually slowed
down from 380 MB/s to 250 MB/s over the next 5.5 hours, and then
is started to intermittently drop abruptly to only 75 MB/s:

![Disk I/O chart shows disk sdc becomes gradually slower over a period of 6.5 hours]({{ media }}/6.png)

The rate of read from btrfs itself is significantly lower:

![Process I/O chart shows btrfs read rate also becoming gradually slower over a period of 6.5 hours]({{ media }}/7.png)

After about 12 hours, the read rate on sdc drops even lower, to 45 MB/s:

![Disk I/O chart shows disk sdc becomes even slower]({{ media }}/8.png)

Thousands of operations are flagged by `handle_bad_sector` as
*attempting to read beyond the end of the device*:

```
[63670.333955] attempt to access beyond end of device
               sdc: rw=0, want=11721045248, limit=11721045168
[63670.334052] attempt to access beyond end of device
               sdc: rw=0, want=11721045504, limit=11721045168

[63670.334465] attempt to access beyond end of device
               sdc: rw=0, want=11721047552, limit=11721045168
[63675.334648] handle_bad_sector: 154343 callbacks suppressed
[63675.334652] attempt to access beyond end of device
               sdc: rw=0, want=11760559616, limit=11721045168

[64903.722343] attempt to access beyond end of device
               sdc: rw=0, want=11957755648, limit=11721045168
```

This starts to happen **nearly 12 hours** (11:48:31) after the
start of the scrub and then keeps going with a few more thousand
entries every couple of seconds.

At this point there seems to be no hope left for this process to
finish. If there is a chance that the beyond end of device errors on
`sdc` are being caused by the timeouts on `sdb`, maybe removing `sdb`
from the RAID 1 configuration could make `sdc` usable,
at least for a while.

First, shut down and boot back into the system, so that it is no longer stuck trying to read a bad disk.

Then modified the `btrfs-scrub` script to skip `sdb` and run it on
the SSD and NVME drives. *That was fast* 😁

Turns out, after rebooting the RAID 1 seems to work fine.
Both `sdb` and `sdc` can be read. write operations seem to be
happening successfully on both disks as the mirrors they are.
It appears the problem in `sdb` only triggers after a certain amount
of reading, so the idea now is to read everything from `sdc` while
(if) it can still be read.

There is a method to
[convert the RAID 1 back to a single drive](https://www.kubuntuforums.net/forum/general/miscellaneous/btrfs/67709-btrfs-converting-raid1-to-a-single-drive),
and even though `sdb` times out during the `balance` operation,
[there is a way to avoid reading from that disk](https://wiki.tnonline.net/w/Btrfs/Replacing_a_disk#Disk_is_online_but_is_having_errors).
From here, I was thinking I could just use `-r 2` to read only from
`sdc` by converting it to a single drive, but someone (and I’m sorry
to say, I lost track of the source) figured out it takes more than that:

First, disable automounting the partition in `/etc/fstab` and **reboot**.

Then, spin the drive down with and check with `dmesg` that it does stop:

```
# echo 1 | sudo tee /sys/block/sdb/device/delete

# dmesg | egrep 'sdb|ata'
[  190.547826] sd 1:0:0:0: [sdb] Synchronizing SCSI cache
[  190.548096] sd 1:0:0:0: [sdb] Stopping disk
[  191.560912] ata2.00: disabled
```

Now, with the bad disk truly out of play, mount the RAID with in degraded state:

```
# mount /home/raid -o degraded
# df -h | head -1; df -h | grep home
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p5  1.7T  879G  834G  52% /home
/dev/sde        3.7T  2.7T  988G  74% /home/ssd
/dev/sdc        5.5T  4.6T  511G  91% /home/raid
```

Note the messages in `dmesg` when doing this:

```
[   70.409785] BTRFS info (device sdc): flagging fs with big metadata feature
[   70.409792] BTRFS info (device sdc): allowing degraded mounts
[   70.409795] BTRFS info (device sdc): disk space caching is enabled
[   70.409797] BTRFS info (device sdc): has skinny extents
[   70.422129] BTRFS warning (device sdc): devid 1 uuid 3941c5c1-08b5-4cb8-98bd-455ffa38ad2b is missing
[   70.668878] BTRFS info (device sdc): bdev /dev/sdb errs: wr 0, rd 0, flush 0, corrupt 4, gen 0
[   70.668884] BTRFS info (device sdc): bdev /dev/sdc errs: wr 0, rd 0, flush 0, corrupt 2, gen 0
```

Next, begin the rebalancing operation to convert `/home/raid`
from RAID 1 to single disk. This would take about **14 hours**:

```
# btrfs balance start -f -mconvert=single -dconvert=single /home/raid
```

Note the messages in `dmesg` when doing this:

```
[  206.773574] BTRFS info (device sdc): balance: force reducing metadata redundancy
[  207.233822] BTRFS info (device sdc): balance: start -f -dconvert=single -mconvert=single -sconvert=single
[  207.237653] BTRFS info (device sdc): relocating block group 17087791628288 flags data
[  207.610162] BTRFS info (device sdc): found 5 extents, stage: move data extents
[  207.707770] BTRFS info (device sdc): found 5 extents, stage: update data pointers
[  207.812576] BTRFS info (device sdc): relocating block group 17087758073856 flags system
[  207.895458] BTRFS info (device sdc): found 3 extents, stage: move data extents
[  207.982061] BTRFS info (device sdc): relocating block group 17086684332032 flags metadata
[  208.075232] BTRFS info (device sdc): relocating block group 17086650777600 flags system|raid1
[  208.171430] BTRFS info (device sdc): found 51 extents, stage: move data extents
[  208.245601] BTRFS info (device sdc): relocating block group 17085577035776 flags data|raid1
```

Progress can be checked with

```
# btrfs balance status -v /home/raid
Balance on '/home/raid' is running
14 out of about 4840 chunks balanced (15 considered), 100% left
Dumping filters: flags 0xf, state 0x1, force is on
  DATA (flags 0x100): converting, target=281474976710656, soft is off
  METADATA (flags 0x100): converting, target=281474976710656, soft is off
  SYSTEM (flags 0x100): converting, target=281474976710656, soft is off
```

Sadly, this **failed with I/O errors** after 9h 20min.:

```
ERROR: error during balancing '/home/raid': Read-only file system
There may be more info in syslog - try dmesg | tail
```

Indeed there was plenty of details in `dmesg`:

```
[34044.600737] BTRFS info (device sdc): found 227 extents, stage: move data extents
[34045.507151] BTRFS info (device sdc): found 227 extents, stage: update data pointers
[34046.546190] BTRFS info (device sdc): relocating block group 10686444863488 flags data|raid1
[34057.848567] BTRFS info (device sdc): found 318 extents, stage: move data extents
[34059.062519] BTRFS info (device sdc): found 318 extents, stage: update data pointers
[34060.419531] BTRFS info (device sdc): relocating block group 10685371121664 flags metadata|raid1
[34067.303863] ------------[ cut here ]------------
[34067.303865] WARNING: CPU: 10 PID: 55370 at fs/btrfs/extent-tree.c:862 lookup_inline_extent_backref+0x638/0x700 [btrfs]
[34067.303898] Modules linked in: nvme_fabrics rfcomm bnep intel_rapl_msr snd_hda_codec_realtek snd_hda_codec_generic snd_hda_codec_hdmi intel_rapl_common ledtrig_audio joydev edac_mce_amd snd_hda_intel snd_intel_dspcfg snd_intel_sdw_acpi snd_hda_codec kvm snd_hda_core snd_hwdep btusb btrtl snd_pcm rapl btbcm input_leds snd_seq_midi snd_seq_midi_event btintel snd_rawmidi ath9k bluetooth ath9k_common eeepc_wmi wmi_bmof ecdh_generic mxm_wmi ecc snd_seq ath9k_hw snd_seq_device ath k10temp snd_timer mac80211 snd ccp soundcore cfg80211 libarc4 mac_hid nvidia_uvm(POE) sch_fq_codel cuse ipmi_devintf ipmi_msghandler msr parport_pc ppdev lp parport ramoops reed_solomon pstore_blk pstore_zone mtd efi_pstore ip_tables x_tables autofs4 btrfs blake2b_generic xor zstd_compress raid6_pq libcrc32c dm_mirror dm_region_hash dm_log nvidia_drm(POE) nvidia_modeset(POE) nvidia(POE) r8153_ecm cdc_ether usbnet drm_kms_helper r8152 mii syscopyarea mfd_aaeon sysfillrect hid_generic sysimgblt asus_wmi fb_sys_fops
[34067.303942]  sparse_keymap cec video usbhid hid crct10dif_pclmul crc32_pclmul ghash_clmulni_intel aesni_intel crypto_simd rc_core platform_profile igb cryptd drm i2c_piix4 nvme ahci dca gpio_amdpt xhci_pci i2c_algo_bit nvme_core libahci xhci_pci_renesas wmi gpio_generic
[34067.303958] CPU: 10 PID: 55370 Comm: btrfs Tainted: P           OE     5.15.0-47-lowlatency #53-Ubuntu
[34067.303960] Hardware name: System manufacturer System Product Name/PRIME X370-PRO, BIOS 5220 09/12/2019
[34067.303961] RIP: 0010:lookup_inline_extent_backref+0x638/0x700 [btrfs]
[34067.303999] Code: e8 5d 6e 03 00 e9 6c fe ff ff 48 83 c3 01 48 83 fb 08 0f 85 ca fd ff ff e9 67 fe ff ff 4d 89 e6 31 db 4d 89 ec e9 15 fc ff ff <0f> 0b b8 fb ff ff ff e9 1c fb ff ff 80 7d c7 bf 0f 87 44 fe ff ff
[34067.304000] RSP: 0018:ffffaaf2d40bf6b8 EFLAGS: 00010202
[34067.304002] RAX: 0000000000000001 RBX: 0000000000000000 RCX: 0002ca395bedce00
[34067.304003] RDX: 0000000000000001 RSI: 0000000000000002 RDI: ffff99f79a123548
[34067.304004] RBP: ffffaaf2d40bf750 R08: 00000000000000b5 R09: ffff99f7ce4bbe00
[34067.304005] R10: 0000000000000001 R11: 0000000000000001 R12: ffff99f7ce4bbe00
[34067.304006] R13: ffff99f7ce4bbe00 R14: ffff99f8c7941a10 R15: ffff99f78ce2c800
[34067.304008] FS:  00007f017cee58c0(0000) GS:ffff99fe7ec80000(0000) knlGS:0000000000000000
[34067.304009] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[34067.304010] CR2: 0000558bc3908340 CR3: 0000000246ad0000 CR4: 00000000003506e0
[34067.304012] Call Trace:
[34067.304013]  <TASK>
[34067.304016]  insert_inline_extent_backref+0x5c/0xf0 [btrfs]
[34067.304041]  ? kmem_cache_alloc+0x1b3/0x330
[34067.304046]  __btrfs_inc_extent_ref.isra.0+0x77/0x240 [btrfs]
[34067.304071]  run_delayed_data_ref+0x15f/0x180 [btrfs]
[34067.304105]  btrfs_run_delayed_refs_for_head+0x185/0x500 [btrfs]
[34067.304139]  __btrfs_run_delayed_refs+0x8c/0x1d0 [btrfs]
[34067.304174]  btrfs_run_delayed_refs+0x73/0x200 [btrfs]
[34067.304205]  ? btrfs_update_root+0x1a2/0x2d0 [btrfs]
[34067.304229]  btrfs_commit_transaction+0x63/0xb50 [btrfs]
[34067.304255]  ? btrfs_update_reloc_root+0x126/0x230 [btrfs]
[34067.304289]  prepare_to_merge+0x29b/0x320 [btrfs]
[34067.304323]  relocate_block_group+0x2c7/0x570 [btrfs]
[34067.304356]  btrfs_relocate_block_group+0x1e1/0x390 [btrfs]
[34067.304389]  btrfs_relocate_chunk+0x2c/0x100 [btrfs]
[34067.304420]  __btrfs_balance+0x2fc/0x4d0 [btrfs]
[34067.304452]  btrfs_balance+0x4cb/0x7d0 [btrfs]
[34067.304482]  ? kmem_cache_alloc_trace+0x1a6/0x320
[34067.304485]  btrfs_ioctl_balance+0x325/0x3e0 [btrfs]
[34067.304516]  btrfs_ioctl+0x340/0x1250 [btrfs]
[34067.304547]  ? rseq_ip_fixup+0x72/0x1a0
[34067.304550]  ? __fput+0x123/0x260
[34067.304553]  __x64_sys_ioctl+0x95/0xd0
[34067.304556]  do_syscall_64+0x5c/0xc0
[34067.304560]  ? __x64_sys_close+0x11/0x50
[34067.304562]  ? do_syscall_64+0x69/0xc0
[34067.304564]  ? __fput+0x123/0x260
[34067.304566]  ? __rseq_handle_notify_resume+0x2d/0xd0
[34067.304568]  ? exit_to_user_mode_loop+0x10d/0x160
[34067.304571]  ? exit_to_user_mode_prepare+0x37/0xb0
[34067.304573]  ? syscall_exit_to_user_mode+0x27/0x50
[34067.304575]  ? __x64_sys_close+0x11/0x50
[34067.304576]  ? do_syscall_64+0x69/0xc0
[34067.304578]  entry_SYSCALL_64_after_hwframe+0x61/0xcb
[34067.304581] RIP: 0033:0x7f017d002aff
[34067.304583] Code: 00 48 89 44 24 18 31 c0 48 8d 44 24 60 c7 04 24 10 00 00 00 48 89 44 24 08 48 8d 44 24 20 48 89 44 24 10 b8 10 00 00 00 0f 05 <41> 89 c0 3d 00 f0 ff ff 77 1f 48 8b 44 24 18 64 48 2b 04 25 28 00
[34067.304584] RSP: 002b:00007ffeb2da5f10 EFLAGS: 00000246 ORIG_RAX: 0000000000000010
[34067.304586] RAX: ffffffffffffffda RBX: 00007ffeb2da6008 RCX: 00007f017d002aff
[34067.304587] RDX: 00007ffeb2da6008 RSI: 00000000c4009420 RDI: 0000000000000003
[34067.304588] RBP: 0000000000000003 R08: 000000000000006e R09: 000000007fffffff
[34067.304590] R10: 0000000000000000 R11: 0000000000000246 R12: 0000000000000000
[34067.304591] R13: 0000000000000000 R14: 00007ffeb2da828b R15: 0000000000000001
[34067.304594]  </TASK>
[34067.304594] ---[ end trace 41f3048f844218d9 ]---
[34067.304597] BTRFS: error (device sdc) in btrfs_run_delayed_refs:2150: errno=-5 IO failure
[34067.304600] BTRFS info (device sdc): forced readonly
[34067.304742] BTRFS info (device sdc): balance: ended with status: -30
```

The file system seems to be at least somewhat readable, for now...

```
# btrfs filesystem show /home/raid
Label: 'HomeDepot6TB'  uuid: 300760fb-f533-4513-9710-50f283f3dbf4
	Total devices 2 FS bytes used 4.59TiB
	devid    2 size 5.46TiB used 4.73TiB path /dev/sdc
	*** Some devices missing

# df -h | head -1; df -h | grep raid
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc         11T  6.6T  893G  89% /home/raid

# time du -sh /home/raid/*
366G	/home/raid/audio
569G	/home/raid/backups
2.1T	/home/raid/depot
148K	/home/raid/dmesg
1.6T	/home/raid/video
```

### What happened to the Data

*Remember, kids: **a RAID is not a backup!***

Depending on the array configuration, a RAID can give you performance
(RAID 0), **uptime** (RAID 1), or a mix of both (RAID 5, 10, etc.).

An array with redundancy (RAID 1, 5, 10, etc.) gives you **uptime**;
a single disk failing doesn’t *immediately* make the entire array
unavailable. As seen above, the file system is still readable despite
taking one drive out. However, it can make data *unrecoverable*, if a
second disk *also* becomes unreadable or contains corrupt data that
can no longer be corrected.

You may find it ironic that my RAID 1 *which is not a backup*
contains a backups folder. You may be amused to hear that this RAID
is the first backup for my most precious files (family photos and
other personal files), as well as the main storage for the heaviest
files (audiobooks, podcasts, home videos and really old backups I
never find time to clean up).

But this is only one piece in a broader
[3-2-1 backup setup](https://www.cisa.gov/sites/default/files/publications/data_backup_options.pdf),
where everything important is stored in at least 3 copies,
on at least 2 types of disks and at least 1 remote location.
My setup is far from perfect but, given the constrains I have to
work with, works well enough and has allowed to me recover from
disk failures multiple times over the last 10 years.

Despite being the biggest file system around the house, everything
that was stored in this RAID was also stored in at least one other
disk, so there was no need to attempt data recovery. Much time had
been sunk already into trying, which was useful to show the disks were dead.

## Regaining Control

The above results showed at least one disk was *probably dead*,
possibly both. However, there was yet more to test in order to
confirm the disks were truly dead and one of those tests would be
required before sending the disks out for RMA:
**wipe out all the data**.

Following Arch Linux
[Securely wipe disk](https://wiki.archlinux.org/title/Securely_wipe_disk),
I settled for a single pass of shred with urandom followed by zeroes:

```
# time shred --verbose --random-source=/dev/urandom -n1 --zero /dev/sdc
…
shred: /dev/sdc: pass 1/2 (random)...32GiB/5.5TiB 0%
…
shred: /dev/sdc: pass 1/2 (random)...4.9TiB/5.5TiB 89%
shred: /dev/sdc: pass 1/2 (random)...5.0TiB/5.5TiB 91%
…
shred: /dev/sdc: pass 1/2 (random)...5.4TiB/5.5TiB 98%
shred: /dev/sdc: pass 1/2 (random)...5.5TiB/5.5TiB 100%
shred: /dev/sdc: pass 2/2 (000000)...
shred: /dev/sdc: pass 2/2 (000000)...1.2GiB/5.5TiB 0%
…
shred: /dev/sdc: pass 2/2 (000000)...5.5TiB/5.5TiB 100%
```

The progress indication would come in handy later. If this operation also failed after 9-12 hours (that’d be between 18:30 and 21:30 on 9/21) it would be a stronger indication that the disk was truly dead. Starting just 6.5 hours later (around 4pm) the chart of Disk I/O started to show (a) general slowdown from sustained 215-220 MB/s to (b) slightly less sustained 195-210 MB/s in about 45 min. If this trend was to continue linearly, in the following 3 hours Disk I/O would slow further down to somewhere between 95 and 170 MB/s.

### *3 Hours Later…*

Pretty much exactly 3 hours later, the writing of random patterns
finished at 140-150 MB/s and, when the writing of zeros started,
transfer rate jumped up to 310-330 MB/s:

![Disk I/O chart shows disk sdc transfer rate jumping up]({{ media }}/9.png)

35 minutes later the downward trend is still there, just displaced by the upward jump. Eventually the operation finished, slowing down to 140-150 MB/s by the end of it.

![Disk I/O chart shows disk sdc becomes even slower again]({{ media }}/10.png)

The next step *would have been* wiping ~1% of the disk in each pass,
starting from the end, in batches of 118394372 bytes:

```
# fdisk -l /dev/sdc
Disk /dev/sdc: 5.46 TiB, 6001175126016 bytes, 11721045168 sectors
Disk model: WDC WD6003FZBX-0
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

```bash
for i in $(seq 99); do
  start=$((11721045168-118394372*i));
  echo "Starting on sector $start..."; 
  time dd if=/dev/urandom of=/dev/sdc bs=512 \
    seek=$start count=118394372 status=progress;
done
```

As it turned out, this was not necessary. Surprised by the success
in the first operation, I decided to do the same on `sdb` first.
It took 20 hours, but it worked!

![Disk I/O chart shows disk sdb with the same pattern as sdc]({{ media }}/11.png)

Both disks slowed down in the same pattern, but never timed out.
This was very surprising given the I/O errors and timeouts they
had caused before.

**Reading** from both disks was also successful,
reading the entire disk in one go:

```
# time dd if=/dev/sdb of=/dev/null bs=512 status=progress
134832546304 bytes (135 GB, 126 GiB) copied, 603 s, 224 MB/s
6001174704640 bytes (6.0 TB, 5.5 TiB) copied, 34417 s, 174 MB/s
11721045168+0 records in
11721045168+0 records out
6001175126016 bytes (6.0 TB, 5.5 TiB) copied, 34423.2 s, 174 MB/s

real    573m43.223s
user    32m34.391s
sys     199m35.531s

# time dd if=/dev/sdc of=/dev/null bs=512 status=progress
134832546304 bytes (135 GB, 126 GiB) copied, 603 s, 224 MB/s
6001103282688 bytes (6.0 TB, 5.5 TiB) copied, 29290 s, 205 MB/s
11721045168+0 records in
11721045168+0 records out
6001175126016 bytes (6.0 TB, 5.5 TiB) copied, 29293.3 s, 205 MB/s

real    488m13.316s
user    28m54.533s
sys     176m39.930s
```

Once again, the data transfer rate slowed down over time at a pretty linear rate:

![Disk I/O chart shows disks sdb and sdc becomes slower over time]({{ media }}/12.png)

As I learned from a colleague, the progressive slow down is normal, caused by
[outer cylinders being “faster”](https://superuser.com/questions/643013/are-partitions-to-the-inner-outer-edge-significantly-faster).

The good news was: after 9 hours for `sdb` and 10.5 hours for `sdc`,
there were no I/O errors or timeouts. Not a single one, after having
written through both entire disks twice, and then read it back.

At this point I started considering, should I just rebuild the original setup and restore the data?

### Get S.M.A.R.T.

To further check the hard drives’ hardware, I decided to run
[S.M.A.R.T.](https://wiki.archlinux.org/title/S.M.A.R.T.)
long test on both drives:

```
# smartctl -t long /dev/sdb
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.15.0-48-lowlatency] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Extended self-test routine immediately in off-line mode".
Drive command "Execute SMART Extended self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 713 minutes for test to complete.
Test will complete after Sat Sep 24 09:13:06 2022 CEST
Use smartctl -X to abort test.

# smartctl -t long /dev/sdc
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.15.0-48-lowlatency] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Extended self-test routine immediately in off-line mode".
Drive command "Execute SMART Extended self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 645 minutes for test to complete.
Test will complete after Sat Sep 24 08:05:10 2022 CEST
Use smartctl -X to abort test.
```

This also took several hours to complete, eventually to show No `Errors Logged` and `Completed without error` on both disks.

```
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.15.0-48-lowlatency] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     WDC WD6002FZWX-00GBGB0
Serial Number:    K1H9M1TD
LU WWN Device Id: 5 000cca 255d27662
Firmware Version: 81.H0A81
User Capacity:    6,001,175,126,016 bytes [6.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-2, ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Sat Sep 24 09:57:52 2022 CEST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x82)	Offline data collection activity
					was completed without error.
					Auto Offline Data Collection: Enabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever 
					been run.
Total time to complete Offline 
data collection: 		(  113) seconds.
Offline data collection
capabilities: 			 (0x5b) SMART execute Offline immediate.
					Auto Offline data collection on/off support.
					Suspend Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					No Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine 
recommended polling time: 	 (   2) minutes.
Extended self-test routine
recommended polling time: 	 ( 713) minutes.
SCT capabilities: 	       (0x0035)	SCT Status supported.
					SCT Feature Control supported.
					SCT Data Table supported.

SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   100   100   016    Pre-fail  Always       -       0
  2 Throughput_Performance  0x0005   137   137   054    Pre-fail  Offline      -       104
  3 Spin_Up_Time            0x0007   131   131   024    Pre-fail  Always       -       497 (Average 503)
  4 Start_Stop_Count        0x0012   100   100   000    Old_age   Always       -       1908
  5 Reallocated_Sector_Ct   0x0033   100   100   005    Pre-fail  Always       -       0
  7 Seek_Error_Rate         0x000b   100   100   067    Pre-fail  Always       -       0
  8 Seek_Time_Performance   0x0005   128   128   020    Pre-fail  Offline      -       18
  9 Power_On_Hours          0x0012   098   098   000    Old_age   Always       -       17768
 10 Spin_Retry_Count        0x0013   100   100   060    Pre-fail  Always       -       0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       1903
192 Power-Off_Retract_Count 0x0032   098   098   000    Old_age   Always       -       2427
193 Load_Cycle_Count        0x0012   098   098   000    Old_age   Always       -       2427
194 Temperature_Celsius     0x0002   142   142   000    Old_age   Always       -       42 (Min/Max 18/49)
196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       0
197 Current_Pending_Sector  0x0022   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0008   100   100   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x000a   200   200   000    Old_age   Always       -       0

SMART Error Log Version: 1
No Errors Logged

SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed without error       00%     17767         -
# 2  Extended offline    Completed without error       00%     17354         -
# 3  Extended offline    Completed without error       00%      8902         -
# 4  Extended offline    Completed without error       00%      7409         -
# 5  Extended offline    Interrupted (host reset)      20%      7397         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```

```
# smartctl -a /dev/sdc
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.15.0-48-lowlatency] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     WDC WD6003FZBX-00K5WB0
Serial Number:    V8HD3R0R
LU WWN Device Id: 5 000cca 098d399e2
Firmware Version: 01.01A01
User Capacity:    6,001,175,126,016 bytes [6.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-2, ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.2, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Sat Sep 24 09:58:12 2022 CEST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x82)	Offline data collection activity
					was completed without error.
					Auto Offline Data Collection: Enabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever 
					been run.
Total time to complete Offline 
data collection: 		(   87) seconds.
Offline data collection
capabilities: 			 (0x5b) SMART execute Offline immediate.
					Auto Offline data collection on/off support.
					Suspend Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					No Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine 
recommended polling time: 	 (   2) minutes.
Extended self-test routine
recommended polling time: 	 ( 645) minutes.
SCT capabilities: 	       (0x0035)	SCT Status supported.
					SCT Feature Control supported.
					SCT Data Table supported.

SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   100   100   016    Pre-fail  Always       -       0
  2 Throughput_Performance  0x0004   132   132   054    Old_age   Offline      -       96
  3 Spin_Up_Time            0x0007   165   165   024    Pre-fail  Always       -       401 (Average 330)
  4 Start_Stop_Count        0x0012   100   100   000    Old_age   Always       -       670
  5 Reallocated_Sector_Ct   0x0033   100   100   005    Pre-fail  Always       -       0
  7 Seek_Error_Rate         0x000a   100   100   067    Old_age   Always       -       0
  8 Seek_Time_Performance   0x0004   128   128   020    Old_age   Offline      -       18
  9 Power_On_Hours          0x0012   099   099   000    Old_age   Always       -       9962
 10 Spin_Retry_Count        0x0012   100   100   060    Old_age   Always       -       0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       664
192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       1030
193 Load_Cycle_Count        0x0012   100   100   000    Old_age   Always       -       1030
194 Temperature_Celsius     0x0002   144   144   000    Old_age   Always       -       38 (Min/Max 20/46)
196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       0
197 Current_Pending_Sector  0x0022   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0008   100   100   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x000a   200   200   000    Old_age   Always       -       0

SMART Error Log Version: 1
No Errors Logged

SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed without error       00%      9959         -
# 2  Extended offline    Aborted by host               90%      9569         -
# 3  Extended offline    Completed without error       00%      9569         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```

### Got Bad Blocks?

Even if S.M.A.R.T. extended tests passed there may still be bad
blocks that escaped detection, so the next step is to
[check for bad blocks](https://forums.linuxmint.com/viewtopic.php?t=255457)
on both disks. Need to use `-b 4096`
[to avoid 32-bit limitation](https://askubuntu.com/questions/548945/how-to-check-badsector-on-ext4-6tb):

```
# time badblocks -b 4096 -o sdb_bb.txt /dev/sdb
real    573m30.844s
user    0m32.405s
sys     11m49.075s

# time badblocks -b 4096 -o sdb_bb.txt /dev/sdc
real    488m9.397s
user    0m29.813s
sys     10m56.105s
```

Left both commands running in parallel, both running pretty fast, for the next ~10 hours:

![Disk I/O chart shows disks sdb and sdc becomes slower over time]({{ media }}/13.png)

The output is a list of bad blocks in each drive, which
[btrfs can’t use](https://unix.stackexchange.com/questions/362241/can-btrfs-track-avoid-bad-blocks/362242#362242).
However, both lists came out empty, i.e. no bad blocks were detected.

## Back To Normal

Both disks being now empty and seemingly undead, it’s time to[recreate the RAID1 array](https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.html#profiles)
and copy data back into it:

```
# mkfs.btrfs -d raid1 /dev/sdb /dev/sdc
btrfs-progs v5.16.2
See http://btrfs.wiki.kernel.org for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              (null)
UUID:               a4ee872d-b985-445f-94a2-15232e93dcd5
Node size:          16384
Sector size:        4096
Filesystem size:    10.92TiB
Block group profiles:
  Data:             RAID1             1.00GiB
  Metadata:         RAID1             1.00GiB
  System:           RAID1             8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  2
Devices:
   ID        SIZE  PATH
    1     5.46TiB  /dev/sdb
    2     5.46TiB  /dev/sdc

# dmesg | tail
BTRFS: device fsid a4ee872d-b985-445f-94a2-15232e93dcd5 devid 1 transid 6 /dev/sdb scanned by mkfs.btrfs (3135708)
BTRFS: device fsid a4ee872d-b985-445f-94a2-15232e93dcd5 devid 2 transid 6 /dev/sdc scanned by mkfs.btrfs (3135708)
BTRFS info (device sdb): flagging fs with big metadata feature
BTRFS info (device sdb): using free space tree
BTRFS info (device sdb): has skinny extents
BTRFS info (device sdb): checking UUID tree

# vi /etc/fstab
UUID=a4ee872d-b985-445f-94a2-15232e93dcd5 /home/raid  btrfs defaults 0 0

# btrfs filesystem label /home/raid HomeDepot6TB
# btrfs filesystem show /home/raid
Label: 'HomeDepot6TB'  uuid: a4ee872d-b985-445f-94a2-15232e93dcd5
        Total devices 2 FS bytes used 192.00KiB
        devid    1 size 5.46TiB used 2.01GiB path /dev/sdb
        devid    2 size 5.46TiB used 2.01GiB path /dev/sdc
```

From here, all that was left was `rsync`‘ing files across disks and computers.

## What Next?

In the last 10 years I’ve had 3 or more spinning-plate hard drives
fail, at least one of them abruptly enough to lose data. In the same
10 years, I’ve worked with more SSD (and NVME) disks and only one
failed, so at this point I’m really eager to do get rid of all
spinning-plate disks and move to 100% SSD.

