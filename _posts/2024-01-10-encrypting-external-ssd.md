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
and formatted with `ext4` (no need for `btrfs`).

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
# fdisk -l /dev/sdf
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

## Follow up: Samsung T5 2TB SSD

After the above setup has been working very well on the new 4TB SSD,
it was time to apply the same to the old 2TB SSD. In this case,
there will be only one partition, encrypted with LUKS:

```
# fdisk /dev/sdf

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/sdf: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: Portable SSD T5 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0x9dfc017f

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdf1        2048 3907026112 3907024065  1.8T  7 HPFS/NTFS/exFAT

Command (m for help): l

00 Empty            24 NEC DOS          81 Minix / old Lin  bf Solaris        
01 FAT12            27 Hidden NTFS Win  82 Linux swap / So  c1 DRDOS/sec (FAT-
02 XENIX root       39 Plan 9           83 Linux            c4 DRDOS/sec (FAT-
03 XENIX usr        3c PartitionMagic   84 OS/2 hidden or   c6 DRDOS/sec (FAT-
04 FAT16 <32M       40 Venix 80286      85 Linux extended   c7 Syrinx         
05 Extended         41 PPC PReP Boot    86 NTFS volume set  da Non-FS data    
06 FAT16            42 SFS              87 NTFS volume set  db CP/M / CTOS / .
07 HPFS/NTFS/exFAT  4d QNX4.x           88 Linux plaintext  de Dell Utility   
08 AIX              4e QNX4.x 2nd part  8e Linux LVM        df BootIt         
09 AIX bootable     4f QNX4.x 3rd part  93 Amoeba           e1 DOS access     
0a OS/2 Boot Manag  50 OnTrack DM       94 Amoeba BBT       e3 DOS R/O        
0b W95 FAT32        51 OnTrack DM6 Aux  9f BSD/OS           e4 SpeedStor      
0c W95 FAT32 (LBA)  52 CP/M             a0 IBM Thinkpad hi  ea Linux extended 
0e W95 FAT16 (LBA)  53 OnTrack DM6 Aux  a5 FreeBSD          eb BeOS fs        
0f W95 Ext'd (LBA)  54 OnTrackDM6       a6 OpenBSD          ee GPT            
10 OPUS             55 EZ-Drive         a7 NeXTSTEP         ef EFI (FAT-12/16/
11 Hidden FAT12     56 Golden Bow       a8 Darwin UFS       f0 Linux/PA-RISC b
12 Compaq diagnost  5c Priam Edisk      a9 NetBSD           f1 SpeedStor      
14 Hidden FAT16 <3  61 SpeedStor        ab Darwin boot      f4 SpeedStor      
16 Hidden FAT16     63 GNU HURD or Sys  af HFS / HFS+       f2 DOS secondary  
17 Hidden HPFS/NTF  64 Novell Netware   b7 BSDI fs          fb VMware VMFS    
18 AST SmartSleep   65 Novell Netware   b8 BSDI swap        fc VMware VMKCORE 
1b Hidden W95 FAT3  70 DiskSecure Mult  bb Boot Wizard hid  fd Linux raid auto
1c Hidden W95 FAT3  75 PC/IX            bc Acronis FAT32 L  fe LANstep        
1e Hidden W95 FAT1  80 Old Minix        be Solaris boot     ff BBT            

Aliases:
   linux          - 83
   swap           - 82
   extended       - 05
   uefi           - EF
   raid           - FD
   lvm            - 8E
   linuxex        - 85

Command (m for help): p
Disk /dev/sdf: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: Portable SSD T5 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0x9dfc017f

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdf1        2048 3907026112 3907024065  1.8T  7 HPFS/NTFS/exFAT

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-3907029167, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-3907029167, default 3907029167): 

Created a new partition 1 of type 'Linux' and of size 1.8 TiB.
Partition #1 contains a exfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): p
Disk /dev/sdf: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: Portable SSD T5 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0x9dfc017f

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdf1        2048 3907029167 3907027120  1.8T 83 Linux

Filesystem/RAID signature on partition 1 will be wiped.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

**Carefully** run the above commands on the new disk,
making sure to apply the commands to the correct device, e.g.
**`/dev/sdf1`** (not that `/dev/sdf2` would exist).

```
# cryptsetup --verbose --verify-passphrase luksFormat /dev/sdf1
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
Device luks is still in use.

# umount /dev/mapper/luks
# cryptsetup luksClose luks
Device luks is still in use.
# ls -la /mnt/encrypted
total 8
drwxr-xr-x 2 root root 4096 Nov 27 23:02 .
drwxr-xr-x 3 root root 4096 Nov 27 23:02 ..
# umount /mnt/encrypted
umount: /mnt/encrypted: not mounted.
# cryptsetup luksClose luks
Device luks is still in use.
# cryptsetup luksClose luks
Device luks is still in use.
```

At this point something weird happened, the disk went into a
state were it kept producing `I/O error` messages even though
it was essentially empty and ummounted.

```
[47088.440687] EXT4-fs (dm-0): mounted filesystem with ordered data mode. Opts: errors=remount-ro. Quota mode: none.

[47199.602516] usb 4-4: USB disconnect, device number 7
[47199.616305] sd 10:0:0:0: [sdf] Synchronizing SCSI cache
[47199.627339] blk_update_request: I/O error, dev sdf, sector 356612464 op 0x1:(WRITE) flags 0x800 phys_seg 1 prio class 0
[47199.627348] blk_update_request: I/O error, dev sdf, sector 356608368 op 0x1:(WRITE) flags 0x4000 phys_seg 128 prio class 0
[47199.627352] blk_update_request: I/O error, dev sdf, sector 356609392 op 0x1:(WRITE) flags 0x0 phys_seg 128 prio class 0
[47199.627381] blk_update_request: I/O error, dev sdf, sector 356610416 op 0x1:(WRITE) flags 0x4000 phys_seg 128 prio class 0
[47199.627384] blk_update_request: I/O error, dev sdf, sector 356611440 op 0x1:(WRITE) flags 0x0 phys_seg 128 prio class 0
[47199.737269] sd 10:0:0:0: [sdf] Synchronize Cache(10) failed: Result: hostbyte=DID_ERROR driverbyte=DRIVER_OK
[47202.969163] Aborting journal on device dm-0-8.
[47202.969184] Buffer I/O error on dev dm-0, logical block 243826688, lost sync page write
[47202.969194] JBD2: Error -5 detected when updating journal superblock for dm-0-8.
```

After waiting a little, unplugged the disk and plugged it again:

```
[47222.937094] Buffer I/O error on dev dm-0, logical block 20, lost async page write
[47222.937105] Buffer I/O error on dev dm-0, logical block 21, lost async page write

[47343.977469] EXT4-fs (dm-1): recovery complete
[47343.984397] EXT4-fs (dm-1): mounted filesystem with ordered data mode. Opts: errors=remount-ro. Quota mode: none.
```

A different spur of errors showed up later, although this was not
caused by unplugging the disk:

```
[58660.961828] EXT4-fs error (device dm-0): __ext4_find_entry:1682: inode #2: comm winedevice.exe: reading directory lblock 0
[58660.961872] Buffer I/O error on dev dm-0, logical block 0, lost sync page write
[58660.961882] EXT4-fs (dm-0): I/O error while writing superblock
[58660.961885] EXT4-fs (dm-0): Remounting filesystem read-only
[58660.961904] EXT4-fs error (device dm-0): __ext4_find_entry:1682: inode #2: comm winedevice.exe: reading directory lblock 0
[58660.961924] Buffer I/O error on dev dm-0, logical block 0, lost sync page write
[58660.961930] EXT4-fs (dm-0): I/O error while writing superblock
[58666.912824] EXT4-fs error (device dm-0): __ext4_find_entry:1682: inode #2: comm winedevice.exe: reading directory lblock 0
[58666.912861] Buffer I/O error on dev dm-0, logical block 0, lost sync page write
[58666.912868] EXT4-fs (dm-0): I/O error while writing superblock
[58666.912887] EXT4-fs error (device dm-0): __ext4_find_entry:1682: inode #2: comm winedevice.exe: reading directory lblock 0
[58666.912900] Buffer I/O error on dev dm-0, logical block 0, lost sync page write
[58666.912904] EXT4-fs (dm-0): I/O error while writing superblock
```
