---
title: TODO
permalink: /todo/
---

This a loose collections of things or ideas to try.

Learning German

A2 playlist
https://www.youtube.com/watch?v=58U2IvsjSrk&list=PLk1fjOl39-5201BUdhtOM_x23poNvLouT&index=125&ab_channel=EasyGerman

https://www.youtube.com/@yourgermanteacher/playlists
https://www.youtube.com/@MrLAntrim/playlists
https://www.youtube.com/@GetGermanized/playlists

Speech <->
https://cloud.google.com/speech-to-text/docs/speech-to-text-requests#synchronous-requests
https://cloud.google.com/text-to-speech/docs/ssml

Protoprims
https://github.com/p80n/photoprism-helm

terry pratchett quotes about coffee?
https://www.google.com/search?q=terry+pratchett+quotes+about+coffee&sca_esv=581440190&tbm=isch&sxsrf=AM9HkKm4ECaW8ImKXuemc0R-SjqSTGkQyQ:1699683531771&source=lnms&sa=X&ved=2ahUKEwix6JDJpruCAxUy1AIHHZkKDOQQ_AUoAXoECAIQAw&biw=1567&bih=911&dpr=1.41


Encrypt portable external disks
- https://opensource.com/article/21/3/encryption-luks
- https://www.jwillikers.com/encrypt-an-external-disk-on-linux
- https://www.wikihow.com/Encrypt-an-External-Hard-Drive-on-Linux
- https://linux.fernandocejas.com/docs/how-to/encrypt-an-external-hard-drive
- https://linuxhint.com/encrypt-data-usb-linux/
- https://superuser.com/questions/1602752/encrypt-an-external-hard-drive-with-readwrite-access-on-both-windows-and-linux
- https://forums.linuxmint.com/viewtopic.php?t=377430
- https://theawesomegarage.com/blog/encrypt-external-hard-drives-with-linux
- https://proton.me/blog/usb-encryption
- https://linux.tips/tutorials/how-to-encrypt-a-usb-drive-on-linux-operating-system


12582  dpkg -l cryptsetup
12583  lsblk
12584  dpkg -l sgdisk
12585  apt show sgdisk
12586  sudo fdisk  /dev/sdf
12587  sudo umount /media/miguev/T7\ Shield 
12588  sudo fdisk  /dev/sdf
12589  sudo fdisk -l /dev/sdf
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


12590  sudo mkfs -t ntfs /dev/sdf1
12591  sudo cryptsetup --verbose --verify-passphrase luksFormat /dev/sdf2
12592  python
12593  sudo cryptsetup --verbose --verify-passphrase luksFormat /dev/sdf2
12594  sudo cryptsetup luksOpen /dev/sdf2 luks
12595  ls -l /dev/mapper/luks 
12596  sudo mkfs.ext4 /dev/mapper/luks
12597  sudo mkdir /mnt/encrypted
12598  sudo mount /dev/mapper/luks /mnt/encrypted
12599  sudo touch /mnt/encrypted/file1.txt
12600  sudo chown -R `whoami` /mnt/encrypted
12601  ls -la /mnt/encrypted
12602  touch /mnt/encrypted/file2.txt
12603  sudo umount /dev/mapper/luks
12604  sudo cryptsetup luksClose luks


15469  time rsync -turv Audiobooks /media/miguev/55AB13A823196DF0/
15470  du -sh /media/miguev/55AB13A823196DF0/Audiobooks/
15471  sudo mkfs -t exfat /dev/sdf1
15472  ls /media/miguev/55AB13A823196DF0/
15473  sudo umount /media/miguev/55AB13A823196DF0/
15474  sudo mkfs -t exfat /dev/sdf1
15475  df |grep sdf
15476  time rsync -turv Audiobooks /media/miguev/E7BF-EE0F/

15765  time rsync -turv Audiobooks /media/miguev/E7BF-EE0F/
15766  sudo ls /media/miguev/E7BF-EE0F
15767  sudo find /media/miguev/E7BF-EE0F
15768  df -h|grep sdf
15769  sudo umount /media/miguev/E7BF-EE0F
15770  sudo mkfs -t vfat /dev/sdf1
15771  #time rsync -turv Audiobooks /media/miguev/10ED-FAC7/
15772  df -h|grep sdf
15773  time rsync -turv Audiobooks /media/miguev/10ED-FAC7/



Mod Skyrim on Steam

`PROTON_USE_WINED3D=1 %command%`

`$(echo %command% | sed -r "s/proton waitforexitandrun .*/proton waitforexitandrun/") "$STEAM_COMPAT_INSTALL_PATH/skse64_loader.exe"`


Kiosk browser + Grafana for physical display on lexicon
- Maybe run X + single-command in .xsession (pi-z2?)
- Can't power display off/on with vbetool dpms because can't disable Secure Boot 
[1](https://access.redhat.com/solutions/6969947);
remounting /dev won't help 
[2](https://superuser.com/questions/1555396/trouble-with-shutting-down-screen-in-ubuntu-server)

[RetroDECK](http://retrodeck.net)

Flameshot (screenshots app)

Automated the boring stuff with Python

Godot Engine? Cocos2d?

Find good, child-friendly article about inclusive language

Try Mass Effect with mods, e.g. MEUITM 2.

[Monigote Fantasy](https://bitbrosgames.itch.io/monigote-fantasy)
