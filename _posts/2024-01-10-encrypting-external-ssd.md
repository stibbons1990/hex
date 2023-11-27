---
title:  "Encrypting external SSD"
date:   2024-01-10 10:01:24 +0200
categories: linux ubuntu encryption veracrypt luks cryptsetup
---

There are least two major approaches to encrypt external
SSD drivers; using LUKS seems to be most convenient when
using the drive only on Linux, while using VeryCrypt is
the one that keeps the disk readable on all major OSes.

*Why not both?*

## Starting point: Samsung T7 Shield 4TB SSD

Brand new, this SSD has 2 partitions:

```
# fdisk -l /dev/sdf
Disk /dev/sdf: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: PSSD T7 Shield  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: gpt
Disk identifier: 5D51395C-647D-4B86-83C1-BE9E1633491C

Device     Start        End    Sectors  Size Type
/dev/sdf1     34      32767      32734   16M Microsoft reserved
/dev/sdf2  32768 7814035455 7814002688  3.6T Microsoft basic data

# mount | grep sdf
/dev/sdf2 on /media/coder/T7 Shield type exfat (rw,nosuid,nodev,relatime,uid=1000,gid=1000,fmask=0022,dmask=0022,iocharset=utf8,errors=remount-ro,uhelper=udisks2)

# df -h | grep sdf
/dev/sdf2       3.7T   35M  3.7T   1% /media/coder/T7 Shield

# ls -l /media/coder/T7\ Shield/
total 31488
-rwxr-xr-x 1 coder coder 23890677 Sep 29  2021  SamsungPortableSSD_Setup_Mac_1.0.pkg
-rwxr-xr-x 1 coder coder  8106168 Sep 29  2021  SamsungPortableSSD_Setup_Win_1.0.exe
-rwxr-xr-x 1 coder coder      118 Dec 27  2019 'Samsung Portable SSD SW for Android.txt'
```

None of the above was going to be useful when using the
disk from Linux systems most of the time, with *perhaps*
the chance of needing to read a few files form others.

## LUKS

Most of the disk will be encrypted with [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup)
an dformatted with `ext4` (no need for `btrfs`).

There is no *real* need for a multi-OS setup, we'll just
create a small 32 GB partition at the start of the disk
*just in case* this ever becomes necessary or desirable:

For the most part, we follow the
**Encrypt USB Data Using cryptsetup**
section of the LinuxHint article to
[Encrypt Data on USB from Linux](https://linuxhint.com/encrypt-data-usb-linux/).

Install `cryptsetup` (if not already installed):

```
# apt install cryptsetup -y
```

The disk comes with 2 partitions but the first one is too
small, so we replace those partitions with new ones so
that the first one is a generous 32 GB:

```
# fdisk /dev/sdf
...
Disk /dev/sdf: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
Disk model: PSSD T7 Shield  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: gpt
Disk identifier: 5D51395C-647D-4B86-83C1-BE9E1633491C

Device        Start        End    Sectors  Size Type
/dev/sdf1      2048   67110911   67108864   32G Microsoft basic data
/dev/sdf2  67110912 7814037134 7746926223  3.6T Linux filesystem
```

Format the 32 GB partition as NTFS (not encrypted):

```
# mkfs -t ntfs /dev/sdf1
Cluster size has been automatically set to 4096 bytes.
Initializing device with zeroes: 100% - Done.
Creating NTFS volume structures.
mkntfs completed successfully. Have a nice day.
```

Encrypt the rest of the disk (will format later):

```
# cryptsetup --verbose --verify-passphrase luksFormat /dev/sdf2

WARNING!
========
This will overwrite data on /dev/sdf2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdf2: 
Verify passphrase: 
Ignoring bogus optimal-io size for data device (33553920 bytes).
Key slot 0 created.
Command successful.
```

```
# sudo cryptsetup luksOpen /dev/sdf2 luks
Enter passphrase for /dev/sdf2: 

# ls -l /dev/mapper/luks 
lrwxrwxrwx 1 root root 7 Nov 27 23:00 /dev/mapper/luks -> ../dm-0

# sudo mkfs.ext4 /dev/mapper/luks
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 968361681 4k blocks and 242098176 inodes
Filesystem UUID: bf17a8ca-56e7-4521-a9ba-2d75895251fd
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done       
```

Open the encrypted disk, mount the file system and setup
permissions for the desired user/s:

```
# sudo mkdir /mnt/encrypted
# sudo mount /dev/mapper/luks /mnt/encrypted
# sudo touch /mnt/encrypted/file1.txt
# sudo chown -R `whoami` /mnt/encrypted
# ls -la /mnt/encrypted
total 24
drwxr-xr-x 3 coder root  4096 Nov 27 23:03 .
drwxr-xr-x 3 root   root  4096 Nov 27 23:02 ..
-rw-r--r-- 1 coder root     0 Nov 27 23:03 file1.txt
drwx------ 2 coder root 16384 Nov 27 23:01 lost+found
```

Once done, umount the filesystem and *close* the
encrypted disk:

```
# sudo umount /dev/mapper/luks
# sudo cryptsetup luksClose luks
```

### Everyday use

After this setup, the disk is recognized as an encrypted
volume when plugged in and KDE Plasma will prompt for
the password (and remember it, if desired). The disk will
be mounted under `/media/coder` but somehow the option to
*Open with File Manager* is not available for it. Still,
the same option can be used in any other disk, and the
File Manager (Dolphin) does show the mounted filesystem
as **3.6 TiB Internal Drive (dm-0)** and is ready to use.

Copying 140 GB of Audiobooks, reading from a Crucial MX
SSD (SATA) that *should* be able to keep up, takes about
8 minutes.

## VeraCrypt

To create an Mac/Windows-friendly encrypted volume,
[download VeraCrypt](https://www.veracrypt.fr/en/Downloads.html)
for Ubuntu 22.04 an install it, then setup up the small
32 GB partition following the
**Encrypt USB Data Using VeraCrypt**
section of the same LinuxHint article to
[Encrypt Data on USB from Linux](https://linuxhint.com/encrypt-data-usb-linux/).
