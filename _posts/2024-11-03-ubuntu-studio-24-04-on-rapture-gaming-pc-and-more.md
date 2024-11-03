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
(`label`), then create
the partitions
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

#### New BtrFS Home

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

### Installation

With the above partitions prepared well in advance, to
[Install Ubuntu Studio 24.04]({{ baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#install-ubuntu-studio-2404)
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
