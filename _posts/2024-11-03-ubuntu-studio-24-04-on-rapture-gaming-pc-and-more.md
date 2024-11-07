---
title:  "Ubuntu Studio 24.04 on Rapture, Gaming PC (and more)"
date:   2024-11-03 12:11:12 +0200
categories: linux ubuntu installation setup
---

Installing Ubuntu Studio 24.04 on my main PC, for Gaming, coding, media and more."

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

## Considering Timing

[Ubuntu Studio 24.04 LTS Released](https://ubuntustudio.org/2024/04/ubuntu-studio-24-04-lts-released/)
on April 25 but, as they themselves put it
*since it’s just out, you may experience some issues, so you might want to wait a bit before upgrading*.

There doesn't seem to be anything particular scarey in release notes:

*  [Ubuntu Studio 24.04 LTS Release Notes](https://ubuntustudio.org/ubuntu-studio-24-04-LTS-release-notes/)
*  [Kubuntu 24.04 Release Notes](https://wiki.ubuntu.com/NobleNumbat/ReleaseNotes/Kubuntu)
*  [Noble Numbat Release Notes](https://discourse.ubuntu.com/t/noble-numbat-release-notes/39890/1)

And my plan is not to upgrade in place; I like to keep the previous
version around just in case I need a stable system to fall back to.

## Preparation

In preparation to upgrade my main PC to
[**Ubuntu Studio**](https://ubuntustudio.org/) **24.04**,
I installed a new
[Kingston FURY Renegade 4000 GB M.2 SSD](https://www.kingston.com/en/ssd/gaming/kingston-fury-renegade-nvme-m2-ssd)
and prepared partitions as follows:

1. 260 MB EFI System for the EFI boot.
2. 75 GB Linux filesystem for the root (ext4).
3. 75 GB Linux filesystem for the alternative root (ext4).
4. 3.5T GB Linux filesystem for `/home` (btrfs).

**Note:** 75 GB has proven to be a reasonable size for the
root partition for the amount of software that tends to be
installed in my PC, including 20 GB in `/usr` and 13 GB in
`/snap`.

### Partitions

First,
[create a GPT partition table](https://serverfault.com/a/709952)
(`label`), then create the partitions
[with `optimal` alignment](https://unix.stackexchange.com/a/49274)

```
# parted /dev/nvme1n1 --script -- mklabel gpt
# parted -a optimal /dev/nvme1n1 mkpart primary fat32 0% 260MiB
# parted -a optimal /dev/nvme1n1 mkpart primary ext4 260MiB 75GiB
# parted -a optimal /dev/nvme1n1 mkpart primary ext4 75GiB 150GiB
# parted -a optimal /dev/nvme1n1 mkpart primary btrfs 150GiB 100%

# parted /dev/nvme1n1 print
Model: KINGSTON SFYRD4000G (nvme)
Disk /dev/nvme1n1: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  273MB   272MB                primary  msftdata
 2      273MB   80.5GB  80.3GB               primary
 3      80.5GB  161GB   80.5GB               primary
 4      161GB   4001GB  3840GB               primary

# fdisk -l /dev/nvme1n1 
Disk /dev/nvme1n1: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: KINGSTON SFYRD4000G                     
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 3C935B5D-3EFC-4683-9483-BC110B2AEB17

Device             Start        End    Sectors  Size Type
/dev/nvme1n1p1      2048     532479     530432  259M Microsoft basic data
/dev/nvme1n1p2    532480  157286399  156753920 74.7G Linux filesystem
/dev/nvme1n1p3 157286400  314572799  157286400   75G Linux filesystem
/dev/nvme1n1p4 314572800 7814035455 7499462656  3.5T Linux filesystem
```

**Note:** in retrospect, it seems to be necessary to also apply the
`boot` flag to the EFI partition; otherwise the Ubuntu installer will
not offer the possibility of installing the boot loader in this disk:

```
# parted /dev/nvme1n1 toggle 1 boot

# parted /dev/nvme1n1 print
Model: KINGSTON SFYRD4000G (nvme)
Disk /dev/nvme1n1: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  273MB   272MB                primary  boot, esp
 2      273MB   80.5GB  80.3GB               primary
 3      80.5GB  161GB   80.5GB               primary
 4      161GB   4001GB  3840GB               primary
```

Partitions can be created during the installation of the
system, there is no need to create them before hand because
the previous M.2 SSD is not going anywhere any time soon.

In the future, the previous (2 TB) M.2 SSD may be replaced with another 4 TB SSD, e.g.
- [Kingston FURY Renegade with Heatsink](https://www.digitec.ch/en/s1/product/kingston-fury-renegade-with-heatsink-4000-gb-m2-2280-ssd-22903765) ($320+)
- [Samsung 990 Pro with Heatsink](https://www.digitec.ch/en/s1/product/samsung-990-pro-with-heatsink-4000-gb-m2-2280-ssd-37728060) ($300+)

However, that replacement will likely not happen under 2026,
once Ubuntu Studio 26.04 is installed and there is no longer
a point to keep the old Ubuntu 22.04 around.

In the meantime, the new SSD can be converted into being the new
`/home` and, while the old one is still around, this could be a
good time to try [bcachefs](https://bcachefs.org/) on the new one.
However, that requires either [building a kernel](https://kernelnewbies.org/KernelBuild)
or waiting for Linux **6.7** or later.
[Ubuntu 24.04 LTS Will Aim To Ship With The Linux 6.8 Kernel](https://www.phoronix.com/news/Ubuntu-24.04-Will-Use-Linux-6.8),
so the easier approach would be to wait for the installation of Ubuntu 24.04
to move `/home` to the new SSD.

Listen to [Linux Matters #23: An Exodus of Bitcoin](https://linuxmatters.sh/23/)
were *Martin has excitedly installed it on everything!*
In particular, there seems to be options to configure large
file systems across multiple devices **with redundancy**
(details to be confirmed) and *tiered storage* to keep
*hot data* on the faster storage (NVME) and rebalance unused
data back to slower storage (SATA).

### New BtrFS Home

Create a new `btrfs` file system to test the new SSD, while
we're waiting for the upcoming Ubuntu 24.04 release:

```
# mkfs.btrfs /dev/nvme1n1p4
btrfs-progs v5.16.2
See http://btrfs.wiki.kernel.org for more information.

Performing full device TRIM /dev/nvme1n1p4 (3.49TiB) ...
NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              (null)
UUID:               8edfc3ba-4981-4423-8730-7e229bfa63f3
Node size:          16384
Sector size:        4096
Filesystem size:    3.49TiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       yes
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1     3.49TiB  /dev/nvme1n1p4

# mkdir /home/new-m2
# tail -1 /etc/fstab 
UUID=8edfc3ba-4981-4423-8730-7e229bfa63f3 /home/new-m2      btrfs   defaults        0       0
```

Suddently there is a lot of space available!

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p5  1.7T  1.2T  517G  70% /home
/dev/sdc        3.7T  2.7T  1.1T  72% /home/ssd
/dev/sda        5.5T  5.3T  254G  96% /home/raid
/dev/sdb        3.7T  3.0T  667G  83% /home/new-ssd
/dev/nvme1n1p4  3.5T  3.7M  3.5T   1% /home/new-m2
```

### Transfer Speed Test

First, transfer 1.4 GB of media (family photos and videos)
from the SATA SSD. Theoretical maximum transfer speed is
around 550 MB/s, but in practice the transfer speed fluctuates
between 500 and 540 MB/s and the transfer took 48 minutes.
There was little difference between using `rync -uva` vs
`cp -av`.

```
# time cp -a /home/ssd/Fotos /home/new-m2/

real    48m39.874s
user    0m3.933s
sys     13m10.019s
```

Then, transfer 1.1 GB of personal files from the old
NVME SSD. Theoretical maximum transfer speed is
around 3000 MB/s, in practice the transfer speed fluctuates
between 800 and 2500 MB/s and the transfer took 25 minutes,
so just about twice as fast as the previous one.

```
# time rsync -a /home/coder /home/new-m2/

real    24m54.444s
user    4m31.149s
sys     17m54.582s
```

With these 2 transfers, the new M.2 SSD is nearly at 70%:

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p5  1.7T  1.2T  517G  70% /home
/dev/sdc        3.7T  2.7T  1.1T  72% /home/ssd
/dev/sda        5.5T  5.3T  254G  96% /home/raid
/dev/sdb        3.7T  3.0T  667G  83% /home/new-ssd
/dev/nvme1n1p4  3.5T  2.4T  1.2T  69% /home/new-m2
```

## Installation

With the above partitions prepared well in advance, to
[Install Ubuntu Studio 24.04]({{ site.baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#install-ubuntu-studio-2404)
the process *should* be as simple, easy and smooth as it
was with other systems.

Alas, it wasn't. Even after setting up all the partition
correctly for the new install, the installer would not
allow selecting the correct device for to install the
boot loader in it: `nvme1n1` is grayed out!

This problem is one I had seen recently, but didn't write
down what the solution was.

While search (in vain) for others facing the same issue,
I took advantage of being in a live USB system and
cloned the current Ubuntu Studio 22.04 root partition
(`nvme0n1p6`) onto what *would* have been the new Ubuntu
Studio 24.04 root partition (`nvme1n1p2`), in case this
may come in handy later.

**Important:** after cloning a root file system, the
`/etc/fstab` file in the new clone must be updated with
the correct UUID of that partition.

If the (new) EFI partition has not been formatted yet;
format it as **FAT32**:

```
# mkfs.fat -F32 /dev/nvme1n1p1
mkfs.fat 4.2 (2021-01-31)

# lsblk -f
nvme1n1
│
├─nvme1n1p1
│    vfat   FAT32       73CC-6E86
...
├─nvme1n1p2
│    ext4   1.0         409501ea-d63d-49b2-bd45-3b876404dc53

nvme0n1
│
...
└─nvme0n1p6
     ext4   1.0         de317ca5-96dd-49a7-b72b-4bd050a8d15c   20.8G    67% /var/snap/firefox/common/host-hunspell
```

The mount the new root and edit `/etc/fstab` in it:

```
# mount /dev/nvme1n1p2 /media/cdrom/
# vi /media/cdrom/etc/fstab
...
# <file system>             <mount point>  <type>  <options>  <dump>  <pass>
UUID=73CC-6E86                            /boot/efi      vfat    umask=0077 0 2
UUID=409501ea-d63d-49b2-bd45-3b876404dc53 /              ext4    defaults,discard 0 1
...
```

One potential problem is that no partition in `nvme1n1` 
has tbe `boot` flag. It appears all bootable partitions
that are working have flags `boot, esp` so the solution
may be simply to add those flags. At the very least, this
seems like it is necessary, if not sufficient:

```
# parted /dev/nvme1n1
GNU Parted 3.4
Using /dev/nvme1n1
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model: KINGSTON SFYRD4000G (nvme)
Disk /dev/nvme1n1: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  273MB   272MB                primary  msftdata
 2      273MB   80.5GB  80.3GB  ext4         primary
 4      161GB   4001GB  3840GB  btrfs        primary

(parted) toggle 1 boot
(parted) print
Model: KINGSTON SFYRD4000G (nvme)
Disk /dev/nvme1n1: 4001GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  273MB   272MB                primary  boot, esp
 2      273MB   80.5GB  80.3GB  ext4         primary
 4      161GB   4001GB  3840GB  btrfs        primary
```

**Note:** `boot` and `esp` are the same flag; if you
toggle both you end up with the initial state.

At this point, decided to follow 
[askubuntu.com/a/1463655](https://askubuntu.com/a/1463655)
to fully cloned the current system onto the new 4TB
NVME and try to boot from it.

```
# mount /dev/nvme1n1p2 /media/cdrom/
# mount /dev/nvme1n1p1 /media/cdrom/boot/efi/

# grub-install \
  --target x86_64-efi \
  --efi-directory /media/cdrom/boot/efi \
  --boot-directory /media/cdrom/boot
Installing for x86_64-efi platform.
Installation finished. No error reported.

# grub-mkconfig -o /media/cdrom/boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.15.0-124-lowlatency
Found initrd image: /boot/initrd.img-5.15.0-124-lowlatency
Found linux image: /boot/vmlinuz-5.15.0-124-lowlatency
Found initrd image: /boot/initrd.img-5.15.0-124-lowlatency
Found linux image: /boot/vmlinuz-5.15.0-122-lowlatency
Found initrd image: /boot/initrd.img-5.15.0-122-lowlatency
Found linux image: /boot/vmlinuz-5.15.0-60-lowlatency
Found initrd image: /boot/initrd.img-5.15.0-60-lowlatency
Memtest86+ needs a 16-bit boot, that is not available on EFI, exiting
Warning: os-prober will be executed to detect other bootable partitions.
Its output will be used to detect bootable binaries on them and create new boot entries.
Found Ubuntu 22.04.1 LTS (22.04) on /dev/nvme0n1p2
Found Ubuntu 22.04.5 LTS (22.04) on /dev/nvme1n1p2
Adding boot menu entry for UEFI Firmware Settings ...
done
```

At this point there *should* be a boot loader, ready to
boot, on thew new NVME. If anything, it looks like it
would boot the *old old* 22.04.1 `nvme0n1p2` instead of
the *old* (current) 22.04.5 `nvme0n1p6`.

Indeed upon reboot, the UEFI boot menu now shows both
NVME disks and selecting the 4TB disk the new bootloader
shows those entries. After this, the installation process
was, finally, smooth and successful, installing the new
boot loader in `nvme1n1`.

This *new new* bootloader in `nvme1n1` now shows all 4
systems available to boot
- Ubuntu 22.04.1 in `nvme0n1p2` (have not used in some time)
- Ubuntu 22.04.5 in `nvme0n1p6` (current daily driver)
- Ubuntu 22.04.5 in `nvme1n1p2` (future backup daily driver)
- Ubuntu 24.04.1 in `nvme1n1p3` (future daily driver)

### Adjusting all `/etc/fstab` files

To make sure all those root partitions are usable, their
`/etc/fstab` files need to be adjusted to point to the
correct `/boot/efi` partitions (the one in the same disk)
and the new one will has a new UUID after the last
installation:

```
# lsblk -f
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
...
nvme0n1
│
├─nvme0n1p1
│                  
├─nvme0n1p2
│    ext4   1.0         833c6403-a771-46b2-bde8-704f2ab7e88b
├─nvme0n1p3
│    ext4   1.0         343f75fe-ec96-49fa-a4f8-0d32c69c1424
├─nvme0n1p4
│    vfat   FAT32 NO_LABEL
│                       C38B-C318                             293.3M     2% /boot/efi
├─nvme0n1p5
│    btrfs              18238846-d411-4dcb-af87-a2d19a17fef3  654.9G    62% /home
└─nvme0n1p6
     ext4   1.0         de317ca5-96dd-49a7-b72b-4bd050a8d15c

nvme1n1
│
├─nvme1n1p1
│    vfat   FAT16       4485-0F5E
├─nvme1n1p2
│    ext4   1.0         409501ea-d63d-49b2-bd45-3b876404dc53   18.7G    69%
├─nvme1n1p3
│    ext4   1.0         1d30a16e-b4f6-4459-9b19-8c9093b0d047                
└─nvme1n1p4
     btrfs              8edfc3ba-4981-4423-8730-7e229bfa63f3      1T    71% /home/new-m2
```

Following the above order, make sure that each
`/etc/fstab` file points to the correct partition/s:

Ubuntu 22.04.1 in `nvme0n1p2` (not used in some time) is
not even using an EFI partition at all; this one dates
back to a time when this disk was used in legacy BIOS
mode:

```
# df -h | grep nvme0n1p2
/dev/nvme0n1p2   50G   27G   21G  57% /jellyfish
# grep -E '/ |/bo'  /jellyfish/etc/fstab 
UUID=833c6403-a771-46b2-bde8-704f2ab7e88b /              ext4    defaults   0 1
```

Ubuntu 22.04.5 in `nvme0n1p6` (current daily driver) has
not changed, as expected:

```
# grep -E '/ |/bo'  /etc/fstab 
UUID=C38B-C318                            /boot/efi      vfat    umask=0077 0 2
UUID=de317ca5-96dd-49a7-b72b-4bd050a8d15c /              ext4    defaults,discard 0 1
```

Ubuntu 22.04.5 in `nvme1n1p2` (the *future backup* daily
driver) is a clone of the one in `nvme0n1p6` **but** it
is also on the newer NVME disk, so both partitions must
be updated:
- `/boot/efi` must point to `nvme1n1p1`
- `/` must point to `nvme1n1p2`

```
# mount /dev/nvme1n1p2 /media/cdrom/
# grep -E '/ |/bo'  /media/cdrom/etc/fstab 
UUID=4485-0F5E                            /boot/efi      vfat    umask=0077 0 2
UUID=409501ea-d63d-49b2-bd45-3b876404dc53 /              ext4    defaults,discard 0 1
# umount /dev/nvme1n1p2
```

This being an updated copy of the current `/etc/fstab`
file, it already has all the partitions currently in use.

**Ubuntu 24.04.1** in `nvme1n1p3` (the **future** daily
driver) should be already good to go; since it was just
installed:

```
# mount /dev/nvme1n1p3 /noble
# grep -E '/ |/bo'  /noble/etc/fstab 
# / was on /dev/nvme1n1p3 during curtin installation
/dev/disk/by-uuid/1d30a16e-b4f6-4459-9b19-8c9093b0d047 / ext4 defaults 0 1
# /boot/efi was on /dev/nvme1n1p1 during curtin installation
/dev/disk/by-uuid/4485-0F5E /boot/efi vfat defaults 0 1
```

### New mount points for all partitions

The `/etc/fstab` in **Ubuntu 24.04.1** (`nvme1n1p3`) is
missing all the *other* partitions currently in use, and
includes a swap file that is unnecessary in a system with
32 GB of RAM:

```bash
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/nvme1n1p3 during curtin installation
/dev/disk/by-uuid/1d30a16e-b4f6-4459-9b19-8c9093b0d047 / ext4 defaults 0 1
# /boot/efi was on /dev/nvme1n1p1 during curtin installation
/dev/disk/by-uuid/4485-0F5E /boot/efi vfat defaults 0 1
# /home was on /dev/nvme1n1p4 during curtin installation
/dev/disk/by-uuid/8edfc3ba-4981-4423-8730-7e229bfa63f3 /home btrfs defaults 0 1
/swap.img       none    swap    sw      0       0
```

Because this PC has been *collecting* hard drives over
the years, there are many additional partitions in use:

```
# df -h | head -1; df -h | grep -E 'nvme|sd.'
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p6   74G   49G   21G  71% /
/dev/nvme0n1p4  300M  6.1M  294M   3% /boot/efi
/dev/nvme0n1p5  1.7T  1.1T  655G  62% /home
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/nvme1n1p4  3.5T  2.5T  1.1T  71% /home/new-m2
/dev/sda        5.5T  5.1T  370G  94% /home/raid
/dev/nvme1n1p3   74G   22G   49G  31% /noble
```

For just about the same reason, there are also several
symlinks strategically pointing from where thigns used
to be to where they are now:

```
# ls -l /home/
total 80
lrwxrwxrwx 1 root   root      17 Sep 24  2022 depot -> /home/raid/depot/
lrwxrwxrwx 1 root   root      16 May 12 19:37 k8s -> /home/new-m2/k8s
lrwxrwxrwx 1 root   root      16 May 12 10:02 lib -> /home/new-m2/lib

# ls -l /home/raid/depot/[av]*
lrwxrwxrwx 1 coder coder 16 Aug 20  2023 /home/raid/depot/audio -> /home/raid/audio
lrwxrwxrwx 1 coder coder 16 Aug 20  2023 /home/raid/depot/video -> /home/raid/video

# ls -l /home/new-ssd/video
lrwxrwxrwx 1 root root 16 Aug 20  2023 /home/new-ssd/video -> /home/raid/video
```

In the future daily driver, the old 2TB NVME should not
be used, so it can be replaced later by a newer 4TB disk.

To that effect, all data in `/dev/nvme0n1p5` must be
copied over to `/dev/nvme1n1p4` and in fact most of it
is already there. There are only a few users' home
directories and empty directories (mount points):

```
# du -sh /home/*
952G    /home/coder
18G     /home/ernest
134G    /home/manuel
1.5G    /home/minecraft
44K     /home/sam

# du -sh /home/new-m2/*
952G    /home/new-m2/coder
18G     /home/new-m2/ernest
1.5T    /home/new-m2/Fotos
26G     /home/new-m2/k8s
35G     /home/new-m2/lib
134G    /home/new-m2/manuel
1.5G    /home/new-m2/minecraft
44K     /home/new-m2/sam

# cd /home/new-m2/
root@rapture:/home/new-m2# mkdir new-ssd raid ssd
```

#### Boot cloned Ubuntu Studio 22.04 without old NVME

As an intermediate step to booting the new Ubuntu Studio
24.04 later, boot the newly cloned Ubuntu Studio 22.04
in `nvme1n1p3` **without** mounting the old NVME.

With the above preparetions done in the new NVME, this
*should* be as simple as mounting the new NVME on
`/home` and simply *not mounting* the old NVME:

```
# mount /dev/nvme1n1p2 /media/cdrom/

# vi /media/cdrom/etc/fstab
...
# Previous-new (June 2022) 2TB NVME SSD (/home)
#UUID=18238846-d411-4dcb-af87-a2d19a17fef3 /home          btrfs   defaults,noatime,autodefrag,discard,compress=lzo 0 0

# New-new 4TB M.2 SSD (newer /home; previously /home/new-m2)
UUID=8edfc3ba-4981-4423-8730-7e229bfa63f3 /home      btrfs   defaults        0       0

# umount /media/cdrom/
```

Reboot into the *newly cloned* **Ubuntu Studio 22.04**
and check that everything works as usual. This would be
the first green light to removing the old 2TB NVME disk
from the system.

Despite booting from the new bootloader in the 4TB NVME
the **Ubuntu Studio 22.04.5 on /dev/nvme1n1p2** option, 
somehow the system boots the *old old* root:

```
$ df-h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p6   74G   49G   22G  70% /
/dev/nvme0n1p2   50G   27G   21G  57% /jellyfish
/dev/nvme0n1p4  300M  6.1M  294M   3% /boot/efi
/dev/nvme0n1p5  1.7T  1.1T  655G  62% /home
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/nvme1n1p4  3.5T  2.7T  887G  76% /home/new-m2
/dev/sda        5.5T  5.1T  370G  94% /home/raid
```

The bootloader entry specifies the root file system as
`409501ea-d63d-49b2-bd45-3b876404dc53` but it boots on
`de317ca5-96dd-49a7-b72b-4bd050a8d15c`; despite the
correct UUID in the new `/etc/fstab`.

The problem is in precisely *this* entry in the boot
loader, as is the only one that boots the kernel with
the wrong `root=` parameter. This can be confirmed in the
bootloader by pressing `e` after selecting the entry to
boot the newly cloned Ubuntu 22.04:

```
setparams 'Ubuntu 22.04.5 LTS (22.04) (on /dev/nvme1n1p2)'

        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root 409501ea-d63d-49b2-bd45-3b876404dc53
        linux /boot/vmlinuz-5.15.0-124-lowlatency root=UUID=de317ca5-96dd-49a7-b72b-4bd050a8d15c ro threadirqs noquiet nosplash
        initrd /boot/initrd.img-5.15.0-124-lowlatency
```

There is why the kernel is mounting the root from the old NMVE:
the UUID in the `search` line is the desired root, the newly
cloned 22.04 in the 4TB NVME, but the `linux` line is pointing
to the old root in the 2TB NVME. Editing this value and then
pressing `Ctrl+x` finally boots into the newly cloned 22.04
and the old 2TB NVMe is not mounted or used at all:

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1p2   74G   51G   19G  74% /
/dev/nvme1n1p1  259M  6.2M  253M   3% /boot/efi
/dev/nvme1n1p4  3.5T  2.7T  887G  76% /home
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/sda        5.5T  5.1T  370G  94% /home/raid
```

To make this change permanent, the answer (from **2019**) in
https://askubuntu.com/a/1140397 seems to suggest that simply
running `sudo update-grub` would pick up the correct root.
This would make sense, but perhaps would be better to do it
from the new 24.04 system. Perhaps the old root UUID has
been left somewhere in the new root UUID, which would be
updated by now.

So the next move is to boot the new 24.04 system, but before
doing that its `/etc/fstab` needs to be updated to **add** the
additional partitions in the old SATA disks.

First, lets add the 24.04 root as a read-only mount:

```
# vi /etc/fstab
...
# NEW Ubuntu Studio 24.04 root (nvme1n1p3)
UUID=1d30a16e-b4f6-4459-9b19-8c9093b0d047 /noble      ext4   defaults,ro        0       0
```

Then update its `/etc/fstab` after remounting as read-write:

```
# mount /noble/ -o remount,rw

# cat /etc/fstab \
  | grep --color=no -C2 sd. \
  >> /noble/etc/fstab 

# cat /noble/etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/nvme1n1p3 during curtin installation
/dev/disk/by-uuid/1d30a16e-b4f6-4459-9b19-8c9093b0d047 / ext4 defaults 0 1
# /boot/efi was on /dev/nvme1n1p1 during curtin installation
/dev/disk/by-uuid/4485-0F5E /boot/efi vfat defaults 0 1
# /home was on /dev/nvme1n1p4 during curtin installation
/dev/disk/by-uuid/8edfc3ba-4981-4423-8730-7e229bfa63f3 /home btrfs defaults 0 1
/swap.img       none    swap    sw      0       0

# Previous (June 2021) 4TB SSD (previous-previous /home)
# /home/ssd is now on /dev/sde with a new UUID
UUID=5cf65a95-4ae5-41ed-9a14-7d7fbeee1951 /home/ssd       btrfs   defaults        0       2

# /home/raid was on /dev/sdb during installation
UUID=a4ee872d-b985-445f-94a2-15232e93dcd5 /home/raid      btrfs   defaults        0       0

# New (Jan 2023) 4TB SSD (Crucial MX)
# Always write slowly! e.g.
# rsync -turva --bwlimit=500000 /home/depot/audio/* /home/ssd/audio/
UUID=6b809fc0-0b85-4041-ac25-47ec4682f5f5 /home/new-ssd      btrfs   defaults        0       0
```

Finally, comment out the line for the swap file (`/swap.img`),
because that won't be necessary in a system with 32GB of RAM,
and add a line to mount the newly clone 22.04 as read-only:

```
# mkdir /noble/jammy
# vi /noble/etc/fstab
...
# Ubuntu Studio 22.04 root (ro)
UUID=409501ea-d63d-49b2-bd45-3b876404dc53 /jammy      ext4   defaults,ro        0       0
```

With all these adjustments, after booting into the new
**Ubuntu Studio 24.04** this should be what is mounted:

```
# df -h | head -1; df -h | grep -E 'nvme|sd.'
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1p3   74G   22G   49G  31% /
/dev/nvme1n1p1  300M  6.1M  294M   3% /boot/efi
/dev/nvme1n1p4  3.5T  2.5T  1.1T  71% /home/
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/sda        5.5T  5.1T  370G  94% /home/raid
/dev/nvme1n1p2   74G   51G   19G  74% /jammy
```

And then it will be a better time to run `sudo update-grub`
to hopefully pick up the correct root for the new 22.04.

## First boot into Ubuntu Studio 24.04

The first time booting into the new system, right after login for
the first time an additional reboot is required for the
[Ubuntu Studio Audio Configuration]({{ site.baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html##ubuntu-studio-audio-configuration).

After rebooting again, `df` shows partitions are mounted like this:

```
# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           3.2G  2.6M  3.2G   1% /run
/dev/nvme1n1p3   74G   22G   48G  32% /
tmpfs            16G     0   16G   0% /dev/shm
tmpfs           5.0M   24K  5.0M   1% /run/lock
efivarfs        128K   51K   73K  42% /sys/firmware/efi/efivars
/dev/nvme1n1p2   74G   51G   19G  74% /jammy
/dev/nvme1n1p1  259M  6.2M  253M   3% /boot/efi
/dev/nvme1n1p4  3.5T  2.7T  887G  76% /home
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/sda        5.5T  5.1T  370G  94% /home/raid
tmpfs           3.2G  136K  3.2G   1% /run/user/1000
```
### Fix boot for cloned Ubuntu Studio 22.04

Contrary to initial expectaions, running `sudo update-grub` on the
new 24.04 system did not help fix the root UUID for `nvme1n1p2`;
it actually *unfixed* the one that was correct!

Instead, it is necessary to edit `/boot/grub/grub.cfg` as
suggested in https://superuser.com/a/485763 and replace
`nvme0n1p6` UUID with that of `nvme1n1p2` for all entries
under the name `Ubuntu 22.04.5 LTS (22.04) (on /dev/nvme1n1p2)`
*and then simply reboot*. This does lead to the *new old*
system (22.04 in the new NVME) to boot correctly and `df -h`
shows the desired partitions mounted:

```
$ df-h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1p2   74G   51G   19G  73% /
/dev/nvme1n1p1  259M  6.2M  253M   3% /boot/efi
/dev/nvme1n1p3   74G   24G   47G  34% /noble
/dev/nvme1n1p4  3.5T  2.7T  886G  76% /home
/dev/sdc        3.7T  2.9T  833G  78% /home/ssd
/dev/sdb        3.7T  2.1T  1.6T  57% /home/new-ssd
/dev/sda        5.5T  5.1T  370G  94% /home/raid
```

**Warning:** running `sudo update-grub` will *unfix* the root UUID for `nvme1n1p2` *again*; and this will happen each time
a new kernel is installed.

Once the *new old* 22.04 system is *reliably* bootable, it can be
left alone as a fallback system, and continue setting up the new one.


### Multiple IPs on LAN

Connecting to the local wired network provides a dynamic IP address
that may change over time, but it is more convenient to have fixed
IP addresses. Moreover, the DHCP range is shared with the wireless
network, we want to have an additional wired-only LAN and set *both*
IP addresses on the same NIC.

To this effect, copy `/etc/netplan/01-network-manager-all.yaml` from
the old system, change the network interface name if different (run
`ip a` to check) and apply the changes with `netplan apply`:

```
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 04:42:1a:97:4e:47 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global dynamic noprefixroute enp5s0
       valid_lft 78508sec preferred_lft 78508sec
    inet6 fe80::642:1aff:fe97:4e47/64 scope link 
       valid_lft forever preferred_lft forever
```

Network interface is `enp5s0`; 


```yaml
# Dual static IP on LAN, nothing else.
network:
  version: 2
  renderer: networkd
  ethernets:
    enp4s0:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.2/24, 192.168.0.2/24 ]
      nameservers:
      # Set DNS name servers
        search: [v.cablecom.net]
        addresses: [62.2.24.158, 62.2.17.61]
    enp5s0:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.2/24, 192.168.0.2/24 ]
      # Set default gateway
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
      # Set DNS name servers
        search: [v.cablecom.net]
        addresses: [62.2.24.158, 62.2.17.61]
```

```
# chmod 400 /etc/netplan/01-network-manager-all.yam
# netplan apply

# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 04:42:1a:97:4e:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global enp5s0
       valid_lft forever preferred_lft forever
    inet 192.168.0.2/24 metric 100 brd 192.168.0.255 scope global dynamic enp5s0
       valid_lft 86382sec preferred_lft 86382sec
```

### SSH Server

Ubuntu Studio doesn't enable the SSH server by default, but we want
this to adjust the system remotely:

```
# apt install ssh -y
# sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
# systemctl enable --now ssh
```

**Note:** remember to copy over files under `/root` from
previous system/s, in case it contains useful scripts (and/or
SSH keys worth keeping under `.ssh`).

```
# rm /root/.ssh/authorized_keys 
# rmdir /root/.ssh/ 
# cp -a /mnt/root/.ssh/ /root/
```

### `/etc/hosts`

Having the old system's root partition mounted (see above),
copy over `/etc/hosts` so that connections to local hosts work as
smoothly as in the old system (e.g. for
[Continuous Monitoring](#continuous-monitoring)).

Better yet, **append** the old `/etc/hosts` to the new one, then
edit the new one to remove redundant lines. There are a few
interesting lines in the new one:

```
# Standard host addresses
127.0.0.1 localhost
127.0.1.1 rapture

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### Install Essential Packages

Start by installing a few
[essential packages]({{ site.baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#install-essential-packages):

```
# apt install gdebi-core wget gkrellm vim curl gkrellm-leds \
  gkrellm-xkb gkrellm-cpufreq geeqie playonlinux exfat-fuse \
  clementine id3v2 htop vnstat neofetch tigervnc-viewer sox \
  scummvm wine gamemode python-is-python3 exiv2 rename scrot \
  speedtest-cli xcalib python3-pip netcat-openbsd jstest-gtk \
  etherwake python3-selenium lm-sensors sysstat tor unrar \
  ttf-mscorefonts-installer winetricks icc-profiles ffmpeg \
  iotop-c xdotool redshift-qt inxi vainfo vdpauinfo mpv \
  tigervnc-tools screen lutris xsane -y
```

After installing these **Redshift** is immediately available.

Even before installing any packages **KDE Connect** is already
running, which comes in very handy if it was already setup.

At this point a reboot should not be necessary.

## Install Additional Software

### Brave browser

Install from the
[Release Channel](https://brave.com/linux/#release-channel-installation):

```
# curl -fsSLo \
  /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

# echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" \
| tee /etc/apt/sources.list.d/brave-browser-release.list

# apt update && apt install brave-browser -y
```

### Google Chrome

Installing [Google Chrome](https://google.com/chrome) is as
simple as downloading the Debian package and installing it:

```
# dpkg -i google-chrome-stable_current_amd64.deb
```

### ActivityWatch

As soon as Chrome is started, the ActivityWatch extension complains
that it can't connect to the server, because it only runs locally.

Install the latest version of [ActivityWatch]({{ site.baseurl }}/2024/06/30/self-hosted-time-tracking-with-activitywatch.html)
and then run `/opt/activitywatch/aw-qt` manually once.
This should already be in the **Autostart** settings, if it was
setup previously, so it only needs to be run manually this once. 

### Steam

Installing Steam from Snap 
[couldn't be simplers](https://unixhint.com/install-steam-on-ubuntu-24-04/):

```
# snap install steam
steam 1.0.0.81 from Canonical✓ installed
```

**Note:** [snapcraft.io/steam](https://snapcraft.io/steam) is
provided by Canonical.

When runing the Steam client for the first time, a pop-up shows up
advising to install additional 32-bit drivers *for best experience*

```
# dpkg --add-architecture i386
# apt update
# apt install libnvidia-gl-550:i386 -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  gcc-14-base:i386 libbsd0:i386 libc6:i386 libdrm2:i386 libffi8:i386 libgcc-s1:i386 libidn2-0:i386 libmd0:i386
  libnvidia-egl-wayland1:i386 libunistring5:i386 libwayland-client0:i386 libwayland-server0:i386 libx11-6:i386
  libxau6:i386 libxcb1:i386 libxdmcp6:i386 libxext6:i386
Suggested packages:
  glibc-doc:i386 locales:i386 libnss-nis:i386 libnss-nisplus:i386
The following NEW packages will be installed:
  gcc-14-base:i386 libbsd0:i386 libc6:i386 libdrm2:i386 libffi8:i386 libgcc-s1:i386 libidn2-0:i386 libmd0:i386
  libnvidia-egl-wayland1:i386 libnvidia-gl-550:i386 libunistring5:i386 libwayland-client0:i386 libwayland-server0:i386
  libx11-6:i386 libxau6:i386 libxcb1:i386 libxdmcp6:i386 libxext6:i386
0 upgraded, 18 newly installed, 0 to remove and 6 not upgraded.
```

It should also be possible to install the official Steam client, with
[the non-snap alternative]({{ site.baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#non-snap-alternative). This doesn't seems necessary anymore.

### Minecraft Java Edition

To avoid taking chances, copy the Minecraft launcher from the
previous system:

```
# cp -a /jammy/opt/minecraft-launcher/ /opt/
```

It works perfectly right after installing; no need to login again.

In contrast, trying to re-download Minecraft Java Edition
[seems to lead nowhere good]({{ site.baseurl }}/2024/11/03/ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.html).

### Minecraft Bedrock Edition

There is an **unofficial**
[Minecraft Bedrock Launcher](https://flathub.org/apps/io.mrarm.mcpelauncher),
including smiple steps to install it on
[Debian / Ubuntu](https://mcpelauncher.readthedocs.io/en/latest/getting_started/index.html#debian-ubuntu).
This has not seemed necessary so far, since the family enjoys
playing the Java edition more.

### Blender

[Blender 4.2 LTS](https://www.blender.org/) is already available
even for Ubuntu 22.04 via 
[snapcraft.io/blender](https://snapcraft.io/blender)
so there is no reason to install it any other way:

```
# snap install blender --classic
blender 4.2.3 from Blender Foundation (blenderfoundation✓) installed
```

### Continuous Monitoring

Install the
[multi-thread version]({{ site.baseurl }}/conmon/#deploy-to-pcs)
of the `conmon` script as `/usr/local/bin/conmon` and
[run it as a service]({{ site.baseurl }}/conmon/#install-conmon);
create `/etc/systemd/system/conmon.service` as follows:

```ini
[Unit]
Description=Continuous Monitoring

[Service]
ExecStart=/usr/local/bin/conmon
Restart=on-failure
StandardOutput=null

[Install]
WantedBy=multi-user.target
```

Then enable and start the services in `systemd`:

```
# systemctl enable conmon.service
# systemctl daemon-reload
# systemctl start conmon.service
# systemctl status conmon.service
```

#### Hardware Sensors

Initially there is only a limited amount of hardware sensors:

```
# sensors -A
nvme-pci-0400
Composite:    +36.9°C  (low  = -20.1°C, high = +83.8°C)
                       (crit = +88.8°C)
Sensor 2:     +72.8°C  

k10temp-pci-00c3
Tctl:         +42.5°C  
Tccd2:        +36.0°C  

nvme-pci-0100
Composite:    +42.9°C  (low  = -273.1°C, high = +84.8°C)
                       (crit = +84.8°C)
Sensor 1:     +42.9°C  (low  = -273.1°C, high = +65261.8°C)
Sensor 2:     +42.9°C  (low  = -273.1°C, high = +65261.8°C) 
```

HDD temperatures are available by loading the drivetemp kernel module:

```
# echo drivetemp > /etc/modules-load.d/drivetemp.conf
# modprobe drivetemp
# # sensors -A
drivetemp-scsi-9-0
temp1:        +40.0°C  (low  =  +0.0°C, high = +55.0°C)
                       (crit low = -40.0°C, crit = +70.0°C)
                       (lowest = +24.0°C, highest = +40.0°C)

drivetemp-scsi-5-0
temp1:        +29.0°C  (low  =  +0.0°C, high = +70.0°C)
                       (crit low =  +0.0°C, crit = +70.0°C)
                       (lowest = +25.0°C, highest = +34.0°C)

drivetemp-scsi-2-0
temp1:        +40.0°C  (low  =  +0.0°C, high = +60.0°C)
                       (crit low = -40.0°C, crit = +70.0°C)
                       (lowest = +24.0°C, highest = +40.0°C)

nvme-pci-0400
Composite:    +36.9°C  (low  = -20.1°C, high = +83.8°C)
                       (crit = +88.8°C)
Sensor 2:     +72.8°C  

k10temp-pci-00c3
Tctl:         +42.5°C  
Tccd2:        +36.0°C  

drivetemp-scsi-8-0
temp1:        +36.0°C  (low  =  +0.0°C, high = +60.0°C)
                       (crit low = -41.0°C, crit = +85.0°C)
                       (lowest = +22.0°C, highest = +36.0°C)

drivetemp-scsi-4-0
temp1:        +29.0°C  (low  =  +0.0°C, high = +100.0°C)
                       (crit low =  +0.0°C, crit = +100.0°C)
                       (lowest = +25.0°C, highest = +29.0°C)

nvme-pci-0100
Composite:    +42.9°C  (low  = -273.1°C, high = +84.8°C)
                       (crit = +84.8°C)
Sensor 1:     +42.9°C  (low  = -273.1°C, high = +65261.8°C)
Sensor 2:     +42.9°C  (low  = -273.1°C, high = +65261.8°C)
```

## System Configuration

The above having covered **installing** software, there are still
system configurations that need to be tweaked.

### APT respositories clean-up

Ubuntu Studio 24.04 seems to consistently need a little
[APT respositories clean-up]({{ site.baseurl }}2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#apt-respositories-clean-up); just comment out the last line
in `/etc/apt/sources.list.d/dvd.list` to let `noble-security` be
defined (only) in `ubuntu.sources`.

### Ubuntu Pro

When updating the system with `apt full-upgrade -y` a notice comes
up about additional security updates:

```
Get more security updates through Ubuntu Pro with 'esm-apps' enabled:
  libdcmtk17t64 libcjson1 libavdevice60 ffmpeg libpostproc57 libavcodec60
  libavutil58 libswscale7 libswresample4 libavformat60 libavfilter9
Learn more about Ubuntu Pro at https://ubuntu.com/pro
```

This being a new system, indeed it's not attached to an Ubuntu Pro
account (the old system was):

```
# pro security-status
3213 packages installed:
     1642 packages from Ubuntu Main/Restricted repository
     1569 packages from Ubuntu Universe/Multiverse repository
     1 package from a third party
     1 package no longer available for download

To get more information about the packages, run
    pro security-status --help
for a list of available options.

This machine is receiving security patching for Ubuntu Main/Restricted
repository until 2029.
This machine is NOT attached to an Ubuntu Pro subscription.

Ubuntu Pro with 'esm-infra' enabled provides security updates for
Main/Restricted packages until 2034.

Ubuntu Pro with 'esm-apps' enabled provides security updates for
Universe/Multiverse packages until 2034. There are 11 pending security updates.

Try Ubuntu Pro with a free personal subscription on up to 5 machines.
Learn more at https://ubuntu.com/pro
```

After creating an Ubuntu account a token is available to use with
`pro attach`:

```
# pro attach ...
Enabling Ubuntu Pro: ESM Apps
Ubuntu Pro: ESM Apps enabled
Enabling Ubuntu Pro: ESM Infra
Ubuntu Pro: ESM Infra enabled
Enabling Livepatch
Livepatch enabled
This machine is now attached to 'Ubuntu Pro - free personal subscription'

SERVICE          ENTITLED  STATUS       DESCRIPTION
anbox-cloud      yes       disabled     Scalable Android in the cloud
esm-apps         yes       enabled      Expanded Security Maintenance for Applications
esm-infra        yes       enabled      Expanded Security Maintenance for Infrastructure
landscape        yes       disabled     Management and administration tool for Ubuntu
livepatch        yes       warning      Current kernel is not covered by livepatch
realtime-kernel* yes       disabled     Ubuntu kernel with PREEMPT_RT patches integrated

 * Service has variants

NOTICES
Operation in progress: pro attach
The current kernel (6.8.0-47-lowlatency, x86_64) is not covered by livepatch.
Covered kernels are listed here: https://ubuntu.com/security/livepatch/docs/kernels
Either switch to a covered kernel or `sudo pro disable livepatch` to dismiss this warning.

For a list of all Ubuntu Pro services and variants, run 'pro status --all'
Enable services with: pro enable <service>

     Account: ponder.stibbons@uu.am
Subscription: Ubuntu Pro - free personal subscription

# pro status --all
SERVICE          ENTITLED  STATUS       DESCRIPTION
anbox-cloud      yes       disabled     Scalable Android in the cloud
cc-eal           yes       n/a          Common Criteria EAL2 Provisioning Packages
esm-apps         yes       enabled      Expanded Security Maintenance for Applications
esm-infra        yes       enabled      Expanded Security Maintenance for Infrastructure
fips             yes       n/a          NIST-certified FIPS crypto packages
fips-preview     yes       n/a          Preview of FIPS crypto packages undergoing certification with NIST
fips-updates     yes       n/a          FIPS compliant crypto packages with stable security updates
landscape        yes       disabled     Management and administration tool for Ubuntu
livepatch        yes       warning      Current kernel is not covered by livepatch
realtime-kernel  yes       disabled     Ubuntu kernel with PREEMPT_RT patches integrated
├ generic        yes       disabled     Generic version of the RT kernel (default)
├ intel-iotg     yes       n/a          RT kernel optimized for Intel IOTG platform
└ raspi          yes       n/a          24.04 Real-time kernel optimised for Raspberry Pi
ros              yes       n/a          Security Updates for the Robot Operating System
ros-updates      yes       n/a          All Updates for the Robot Operating System
usg              yes       n/a          Security compliance and audit tools

NOTICES
The current kernel (6.8.0-47-lowlatency, x86_64) is not covered by livepatch.
Covered kernels are listed here: https://ubuntu.com/security/livepatch/docs/kernels
Either switch to a covered kernel or `sudo pro disable livepatch` to dismiss this warning.

Enable services with: pro enable <service>
```

Now the system can be updated *again* with `apt full-upgrade -y`
to receive those additional security updates:

```
# apt full-upgrade -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following upgrades have been deferred due to phasing:
  python3-distupgrade ubuntu-release-upgrader-core
  ubuntu-release-upgrader-qt
The following packages will be upgraded:
  ffmpeg libavcodec60 libavdevice60 libavfilter9 libavformat60 libavutil58
  libcjson1 libdcmtk17t64 libpostproc57 libswresample4 libswscale7
11 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
11 esm-apps security updates
```

### Fix failed locale settings

Every time `dpkg` runs, `perl` reports failed locale settings:

```
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
        LANGUAGE = "en_US:en",
        LC_ALL = (unset),
        LC_TIME = "en_CH.UTF-8",
        LC_MONETARY = "de_CH.UTF-8",
        LC_ADDRESS = "C.UTF-8",
        LC_TELEPHONE = "C.UTF-8",
        LC_NAME = "C.UTF-8",
        LC_MEASUREMENT = "de_CH.UTF-8",
        LC_IDENTIFICATION = "C.UTF-8",
        LC_NUMERIC = "de_CH.UTF-8",
        LC_PAPER = "C.UTF-8",
        LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_ALL to default locale: No such file or directory
```

To fix this, set `LC_ALL` globally:

```
# echo 'LC_ALL="en_US.UTF-8"' >> /etc/environment
```

Re/generate the desired locales, e.g. at least `en_US.UTF-8`.

```
# dpkg-reconfigure locales
Generating locales (this might take a while)...
  en_US.UTF-8... done
  ...
Generation complete.
```

### Make SDDM Look Good

Ubuntu Studio 24.04 uses 
[Simple Desktop Display Manager (SDDM)](https://wiki.archlinux.org/title/SDDM)
([sddm/sddm](https://github.com/sddm/sddm) in GitHub)
which is quite good looking out of the box, but I like to
customize this for each computer.

For most computers my favorite SDDM theme is
[Breeze-Noir-Dark](https://store.kde.org/p/1361460),
which I like to install system-wide.

```
# unzip -d /usr/share/sddm/themes Breeze-Noir-Dark.zip
```

**Note:** action icons won’t render if the directory name is
changed. If needed, change the directory name in the `iconSource` fields in `Main.qml` to match final directory name
so icons show. This is not the only thing that breaks when
changing the directory name.

Other than installing this theme, all I really change in it
is the background image to use 
[The Rapture [3440x1440]](https://www.flickr.com/photos/douglastofoli/27740699244/).

```
# mv welcome-to-rapture-opportunity-awaits-3440x1440.jpg \
  /usr/share/sddm/themes/Breeze-Noir-Dark/
# cd /usr/share/sddm/themes/Breeze-Noir-Dark/

# vi theme.conf
[General]
type=image
color=#132e43
background=/usr/share/sddm/themes/Breeze-Noir-Dark/welcome-to-rapture-opportunity-awaits-3440x1440.jpg

# vi theme.conf.user
[General]
type=image
background=welcome-to-rapture-opportunity-awaits-3440x1440.jpg
```

**Additionally**, as this is new in Ubuntu 24.04, the theme has to
be selected by adding a `[Theme]` section in the system config
in `/usr/lib/sddm/sddm.conf.d/ubuntustudio.conf`

```ini
[General]
InputMethod=

[Theme]
Current="Breeze-Noir-Dark"
EnableAvatars=True
```

[Reportedly](https://superuser.com/questions/1720931/how-to-configure-sddm-in-kubuntu-22-04-where-is-config-file),
you have to **create** the `/etc/sddm.conf.d` directory to add
the *Local configuration* file that allows setting the theme:

```
# mkdir /etc/sddm.conf.d
# vi /etc/sddm.conf.d/ubuntustudio.conf
```

Besides setting the theme, it is also good to limit the range of
user ids so that only human users show up:

```ini
[Theme]
Current=Breeze-Noir-Dark

[Users]
MaximumUid=1003
MinimumUid=1000
```

It seems no longer necessary to manually add Redshift to one's
desktop session. Previously, it would be necessary to launch
**Autostart** and **Add Application…** to add Redshift.

### Make `dmesg` non-privileged

Since Ubuntu 22.04, `dmesg` has become a privileged operation
by default:

```
$ dmesg
dmesg: read kernel buffer failed: Operation not permitted
```

This is controlled by 

```
# sysctl kernel.dmesg_restrict
kernel.dmesg_restrict = 1
```

To revert this default, and make it permanent
([source](https://archived.forum.manjaro.org/t/why-did-dmesg-become-a-priveleged-operation-suddenly/86468/3)):

```
# echo 'kernel.dmesg_restrict=0' | tee -a /etc/sysctl.d/99-sysctl.conf
```

### Waiting for initial location to become available...

However, redshift in Ubuntu 24.04 seems to be susceptible to
crashing whenever toggleing it on/off or adjusting the color
temperature. *Despite* the setting above to force specific
coordinates manually, it tries to fetch location from an online
service that is not available: a pop-up error tells:

> Redshift has terminated unexpectedly with exit code 0:  
> Waiting for initial location to become available...

The same is seen when running `redshift-qt` in a shell, then trying
to adjust the color temperature:

```
$ redshift-qt
"Solar elevations: day above 3.0, night below -6.0"
"Temperatures: 4500K at day, 3500K at night"
"Brightness: 1.00:0.70"
"Gamma (Daytime): 0.700, 0.700, 0.700"
"Gamma (Night): 0.700, 0.700, 0.700"
"Location: 48.00 N, 8.00 E"
"Color temperature: 6500K"
"Brightness: 1.00"
"Status: Enabled"
"Period: Night"
"Color temperature: 3500K"
"Brightness: 0.70"

"Status: Disabled"
"Period: None"
"Color temperature: 6500K"
"Brightness: 1.00"
"Waiting for initial location to become available..."
```

In the end, it was again necessary to manually add Redshift to the
desktop session. Previously, it would be necessary to launch
**Autostart** and **Add Application…** to add `redshift-qt`.

This *may* be related to the **Update** app failing with
[Cannot Refresh Cache Whilst Offline](https://www.reddit.com/r/Actualfixes/comments/1cek3rg/fix_cockpit_cannot_refresh_cache_whilst_offline/),
apparently becuase 
[Cockpit just 'needs' Network Manager](https://askubuntu.com/a/1336040), but there is a
[Solution](https://cockpit-project.org/faq#error-message-about-being-offline),
involving the creation of a *fake* network interface.

```
root@rapture:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 04:42:1a:97:4e:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global enp5s0
       valid_lft forever preferred_lft forever
    inet 192.168.0.2/24 metric 100 brd 192.168.0.255 scope global dynamic enp5s0
       valid_lft 85940sec preferred_lft 85940sec
    inet6 fe80::642:1aff:fe97:4e47/64 scope link 
       valid_lft forever preferred_lft forever

# nmcli con add type dummy con-name fake ifname fake0 ip4 1.2.3.4/24 gw4 1.2.3.1
Connection 'fake' (f7fe724b-5aa8-4988-b21b-aa9dc96dae1a) successfully added.

# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 04:42:1a:97:4e:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global enp5s0
       valid_lft forever preferred_lft forever
    inet 192.168.0.2/24 metric 100 brd 192.168.0.255 scope global dynamic enp5s0
       valid_lft 85937sec preferred_lft 85937sec
    inet6 fe80::642:1aff:fe97:4e47/64 scope link 
       valid_lft forever preferred_lft forever
3: fake0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ce:56:b8:58:f8:32 brd ff:ff:ff:ff:ff:ff
    inet 1.2.3.4/24 brd 1.2.3.255 scope global noprefixroute fake0
       valid_lft forever preferred_lft forever
```

This doesn't seem to affect anything else's connectivity, but
at least now the **Updates** application no longer fails.
