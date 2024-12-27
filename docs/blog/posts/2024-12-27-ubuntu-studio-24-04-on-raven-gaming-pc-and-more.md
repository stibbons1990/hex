---
date: 2024-12-27
categories:
 - linux
 - ubuntu
 - installation
 - setup
title: Ubuntu Studio 24.04 on Raven, Gaming PC (and more)
---

The time has came to update *my other PC*, which I use also for gaming,
coding, media production and just about everything, to 
[Ubuntu Studio 24.04](https://ubuntustudio.org/2024/04/ubuntu-studio-24-04-lts-released/).

This is a smaller PC build based on (maybe describe hardware here)

<!-- more -->

## Preparation

This PC was already prepared to install
[Ubuntu Studio 24.04](https://ubuntustudio.org/2024/04/ubuntu-studio-24-04-lts-released/)
as part of installing 22.04 on a new M.2 SSD a year ago,
with partitions as follows:

1. 300 MB EFI System for the EFI boot.
2. 72 GB Linux filesystem for the root (ext4).
3. 72 GB Linux filesystem for the alternative root (ext4).
4. 1.7 TB Linux filesystem for `/home` (btrfs).

``` console
# parted /dev/nvme0n1 print
Model: Samsung SSD 970 EVO Plus 2TB (nvme)
Disk /dev/nvme0n1: 2000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  316MB   315MB   fat32              boot, esp
 2      316MB   79.0GB  78.6GB  ext4
 3      79.0GB  158GB   78.6GB  ext4
 4      158GB   2000GB  1843GB  btrfs
```

!!! note 

    78 GB has proven to be a reasonable size for the root partition
    for the amount of software that tends to be installed in my PC,
    including 20 GB in `/usr` and 13 GB in `/snap`.

Ubuntu Studio **22.04** is already installed in `/dev/nvme0n1p2` and now
Ubuntu Studio **24.04** will be installed in `/dev/nvme0n1p3`.

## Installation

With the above partitions prepared well in advance, to
[Install Ubuntu Studio 24.04](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#install-ubuntu-studio-2404)
the process *should* be as simple, easy and smooth as it
was with other systems.

And *this time* it was, thanks to the lesson learned when installing
[Ubuntu Studio 24.04 on Super Tuna (NUC PC)](2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc.md#install-ubuntu-studio-2404) to
**Disable screen locking** to prevent
[KDE Plasma live lock screen rendering the session useless](https://launchpad.net/bugs/2062402).
With that in mind, and the USB stick already prepared with
`usb-creator-kde`, booted  into it and used “Install Ubuntu” launcher
on the desktop after disabling screen locking.

1.  Plug the USB stick and turn the PC on. This PC already defaults to
    boot from USB when present.
1.  In the Grub menu, choose to **Try or Install Ubuntu**.
1.  Select language (English) and then **Install Ubuntu**.
1.  **Connect to a Wi-Fi network** because there is no LAN here.
1.  Select keyboard layout (can be from a different language).
1.  Select **Try Ubuntu Studio**.
1.  **Disable screen locking** to prevent
    [KDE Plasma live lock screen rendering the session useless](https://launchpad.net/bugs/2062402):
    *   Press `Alt+Space` to invoke Krunner and type `System Settings`.
    *   From there, search for **Screen Locking** and
    *   deactivate **Lock automatically after…**.
1.  Launch **Install Ubuntu Studio** from the desktop.
1.  Select Type of install: **Interactive Installation**.
1.  Enable the options to
    *   **Install third-party software for graphics and Wifi hardware** and
    *   **Download and install support for additional media formats**.
1.  Select **Manual Installation**
    *  Use the arrow keys to navigate down to the **nvme0n1** disk.
    *  Set **nvme0n1p1** (300 MB) as **EFI System Partition** mounted
       on `/boot/efi`
    *  Leave **nvme0n1p2** (78 GB) alone (here lives Ubuntu 22.04)
    *  Set **nvme0n1p3** (78 GB) as **ext4** mounted on `/`
    *  Set **nvme0n1p4** (1.7 TB) as **Leave formatted as Btrfs**
       mounted on `/home`
    *  Set **Device for boot loader installation** to **nvme0n1**
1.  Click on **Next** to confirm the partition selection.
1.  Confirm first non-root user name (`coder`) and computer
    name (`raven`).
1.  Select time zone (seems to be detected correctly).
1.  Review the choices and click on **Install** to start copying files.
1.  Once it's done, select **Restart** and, once prompted later
    remove the USB stick and hit `Enter`.

### First boot into Ubuntu Studio 24.04

The first time booting into the new system, right after login for
the first time an additional reboot is required for the
[Ubuntu Studio Audio Configuration](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#ubuntu-studio-audio-configuration).

### Second boot into Ubuntu Studio 24.04

#### Mount Ubuntu Studio 22.04 root

To have easy access to files from the old Ubuntu Studio 22.04 root
partition, mount it on `/jammy` and add that to `/etc/fstab`

``` console
# ls -l /dev/disk/by-uuid/ | grep nvme0n1p2
lrwxrwxrwx 1 root root 15 Dec 27 10:44 ffeabdbb-ac78-46db-8cf8-1ef6efe7db88 -> ../../nvme0n1p2

# vi /etc/fstab
...
/dev/disk/by-uuid/ffeabdbb-ac78-46db-8cf8-1ef6efe7db88 /jammy ext4 defaults 0 1
...

# systemctl daemon-reload
# mount /jammy
# df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/nvme0n1p2   72G   32G   37G  47% /jammy
```

#### Tweak Grub

Make sure Grub will show the menu and wait a bit, by tweaking
`/etc/default/grub` as follows (and updating Grub):

``` ini linenums="6" hl_lines="7 8 10" title="/etc/default/grub"
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR=`( . /etc/os-release; echo ${NAME:-Ubuntu} ) 2>/dev/null || echo Ubuntu`
GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash"
GRUB_CMDLINE_LINUX=""
```

``` console
# update-grub2
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/ubuntustudio.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-50-lowlatency
Found initrd image: /boot/initrd.img-6.8.0-50-lowlatency
Found memtest86+ 64bit EFI image: /boot/memtest86+x64.efi
Warning: os-prober will be executed to detect other bootable partitions.
Its output will be used to detect bootable binaries on them and create new boot entries.
Found Ubuntu 22.04.5 LTS (22.04) on /dev/nvme0n1p2
Adding boot menu entry for UEFI Firmware Settings ...
done
```

??? note "The latest NVidia driver is already installed."

    Ubuntu Studio 24.04 already installed the latest NVidia driver so
    there is no need to reboot just yet.

    ``` console
    # nvidia-smi 
    Fri Dec 27 11:43:33 2024       
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 550.120                Driver Version: 550.120        CUDA Version: 12.4     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  NVIDIA GeForce GTX 1660 Ti     Off |   00000000:29:00.0  On |                  N/A |
    | 25%   27C    P8              7W /  120W |     503MiB /   6144MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
    ```

Since a reboot is not really necessary just yet, start by updating the
system and installing essential packages.

#### APT respositories clean-up

Ubuntu Studio 24.04 seems to consistently need a little
[APT respositories clean-up](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#apt-respositories-clean-up); just comment out the last line
in `/etc/apt/sources.list.d/dvd.list` to let `noble-security` be
defined (only) in `ubuntu.sources`.

#### Update Installed Packages

Updates will be available after installing from the USB installer.

??? terminal "`# apt update && apt full-upgrade -y`"

    ``` console
    # apt update && apt full-upgrade -y
    Hit:1 http://es.archive.ubuntu.com/ubuntu noble InRelease
    Hit:2 http://security.ubuntu.com/ubuntu noble-security InRelease
    Hit:3 http://es.archive.ubuntu.com/ubuntu noble-updates InRelease
    Hit:4 http://es.archive.ubuntu.com/ubuntu noble-backports InRelease
    Hit:5 http://archive.ubuntu.com/ubuntu noble InRelease
    Hit:6 https://dl.google.com/linux/chrome/deb stable InRelease
    Hit:7 http://archive.ubuntu.com/ubuntu noble-updates InRelease
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    105 packages can be upgraded. Run 'apt list --upgradable' to see them.
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Calculating upgrade... Done
    Get more security updates through Ubuntu Pro with 'esm-apps' enabled:
      libdcmtk17t64 python3-waitress libcjson1 libavdevice60 ffmpeg libpostproc57
      libavcodec60 libavutil58 libswscale7 libswresample4 libavformat60
      libavfilter9
    Learn more about Ubuntu Pro at https://ubuntu.com/pro
    The following upgrades have been deferred due to phasing:
      python3-distupgrade ubuntu-release-upgrader-core ubuntu-release-upgrader-qt
    The following packages will be upgraded:
      acl alsa-ucm-conf apparmor apport apport-core-dump-handler apport-kde cloud-init
      cryptsetup cryptsetup-bin cryptsetup-initramfs digikam digikam-data digikam-private-libs
      distro-info-data dmidecode dmsetup fwupd gir1.2-gtk-3.0 gir1.2-packagekitglib-1.0
      gir1.2-udisks-2.0 gtk-update-icon-cache initramfs-tools initramfs-tools-bin
      initramfs-tools-core krb5-locales libacl1 libapparmor1 libastro1 libaudit-common
      libaudit1 libboost-chrono1.83.0t64 libboost-filesystem1.83.0 libboost-iostreams1.83.0
      libboost-locale1.83.0 libboost-program-options1.83.0 libboost-thread1.83.0
      libcryptsetup12 libdevmapper1.02.1 libegl-mesa0 libegl1-mesa-dev libfwupd2 libgbm1
      libgl1-mesa-dri libglapi-mesa libglx-mesa0 libgssapi-krb5-2 libgtk-3-0t64 libgtk-3-bin
      libgtk-3-common libgtk-3-dev libk5crypto3 libkrb5-3 libkrb5support0 libldap-common
      libldap2 libmarblewidget-qt5-28 libnm0 libpackagekit-glib2-18 libpipewire-0.3-0t64
      libpipewire-0.3-common libpipewire-0.3-modules libproc2-0 libspa-0.2-bluetooth
      libspa-0.2-modules libspeex1 libudisks2-0 libxatracker2 login lp-solve marble-plugins
      marble-qt-data mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers mtr-tiny
      network-manager packagekit packagekit-tools passwd pipewire pipewire-alsa pipewire-audio
      pipewire-bin pipewire-jack pipewire-pulse plasma-distro-release-notifier procps
      python3-apport python3-problem-report python3-software-properties python3-update-manager
      snapd software-properties-common software-properties-qt systemd-hwe-hwdb
      ubuntu-drivers-common ubuntu-pro-client ubuntu-pro-client-l10n udisks2
      update-manager-core xdg-desktop-portal zip
    102 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
    Need to get 0 B/133 MB of archives.
    After this operation, 5,865 kB of additional disk space will be used.
    Extracting templates from packages: 100%
    Preconfiguring packages ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../login_1%3a4.13+dfsg1-4ubuntu3.2_amd64.deb ...
    Unpacking login (1:4.13+dfsg1-4ubuntu3.2) over (1:4.13+dfsg1-4ubuntu3) ...
    Setting up login (1:4.13+dfsg1-4ubuntu3.2) ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../0-python3-problem-report_2.28.1-0ubuntu3.3_all.deb ...
    Unpacking python3-problem-report (2.28.1-0ubuntu3.3) over (2.28.1-0ubuntu3.1) ...
    Preparing to unpack .../1-python3-apport_2.28.1-0ubuntu3.3_all.deb ...
    Unpacking python3-apport (2.28.1-0ubuntu3.3) over (2.28.1-0ubuntu3.1) ...
    Preparing to unpack .../2-apport-core-dump-handler_2.28.1-0ubuntu3.3_all.deb ...
    Unpacking apport-core-dump-handler (2.28.1-0ubuntu3.3) over (2.28.1-0ubuntu3.1) ...
    Preparing to unpack .../3-apport_2.28.1-0ubuntu3.3_all.deb ...
    Unpacking apport (2.28.1-0ubuntu3.3) over (2.28.1-0ubuntu3.1) ...
    Preparing to unpack .../4-ubuntu-drivers-common_1%3a0.9.7.6ubuntu3.1_amd64.deb ...
    Unpacking ubuntu-drivers-common (1:0.9.7.6ubuntu3.1) over (1:0.9.7.6ubuntu3) ...
    Preparing to unpack .../5-acl_2.3.2-1build1.1_amd64.deb ...
    Unpacking acl (2.3.2-1build1.1) over (2.3.2-1build1) ...
    Preparing to unpack .../6-libacl1_2.3.2-1build1.1_amd64.deb ...
    Unpacking libacl1:amd64 (2.3.2-1build1.1) over (2.3.2-1build1) ...
    Setting up libacl1:amd64 (2.3.2-1build1.1) ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../libaudit-common_1%3a3.1.2-2.1build1.1_all.deb ...
    Unpacking libaudit-common (1:3.1.2-2.1build1.1) over (1:3.1.2-2.1build1) ...
    Setting up libaudit-common (1:3.1.2-2.1build1.1) ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../libaudit1_1%3a3.1.2-2.1build1.1_amd64.deb ...
    Unpacking libaudit1:amd64 (1:3.1.2-2.1build1.1) over (1:3.1.2-2.1build1) ...
    Setting up libaudit1:amd64 (1:3.1.2-2.1build1.1) ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../passwd_1%3a4.13+dfsg1-4ubuntu3.2_amd64.deb ...
    Unpacking passwd (1:4.13+dfsg1-4ubuntu3.2) over (1:4.13+dfsg1-4ubuntu3) ...
    Setting up passwd (1:4.13+dfsg1-4ubuntu3.2) ...
    (Reading database ... 417845 files and directories currently installed.)
    Preparing to unpack .../00-libproc2-0_2%3a4.0.4-4ubuntu3.2_amd64.deb ...
    Unpacking libproc2-0:amd64 (2:4.0.4-4ubuntu3.2) over (2:4.0.4-4ubuntu3) ...
    Preparing to unpack .../01-procps_2%3a4.0.4-4ubuntu3.2_amd64.deb ...
    Unpacking procps (2:4.0.4-4ubuntu3.2) over (2:4.0.4-4ubuntu3) ...
    Preparing to unpack .../02-distro-info-data_0.60ubuntu0.2_all.deb ...
    Unpacking distro-info-data (0.60ubuntu0.2) over (0.60ubuntu0.1) ...
    Preparing to unpack .../03-libdevmapper1.02.1_2%3a1.02.185-3ubuntu3.1_amd64.deb ...
    Unpacking libdevmapper1.02.1:amd64 (2:1.02.185-3ubuntu3.1) over (2:1.02.185-3ubuntu3) ...
    Preparing to unpack .../04-dmsetup_2%3a1.02.185-3ubuntu3.1_amd64.deb ...
    Unpacking dmsetup (2:1.02.185-3ubuntu3.1) over (2:1.02.185-3ubuntu3) ...
    Preparing to unpack .../05-krb5-locales_1.20.1-6ubuntu2.2_all.deb ...
    Unpacking krb5-locales (1.20.1-6ubuntu2.2) over (1.20.1-6ubuntu2.1) ...
    Preparing to unpack .../06-libapparmor1_4.0.1really4.0.1-0ubuntu0.24.04.3_amd64.deb ...
    Unpacking libapparmor1:amd64 (4.0.1really4.0.1-0ubuntu0.24.04.3) over (4.0.1really4.0.0-beta3-0ubuntu0.1) ...
    Preparing to unpack .../07-libcryptsetup12_2%3a2.7.0-1ubuntu4.1_amd64.deb ...
    Unpacking libcryptsetup12:amd64 (2:2.7.0-1ubuntu4.1) over (2:2.7.0-1ubuntu4) ...
    Preparing to unpack .../08-libgssapi-krb5-2_1.20.1-6ubuntu2.2_amd64.deb ...
    Unpacking libgssapi-krb5-2:amd64 (1.20.1-6ubuntu2.2) over (1.20.1-6ubuntu2.1) ...
    Preparing to unpack .../09-libkrb5-3_1.20.1-6ubuntu2.2_amd64.deb ...
    Unpacking libkrb5-3:amd64 (1.20.1-6ubuntu2.2) over (1.20.1-6ubuntu2.1) ...
    Preparing to unpack .../10-libkrb5support0_1.20.1-6ubuntu2.2_amd64.deb ...
    Unpacking libkrb5support0:amd64 (1.20.1-6ubuntu2.2) over (1.20.1-6ubuntu2.1) ...
    Preparing to unpack .../11-libk5crypto3_1.20.1-6ubuntu2.2_amd64.deb ...
    Unpacking libk5crypto3:amd64 (1.20.1-6ubuntu2.2) over (1.20.1-6ubuntu2.1) ...
    Preparing to unpack .../12-systemd-hwe-hwdb_255.1.4_all.deb ...
    Unpacking systemd-hwe-hwdb (255.1.4) over (255.1.3) ...
    Preparing to unpack .../13-ubuntu-pro-client-l10n_34~24.04_amd64.deb ...
    Unpacking ubuntu-pro-client-l10n (34~24.04) over (32.3.1~24.04) ...
    Preparing to unpack .../14-ubuntu-pro-client_34~24.04_amd64.deb ...
    Unpacking ubuntu-pro-client (34~24.04) over (32.3.1~24.04) ...
    Preparing to unpack .../15-apparmor_4.0.1really4.0.1-0ubuntu0.24.04.3_amd64.deb ...
    Unpacking apparmor (4.0.1really4.0.1-0ubuntu0.24.04.3) over (4.0.1really4.0.0-beta3-0ubuntu0.1) ...
    Preparing to unpack .../16-dmidecode_3.5-3ubuntu0.1_amd64.deb ...
    Unpacking dmidecode (3.5-3ubuntu0.1) over (3.5-3build1) ...
    Preparing to unpack .../17-mtr-tiny_0.95-1.1ubuntu0.1_amd64.deb ...
    Unpacking mtr-tiny (0.95-1.1ubuntu0.1) over (0.95-1.1build2) ...
    Preparing to unpack .../18-python3-update-manager_1%3a24.04.9_all.deb ...
    Unpacking python3-update-manager (1:24.04.9) over (1:24.04.6) ...
    Preparing to unpack .../19-update-manager-core_1%3a24.04.9_all.deb ...
    Unpacking update-manager-core (1:24.04.9) over (1:24.04.6) ...
    Preparing to unpack .../20-alsa-ucm-conf_1.2.10-1ubuntu5.3_all.deb ...
    Unpacking alsa-ucm-conf (1.2.10-1ubuntu5.3) over (1.2.10-1ubuntu5) ...
    Preparing to unpack .../21-apport-kde_2.28.1-0ubuntu3.3_all.deb ...
    Unpacking apport-kde (2.28.1-0ubuntu3.3) over (2.28.1-0ubuntu3.1) ...
    Preparing to unpack .../22-initramfs-tools_0.142ubuntu25.4_all.deb ...
    Unpacking initramfs-tools (0.142ubuntu25.4) over (0.142ubuntu25.1) ...
    Preparing to unpack .../23-initramfs-tools-core_0.142ubuntu25.4_all.deb ...
    Unpacking initramfs-tools-core (0.142ubuntu25.4) over (0.142ubuntu25.1) ...
    Preparing to unpack .../24-initramfs-tools-bin_0.142ubuntu25.4_amd64.deb ...
    Unpacking initramfs-tools-bin (0.142ubuntu25.4) over (0.142ubuntu25.1) ...
    Preparing to unpack .../25-cryptsetup-initramfs_2%3a2.7.0-1ubuntu4.1_all.deb ...
    Unpacking cryptsetup-initramfs (2:2.7.0-1ubuntu4.1) over (2:2.7.0-1ubuntu4) ...
    Preparing to unpack .../26-cryptsetup-bin_2%3a2.7.0-1ubuntu4.1_amd64.deb ...
    Unpacking cryptsetup-bin (2:2.7.0-1ubuntu4.1) over (2:2.7.0-1ubuntu4) ...
    Preparing to unpack .../27-cryptsetup_2%3a2.7.0-1ubuntu4.1_amd64.deb ...
    Unpacking cryptsetup (2:2.7.0-1ubuntu4.1) over (2:2.7.0-1ubuntu4) ...
    Preparing to unpack .../28-marble-plugins_4%3a23.08.5-0ubuntu3.2_amd64.deb ...
    Unpacking marble-plugins:amd64 (4:23.08.5-0ubuntu3.2) over (4:23.08.5-0ubuntu3) ...
    Preparing to unpack .../29-libmarblewidget-qt5-28_4%3a23.08.5-0ubuntu3.2_amd64.deb ...
    Unpacking libmarblewidget-qt5-28:amd64 (4:23.08.5-0ubuntu3.2) over (4:23.08.5-0ubuntu3) ...
    Preparing to unpack .../30-libastro1_4%3a23.08.5-0ubuntu3.2_amd64.deb ...
    Unpacking libastro1:amd64 (4:23.08.5-0ubuntu3.2) over (4:23.08.5-0ubuntu3) ...
    Preparing to unpack .../31-marble-qt-data_4%3a23.08.5-0ubuntu3.2_all.deb ...
    Unpacking marble-qt-data (4:23.08.5-0ubuntu3.2) over (4:23.08.5-0ubuntu3) ...
    Preparing to unpack .../32-digikam_4%3a8.2.0-0ubuntu6.2_amd64.deb ...
    Unpacking digikam (4:8.2.0-0ubuntu6.2) over (4:8.2.0-0ubuntu6) ...
    Preparing to unpack .../33-digikam-private-libs_4%3a8.2.0-0ubuntu6.2_amd64.deb ...
    Unpacking digikam-private-libs (4:8.2.0-0ubuntu6.2) over (4:8.2.0-0ubuntu6) ...
    Preparing to unpack .../34-digikam-data_4%3a8.2.0-0ubuntu6.2_all.deb ...
    Unpacking digikam-data (4:8.2.0-0ubuntu6.2) over (4:8.2.0-0ubuntu6) ...
    Preparing to unpack .../35-libfwupd2_1.9.27-0ubuntu1~24.04.1_amd64.deb ...
    Unpacking libfwupd2:amd64 (1.9.27-0ubuntu1~24.04.1) over (1.9.16-1) ...
    Preparing to unpack .../36-fwupd_1.9.27-0ubuntu1~24.04.1_amd64.deb ...
    Unpacking fwupd (1.9.27-0ubuntu1~24.04.1) over (1.9.16-1) ...
    Preparing to unpack .../37-libgtk-3-common_3.24.41-4ubuntu1.2_all.deb ...
    Unpacking libgtk-3-common (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../38-libegl1-mesa-dev_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libegl1-mesa-dev:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../39-libgtk-3-dev_3.24.41-4ubuntu1.2_amd64.deb ...
    Unpacking libgtk-3-dev:amd64 (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../40-libgtk-3-0t64_3.24.41-4ubuntu1.2_amd64.deb ...
    Unpacking libgtk-3-0t64:amd64 (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../41-gir1.2-gtk-3.0_3.24.41-4ubuntu1.2_amd64.deb ...
    Unpacking gir1.2-gtk-3.0:amd64 (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../42-libpackagekit-glib2-18_1.2.8-2ubuntu1.1_amd64.deb ...
    Unpacking libpackagekit-glib2-18:amd64 (1.2.8-2ubuntu1.1) over (1.2.8-2build3) ...
    Preparing to unpack .../43-gir1.2-packagekitglib-1.0_1.2.8-2ubuntu1.1_amd64.deb ...
    Unpacking gir1.2-packagekitglib-1.0 (1.2.8-2ubuntu1.1) over (1.2.8-2build3) ...
    Preparing to unpack .../44-udisks2_2.10.1-6ubuntu1_amd64.deb ...
    Unpacking udisks2 (2.10.1-6ubuntu1) over (2.10.1-6build1) ...
    Preparing to unpack .../45-libudisks2-0_2.10.1-6ubuntu1_amd64.deb ...
    Unpacking libudisks2-0:amd64 (2.10.1-6ubuntu1) over (2.10.1-6build1) ...
    Preparing to unpack .../46-gir1.2-udisks-2.0_2.10.1-6ubuntu1_amd64.deb ...
    Unpacking gir1.2-udisks-2.0:amd64 (2.10.1-6ubuntu1) over (2.10.1-6build1) ...
    Preparing to unpack .../47-gtk-update-icon-cache_3.24.41-4ubuntu1.2_amd64.deb ...
    Unpacking gtk-update-icon-cache (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../48-libboost-chrono1.83.0t64_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-chrono1.83.0t64:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../49-libboost-filesystem1.83.0_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-filesystem1.83.0:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../50-libboost-iostreams1.83.0_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-iostreams1.83.0:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../51-libboost-thread1.83.0_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-thread1.83.0:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../52-libboost-locale1.83.0_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-locale1.83.0:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../53-libboost-program-options1.83.0_1.83.0-2.1ubuntu3.1_amd64.deb ...
    Unpacking libboost-program-options1.83.0:amd64 (1.83.0-2.1ubuntu3.1) over (1.83.0-2.1ubuntu3) ...
    Preparing to unpack .../54-libegl-mesa0_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libegl-mesa0:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../55-libgbm1_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libgbm1:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../56-libgl1-mesa-dri_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libgl1-mesa-dri:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../57-libglx-mesa0_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libglx-mesa0:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../58-libglapi-mesa_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libglapi-mesa:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../59-libgtk-3-bin_3.24.41-4ubuntu1.2_amd64.deb ...
    Unpacking libgtk-3-bin (3.24.41-4ubuntu1.2) over (3.24.41-4ubuntu1.1) ...
    Preparing to unpack .../60-libldap-common_2.6.7+dfsg-1~exp1ubuntu8.1_all.deb ...
    Unpacking libldap-common (2.6.7+dfsg-1~exp1ubuntu8.1) over (2.6.7+dfsg-1~exp1ubuntu8) ...
    Preparing to unpack .../61-libldap2_2.6.7+dfsg-1~exp1ubuntu8.1_amd64.deb ...
    Unpacking libldap2:amd64 (2.6.7+dfsg-1~exp1ubuntu8.1) over (2.6.7+dfsg-1~exp1ubuntu8) ...
    Preparing to unpack .../62-network-manager_1.46.0-1ubuntu2.2_amd64.deb ...
    Unpacking network-manager (1.46.0-1ubuntu2.2) over (1.46.0-1ubuntu2) ...
    Preparing to unpack .../63-libnm0_1.46.0-1ubuntu2.2_amd64.deb ...
    Unpacking libnm0:amd64 (1.46.0-1ubuntu2.2) over (1.46.0-1ubuntu2) ...
    Preparing to unpack .../64-pipewire-pulse_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking pipewire-pulse (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../65-pipewire-jack_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking pipewire-jack:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../66-pipewire-alsa_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking pipewire-alsa:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../67-libpipewire-0.3-modules_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking libpipewire-0.3-modules:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../68-libpipewire-0.3-0t64_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking libpipewire-0.3-0t64:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../69-libspa-0.2-bluetooth_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking libspa-0.2-bluetooth:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../70-libspa-0.2-modules_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking libspa-0.2-modules:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../71-pipewire_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking pipewire:amd64 (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../72-pipewire-bin_1.0.5-1ubuntu2_amd64.deb ...
    Unpacking pipewire-bin (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../73-libpipewire-0.3-common_1.0.5-1ubuntu2_all.deb ...
    Unpacking libpipewire-0.3-common (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../74-libspeex1_1.2.1-2ubuntu2.24.04.1_amd64.deb ...
    Unpacking libspeex1:amd64 (1.2.1-2ubuntu2.24.04.1) over (1.2.1-2ubuntu2) ...
    Preparing to unpack .../75-libxatracker2_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libxatracker2:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../76-lp-solve_5.5.2.5-2ubuntu0.1_amd64.deb ...
    Unpacking lp-solve (5.5.2.5-2ubuntu0.1) over (5.5.2.5-2build4) ...
    Preparing to unpack .../77-mesa-va-drivers_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking mesa-va-drivers:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../78-mesa-vdpau-drivers_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking mesa-vdpau-drivers:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../79-mesa-vulkan-drivers_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking mesa-vulkan-drivers:amd64 (24.0.9-0ubuntu0.3) over (24.0.9-0ubuntu0.1) ...
    Preparing to unpack .../80-packagekit-tools_1.2.8-2ubuntu1.1_amd64.deb ...
    Unpacking packagekit-tools (1.2.8-2ubuntu1.1) over (1.2.8-2build3) ...
    Preparing to unpack .../81-packagekit_1.2.8-2ubuntu1.1_amd64.deb ...
    Unpacking packagekit (1.2.8-2ubuntu1.1) over (1.2.8-2build3) ...
    Preparing to unpack .../82-pipewire-audio_1.0.5-1ubuntu2_all.deb ...
    Unpacking pipewire-audio (1.0.5-1ubuntu2) over (1.0.5-1ubuntu1) ...
    Preparing to unpack .../83-plasma-distro-release-notifier_20220915-0ubuntu6.1_amd64.deb ...
    Unpacking plasma-distro-release-notifier (20220915-0ubuntu6.1) over (20220915-0ubuntu6) ...
    Preparing to unpack .../84-software-properties-common_0.99.49.1_all.deb ...
    Unpacking software-properties-common (0.99.49.1) over (0.99.48) ...
    Preparing to unpack .../85-software-properties-qt_0.99.49.1_all.deb ...
    Unpacking software-properties-qt (0.99.49.1) over (0.99.48.1) ...
    Preparing to unpack .../86-python3-software-properties_0.99.49.1_all.deb ...
    Unpacking python3-software-properties (0.99.49.1) over (0.99.48) ...
    Preparing to unpack .../87-snapd_2.66.1+24.04_amd64.deb ...
    Unpacking snapd (2.66.1+24.04) over (2.63.1+24.04) ...
    Preparing to unpack .../88-xdg-desktop-portal_1.18.4-1ubuntu2.24.04.1_amd64.deb ...
    Unpacking xdg-desktop-portal (1.18.4-1ubuntu2.24.04.1) over (1.18.4-1ubuntu2) ...
    Preparing to unpack .../89-zip_3.0-13ubuntu0.1_amd64.deb ...
    Unpacking zip (3.0-13ubuntu0.1) over (3.0-13build1) ...
    Preparing to unpack .../90-cloud-init_24.4-0ubuntu1~24.04.2_all.deb ...
    Unpacking cloud-init (24.4-0ubuntu1~24.04.2) over (24.1.3-0ubuntu3.3) ...
    dpkg: warning: unable to delete old directory '/etc/systemd/system/sshd-keygen@.service.d': Directory not empty
    Setting up libpipewire-0.3-common (1.0.5-1ubuntu2) ...
    Setting up marble-qt-data (4:23.08.5-0ubuntu3.2) ...
    Setting up libboost-program-options1.83.0:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up mesa-vulkan-drivers:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up mesa-vdpau-drivers:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up gtk-update-icon-cache (3.24.41-4ubuntu1.2) ...
    Setting up libapparmor1:amd64 (4.0.1really4.0.1-0ubuntu0.24.04.3) ...
    Setting up libspeex1:amd64 (1.2.1-2ubuntu2.24.04.1) ...
    Setting up ubuntu-drivers-common (1:0.9.7.6ubuntu3.1) ...
    Setting up libgbm1:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up alsa-ucm-conf (1.2.10-1ubuntu5.3) ...
    Setting up python3-problem-report (2.28.1-0ubuntu3.3) ...
    Setting up distro-info-data (0.60ubuntu0.2) ...
    Setting up libfwupd2:amd64 (1.9.27-0ubuntu1~24.04.1) ...
    Setting up lp-solve (5.5.2.5-2ubuntu0.1) ...
    Setting up libboost-thread1.83.0:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up libpackagekit-glib2-18:amd64 (1.2.8-2ubuntu1.1) ...
    Setting up krb5-locales (1.20.1-6ubuntu2.2) ...
    Setting up libboost-filesystem1.83.0:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up libldap-common (2.6.7+dfsg-1~exp1ubuntu8.1) ...
    Setting up libxatracker2:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up python3-apport (2.28.1-0ubuntu3.3) ...
    Setting up acl (2.3.2-1build1.1) ...
    Setting up libkrb5support0:amd64 (1.20.1-6ubuntu2.2) ...
    Setting up apparmor (4.0.1really4.0.1-0ubuntu0.24.04.3) ...
    Installing new version of config file /etc/apparmor.d/abstractions/authentication ...
    Installing new version of config file /etc/apparmor.d/abstractions/samba ...
    Installing new version of config file /etc/apparmor.d/firefox ...
    Reloading AppArmor profiles 
    Setting up gir1.2-packagekitglib-1.0 (1.2.8-2ubuntu1.1) ...
    Setting up zip (3.0-13ubuntu0.1) ...
    Setting up python3-software-properties (0.99.49.1) ...
    Setting up mtr-tiny (0.95-1.1ubuntu0.1) ...
    Setting up libboost-chrono1.83.0t64:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up libspa-0.2-modules:amd64 (1.0.5-1ubuntu2) ...
    Setting up libboost-iostreams1.83.0:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up libastro1:amd64 (4:23.08.5-0ubuntu3.2) ...
    Setting up libproc2-0:amd64 (2:4.0.4-4ubuntu3.2) ...
    Setting up libk5crypto3:amd64 (1.20.1-6ubuntu2.2) ...
    Setting up libglapi-mesa:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up libnm0:amd64 (1.46.0-1ubuntu2.2) ...
    Setting up systemd-hwe-hwdb (255.1.4) ...
    Setting up libdevmapper1.02.1:amd64 (2:1.02.185-3ubuntu3.1) ...
    Setting up dmsetup (2:1.02.185-3ubuntu3.1) ...
    Setting up python3-update-manager (1:24.04.9) ...
    Setting up procps (2:4.0.4-4ubuntu3.2) ...
    Setting up libspa-0.2-bluetooth:amd64 (1.0.5-1ubuntu2) ...
    Setting up libcryptsetup12:amd64 (2:2.7.0-1ubuntu4.1) ...
    Setting up packagekit (1.2.8-2ubuntu1.1) ...
    Setting up digikam-data (4:8.2.0-0ubuntu6.2) ...
    Setting up libkrb5-3:amd64 (1.20.1-6ubuntu2.2) ...
    Setting up mesa-va-drivers:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up dmidecode (3.5-3ubuntu0.1) ...
    Setting up libpipewire-0.3-0t64:amd64 (1.0.5-1ubuntu2) ...
    Setting up ubuntu-pro-client (34~24.04) ...
    Installing new version of config file /etc/apparmor.d/ubuntu_pro_apt_news ...
    Installing new version of config file /etc/apt/apt.conf.d/20apt-esm-hook.conf ...
    Setting up libldap2:amd64 (2.6.7+dfsg-1~exp1ubuntu8.1) ...
    Setting up fwupd (1.9.27-0ubuntu1~24.04.1) ...
    fwupd-offline-update.service is a disabled or a static unit not running, not starting it.
    fwupd-refresh.service is a disabled or a static unit not running, not starting it.
    fwupd.service is a disabled or a static unit not running, not starting it.
    Setting up libgtk-3-common (3.24.41-4ubuntu1.2) ...
    Setting up libudisks2-0:amd64 (2.10.1-6ubuntu1) ...
    Setting up initramfs-tools-bin (0.142ubuntu25.4) ...
    Setting up libegl1-mesa-dev:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up libmarblewidget-qt5-28:amd64 (4:23.08.5-0ubuntu3.2) ...
    Setting up cryptsetup-bin (2:2.7.0-1ubuntu4.1) ...
    Setting up ubuntu-pro-client-l10n (34~24.04) ...
    Setting up snapd (2.66.1+24.04) ...
    Installing new version of config file /etc/apparmor.d/usr.lib.snapd.snap-confine.real ...
    snapd.failure.service is a disabled or a static unit not running, not starting it.
    snapd.snap-repair.service is a disabled or a static unit not running, not starting it.
    Setting up packagekit-tools (1.2.8-2ubuntu1.1) ...
    Setting up udisks2 (2.10.1-6ubuntu1) ...
    Setting up xdg-desktop-portal (1.18.4-1ubuntu2.24.04.1) ...
    Setting up cloud-init (24.4-0ubuntu1~24.04.2) ...
    Installing new version of config file /etc/cloud/cloud.cfg ...
    Installing new version of config file /etc/cloud/templates/sources.list.ubuntu.deb822.tmpl ...
    Setting up cryptsetup (2:2.7.0-1ubuntu4.1) ...
    Setting up gir1.2-udisks-2.0:amd64 (2.10.1-6ubuntu1) ...
    Setting up libgl1-mesa-dri:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up libboost-locale1.83.0:amd64 (1.83.0-2.1ubuntu3.1) ...
    Setting up plasma-distro-release-notifier (20220915-0ubuntu6.1) ...
    Setting up network-manager (1.46.0-1ubuntu2.2) ...
    Warning: The unit file, source configuration file or drop-ins of NetworkManager.service changed on disk. Run 'systemctl daemon-reload' to reload units.
    Setting up software-properties-common (0.99.49.1) ...
    Setting up libpipewire-0.3-modules:amd64 (1.0.5-1ubuntu2) ...
    Setting up libegl-mesa0:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up libgssapi-krb5-2:amd64 (1.20.1-6ubuntu2.2) ...
    Setting up marble-plugins:amd64 (4:23.08.5-0ubuntu3.2) ...
    Setting up update-manager-core (1:24.04.9) ...
    Setting up digikam-private-libs (4:8.2.0-0ubuntu6.2) ...
    Setting up initramfs-tools-core (0.142ubuntu25.4) ...
    Setting up software-properties-qt (0.99.49.1) ...
    Setting up libglx-mesa0:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up initramfs-tools (0.142ubuntu25.4) ...
    update-initramfs: deferring update (trigger activated)
    Setting up pipewire-bin (1.0.5-1ubuntu2) ...
    Setting up digikam (4:8.2.0-0ubuntu6.2) ...
    Installing new version of config file /etc/apparmor.d/usr.bin.digikam ...
    Setting up pipewire:amd64 (1.0.5-1ubuntu2) ...
    Setting up cryptsetup-initramfs (2:2.7.0-1ubuntu4.1) ...
    update-initramfs: deferring update (trigger activated)
    Setting up pipewire-jack:amd64 (1.0.5-1ubuntu2) ...
    Setting up pipewire-alsa:amd64 (1.0.5-1ubuntu2) ...
    Setting up pipewire-pulse (1.0.5-1ubuntu2) ...
    Setting up pipewire-audio (1.0.5-1ubuntu2) ...
    Setting up apport (2.28.1-0ubuntu3.3) ...
    apport-autoreport.service is a disabled or a static unit not running, not starting it.
    Setting up apport-kde (2.28.1-0ubuntu3.3) ...
    Setting up apport-core-dump-handler (2.28.1-0ubuntu3.3) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for libglib2.0-0t64:amd64 (2.80.0-6ubuntu3.2) ...
    Processing triggers for dbus (1.14.10-4ubuntu4.1) ...
    Setting up libgtk-3-0t64:amd64 (3.24.41-4ubuntu1.2) ...
    Processing triggers for udev (255.4-1ubuntu8.4) ...
    Processing triggers for desktop-file-utils (0.27-2build1) ...
    Processing triggers for hicolor-icon-theme (0.17-2) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
    Processing triggers for rsyslog (8.2312.0-3ubuntu9) ...
    Warning: The unit file, source configuration file or drop-ins of rsyslog.service changed on disk. Run 'systemctl daemon-reload' to reload units.
    Setting up gir1.2-gtk-3.0:amd64 (3.24.41-4ubuntu1.2) ...
    Setting up libgtk-3-bin (3.24.41-4ubuntu1.2) ...
    Setting up libgtk-3-dev:amd64 (3.24.41-4ubuntu1.2) ...
    Processing triggers for initramfs-tools (0.142ubuntu25.4) ...
    update-initramfs: Generating /boot/initrd.img-6.8.0-50-lowlatency
    ```

After this the system will require a reboot, but before that a few more
[essential packages](#install-essential-packages) can be installed.

#### Install Essential Packages

Start by installing a few
[essential packages](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#install-essential-packages), plus a few more 
that have been found necessary later (e.g. `auditd` to
[stop apparmor spew in the logs](2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#stop-apparmor-spew-in-the-logs)):

??? terminal "`# apt install ...`"

    ``` console
    # apt install gdebi-core wget gkrellm vim curl gkrellm-leds \
    gkrellm-xkb gkrellm-cpufreq geeqie playonlinux exfat-fuse \
    clementine id3v2 htop vnstat neofetch tigervnc-viewer sox \
    scummvm wine gamemode python-is-python3 exiv2 rename scrot \
    speedtest-cli xcalib python3-pip netcat-openbsd jstest-gtk \
    etherwake python3-selenium lm-sensors sysstat tor unrar \
    ttf-mscorefonts-installer winetricks icc-profiles ffmpeg \
    iotop-c xdotool redshift-qt inxi vainfo vdpauinfo mpv xsane \
    tigervnc-tools screen lutris libxxf86vm-dev displaycal \
    python3-absl python3-unidecode auditd -y

    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    wget is already the newest version (1.21.4-1ubuntu4.1).
    wget set to manually installed.
    netcat-openbsd is already the newest version (1.226-1ubuntu2).
    netcat-openbsd set to manually installed.
    sysstat is already the newest version (12.6.1-2).
    sysstat set to manually installed.
    ffmpeg is already the newest version (7:6.1.1-3ubuntu5).
    ffmpeg set to manually installed.
    displaycal is already the newest version (3.9.11-2ubuntu0.24.04.1).
    displaycal set to manually installed.
    The following additional packages will be installed:
      auditd cabextract caca-utils chafa chromium-browser chromium-chromedriver evemu-tools
      evtest exiftran fluid-soundfont-gs fonts-wine fuseiso gamemode-daemon geeqie-common
      gir1.2-javascriptcoregtk-4.1 gir1.2-soup-3.0 gir1.2-webkit2-4.1 icoutils joystick jp2a 
      libasound2-plugins libcapi20-3t64 libchafa0t64 libcpufreq0
      libevemu3t64 libgamemode0 libgamemodeauto0 libgnutls-openssl27t64 libgpod-common
      libgpod4t64 libinih1 libjavascriptcoregtk-4.1-0 liblastfm5-1 liblua5.3-0
      libmanette-0.2-0 libmikmod3 libmspack0t64 libmygpo-qt5-1 libntlm0 libosmesa6
      libpython3-dev libpython3.12-dev libsdl2-net-2.0-0 libsgutils2-1.46-2
      libsixel-bin libsonivox3 libutempter0 libwebkit2gtk-4.1-0 libwine libxdo3
      libxkbregistry0 libz-mingw-w64 python3-dev python3-evdev python3-exceptiongroup
      python3-h11 python3-magic python3-natsort python3-outcome python3-setproctitle
      python3-sniffio python3-trio python3-trio-websocket python3-wsproto python3.12-dev
      redshift scummvm-data toilet toilet-fonts tor-geoipdb torsocks tree vim-runtime w3m
      w3m-img wakeonlan webp-pixbuf-loader wine64 xdg-dbus-proxy xsane-common
    Suggested packages:
      audispd-plugins gnome-shell-extension-gamemode xpaint libjpeg-progs
      libterm-readline-gnu-perl | libterm-readline-perl-perl
      libxml-dumper-perl sg3-utils fancontrol read-edid i2c-tools gamescope libcuda1 winbind
      python-evdev-doc python-natsort-doc byobu | screenie | iselect ncurses-term
      beneath-a-steel-sky drascula flight-of-the-amazon-queen lure-of-the-temptress
      libsox-fmt-all figlet mixmaster torbrowser-launcher apparmor-utils nyx obfs4proxy
      ctags vim-doc vim-scripts vnstati brotli cmigemo compface dict dict-wn dictd mailcap
      w3m-el xsel q4wine wine-binfmt dosbox wine64-preloader
      gocr | cuneiform | tesseract-ocr | ocrad gv hylafax-client | mgetty-fax
    Recommended packages:
      libgamemode0:i386 libgamemodeauto0:i386 wine32
    The following NEW packages will be installed:
      auditd cabextract caca-utils chafa chromium-browser chromium-chromedriver clementine
      etherwake evemu-tools evtest exfat-fuse exiftran exiv2 fluid-soundfont-gs fonts-wine
      fuseiso gamemode gamemode-daemon gdebi-core geeqie geeqie-common
      gir1.2-javascriptcoregtk-4.1 gir1.2-soup-3.0 gir1.2-webkit2-4.1 gkrellm
      gkrellm-cpufreq gkrellm-leds gkrellm-xkb htop icc-profiles icoutils id3v2 inxi iotop-c
      joystick jp2a jstest-gtk libasound2-plugins libauparse0t64 libcapi20-3t64 libchafa0t64
      libcpufreq0 libevemu3t64 libgamemode0 libgamemodeauto0 libgnutls-openssl27t64
      libgpod-common libgpod4t64 libinih1 libjavascriptcoregtk-4.1-0 liblastfm5-1 liblua5.3-0
      libmanette-0.2-0 libmikmod3 libmspack0t64 libmygpo-qt5-1 libntlm0 libosmesa6
      libpython3-dev libpython3.12-dev libsdl2-net-2.0-0 libsgutils2-1.46-2 libsixel-bin
      libsonivox3 libutempter0 libwebkit2gtk-4.1-0 libwine libxdo3 libxkbregistry0
      libxxf86vm-dev libz-mingw-w64 lm-sensors lutris mpv neofetch playonlinux
      python-is-python3 python3-absl python3-dev python3-evdev python3-exceptiongroup
      python3-h11 python3-magic python3-natsort python3-outcome python3-pip python3-selenium
      python3-setproctitle python3-sniffio python3-trio python3-trio-websocket
      python3-unidecode auditd python3-wsproto python3.12-dev redshift
      redshift-qt rename screen scrot scummvm scummvm-data sox speedtest-cli tigervnc-tools
      tigervnc-viewer toilet toilet-fonts tor tor-geoipdb torsocks tree
      ttf-mscorefonts-installer unrar vainfo vdpauinfo vim vim-runtime vnstat w3m w3m-img
      wakeonlan webp-pixbuf-loader wine wine64 winetricks xcalib xdg-dbus-proxy xdotool
      xsane xsane-common
    0 upgraded, 128 newly installed, 0 to remove and 3 not upgraded.
    Need to get 0 B/353 MB of archives.
    After this operation, 1,259 MB of additional disk space will be used.
    Extracting templates from packages: 100%
    Preconfiguring packages ...
    Selecting previously unselected package chromium-browser.
    (Reading database ... 417887 files and directories currently installed.)
    Preparing to unpack .../000-chromium-browser_2%3a1snap1-0ubuntu2_amd64.deb ...
    => Installing the chromium snap
    ==> Checking connectivity with the snap store
    ==> Installing the chromium snap
    Warning: /snap/bin was not found in your $PATH. If you've not restarted your session since you
            installed snapd, try doing that. Please see https://forum.snapcraft.io/t/9469 for more
            details.

    chromium 131.0.6778.139 from Canonical✓ installed
    => Snap installation complete

    Unpacking chromium-browser (2:1snap1-0ubuntu2) ...
    Selecting previously unselected package libmspack0t64:amd64.
    Preparing to unpack .../001-libmspack0t64_0.11-1.1build1_amd64.deb ...
    Unpacking libmspack0t64:amd64 (0.11-1.1build1) ...
    Selecting previously unselected package cabextract.
    Preparing to unpack .../002-cabextract_1.11-2_amd64.deb ...
    Unpacking cabextract (1.11-2) ...
    Selecting previously unselected package ttf-mscorefonts-installer.
    Preparing to unpack .../003-ttf-mscorefonts-installer_3.8.1ubuntu1_all.deb ...
    Unpacking ttf-mscorefonts-installer (3.8.1ubuntu1) ...
    Selecting previously unselected package vnstat.
    Preparing to unpack .../004-vnstat_2.12-1_amd64.deb ...
    Unpacking vnstat (2.12-1) ...
    Selecting previously unselected package caca-utils.
    Preparing to unpack .../005-caca-utils_0.99.beta20-4build2_amd64.deb ...
    Unpacking caca-utils (0.99.beta20-4build2) ...
    Selecting previously unselected package libchafa0t64:amd64.
    Preparing to unpack .../006-libchafa0t64_1.14.0-1.1build1_amd64.deb ...
    Unpacking libchafa0t64:amd64 (1.14.0-1.1build1) ...
    Selecting previously unselected package chafa.
    Preparing to unpack .../007-chafa_1.14.0-1.1build1_amd64.deb ...
    Unpacking chafa (1.14.0-1.1build1) ...
    Selecting previously unselected package chromium-chromedriver.
    Preparing to unpack .../008-chromium-chromedriver_2%3a1snap1-0ubuntu2_amd64.deb ...
    Unpacking chromium-chromedriver (2:1snap1-0ubuntu2) ...
    Selecting previously unselected package libgpod4t64:amd64.
    Preparing to unpack .../009-libgpod4t64_0.8.3-19.1ubuntu4_amd64.deb ...
    Unpacking libgpod4t64:amd64 (0.8.3-19.1ubuntu4) ...
    Selecting previously unselected package liblastfm5-1:amd64.
    Preparing to unpack .../010-liblastfm5-1_1.1.0-5build3_amd64.deb ...
    Unpacking liblastfm5-1:amd64 (1.1.0-5build3) ...
    Selecting previously unselected package libmygpo-qt5-1:amd64.
    Preparing to unpack .../011-libmygpo-qt5-1_1.1.0-4.1build3_amd64.deb ...
    Unpacking libmygpo-qt5-1:amd64 (1.1.0-4.1build3) ...
    Selecting previously unselected package clementine.
    Preparing to unpack .../012-clementine_1.4.0~rc1+git867-g9ef681b0e+dfsg-1ubuntu4_amd64.deb ...
    Unpacking clementine (1.4.0~rc1+git867-g9ef681b0e+dfsg-1ubuntu4) ...
    Selecting previously unselected package etherwake.
    Preparing to unpack .../013-etherwake_1.09-4build1_amd64.deb ...
    Unpacking etherwake (1.09-4build1) ...
    Selecting previously unselected package libevemu3t64:amd64.
    Preparing to unpack .../014-libevemu3t64_2.7.0-4build1_amd64.deb ...
    Unpacking libevemu3t64:amd64 (2.7.0-4build1) ...
    Selecting previously unselected package evemu-tools.
    Preparing to unpack .../015-evemu-tools_2.7.0-4build1_amd64.deb ...
    Unpacking evemu-tools (2.7.0-4build1) ...
    Selecting previously unselected package exfat-fuse.
    Preparing to unpack .../016-exfat-fuse_1.4.0-2_amd64.deb ...
    Unpacking exfat-fuse (1.4.0-2) ...
    Selecting previously unselected package exiftran.
    Preparing to unpack .../017-exiftran_2.10-4ubuntu4_amd64.deb ...
    Unpacking exiftran (2.10-4ubuntu4) ...
    Selecting previously unselected package exiv2.
    Preparing to unpack .../018-exiv2_0.27.6-1build1_amd64.deb ...
    Unpacking exiv2 (0.27.6-1build1) ...
    Selecting previously unselected package fluid-soundfont-gs.
    Preparing to unpack .../019-fluid-soundfont-gs_3.1-5.3_all.deb ...
    Unpacking fluid-soundfont-gs (3.1-5.3) ...
    Selecting previously unselected package fonts-wine.
    Preparing to unpack .../020-fonts-wine_9.0~repack-4build3_all.deb ...
    Unpacking fonts-wine (9.0~repack-4build3) ...
    Selecting previously unselected package fuseiso.
    Preparing to unpack .../021-fuseiso_20070708-3.2build3_amd64.deb ...
    Unpacking fuseiso (20070708-3.2build3) ...
    Selecting previously unselected package libinih1:amd64.
    Preparing to unpack .../022-libinih1_55-1ubuntu2_amd64.deb ...
    Unpacking libinih1:amd64 (55-1ubuntu2) ...
    Selecting previously unselected package gamemode-daemon.
    Preparing to unpack .../023-gamemode-daemon_1.8.1-2build1_amd64.deb ...
    Unpacking gamemode-daemon (1.8.1-2build1) ...
    Selecting previously unselected package libgamemode0:amd64.
    Preparing to unpack .../024-libgamemode0_1.8.1-2build1_amd64.deb ...
    Unpacking libgamemode0:amd64 (1.8.1-2build1) ...
    Selecting previously unselected package libgamemodeauto0:amd64.
    Preparing to unpack .../025-libgamemodeauto0_1.8.1-2build1_amd64.deb ...
    Unpacking libgamemodeauto0:amd64 (1.8.1-2build1) ...
    Selecting previously unselected package gamemode.
    Preparing to unpack .../026-gamemode_1.8.1-2build1_amd64.deb ...
    Unpacking gamemode (1.8.1-2build1) ...
    Selecting previously unselected package gdebi-core.
    Preparing to unpack .../027-gdebi-core_0.9.5.7+nmu7_all.deb ...
    Unpacking gdebi-core (0.9.5.7+nmu7) ...
    Selecting previously unselected package liblua5.3-0:amd64.
    Preparing to unpack .../028-liblua5.3-0_5.3.6-2build2_amd64.deb ...
    Unpacking liblua5.3-0:amd64 (5.3.6-2build2) ...
    Selecting previously unselected package geeqie-common.
    Preparing to unpack .../029-geeqie-common_1%3a2.2-2build4_all.deb ...
    Unpacking geeqie-common (1:2.2-2build4) ...
    Selecting previously unselected package webp-pixbuf-loader:amd64.
    Preparing to unpack .../030-webp-pixbuf-loader_0.2.4-2build2_amd64.deb ...
    Unpacking webp-pixbuf-loader:amd64 (0.2.4-2build2) ...
    Selecting previously unselected package geeqie.
    Preparing to unpack .../031-geeqie_1%3a2.2-2build4_amd64.deb ...
    Unpacking geeqie (1:2.2-2build4) ...
    Selecting previously unselected package libjavascriptcoregtk-4.1-0:amd64.
    Preparing to unpack .../032-libjavascriptcoregtk-4.1-0_2.46.4-0ubuntu0.24.04.1_amd64.deb ...
    Unpacking libjavascriptcoregtk-4.1-0:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Selecting previously unselected package gir1.2-javascriptcoregtk-4.1:amd64.
    Preparing to unpack .../033-gir1.2-javascriptcoregtk-4.1_2.46.4-0ubuntu0.24.04.1_amd64.deb ...
    Unpacking gir1.2-javascriptcoregtk-4.1:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Selecting previously unselected package gir1.2-soup-3.0:amd64.
    Preparing to unpack .../034-gir1.2-soup-3.0_3.4.4-5ubuntu0.1_amd64.deb ...
    Unpacking gir1.2-soup-3.0:amd64 (3.4.4-5ubuntu0.1) ...
    Selecting previously unselected package xdg-dbus-proxy.
    Preparing to unpack .../035-xdg-dbus-proxy_0.1.5-1build2_amd64.deb ...
    Unpacking xdg-dbus-proxy (0.1.5-1build2) ...
    Selecting previously unselected package libmanette-0.2-0:amd64.
    Preparing to unpack .../036-libmanette-0.2-0_0.2.7-1build2_amd64.deb ...
    Unpacking libmanette-0.2-0:amd64 (0.2.7-1build2) ...
    Selecting previously unselected package libwebkit2gtk-4.1-0:amd64.
    Preparing to unpack .../037-libwebkit2gtk-4.1-0_2.46.4-0ubuntu0.24.04.1_amd64.deb ...
    Unpacking libwebkit2gtk-4.1-0:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Selecting previously unselected package gir1.2-webkit2-4.1:amd64.
    Preparing to unpack .../038-gir1.2-webkit2-4.1_2.46.4-0ubuntu0.24.04.1_amd64.deb ...
    Unpacking gir1.2-webkit2-4.1:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Selecting previously unselected package libgnutls-openssl27t64:amd64.
    Preparing to unpack .../039-libgnutls-openssl27t64_3.8.3-1.1ubuntu3.2_amd64.deb ...
    Unpacking libgnutls-openssl27t64:amd64 (3.8.3-1.1ubuntu3.2) ...
    Selecting previously unselected package libntlm0:amd64.
    Preparing to unpack .../040-libntlm0_1.7-1build1_amd64.deb ...
    Unpacking libntlm0:amd64 (1.7-1build1) ...
    Selecting previously unselected package gkrellm.
    Preparing to unpack .../041-gkrellm_2.3.11-2build2_amd64.deb ...
    Unpacking gkrellm (2.3.11-2build2) ...
    Selecting previously unselected package gkrellm-xkb.
    Preparing to unpack .../042-gkrellm-xkb_1.05-5.1build2_amd64.deb ...
    Unpacking gkrellm-xkb (1.05-5.1build2) ...
    Selecting previously unselected package htop.
    Preparing to unpack .../043-htop_3.3.0-4build1_amd64.deb ...
    Unpacking htop (3.3.0-4build1) ...
    Selecting previously unselected package icc-profiles.
    Preparing to unpack .../044-icc-profiles_2.1-2_all.deb ...
    Unpacking icc-profiles (2.1-2) ...
    Selecting previously unselected package icoutils.
    Preparing to unpack .../045-icoutils_0.32.3-4build2_amd64.deb ...
    Unpacking icoutils (0.32.3-4build2) ...
    Selecting previously unselected package id3v2.
    Preparing to unpack .../046-id3v2_0.1.12+dfsg-7_amd64.deb ...
    Unpacking id3v2 (0.1.12+dfsg-7) ...
    Selecting previously unselected package iotop-c.
    Preparing to unpack .../047-iotop-c_1.26-1_amd64.deb ...
    Unpacking iotop-c (1.26-1) ...
    Selecting previously unselected package jp2a.
    Preparing to unpack .../048-jp2a_1.1.1-2ubuntu2_amd64.deb ...
    Unpacking jp2a (1.1.1-2ubuntu2) ...
    Selecting previously unselected package jstest-gtk.
    Preparing to unpack .../049-jstest-gtk_0.1.1~git20180602-2build2_amd64.deb ...
    Unpacking jstest-gtk (0.1.1~git20180602-2build2) ...
    Selecting previously unselected package libasound2-plugins:amd64.
    Preparing to unpack .../050-libasound2-plugins_1.2.7.1-1ubuntu5_amd64.deb ...
    Unpacking libasound2-plugins:amd64 (1.2.7.1-1ubuntu5) ...
    Selecting previously unselected package libcapi20-3t64:amd64.
    Preparing to unpack .../051-libcapi20-3t64_1%3a3.27-3.1build1_amd64.deb ...
    Unpacking libcapi20-3t64:amd64 (1:3.27-3.1build1) ...
    Selecting previously unselected package libcpufreq0.
    Preparing to unpack .../052-libcpufreq0_008-2build2_amd64.deb ...
    Unpacking libcpufreq0 (008-2build2) ...
    Selecting previously unselected package libsgutils2-1.46-2:amd64.
    Preparing to unpack .../053-libsgutils2-1.46-2_1.46-3ubuntu4_amd64.deb ...
    Unpacking libsgutils2-1.46-2:amd64 (1.46-3ubuntu4) ...
    Selecting previously unselected package libgpod-common.
    Preparing to unpack .../054-libgpod-common_0.8.3-19.1ubuntu4_amd64.deb ...
    Unpacking libgpod-common (0.8.3-19.1ubuntu4) ...
    Selecting previously unselected package libmikmod3:amd64.
    Preparing to unpack .../055-libmikmod3_3.3.11.1-7build1_amd64.deb ...
    Unpacking libmikmod3:amd64 (3.3.11.1-7build1) ...
    Selecting previously unselected package libpython3.12-dev:amd64.
    Preparing to unpack .../056-libpython3.12-dev_3.12.3-1ubuntu0.3_amd64.deb ...
    Unpacking libpython3.12-dev:amd64 (3.12.3-1ubuntu0.3) ...
    Selecting previously unselected package libpython3-dev:amd64.
    Preparing to unpack .../057-libpython3-dev_3.12.3-0ubuntu2_amd64.deb ...
    Unpacking libpython3-dev:amd64 (3.12.3-0ubuntu2) ...
    Selecting previously unselected package libsdl2-net-2.0-0:amd64.
    Preparing to unpack .../058-libsdl2-net-2.0-0_2.2.0+dfsg-2_amd64.deb ...
    Unpacking libsdl2-net-2.0-0:amd64 (2.2.0+dfsg-2) ...
    Selecting previously unselected package libsonivox3:amd64.
    Preparing to unpack .../059-libsonivox3_3.6.12-1_amd64.deb ...
    Unpacking libsonivox3:amd64 (3.6.12-1) ...
    Selecting previously unselected package libutempter0:amd64.
    Preparing to unpack .../060-libutempter0_1.2.1-3build1_amd64.deb ...
    Unpacking libutempter0:amd64 (1.2.1-3build1) ...
    Selecting previously unselected package libxkbregistry0:amd64.
    Preparing to unpack .../061-libxkbregistry0_1.6.0-1build1_amd64.deb ...
    Unpacking libxkbregistry0:amd64 (1.6.0-1build1) ...
    Selecting previously unselected package libz-mingw-w64..................................] 
    Preparing to unpack .../062-libz-mingw-w64_1.3.1+dfsg-1_all.deb ........................] 
    Unpacking libz-mingw-w64 (1.3.1+dfsg-1) ...
    Selecting previously unselected package libwine:amd64.
    Preparing to unpack .../063-libwine_9.0~repack-4build3_amd64.deb ...
    Unpacking libwine:amd64 (9.0~repack-4build3) ...
    Selecting previously unselected package libxxf86vm-dev:amd64.
    Preparing to unpack .../064-libxxf86vm-dev_1%3a1.1.4-1build4_amd64.deb ...
    Unpacking libxxf86vm-dev:amd64 (1:1.1.4-1build4) ...
    Selecting previously unselected package python3-setproctitle:amd64.
    Preparing to unpack .../065-python3-setproctitle_1.3.3-1build2_amd64.deb ...
    Unpacking python3-setproctitle:amd64 (1.3.3-1build2) ...
    Selecting previously unselected package python3-magic.
    Preparing to unpack .../066-python3-magic_2%3a0.4.27-3_all.deb ...
    Unpacking python3-magic (2:0.4.27-3) ...
    Selecting previously unselected package lutris.
    Preparing to unpack .../067-lutris_0.5.14-2_all.deb ...
    Unpacking lutris (0.5.14-2) ...
    Selecting previously unselected package mpv.
    Preparing to unpack .../068-mpv_0.37.0-1ubuntu4_amd64.deb ...
    Unpacking mpv (0.37.0-1ubuntu4) ...
    Selecting previously unselected package neofetch.
    Preparing to unpack .../069-neofetch_7.1.0-4_all.deb ...
    Unpacking neofetch (7.1.0-4) ...
    Selecting previously unselected package python3-natsort.
    Preparing to unpack .../070-python3-natsort_8.0.2-2_all.deb ...
    Unpacking python3-natsort (8.0.2-2) ...
    Selecting previously unselected package wine64.
    Preparing to unpack .../071-wine64_9.0~repack-4build3_amd64.deb ...
    Unpacking wine64 (9.0~repack-4build3) ...
    Selecting previously unselected package wine.
    Preparing to unpack .../072-wine_9.0~repack-4build3_all.deb ...
    Unpacking wine (9.0~repack-4build3) ...
    Selecting previously unselected package playonlinux.
    Preparing to unpack .../073-playonlinux_4.3.4-3_all.deb ...
    Unpacking playonlinux (4.3.4-3) ...
    Selecting previously unselected package python-is-python3.
    Preparing to unpack .../074-python-is-python3_3.11.4-1_all.deb ...
    Unpacking python-is-python3 (3.11.4-1) ...
    Selecting previously unselected package python3-absl.
    Preparing to unpack .../075-python3-absl_2.1.0-1_all.deb ...
    Unpacking python3-absl (2.1.0-1) ...
    Selecting previously unselected package python3.12-dev.
    Preparing to unpack .../076-python3.12-dev_3.12.3-1ubuntu0.3_amd64.deb ...
    Unpacking python3.12-dev (3.12.3-1ubuntu0.3) ...
    Selecting previously unselected package python3-dev.
    Preparing to unpack .../077-python3-dev_3.12.3-0ubuntu2_amd64.deb ...
    Unpacking python3-dev (3.12.3-0ubuntu2) ...
    Selecting previously unselected package python3-evdev.
    Preparing to unpack .../078-python3-evdev_1.7.0+dfsg-1build1_amd64.deb ...
    Unpacking python3-evdev (1.7.0+dfsg-1build1) ...
    Selecting previously unselected package python3-exceptiongroup.
    Preparing to unpack .../079-python3-exceptiongroup_1.2.0-1_all.deb ...
    Unpacking python3-exceptiongroup (1.2.0-1) ...
    Selecting previously unselected package python3-h11.
    Preparing to unpack .../080-python3-h11_0.14.0-1_all.deb ...
    Unpacking python3-h11 (0.14.0-1) ...
    Selecting previously unselected package python3-outcome.
    Preparing to unpack .../081-python3-outcome_1.2.0-1.1_all.deb ...
    Unpacking python3-outcome (1.2.0-1.1) ...
    Selecting previously unselected package python3-pip.
    Preparing to unpack .../082-python3-pip_24.0+dfsg-1ubuntu1.1_all.deb ...
    Unpacking python3-pip (24.0+dfsg-1ubuntu1.1) ...
    Selecting previously unselected package python3-sniffio.
    Preparing to unpack .../083-python3-sniffio_1.3.0-2_all.deb ...
    Unpacking python3-sniffio (1.3.0-2) ...
    Selecting previously unselected package python3-trio.
    Preparing to unpack .../084-python3-trio_0.24.0-1ubuntu1_all.deb ...
    Unpacking python3-trio (0.24.0-1ubuntu1) ...
    Selecting previously unselected package python3-wsproto.
    Preparing to unpack .../085-python3-wsproto_1.2.0-1_all.deb ...
    Unpacking python3-wsproto (1.2.0-1) ...
    Selecting previously unselected package python3-trio-websocket.
    Preparing to unpack .../086-python3-trio-websocket_0.11.1-1_all.deb ...
    Unpacking python3-trio-websocket (0.11.1-1) ...
    Selecting previously unselected package python3-selenium.
    Preparing to unpack .../087-python3-selenium_4.18.1+dfsg-1_all.deb ...
    Unpacking python3-selenium (4.18.1+dfsg-1) ...
    Selecting previously unselected package python3-unidecode.
    Preparing to unpack .../088-python3-unidecode_1.3.8-1_all.deb ...
    Unpacking python3-unidecode (1.3.8-1) ...
    Selecting previously unselected package redshift.
    Preparing to unpack .../089-redshift_1.12-4.2ubuntu4_amd64.deb ...
    Unpacking redshift (1.12-4.2ubuntu4) ...
    Selecting previously unselected package redshift-qt.
    Preparing to unpack .../090-redshift-qt_0.6-3build3_amd64.deb ...
    Unpacking redshift-qt (0.6-3build3) ...
    Selecting previously unselected package rename.
    Preparing to unpack .../091-rename_2.02-1_all.deb ...
    Unpacking rename (2.02-1) ...
    Selecting previously unselected package screen.
    Preparing to unpack .../092-screen_4.9.1-1build1_amd64.deb ...
    Unpacking screen (4.9.1-1build1) ...
    Selecting previously unselected package scrot.
    Preparing to unpack .../093-scrot_1.10-1build2_amd64.deb ...
    Unpacking scrot (1.10-1build2) ...
    Selecting previously unselected package scummvm-data.
    Preparing to unpack .../094-scummvm-data_2.8.0+dfsg-1build5_all.deb ...
    Unpacking scummvm-data (2.8.0+dfsg-1build5) ...
    Selecting previously unselected package scummvm.
    Preparing to unpack .../095-scummvm_2.8.0+dfsg-1build5_amd64.deb ...
    Unpacking scummvm (2.8.0+dfsg-1build5) ...
    Selecting previously unselected package sox.
    Preparing to unpack .../096-sox_14.4.2+git20190427-4build4_amd64.deb ...
    Unpacking sox (14.4.2+git20190427-4build4) ...
    Selecting previously unselected package speedtest-cli.
    Preparing to unpack .../097-speedtest-cli_2.1.3-2_all.deb ...
    Unpacking speedtest-cli (2.1.3-2) ...
    Selecting previously unselected package tigervnc-tools.
    Preparing to unpack .../098-tigervnc-tools_1.13.1+dfsg-2build2_amd64.deb ...
    Unpacking tigervnc-tools (1.13.1+dfsg-2build2) ...
    Selecting previously unselected package tigervnc-viewer.
    Preparing to unpack .../099-tigervnc-viewer_1.13.1+dfsg-2build2_amd64.deb ...
    Unpacking tigervnc-viewer (1.13.1+dfsg-2build2) ...
    Selecting previously unselected package toilet-fonts.
    Preparing to unpack .../100-toilet-fonts_0.3-1.4build1_all.deb ...
    Unpacking toilet-fonts (0.3-1.4build1) ...
    Selecting previously unselected package toilet.
    Preparing to unpack .../101-toilet_0.3-1.4build1_amd64.deb ...
    Unpacking toilet (0.3-1.4build1) ...
    Selecting previously unselected package tor.
    Preparing to unpack .../102-tor_0.4.8.10-1build2_amd64.deb ...
    Unpacking tor (0.4.8.10-1build2) ...
    Selecting previously unselected package torsocks.
    Preparing to unpack .../103-torsocks_2.4.0-1_amd64.deb ...
    Unpacking torsocks (2.4.0-1) ...
    Selecting previously unselected package tree.
    Preparing to unpack .../104-tree_2.1.1-2ubuntu3_amd64.deb ...
    Unpacking tree (2.1.1-2ubuntu3) ...
    Selecting previously unselected package unrar.
    Preparing to unpack .../105-unrar_1%3a7.0.7-1build1_amd64.deb ...
    Unpacking unrar (1:7.0.7-1build1) ...
    Selecting previously unselected package vainfo.
    Preparing to unpack .../106-vainfo_2.12.0+ds1-1_amd64.deb ...
    Unpacking vainfo (2.12.0+ds1-1) ...
    Selecting previously unselected package vim-runtime.
    Preparing to unpack .../107-vim-runtime_2%3a9.1.0016-1ubuntu7.5_all.deb ...
    Adding 'diversion of /usr/share/vim/vim91/doc/help.txt to /usr/share/vim/vim91/doc/help.txt.vim-tiny by vim-runtime'
    Adding 'diversion of /usr/share/vim/vim91/doc/tags to /usr/share/vim/vim91/doc/tags.vim-tiny by vim-runtime'
    Unpacking vim-runtime (2:9.1.0016-1ubuntu7.5) ...
    Selecting previously unselected package vim.
    Preparing to unpack .../108-vim_2%3a9.1.0016-1ubuntu7.5_amd64.deb ...
    Unpacking vim (2:9.1.0016-1ubuntu7.5) ...
    Selecting previously unselected package w3m.
    Preparing to unpack .../109-w3m_0.5.3+git20230121-2ubuntu5_amd64.deb ...
    Unpacking w3m (0.5.3+git20230121-2ubuntu5) ...
    Selecting previously unselected package w3m-img.
    Preparing to unpack .../110-w3m-img_0.5.3+git20230121-2ubuntu5_amd64.deb ...
    Unpacking w3m-img (0.5.3+git20230121-2ubuntu5) ...
    Selecting previously unselected package wakeonlan.
    Preparing to unpack .../111-wakeonlan_0.41-12.1_all.deb ...
    Unpacking wakeonlan (0.41-12.1) ...
    Selecting previously unselected package winetricks.
    Preparing to unpack .../112-winetricks_20240105-2_all.deb ...
    Unpacking winetricks (20240105-2) ...
    Selecting previously unselected package xsane-common.
    Preparing to unpack .../113-xsane-common_0.999-12ubuntu4_all.deb ...
    Unpacking xsane-common (0.999-12ubuntu4) ...
    Selecting previously unselected package xsane.
    Preparing to unpack .../114-xsane_0.999-12ubuntu4_amd64.deb ...
    Unpacking xsane (0.999-12ubuntu4) ...
    Selecting previously unselected package evtest.
    Preparing to unpack .../115-evtest_1%3a1.35-1_amd64.deb ...
    Unpacking evtest (1:1.35-1) ...
    Selecting previously unselected package gkrellm-cpufreq.
    Preparing to unpack .../116-gkrellm-cpufreq_0.6.4-4_amd64.deb ...
    Unpacking gkrellm-cpufreq (0.6.4-4) ...
    Selecting previously unselected package gkrellm-leds.
    Preparing to unpack .../117-gkrellm-leds_0.8.0-2build2_amd64.deb ...
    Unpacking gkrellm-leds (0.8.0-2build2) ...
    Selecting previously unselected package inxi.
    Preparing to unpack .../118-inxi_3.3.34-1-1_all.deb ...
    Unpacking inxi (3.3.34-1-1) ...
    Selecting previously unselected package joystick.
    Preparing to unpack .../119-joystick_1%3a1.8.1-2build1_amd64.deb ...
    Unpacking joystick (1:1.8.1-2build1) ...
    Selecting previously unselected package libosmesa6:amd64.
    Preparing to unpack .../120-libosmesa6_24.0.9-0ubuntu0.3_amd64.deb ...
    Unpacking libosmesa6:amd64 (24.0.9-0ubuntu0.3) ...
    Selecting previously unselected package libsixel-bin.
    Preparing to unpack .../121-libsixel-bin_1.10.3-3build1_amd64.deb ...
    Unpacking libsixel-bin (1.10.3-3build1) ...
    Selecting previously unselected package libxdo3:amd64.
    Preparing to unpack .../122-libxdo3_1%3a3.20160805.1-5build1_amd64.deb ...
    Unpacking libxdo3:amd64 (1:3.20160805.1-5build1) ...
    Selecting previously unselected package lm-sensors.
    Preparing to unpack .../123-lm-sensors_1%3a3.6.0-9build1_amd64.deb ...
    Unpacking lm-sensors (1:3.6.0-9build1) ...
    Selecting previously unselected package tor-geoipdb.
    Preparing to unpack .../124-tor-geoipdb_0.4.8.10-1build2_all.deb ...
    Unpacking tor-geoipdb (0.4.8.10-1build2) ...
    Selecting previously unselected package vdpauinfo.
    Preparing to unpack .../125-vdpauinfo_1.5-2_amd64.deb ...
    Unpacking vdpauinfo (1.5-2) ...
    Selecting previously unselected package xcalib.
    Preparing to unpack .../126-xcalib_0.8.dfsg1-3_amd64.deb ...
    Unpacking xcalib (0.8.dfsg1-3) ...
    Selecting previously unselected package xdotool.
    Preparing to unpack .../127-xdotool_1%3a3.20160805.1-5build1_amd64.deb ...
    Unpacking xdotool (1:3.20160805.1-5build1) ...
    Selecting previously unselected package libauparse0t64:amd64.
    (Reading database ... 426177 files and directories currently installed.)
    Preparing to unpack .../libauparse0t64_1%3a3.1.2-2.1build1.1_amd64.deb ...
    Adding 'diversion of /lib/x86_64-linux-gnu/libauparse.so.0 to /lib/x86_64-linux-gnu/libauparse.so.0.usr-is-merged by libauparse0t64'
    Adding 'diversion of /lib/x86_64-linux-gnu/libauparse.so.0.0.0 to /lib/x86_64-linux-gnu/libauparse.so.0.0.0.usr-is-merged by libauparse0t64'
    Unpacking libauparse0t64:amd64 (1:3.1.2-2.1build1.1) ...
    Selecting previously unselected package auditd.
    Preparing to unpack .../auditd_1%3a3.1.2-2.1build1.1_amd64.deb ...
    Unpacking auditd (1:3.1.2-2.1build1.1) ...
    Setting up libauparse0t64:amd64 (1:3.1.2-2.1build1.1) ...
    Setting up auditd (1:3.1.2-2.1build1.1) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/auditd.service → /usr/lib/systemd/system/auditd.service.
    Setting up libsonivox3:amd64 (3.6.12-1) ...
    Setting up libgnutls-openssl27t64:amd64 (3.8.3-1.1ubuntu3.2) ...
    Setting up python3-sniffio (1.3.0-2) ...
    Setting up python3-outcome (1.2.0-1.1) ...
    Setting up libasound2-plugins:amd64 (1.2.7.1-1ubuntu5) ...
    Setting up toilet-fonts (0.3-1.4build1) ...
    Setting up gdebi-core (0.9.5.7+nmu7) ...
    /usr/share/gdebi/GDebi/GDebiCli.py:159: SyntaxWarning: invalid escape sequence '\S'
    c = findall("[[(](\S+)/\S+[])]", msg)[0].lower()
    Setting up caca-utils (0.99.beta20-4build2) ...
    Setting up toilet (0.3-1.4build1) ...
    update-alternatives: using /usr/bin/figlet-toilet to provide /usr/bin/figlet (figlet) in auto mode
    Setting up libgpod4t64:amd64 (0.8.3-19.1ubuntu4) ...
    Setting up libsdl2-net-2.0-0:amd64 (2.2.0+dfsg-2) ...
    Setting up htop (3.3.0-4build1) ...
    Setting up vdpauinfo (1.5-2) ...
    Setting up libinih1:amd64 (55-1ubuntu2) ...
    Setting up joystick (1:1.8.1-2build1) ...
    Setting up geeqie-common (1:2.2-2build4) ...
    Setting up wakeonlan (0.41-12.1) ...
    Setting up libmanette-0.2-0:amd64 (0.2.7-1build2) ...
    Setting up libxxf86vm-dev:amd64 (1:1.1.4-1build4) ...
    Setting up libmygpo-qt5-1:amd64 (1.1.0-4.1build3) ...
    Setting up libmspack0t64:amd64 (0.11-1.1build1) ...
    Setting up jp2a (1.1.1-2ubuntu2) ...
    Setting up libmikmod3:amd64 (3.3.11.1-7build1) ...
    Setting up libsgutils2-1.46-2:amd64 (1.46-3ubuntu4) ...
    Setting up redshift (1.12-4.2ubuntu4) ...
    Setting up inxi (3.3.34-1-1) ...
    Setting up rename (2.02-1) ...
    update-alternatives: using /usr/bin/file-rename to provide /usr/bin/rename (rename) in auto mode
    Setting up libpython3.12-dev:amd64 (3.12.3-1ubuntu0.3) ...
    Setting up unrar (1:7.0.7-1build1) ...
    update-alternatives: using /usr/bin/unrar-nonfree to provide /usr/bin/unrar (unrar) in auto mode
    Setting up libchafa0t64:amd64 (1.14.0-1.1build1) ...
    Setting up libz-mingw-w64 (1.3.1+dfsg-1) ...
    Setting up libsixel-bin (1.10.3-3build1) ...
    Setting up jstest-gtk (0.1.1~git20180602-2build2) ...
    Setting up neofetch (7.1.0-4) ...
    Setting up python3-natsort (8.0.2-2) ...
    Setting up gamemode-daemon (1.8.1-2build1) ...
    Setting up webp-pixbuf-loader:amd64 (0.2.4-2build2) ...
    Setting up exiftran (2.10-4ubuntu4) ...
    Setting up python3-absl (2.1.0-1) ...
    Setting up libxdo3:amd64 (1:3.20160805.1-5build1) ...
    Setting up fonts-wine (9.0~repack-4build3) ...
    Setting up scummvm-data (2.8.0+dfsg-1build5) ...
    Setting up tigervnc-tools (1.13.1+dfsg-2build2) ...
    update-alternatives: using /usr/bin/tigervncpasswd to provide /usr/bin/vncpasswd (vncpasswd) in auto mode
    Setting up libjavascriptcoregtk-4.1-0:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Setting up fuseiso (20070708-3.2build3) ...
    Setting up w3m (0.5.3+git20230121-2ubuntu5) ...
    Setting up libxkbregistry0:amd64 (1.6.0-1build1) ...
    Setting up libntlm0:amd64 (1.7-1build1) ...
    Setting up icoutils (0.32.3-4build2) ...
    Setting up tree (2.1.1-2ubuntu3) ...
    Setting up exiv2 (0.27.6-1build1) ...
    Setting up python3-setproctitle:amd64 (1.3.3-1build2) ...
    Setting up python3.12-dev (3.12.3-1ubuntu0.3) ...
    Setting up python3-h11 (0.14.0-1) ...
    Setting up python3-evdev (1.7.0+dfsg-1build1) ...
    Setting up libevemu3t64:amd64 (2.7.0-4build1) ...
    Setting up python3-pip (24.0+dfsg-1ubuntu1.1) ...
    Setting up icc-profiles (2.1-2) ...
    Setting up libcapi20-3t64:amd64 (1:3.27-3.1build1) ...
    Setting up id3v2 (0.1.12+dfsg-7) ...
    Setting up vainfo (2.12.0+ds1-1) ...
    Setting up xcalib (0.8.dfsg1-3) ...
    Setting up tigervnc-viewer (1.13.1+dfsg-2build2) ...
    update-alternatives: using /usr/bin/xtigervncviewer to provide /usr/bin/vncviewer (vncviewer) in auto mode
    Setting up lm-sensors (1:3.6.0-9build1) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/lm-sensors.service → /usr/lib/systemd/system/lm-sensors.service.
    Setting up gir1.2-soup-3.0:amd64 (3.4.4-5ubuntu0.1) ...
    Setting up libutempter0:amd64 (1.2.1-3build1) ...
    Setting up sox (14.4.2+git20190427-4build4) ...
    Setting up mpv (0.37.0-1ubuntu4) ...
    Setting up xdg-dbus-proxy (0.1.5-1build2) ...
    Setting up libcpufreq0 (008-2build2) ...
    Setting up chromium-browser (2:1snap1-0ubuntu2) ...
    Setting up fluid-soundfont-gs (3.1-5.3) ...
    Setting up speedtest-cli (2.1.3-2) ...
    Setting up tor (0.4.8.10-1build2) ...
    Something or somebody made /var/lib/tor disappear.
    Creating one for you again.
    Something or somebody made /var/log/tor disappear.
    Creating one for you again.
    Created symlink /etc/systemd/system/multi-user.target.wants/tor.service → /usr/lib/systemd/system/tor.service.
    Setting up w3m-img (0.5.3+git20230121-2ubuntu5) ...
    Setting up liblua5.3-0:amd64 (5.3.6-2build2) ...
    Setting up etherwake (1.09-4build1) ...
    Setting up python3-exceptiongroup (1.2.0-1) ...
    Setting up iotop-c (1.26-1) ...
    update-alternatives: using /usr/sbin/iotop-c to provide /usr/sbin/iotop (iotop) in auto mode
    Setting up exfat-fuse (1.4.0-2) ...
    Setting up libwine:amd64 (9.0~repack-4build3) ...
    Setting up vim-runtime (2:9.1.0016-1ubuntu7.5) ...
    Setting up xsane-common (0.999-12ubuntu4) ...
    Setting up redshift-qt (0.6-3build3) ...
    Setting up evtest (1:1.35-1) ...
    Setting up python3-magic (2:0.4.27-3) ...
    Setting up libgamemodeauto0:amd64 (1.8.1-2build1) ...
    Setting up torsocks (2.4.0-1) ...
    Setting up python-is-python3 (3.11.4-1) ...
    Setting up scrot (1.10-1build2) ...
    Setting up vnstat (2.12-1) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/vnstat.service → /usr/lib/systemd/system/vnstat.service.
    Setting up libgamemode0:amd64 (1.8.1-2build1) ...
    Setting up python3-unidecode (1.3.8-1) ...
    Setting up liblastfm5-1:amd64 (1.1.0-5build3) ...
    Setting up scummvm (2.8.0+dfsg-1build5) ...
    Setting up libosmesa6:amd64 (24.0.9-0ubuntu0.3) ...
    Setting up evemu-tools (2.7.0-4build1) ...
    Setting up libwebkit2gtk-4.1-0:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Setting up geeqie (1:2.2-2build4) ...
    Setting up xsane (0.999-12ubuntu4) ...
    Setting up vim (2:9.1.0016-1ubuntu7.5) ...
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/ex (ex) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rview (rview) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rvim (rvim) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vi (vi) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/view (view) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vim (vim) in auto mode
    update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vimdiff (vimdiff) in auto mode
    Setting up libpython3-dev:amd64 (3.12.3-0ubuntu2) ...
    Setting up python3-wsproto (1.2.0-1) ...
    Setting up cabextract (1.11-2) ...
    Setting up xdotool (1:3.20160805.1-5build1) ...
    Setting up libgpod-common (0.8.3-19.1ubuntu4) ...
    Setting up chafa (1.14.0-1.1build1) ...
    Setting up screen (4.9.1-1build1) ...
    Setting up wine64 (9.0~repack-4build3) ...
    Setting up gir1.2-javascriptcoregtk-4.1:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Setting up gkrellm (2.3.11-2build2) ...
    Setting up tor-geoipdb (0.4.8.10-1build2) ...
    Setting up chromium-chromedriver (2:1snap1-0ubuntu2) ...
    Setting up python3-trio (0.24.0-1ubuntu1) ...
    Setting up python3-dev (3.12.3-0ubuntu2) ...
    Setting up gkrellm-cpufreq (0.6.4-4) ...
    Setting up clementine (1.4.0~rc1+git867-g9ef681b0e+dfsg-1ubuntu4) ...
    Setting up ttf-mscorefonts-installer (3.8.1ubuntu1) ...
    Setting up gamemode (1.8.1-2build1) ...
    Setting up wine (9.0~repack-4build3) ...
    Setting up gir1.2-webkit2-4.1:amd64 (2.46.4-0ubuntu0.24.04.1) ...
    Setting up playonlinux (4.3.4-3) ...
    /usr/share/playonlinux/python/configurewindow/ConfigureWindowNotebook.py:463: SyntaxWarning: invalid escape sequence '\|'
    self.supported_files = "All|*.exe;*.EXE;*.msi;*.MSI\
    /usr/share/playonlinux/python/mainwindow.py:710: SyntaxWarning: invalid escape sequence '\|'
    self.SupprotedIconExt = "All|*.xpm;*.XPM;*.png;*.PNG;*.ico;*.ICO;*.jpg;*.JPG;*.jpeg;*.JPEG;*.bmp;*.BMP\
    Setting up python3-trio-websocket (0.11.1-1) ...
    Setting up gkrellm-xkb (1.05-5.1build2) ...
    Setting up gkrellm-leds (0.8.0-2build2) ...
    Setting up winetricks (20240105-2) ...
    Setting up lutris (0.5.14-2) ...
    Setting up python3-selenium (4.18.1+dfsg-1) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for libgdk-pixbuf-2.0-0:amd64 (2.42.10+dfsg-3ubuntu3.1) ...
    Processing triggers for debianutils (5.17build1) ...
    Processing triggers for install-info (7.1-3build2) ...
    Processing triggers for fontconfig (2.15.0-1.1ubuntu2) ...
    Processing triggers for desktop-file-utils (0.27-2build1) ...
    Processing triggers for hicolor-icon-theme (0.17-2) ...
    Processing triggers for update-notifier-common (3.192.68build3) ...
    ttf-mscorefonts-installer: processing...
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/andale32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/andale32.exe [198 kB]
    Fetched 198 kB in 1s (200 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/arial32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/arial32.exe [554 kB]
    Fetched 554 kB in 1s (431 kB/s)                                                             
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/arialb32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/arialb32.exe [168 kB]
    Fetched 168 kB in 4s (43.0 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/comic32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/comic32.exe [246 kB]
    Fetched 246 kB in 1s (271 kB/s)                                                             
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/courie32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/courie32.exe [646 kB]
    Fetched 646 kB in 6s (103 kB/s)                                                                                                                                                                                                                                                                                            
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/georgi32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/georgi32.exe [392 kB]
    Fetched 392 kB in 1s (347 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/impact32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/impact32.exe [173 kB]
    Fetched 173 kB in 1s (202 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/times32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/times32.exe [662 kB]
    Fetched 662 kB in 1s (623 kB/s)                                                             
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/trebuc32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/trebuc32.exe [357 kB]
    Fetched 357 kB in 1s (382 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/verdan32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/verdan32.exe [352 kB]
    Fetched 352 kB in 1s (345 kB/s)                                                              
    ttf-mscorefonts-installer: downloading http://downloads.sourceforge.net/corefonts/webdin32.exe
    Get:1 http://downloads.sourceforge.net/corefonts/webdin32.exe [185 kB]
    Fetched 185 kB in 1s (187 kB/s)                                                              

    These fonts were provided by Microsoft "in the interest of cross-
    platform compatibility".  This is no longer the case, but they are
    still available from third parties.

    You are free to download these fonts and use them for your own use,
    but you may not redistribute them in modified form, including changes
    to the file name or packaging format.

    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/andale32.exe
    extracting fontinst.inf
    extracting andale.inf
    extracting fontinst.exe
    extracting AndaleMo.TTF
    extracting ADVPACK.DLL
    extracting W95INF32.DLL
    extracting W95INF16.DLL

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/arial32.exe
    extracting FONTINST.EXE
    extracting fontinst.inf
    extracting Ariali.TTF
    extracting Arialbd.TTF
    extracting Arialbi.TTF
    extracting Arial.TTF

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/arialb32.exe
    extracting fontinst.exe
    extracting fontinst.inf
    extracting AriBlk.TTF

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/comic32.exe
    extracting fontinst.inf
    extracting Comicbd.TTF
    extracting Comic.TTF
    extracting fontinst.exe

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/courie32.exe
    extracting cour.ttf
    extracting courbd.ttf
    extracting courbi.ttf
    extracting fontinst.inf
    extracting couri.ttf
    extracting fontinst.exe

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/georgi32.exe
    extracting fontinst.inf
    extracting Georgiaz.TTF
    extracting Georgiab.TTF
    extracting Georgiai.TTF
    extracting Georgia.TTF
    extracting fontinst.exe

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/impact32.exe
    extracting fontinst.exe
    extracting Impact.TTF
    extracting fontinst.inf

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/times32.exe
    extracting fontinst.inf
    extracting Times.TTF
    extracting Timesbd.TTF
    extracting Timesbi.TTF
    extracting Timesi.TTF
    extracting FONTINST.EXE

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/trebuc32.exe
    extracting FONTINST.EXE
    extracting trebuc.ttf
    extracting Trebucbd.ttf
    extracting trebucbi.ttf
    extracting trebucit.ttf
    extracting fontinst.inf

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/verdan32.exe
    extracting fontinst.exe
    extracting fontinst.inf
    extracting Verdanab.TTF
    extracting Verdanai.TTF
    extracting Verdanaz.TTF
    extracting Verdana.TTF

    All done, no errors.
    Extracting cabinet: /var/lib/update-notifier/package-data-downloads/partial/webdin32.exe
    extracting fontinst.exe
    extracting Webdings.TTF
    extracting fontinst.inf
    extracting Licen.TXT

    All done, no errors.
    All fonts downloaded and installed.
    Processing triggers for wine (9.0~repack-4build3) ...
    ```

After installing these, **Redshift** is immediately available.

Even before installing any packages **KDE Connect** is already
running, which comes in very handy if it was already setup.

At this point a reboot would be recommended, but we can do one 
more thing before that...

#### Restore Software RAID 5

This PC has a software RAID 5
on 3 old 1TB HDD (not SSD) disks that were already set up as a RAID 5
array in the old system.

To restore this RAID, start by preserving the information reported in
the RAID superblocks on each device:

??? terminal "`# apt install mdadm -y`"

    ``` console
    # apt install mdadm -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
    finalrd
    Suggested packages:
    default-mta | mail-transport-agent
    The following NEW packages will be installed:
    finalrd mdadm
    0 upgraded, 2 newly installed, 0 to remove and 3 not upgraded.
    Need to get 471 kB of archives.
    After this operation, 1,188 kB of additional disk space will be used.
    Get:1 http://es.archive.ubuntu.com/ubuntu noble/main amd64 finalrd all 9build1 [7,306 B]
    Get:2 http://es.archive.ubuntu.com/ubuntu noble-updates/main amd64 mdadm amd64 4.3-1ubuntu2.1 [464 kB]
    Fetched 471 kB in 1s (495 kB/s)
    Preconfiguring packages ...
    Selecting previously unselected package finalrd.
    (Reading database ... 425709 files and directories currently installed.)
    Preparing to unpack .../finalrd_9build1_all.deb ...
    Unpacking finalrd (9build1) ...
    Selecting previously unselected package mdadm.
    Preparing to unpack .../mdadm_4.3-1ubuntu2.1_amd64.deb ...
    Unpacking mdadm (4.3-1ubuntu2.1) ...
    Setting up finalrd (9build1) ...
    Created symlink /etc/systemd/system/sysinit.target.wants/finalrd.service → /usr/lib/systemd/system/finalrd.service.
    Setting up mdadm (4.3-1ubuntu2.1) ...
    Generating mdadm.conf... done.
    update-initramfs: deferring update (trigger activated)
    Sourcing file `/etc/default/grub'
    Sourcing file `/etc/default/grub.d/ubuntustudio.cfg'
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-6.8.0-50-lowlatency
    Found initrd image: /boot/initrd.img-6.8.0-50-lowlatency
    Found memtest86+ 64bit EFI image: /boot/memtest86+x64.efi
    Warning: os-prober will be executed to detect other bootable partitions.
    Its output will be used to detect bootable binaries on them and create new boot entries.
    Found Ubuntu 22.04.5 LTS (22.04) on /dev/nvme0n1p2
    Adding boot menu entry for UEFI Firmware Settings ...
    done
    Created symlink /etc/systemd/system/mdmonitor.service.wants/mdcheck_continue.timer → /usr/lib/systemd/system/mdcheck_continue.timer.
    Created symlink /etc/systemd/system/mdmonitor.service.wants/mdcheck_start.timer → /usr/lib/systemd/system/mdcheck_start.timer.
    Created symlink /etc/systemd/system/mdmonitor.service.wants/mdmonitor-oneshot.timer → /usr/lib/systemd/system/mdmonitor-oneshot.timer.
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for initramfs-tools (0.142ubuntu25.4) ...
    update-initramfs: Generating /boot/initrd.img-6.8.0-50-lowlatency
    ```

!!! note

    Installing `mdadm` already updates Grub so there is no need to
    manually run `update-grub2` afterwards.

??? terminal "`# mdadm --examine /dev/sd[a-c] | tee raid.status`"

    ``` console hl_lines="18 25 30 48 55 60 78 85 90"
    # mdadm --examine /dev/sd[a-c] | tee raid.status
    /dev/sda:
            Magic : a92b4efc
            Version : 1.2
        Feature Map : 0x1
        Array UUID : ba946432:20362047:d8bfa70b:615817a1
            Name : rapture:0
    Creation Time : Fri Jul 26 16:06:17 2019
        Raid Level : raid5
    Raid Devices : 3

    Avail Dev Size : 1953260976 sectors (931.39 GiB 1000.07 GB)
        Array Size : 1953260544 KiB (1862.77 GiB 2000.14 GB)
    Used Dev Size : 1953260544 sectors (931.39 GiB 1000.07 GB)
        Data Offset : 264192 sectors
    Super Offset : 8 sectors
    Unused Space : before=264112 sectors, after=432 sectors
            State : clean
        Device UUID : 0a40bde1:a7089d96:744e726d:5f700b85

    Internal Bitmap : 8 sectors from superblock
        Update Time : Fri Dec 27 09:46:03 2024
    Bad Block Log : 512 entries available at offset 16 sectors
        Checksum : 96af7161 - correct
            Events : 3986

            Layout : left-symmetric
        Chunk Size : 512K

    Device Role : Active device 0
    Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
    /dev/sdb:
            Magic : a92b4efc
            Version : 1.2
        Feature Map : 0x1
        Array UUID : ba946432:20362047:d8bfa70b:615817a1
            Name : rapture:0
    Creation Time : Fri Jul 26 16:06:17 2019
        Raid Level : raid5
    Raid Devices : 3

    Avail Dev Size : 1953260976 sectors (931.39 GiB 1000.07 GB)
        Array Size : 1953260544 KiB (1862.77 GiB 2000.14 GB)
    Used Dev Size : 1953260544 sectors (931.39 GiB 1000.07 GB)
        Data Offset : 264192 sectors
    Super Offset : 8 sectors
    Unused Space : before=264112 sectors, after=432 sectors
            State : clean
        Device UUID : 64dff6fe:57336347:4853f110:1aa160c1

    Internal Bitmap : 8 sectors from superblock
        Update Time : Fri Dec 27 09:46:03 2024
    Bad Block Log : 512 entries available at offset 16 sectors
        Checksum : 4483655b - correct
            Events : 3986

            Layout : left-symmetric
        Chunk Size : 512K

    Device Role : Active device 1
    Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
    /dev/sdc:
            Magic : a92b4efc
            Version : 1.2
        Feature Map : 0x1
        Array UUID : ba946432:20362047:d8bfa70b:615817a1
            Name : rapture:0
    Creation Time : Fri Jul 26 16:06:17 2019
        Raid Level : raid5
    Raid Devices : 3

    Avail Dev Size : 1953260976 sectors (931.39 GiB 1000.07 GB)
        Array Size : 1953260544 KiB (1862.77 GiB 2000.14 GB)
    Used Dev Size : 1953260544 sectors (931.39 GiB 1000.07 GB)
        Data Offset : 264192 sectors
    Super Offset : 8 sectors
    Unused Space : before=264112 sectors, after=432 sectors
            State : clean
        Device UUID : bcf0c123:37b5deb8:74af5640:37aa2be4

    Internal Bitmap : 8 sectors from superblock
        Update Time : Fri Dec 27 09:46:03 2024
    Bad Block Log : 512 entries available at offset 16 sectors
        Checksum : 2cfa5dde - correct
            Events : 3986

            Layout : left-symmetric
        Chunk Size : 512K

    Device Role : Active device 2
    Array State : AAA ('A' == active, '.' == missing, 'R' == replacing)
    ```

All disks show as `clean`, `active` and with the same `Events` count;
that’s good! At this point we want to **Assemble** your array,
**not** *create* it, and with the same `Events` count it shouldn’t
even be necessary to `--force` it:

``` console
# mdadm /dev/md0 --assemble /dev/sda /dev/sdb /dev/sdc
mdadm: /dev/md0 has been started with 3 drives.
```

*Success!* The RAID is now started, as seen in `dmesg`:

```
[ 9649.589111] md: md0 stopped.
[ 9649.593929]  sda:
[ 9649.596065]  sdb:
[ 9649.598005]  sdc:
[ 9649.649447] async_tx: api initialized (async)
[ 9649.661057] md/raid:md0: device sda operational as raid disk 0
[ 9649.661065] md/raid:md0: device sdc operational as raid disk 2
[ 9649.661068] md/raid:md0: device sdb operational as raid disk 1
[ 9649.661956] md/raid:md0: raid level 5 active with 3 out of 3 devices, algorithm 2
[ 9649.691338] md0: detected capacity change from 0 to 3906521088
```

Although not shown in `dmesg`, the BTRFS in it is also found:

```
# btrfs filesystem show
Label: none  uuid: c8d05650-0cbf-48c3-8616-7e942534ab4a
        Total devices 1 FS bytes used 1.12TiB
        devid    1 size 1.68TiB used 1.24TiB path /dev/nvme0n1p4

Label: none  uuid: abd8f19b-aff8-4c27-bbe9-d5b7f18bc18e
        Total devices 1 FS bytes used 1.51TiB
        devid    1 size 1.82TiB used 1.52TiB path /dev/md0
```

Now we can mount the array in `/home/raid` as it used to be:

```
# vi /etc/fstab
/dev/disk/by-uuid/c8d05650-0cbf-48c3-8616-7e942534ab4a /home btrfs defaults 0 1
/dev/disk/by-uuid/abd8f19b-aff8-4c27-bbe9-d5b7f18bc18e /home/raid btrfs defaults 0 1

# systemctl daemon-reload
# mount /home/raid
# df -h 
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/md0        1.9T  1.6T  309G  84% /home/raid
```

At this point a reboot would be in order, if only to make sure all
the above is still working well after a reboot.

!!! note

    In fact quite a few more things can be done before this second
    reboot, and indeed everything that follows down to
    [Continuous Monitoring](#continuous-monitoring) had been done
    before [restoring the RAID](#restore-software-raid-5).

## Install Additional Software

### Brave browser

Install from the
[Release Channel](https://brave.com/linux/#release-channel-installation):

``` console
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

``` console
# dpkg -i google-chrome-stable_current_amd64.deb
```

This could be done later, but it is convenient for me to easily
access services, documents and my self-hosted
[Visual Studio Code Server](2023-05-29-running-vs-code-server-on-kubernetes.md)
to write this blog.

### Continuous Monitoring

!!! warning

    This depends on `curl` being installed as part of the
    [Essential Packages](#install-essential-packages) above.

Install the
[multi-thread version](../../conmon.md#deploy-to-pcs)
of the `conmon` script as `/usr/local/bin/conmon` and
[run it as a service](../../conmon.md#install-conmon);
create `/etc/systemd/system/conmon.service` as follows:

``` ini linenums="1" title="/etc/systemd/system/conmon.service"
[Unit]
Description=Continuous Monitoring

[Service]
ExecStart=/usr/local/bin/conmon
Restart=on-failure
StandardOutput=null

[Install]
WantedBy=multi-user.target
```

**Before** starting the service, copy over the configuration files
under `/jammy/etc/conmon` and update the `TARGET_HTTP` and
`TARGET_HTTPS` in `/usr/local/bin/conmon` so that metrics are sent
to the remote server via **HTTPS**.
Then enable and start the service:

``` console
# vi /usr/local/bin/conmon
TARGET_HTTPS='https://inf.ssl.uu.am:443'

# cp -a /jammy/etc/conmon /etc/
# systemctl enable conmon.service
# systemctl daemon-reload
# systemctl start conmon.service
# systemctl status conmon.service
● conmon.service - Continuous Monitoring
     Loaded: loaded (/etc/systemd/system/conmon.service; enabled; preset: enabled)
     Active: active (running) since Fri 2024-12-27 12:18:57 CET; 3s ago
   Main PID: 1930626 (conmon)
      Tasks: 18 (limit: 38354)
     Memory: 6.0M (peak: 25.3M)
        CPU: 1.255s
     CGroup: /system.slice/conmon.service
```

#### Hardware Sensors

Only a few hardware sensors are supported out of the box,
sensors work for NVME, WiFi and CPU temperatures:

``` console
# sensors -A
iwlwifi_1-virtual-0
temp1:        +50.0°C  

nvme-pci-0100
Composite:    +50.9°C  (low  = -273.1°C, high = +84.8°C)
                       (crit = +84.8°C)
Sensor 1:     +50.9°C  (low  = -273.1°C, high = +65261.8°C)
Sensor 2:     +53.9°C  (low  = -273.1°C, high = +65261.8°C)

k10temp-pci-00c3
Tctl:         +37.9°C  
```

HDD temperatures are available after loading the `drivetemp` kernel module:

``` console
# echo drivetemp > /etc/modules-load.d/drivetemp.conf
# modprobe drivetemp
# sensors -A
drivetemp-scsi-5-0
temp1:        +27.0°C  (low  =  -5.0°C, high = +80.0°C)
                       (crit low = -10.0°C, crit = +85.0°C)
                       (lowest = +16.0°C, highest = +27.0°C)

drivetemp-scsi-0-0
temp1:        +25.0°C  (low  =  -5.0°C, high = +80.0°C)
                       (crit low = -10.0°C, crit = +85.0°C)
                       (lowest = +16.0°C, highest = +25.0°C)

drivetemp-scsi-4-0
temp1:        +29.0°C  (low  =  -5.0°C, high = +80.0°C)
                       (crit low = -10.0°C, crit = +85.0°C)
                       (lowest = +16.0°C, highest = +29.0°C)
```

##### Motherboard Sensors (NCT6795D)

The system board
[MSI B450I Gaming Plus AC](https://es.msi.com/Motherboard/B450I-GAMING-PLUS-AC/Specification)
initially shows only the South bridge (`k10temp`) sensors, but the
`sensors-detect` tool is able to detect that additional sensors are available in a `NCT6795D` compatible chip.

!!! warning

    **Do NOT** attempt scanning the NVidia i2c sensors, they
    [cause errors](https://www.google.com/search?q=nvidia-gpu+0000%3A29%3A00.3%3A+i2c+timeout+error+e0000000&oq=nvidia-gpu+0000%3A29%3A00.3%3A+i2c+timeout+error+e0000000&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIHCAEQIRiPAjIHCAIQIRiPAtIBBzI3NmowajeoAgCwAgA&sourceid=chrome&ie=UTF-8)
    (pending further investigation).

??? terminal "`# sensors-detect`"

    ``` console hl_lines="48 49 103-105 110-114"
    root@raven:~# sensors-detect
    # sensors-detect version 3.6.0
    # System: Micro-Star International Co., Ltd. MS-7A40 [2.0]
    # Board: Micro-Star International Co., Ltd. B450I GAMING PLUS AC (MS-7A40)
    # Kernel: 6.8.0-50-lowlatency x86_64
    # Processor: AMD Ryzen 5 2600X Six-Core Processor (23/8/2)

    This program will help you determine which kernel modules you need
    to load to use lm_sensors most effectively. It is generally safe
    and recommended to accept the default answers to all questions,
    unless you know what you're doing.

    Some south bridges, CPUs or memory controllers contain embedded sensors.
    Do you want to scan for them? This is totally safe. (YES/no): 
    Module cpuid loaded successfully.
    Silicon Integrated Systems SIS5595...                       No
    VIA VT82C686 Integrated Sensors...                          No
    VIA VT8231 Integrated Sensors...                            No
    AMD K8 thermal sensors...                                   No
    AMD Family 10h thermal sensors...                           No
    AMD Family 11h thermal sensors...                           No
    AMD Family 12h and 14h thermal sensors...                   No
    AMD Family 15h thermal sensors...                           No
    AMD Family 16h thermal sensors...                           No
    AMD Family 17h thermal sensors...                           Success!
        (driver `k10temp')
    AMD Family 15h power sensors...                             No
    AMD Family 16h power sensors...                             No
    Hygon Family 18h thermal sensors...                         No
    Intel digital thermal sensor...                             No
    Intel AMB FB-DIMM thermal sensor...                         No
    Intel 5500/5520/X58 thermal sensor...                       No
    VIA C7 thermal sensor...                                    No
    VIA Nano thermal sensor...                                  No

    Some Super I/O chips contain embedded sensors. We have to write to
    standard I/O ports to probe them. This is usually safe.
    Do you want to scan for Super I/O sensors? (YES/no): 
    Probing for Super-I/O at 0x2e/0x2f
    Trying family `National Semiconductor/ITE'...               No
    Trying family `SMSC'...                                     No
    Trying family `VIA/Winbond/Nuvoton/Fintek'...               No
    Trying family `ITE'...                                      No
    Probing for Super-I/O at 0x4e/0x4f
    Trying family `National Semiconductor/ITE'...               No
    Trying family `SMSC'...                                     No
    Trying family `VIA/Winbond/Nuvoton/Fintek'...               Yes
    Found `Nuvoton NCT6795D Super IO Sensors'                   Success!
        (address 0xa20, driver `nct6775')

    Some systems (mainly servers) implement IPMI, a set of common interfaces
    through which system health data may be retrieved, amongst other things.
    We first try to get the information from SMBIOS. If we don't find it
    there, we have to read from arbitrary I/O ports to probe for such
    interfaces. This is normally safe. Do you want to scan for IPMI
    interfaces? (YES/no): 
    Probing for `IPMI BMC KCS' at 0xca0...                      No
    Probing for `IPMI BMC SMIC' at 0xca8...                     No

    Some hardware monitoring chips are accessible through the ISA I/O ports.
    We have to write to arbitrary I/O ports to probe them. This is usually
    safe though. Yes, you do have ISA I/O ports even if you do not have any
    ISA slots! Do you want to scan the ISA I/O ports? (yes/NO): 

    Lastly, we can probe the I2C/SMBus adapters for connected hardware
    monitoring devices. This is the most risky part, and while it works
    reasonably well on most systems, it has been reported to cause trouble
    on some systems.
    Do you want to probe the I2C/SMBus adapters now? (YES/no): 
    Using driver `i2c-piix4' for device 0000:00:14.0: AMD KERNCZ SMBus

    Next adapter: SMBus PIIX4 adapter port 0 at 0b00 (i2c-0)
    Do you want to scan it? (yes/NO/selectively): 

    Next adapter: SMBus PIIX4 adapter port 2 at 0b00 (i2c-1)
    Do you want to scan it? (yes/NO/selectively): 

    Next adapter: SMBus PIIX4 adapter port 1 at 0b20 (i2c-2)
    Do you want to scan it? (yes/NO/selectively): 

    Next adapter: NVIDIA GPU I2C adapter (i2c-3)
    Do you want to scan it? (YES/no/selectively): no

    Next adapter: NVIDIA i2c adapter 1 at 29:00.0 (i2c-4)
    Do you want to scan it? (yes/NO/selectively): no

    Next adapter: NVIDIA i2c adapter 3 at 29:00.0 (i2c-5)
    Do you want to scan it? (yes/NO/selectively): no

    Next adapter: NVIDIA i2c adapter 4 at 29:00.0 (i2c-6)
    Do you want to scan it? (yes/NO/selectively): no

    Next adapter: NVIDIA i2c adapter 5 at 29:00.0 (i2c-7)
    Do you want to scan it? (yes/NO/selectively): no

    Next adapter: NVIDIA i2c adapter 6 at 29:00.0 (i2c-8)
    Do you want to scan it? (yes/NO/selectively): no


    Now follows a summary of the probes I have just done.
    Just press ENTER to continue: 

    Driver `nct6775':
    * ISA bus, address 0xa20
        Chip `Nuvoton NCT6795D Super IO Sensors' (confidence: 9)

    Driver `k10temp' (autoloaded):
    * Chip `AMD Family 17h thermal sensors' (confidence: 9)

    To load everything that is needed, add this to /etc/modules:
    #----cut here----
    # Chip drivers
    nct6775
    #----cut here----
    If you have some drivers built into your kernel, the list above will
    contain too many modules. Skip the appropriate ones!

    Do you want to add these lines automatically to /etc/modules? (yes/NO)

    Unloading cpuid... OK
    ```

So the only other module that needs to be loaded is `nct6775`,
and with that all sensors are available (including fan speeds):

``` console
# echo nct6775 > /etc/modules-load.d/nct6775.conf
# modprobe nct6775
```

??? terminal "`# sensors -A`"

    ``` console
    # sensors -A
    drivetemp-scsi-5-0
    temp1:        +27.0°C  (low  =  -5.0°C, high = +80.0°C)
                        (crit low = -10.0°C, crit = +85.0°C)
                        (lowest = +16.0°C, highest = +27.0°C)

    drivetemp-scsi-0-0
    temp1:        +24.0°C  (low  =  -5.0°C, high = +80.0°C)
                        (crit low = -10.0°C, crit = +85.0°C)
                        (lowest = +16.0°C, highest = +25.0°C)

    iwlwifi_1-virtual-0
    temp1:        +49.0°C  

    nvme-pci-0100
    Composite:    +48.9°C  (low  = -273.1°C, high = +84.8°C)
                        (crit = +84.8°C)
    Sensor 1:     +48.9°C  (low  = -273.1°C, high = +65261.8°C)
    Sensor 2:     +52.9°C  (low  = -273.1°C, high = +65261.8°C)

    nct6795-isa-0a20
    Vcore:                   1.07 V  (min =  +0.00 V, max =  +1.74 V)
    in1:                     1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
    AVCC:                    3.39 V  (min =  +2.98 V, max =  +3.63 V)
    +3.3V:                   3.31 V  (min =  +2.98 V, max =  +3.63 V)
    in4:                     1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
    in5:                   152.00 mV (min =  +0.00 V, max =  +0.00 V)  ALARM
    in6:                   728.00 mV (min =  +0.00 V, max =  +0.00 V)  ALARM
    3VSB:                    3.38 V  (min =  +2.98 V, max =  +3.63 V)
    Vbat:                    3.28 V  (min =  +2.70 V, max =  +3.63 V)
    in9:                     1.84 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
    in10:                    0.00 V  (min =  +0.00 V, max =  +0.00 V)
    in11:                  632.00 mV (min =  +0.00 V, max =  +0.00 V)  ALARM
    in12:                  824.00 mV (min =  +0.00 V, max =  +0.00 V)  ALARM
    in13:                  592.00 mV (min =  +0.00 V, max =  +0.00 V)  ALARM
    in14:                    1.53 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
    fan1:                     0 RPM  (min =    0 RPM)
    fan2:                  1143 RPM  (min =    0 RPM)
    fan3:                   925 RPM  (min =    0 RPM)
    fan4:                     0 RPM  (min =    0 RPM)
    fan5:                     0 RPM  (min =    0 RPM)
    SYSTIN:                 +41.0°C  (high =  +0.0°C, hyst =  +0.0°C)  ALARM  sensor = CPU diode
    CPUTIN:                 +36.0°C  (high = +115.0°C, hyst = +90.0°C)  sensor = thermistor
    AUXTIN0:                +41.0°C  (high = +115.0°C, hyst = +90.0°C)  sensor = thermistor
    AUXTIN1:               -128.0°C    sensor = thermistor
    AUXTIN2:                +47.0°C    sensor = thermistor
    AUXTIN3:                 -2.0°C    sensor = thermistor
    SMBUSMASTER 0:          +34.0°C  
    PCH_CHIP_CPU_MAX_TEMP:   +0.0°C  
    PCH_CHIP_TEMP:           +0.0°C  
    PCH_CPU_TEMP:            +0.0°C  
    PCH_MCH_TEMP:            +0.0°C  
    Agent0 Dimm0:            +0.0°C  
    TSI0_TEMP:              +34.4°C  
    intrusion0:            ALARM
    intrusion1:            ALARM
    beep_enable:           disabled

    drivetemp-scsi-4-0
    temp1:        +28.0°C  (low  =  -5.0°C, high = +80.0°C)
                        (crit low = -10.0°C, crit = +85.0°C)
                        (lowest = +16.0°C, highest = +29.0°C)

    k10temp-pci-00c3
    Tctl:         +34.4°C
    ```

#### Speedtest and Tapo devices

While there is no 24x7 system on-site, this PC also has to run
some of the [additional scripts](../../conmon.md#additional-scripts)
to monitor internet speeds with
[`conmon-speedtest`](../../conmon.md#conmon-speedtest)
and TP-Link Tapo devices wit
[`conmon-tapo.py`](../../conmon.md#conmon-tapopy).

``` console
# cp conmon-speedtest conmon-tapo.py /usr/local/bin/
# crontab -e
*/3 * * * * /usr/local/bin/conmon-speedtest
*   * * * * /usr/local/bin/conmon-tapo.py
```

!!! warning

    Make sure `/etc/conmon/tapo.yaml` maps to the correct IP addresses
    and models for each device, otherwise `conmon-tapo.py`
    [will crash](../../conmon.md#device-ip-dependencies).

### Blender

[Blender 4.3](https://www.blender.org/) is already available
even for Ubuntu 24.04 via 
[snapcraft.io/blender](https://snapcraft.io/blender)
so there is no reason to install it any other way:

``` console
# snap install blender --classic
blender 4.3.2 from Blender Foundation (blenderfoundation✓) installed
```

### Steam

Installing Steam from Snap 
[couldn't be simpler](https://unixhint.com/install-steam-on-ubuntu-24-04/):

``` console
# snap install steam
steam 1.0.0.81 from Canonical✓ installed
```

This [snapcraft.io/steam](https://snapcraft.io/steam) package is provided by Canonical.

When runing the Steam client for the first time, a pop-up shows up
advising to install additional 32-bit drivers *for best experience*

??? terminal "`# apt install libnvidia-gl-550:i386 -y`"

    ``` console
    # dpkg --add-architecture i386
    # apt update
    # apt install libnvidia-gl-550:i386 -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
    gcc-14-base:i386 libbsd0:i386 libc6:i386 libdrm2:i386 libffi8:i386 libgcc-s1:i386
    libidn2-0:i386 libmd0:i386 libnvidia-egl-wayland1:i386 libunistring5:i386
    libwayland-client0:i386 libwayland-server0:i386 libx11-6:i386 libxau6:i386 libxcb1:i386
    libxdmcp6:i386 libxext6:i386
    Suggested packages:
    glibc-doc:i386 locales:i386 libnss-nis:i386 libnss-nisplus:i386
    The following NEW packages will be installed:
    gcc-14-base:i386 libbsd0:i386 libc6:i386 libdrm2:i386 libffi8:i386 libgcc-s1:i386
    libidn2-0:i386 libmd0:i386 libnvidia-egl-wayland1:i386 libnvidia-gl-550:i386
    libunistring5:i386 libwayland-client0:i386 libwayland-server0:i386 libx11-6:i386
    libxau6:i386 libxcb1:i386 libxdmcp6:i386 libxext6:i386
    0 upgraded, 18 newly installed, 0 to remove and 3 not upgraded.
    Need to get 38.6 MB of archives.
    After this operation, 155 MB of additional disk space will be used.
    ...
    Fetched 38.6 MB in 4s (11.0 MB/s)                  
    Preconfiguring packages ...
    Selecting previously unselected package gcc-14-base:i386.
    (Reading database ... 425851 files and directories currently installed.)
    Preparing to unpack .../00-gcc-14-base_14.2.0-4ubuntu2~24.04_i386.deb ...
    Unpacking gcc-14-base:i386 (14.2.0-4ubuntu2~24.04) ...
    Selecting previously unselected package libgcc-s1:i386.
    Preparing to unpack .../01-libgcc-s1_14.2.0-4ubuntu2~24.04_i386.deb ...
    Unpacking libgcc-s1:i386 (14.2.0-4ubuntu2~24.04) ...
    Selecting previously unselected package libc6:i386.
    Preparing to unpack .../02-libc6_2.39-0ubuntu8.3_i386.deb ...
    Unpacking libc6:i386 (2.39-0ubuntu8.3) ...
    Selecting previously unselected package libmd0:i386.
    Preparing to unpack .../03-libmd0_1.1.0-2build1_i386.deb ...
    Unpacking libmd0:i386 (1.1.0-2build1) ...
    Selecting previously unselected package libbsd0:i386.
    Preparing to unpack .../04-libbsd0_0.12.1-1build1_i386.deb ...
    Unpacking libbsd0:i386 (0.12.1-1build1) ...
    Selecting previously unselected package libffi8:i386.
    Preparing to unpack .../05-libffi8_3.4.6-1build1_i386.deb ...
    Unpacking libffi8:i386 (3.4.6-1build1) ...
    Selecting previously unselected package libunistring5:i386.
    Preparing to unpack .../06-libunistring5_1.1-2build1_i386.deb ...
    Unpacking libunistring5:i386 (1.1-2build1) ...
    Selecting previously unselected package libidn2-0:i386.
    Preparing to unpack .../07-libidn2-0_2.3.7-2build1_i386.deb ...
    Unpacking libidn2-0:i386 (2.3.7-2build1) ...
    Selecting previously unselected package libdrm2:i386.
    Preparing to unpack .../08-libdrm2_2.4.120-2build1_i386.deb ...
    Unpacking libdrm2:i386 (2.4.120-2build1) ...
    Selecting previously unselected package libxau6:i386.
    Preparing to unpack .../09-libxau6_1%3a1.0.9-1build6_i386.deb ...
    Unpacking libxau6:i386 (1:1.0.9-1build6) ...
    Selecting previously unselected package libxdmcp6:i386.
    Preparing to unpack .../10-libxdmcp6_1%3a1.1.3-0ubuntu6_i386.deb ...
    Unpacking libxdmcp6:i386 (1:1.1.3-0ubuntu6) ...
    Selecting previously unselected package libxcb1:i386.
    Preparing to unpack .../11-libxcb1_1.15-1ubuntu2_i386.deb ...
    Unpacking libxcb1:i386 (1.15-1ubuntu2) ...
    Selecting previously unselected package libx11-6:i386.
    Preparing to unpack .../12-libx11-6_2%3a1.8.7-1build1_i386.deb ...
    Unpacking libx11-6:i386 (2:1.8.7-1build1) ...
    Selecting previously unselected package libxext6:i386.
    Preparing to unpack .../13-libxext6_2%3a1.3.4-1build2_i386.deb ...
    Unpacking libxext6:i386 (2:1.3.4-1build2) ...
    Selecting previously unselected package libwayland-client0:i386.
    Preparing to unpack .../14-libwayland-client0_1.22.0-2.1build1_i386.deb ...
    Unpacking libwayland-client0:i386 (1.22.0-2.1build1) ...
    Selecting previously unselected package libwayland-server0:i386.
    Preparing to unpack .../15-libwayland-server0_1.22.0-2.1build1_i386.deb ...
    Unpacking libwayland-server0:i386 (1.22.0-2.1build1) ...
    Selecting previously unselected package libnvidia-egl-wayland1:i386.
    Preparing to unpack .../16-libnvidia-egl-wayland1_1%3a1.1.13-1build1_i386.deb ...
    Unpacking libnvidia-egl-wayland1:i386 (1:1.1.13-1build1) ...
    Selecting previously unselected package libnvidia-gl-550:i386.
    Preparing to unpack .../17-libnvidia-gl-550_550.120-0ubuntu0.24.04.1_i386.deb ...
    Unpacking libnvidia-gl-550:i386 (550.120-0ubuntu0.24.04.1) ...
    Setting up gcc-14-base:i386 (14.2.0-4ubuntu2~24.04) ...
    Setting up libgcc-s1:i386 (14.2.0-4ubuntu2~24.04) ...
    Setting up libc6:i386 (2.39-0ubuntu8.3) ...
    Setting up libffi8:i386 (3.4.6-1build1) ...
    Setting up libdrm2:i386 (2.4.120-2build1) ...
    Setting up libmd0:i386 (1.1.0-2build1) ...
    Setting up libbsd0:i386 (0.12.1-1build1) ...
    Setting up libwayland-client0:i386 (1.22.0-2.1build1) ...
    Setting up libwayland-server0:i386 (1.22.0-2.1build1) ...
    Setting up libxau6:i386 (1:1.0.9-1build6) ...
    Setting up libxdmcp6:i386 (1:1.1.3-0ubuntu6) ...
    Setting up libxcb1:i386 (1.15-1ubuntu2) ...
    Setting up libnvidia-egl-wayland1:i386 (1:1.1.13-1build1) ...
    Setting up libunistring5:i386 (1.1-2build1) ...
    Setting up libx11-6:i386 (2:1.8.7-1build1) ...
    Setting up libxext6:i386 (2:1.3.4-1build2) ...
    Setting up libidn2-0:i386 (2.3.7-2build1) ...
    Setting up libnvidia-gl-550:i386 (550.120-0ubuntu0.24.04.1) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
    ```

It should also be possible to install the official Steam client, with
[the non-snap alternative](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#non-snap-alternative).
This doesn't seems necessary anymore.

### Itch.io

There is a binary in `.itch/itch` but it doesn’t work, it seem to have launched the app but the app itself is nowhere to be seen:

??? terminal "`$ .itch/itch`"

    ``` console
    $ .itch/itch
    2024/12/27 20:26:33 itch-setup will log to /tmp/itch-setup-log.txt
    2024/12/27 20:26:33 =========================================
    2024/12/27 20:26:33 itch-setup "v1.26.0, built on Apr 22 2021 @ 03:48:12, ref 48f97b3e7b0b065a2478811b8d0ebcae414845fd" starting up at "2024-12-27 20:26:33.155012082 +0100 CET m=+0.004615449" with arguments:
    2024/12/27 20:26:33 "/home/coder/.itch/itch-setup"
    2024/12/27 20:26:33 "--prefer-launch"
    2024/12/27 20:26:33 "--appname"
    2024/12/27 20:26:33 "itch"
    2024/12/27 20:26:33 "--"
    2024/12/27 20:26:33 =========================================
    2024/12/27 20:26:33 App name specified on command-line: itch
    2024/12/27 20:26:33 Locale:  en-US
    2024/12/27 20:26:33 Initializing installer GUI...
    2024/12/27 20:26:33 Using GTK UI

    (process:748360): Gtk-WARNING **: 20:26:33.157: Locale not supported by C library.
            Using the fallback 'C' locale.
    2024/12/27 20:26:33 Initializing (itch) multiverse @ (/home/coder/.itch)
    2024/12/27 20:26:33 (/home/coder/.itch)(current = "26.1.9", ready = "")
    2024/12/27 20:26:33 Launch preferred, attempting...
    2024/12/27 20:26:33 Launching (26.1.9) from (/home/coder/.itch/app-26.1.9)
    2024/12/27 20:26:33 Kernel should support SUID sandboxing, leaving it enabled
    2024/12/27 20:26:33 App launched, getting out of the way
    ```

The solution, albeit possibly only a temporary one, is to
[disable sandboxing](https://www.reddit.com/r/pop_os/comments/uocj8p/itchio_launcher_crashing_on_startup_ever_sense/)
([source](https://itch.io/t/1760026/itch-app-official-is-closing-at-launch-fedora-linux)) by adding the
`--no-sandbox` in `.itch/itch`:

``` bash linenums="1" title=".itch/itch"
#!/bin/sh
/home/coder/.itch/itch-setup \
  --prefer-launch --appname itch \
  -- --no-sandbox "$@"
```

### Minecraft Java Edition

To avoid taking chances, copy the Minecraft launcher from the
previous system:

``` console
# cp -a /jammy/opt/minecraft-launcher/ /opt/
```

It works perfectly right after installing and logging in again.

In contrast, trying to re-download Minecraft Java Edition
[seems to lead nowhere good](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#minecraft).

### Minecraft Bedrock Edition

There is an **unofficial**
[Minecraft Bedrock Launcher](https://flathub.org/apps/io.mrarm.mcpelauncher),
including smiple steps to install it on
[Debian / Ubuntu](https://mcpelauncher.readthedocs.io/en/latest/getting_started/index.html#debian-ubuntu).
This has not seemed necessary so far, since the family enjoys
playing the Java edition more.

### Arduino IDE

There no Arduino IDE in Ubuntu 22.04 (in `/jammy/opt/arduino`) and 
[even if there was it would be out of date](2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#arduino-ide),
so it pays to install the latest/nightly version:

``` console
# wget https://downloads.arduino.cc/arduino-ide/nightly/arduino-ide_nightly-latest_Linux_64bit.zip
# unzip arduino-ide_nightly-latest_Linux_64bit.zip
# mv arduino-ide_nightly-20241212_Linux_64bit/ /opt/arduino/
# chmod 4755 /opt/arduino/chrome-sandbox
```

Upon launching the Arduino IDE, a notification card offers updating
installed libraries, which comes in vary handy to update them all.

??? warning "Without the `chmod 4755` command, the IDE refuses to run."

    ``` console
    $ /opt/arduino/arduino-ide
    [1917080:1107/234610.122185:FATAL:setuid_sandbox_host.cc(158)] The SUID sandbox helper binary was found, but is not configured correctly. Rather than run without sandboxing I'm aborting now. You need to make sure that /opt/arduino/chrome-sandbox is owned by root and has mode 4755.
    Trace/breakpoint trap (core dumped)
    ```

#### Arduino failed updates

Unfortunately at least one of the libraries in the Arduino IDE hit a
snag and got stuck with an exception, seemingly because NVidia cards
and/or drivers are not supported.

??? terminal "Arduino IDE stuck updating libraries."

    ``` console hl_lines="18"
    $ /opt/arduino/arduino-ide
    Arduino IDE 2.3.5-nightly-20241212
    ...

    2024-12-27T19:33:51.252Z daemon INFO time="2024-12-27T20:33:51+01:00" level=info msg="Loaded tool" tool="esp8266:mklittlefs@3.1.0-gcc10.3-e5f9fec"
    time="2024-12-27T20:33:51+01:00" level=info msg="Loaded tool" tool="esp8266:mkspiffs@3.1.0-gcc10.3-e5f9fec"
    time="2024-12-27T20:33:51+01:00" level=info msg="Loaded tool" tool="esp8266:python3@3.7.2-post1"
    2024-12-27T19:33:51.252Z daemon INFO time="2024-12-27T20:33:51+01:00" level=info msg="Loaded tool" tool="esp8266:xtensa-lx106-elf-gcc@3.1.0-gcc10.3-e5f9fec"
    Warning: loader_get_json: Failed to open JSON file virtio_icd.x86_64.json
    Warning: loader_scanned_icd_add: Could not get 'vkCreateInstance' via 'vk_icdGetInstanceProcAddr' for ICD libGLX_nvidia.so.0
    Warning: loader_get_json: Failed to open JSON file lvp_icd.x86_64.json
    Warning: /usr/lib/x86_64-linux-gnu/libvulkan_intel.so: cannot open shared object file: Permission denied
    Warning: loader_icd_scan: Failed loading library associated with ICD JSON /usr/lib/x86_64-linux-gnu/libvulkan_intel.so. Ignoring this JSON
    Warning: /usr/lib/x86_64-linux-gnu/libvulkan_radeon.so: cannot open shared object file: Permission denied
    Warning: loader_icd_scan: Failed loading library associated with ICD JSON /usr/lib/x86_64-linux-gnu/libvulkan_radeon.so. Ignoring this JSON
    Warning: loader_get_json: Failed to open JSON file intel_hasvk_icd.x86_64.json
    Warning: vkCreateInstance: Found no drivers!
    Warning: vkCreateInstance failed with VK_ERROR_INCOMPATIBLE_DRIVER
        at CheckVkSuccessImpl (../../third_party/dawn/src/dawn/native/vulkan/VulkanError.cpp:88)
        at CreateVkInstance (../../third_party/dawn/src/dawn/native/vulkan/BackendVk.cpp:458)
        at Initialize (../../third_party/dawn/src/dawn/native/vulkan/BackendVk.cpp:344)
        at Create (../../third_party/dawn/src/dawn/native/vulkan/BackendVk.cpp:266)
        at operator() (../../third_party/dawn/src/dawn/native/vulkan/BackendVk.cpp:521)



    ^CisTempSketch: true. Input was /tmp/.arduinoIDE-unsaved20241127-911572-vfe16y.lc64g/sketch_dec27a
    Ignored marking workspace as a closed sketch. The sketch was detected as temporary. Workspace URI: file:///tmp/.arduinoIDE-unsaved20241127-911572-vfe16y.lc64g/sketch_dec27a.
    isTempSketch: true. Input was /tmp/.arduinoIDE-unsaved20241127-911572-vfe16y.lc64g/sketch_dec27a
    Ignored marking workspace as a closed sketch. The sketch was detected as temporary. Workspace URI: file:///tmp/.arduinoIDE-unsaved20241127-911572-vfe16y.lc64g/sketch_dec27a.
    Closing channel on service path '/services/electron-window'.
    Closing channel on service path '/services/ide-updater'.
    Stored workspaces roots: 
    No sketches were scheduled for deletion.
    ```

Searching around for that specific error, all results are about issues
for non-NVidia GPUs and thus not applicable. There is a hint of a
specific library being problematic in the output of 
`vulkaninfo --summary` but that
[seems to be a red herring](https://gitlab.freedesktop.org/mesa/mesa/-/issues/10753#note_2312512).

??? terminal "`$ vulkaninfo --summary`"

    ``` console hl_lines="2"
    $ vulkaninfo --summary
    WARNING: [Loader Message] Code 0 : terminator_CreateInstance: Received return code -3 from call to vkCreateInstance in ICD /usr/lib/x86_64-linux-gnu/libvulkan_virtio.so. Skipping this driver.
    ==========
    VULKANINFO
    ==========

    Vulkan Instance Version: 1.3.275


    Instance Extensions: count = 23
    -------------------------------
    VK_EXT_acquire_drm_display             : extension revision 1
    VK_EXT_acquire_xlib_display            : extension revision 1
    VK_EXT_debug_report                    : extension revision 10
    VK_EXT_debug_utils                     : extension revision 2
    VK_EXT_direct_mode_display             : extension revision 1
    VK_EXT_display_surface_counter         : extension revision 1
    VK_EXT_surface_maintenance1            : extension revision 1
    VK_EXT_swapchain_colorspace            : extension revision 4
    VK_KHR_device_group_creation           : extension revision 1
    VK_KHR_display                         : extension revision 23
    VK_KHR_external_fence_capabilities     : extension revision 1
    VK_KHR_external_memory_capabilities    : extension revision 1
    VK_KHR_external_semaphore_capabilities : extension revision 1
    VK_KHR_get_display_properties2         : extension revision 1
    VK_KHR_get_physical_device_properties2 : extension revision 2
    VK_KHR_get_surface_capabilities2       : extension revision 1
    VK_KHR_portability_enumeration         : extension revision 1
    VK_KHR_surface                         : extension revision 25
    VK_KHR_surface_protected_capabilities  : extension revision 1
    VK_KHR_wayland_surface                 : extension revision 6
    VK_KHR_xcb_surface                     : extension revision 6
    VK_KHR_xlib_surface                    : extension revision 6
    VK_LUNARG_direct_driver_loading        : extension revision 1

    Instance Layers: count = 8
    --------------------------
    VK_LAYER_INTEL_nullhw             INTEL NULL HW                1.1.73   version 1
    VK_LAYER_MESA_device_select       Linux device selection layer 1.3.211  version 1
    VK_LAYER_MESA_overlay             Mesa Overlay layer           1.3.211  version 1
    VK_LAYER_NV_optimus               NVIDIA Optimus layer         1.3.277  version 1
    VK_LAYER_VALVE_steam_fossilize_32 Steam Pipeline Caching Layer 1.3.207  version 1
    VK_LAYER_VALVE_steam_fossilize_64 Steam Pipeline Caching Layer 1.3.207  version 1
    VK_LAYER_VALVE_steam_overlay_32   Steam Overlay Layer          1.3.207  version 1
    VK_LAYER_VALVE_steam_overlay_64   Steam Overlay Layer          1.3.207  version 1

    Devices:
    ========
    GPU0:
            apiVersion         = 1.3.277
            driverVersion      = 550.120.0.0
            vendorID           = 0x10de
            deviceID           = 0x2182
            deviceType         = PHYSICAL_DEVICE_TYPE_DISCRETE_GPU
            deviceName         = NVIDIA GeForce GTX 1660 Ti
            driverID           = DRIVER_ID_NVIDIA_PROPRIETARY
            driverName         = NVIDIA
            driverInfo         = 550.120
            conformanceVersion = 1.3.7.2
            deviceUUID         = f77cc413-629c-e437-1b69-3c3edb0915b8
            driverUUID         = b0238158-7345-5c56-8ef8-c3afdfd4908a
    GPU1:
            apiVersion         = 1.3.274
            driverVersion      = 0.0.1
            vendorID           = 0x10005
            deviceID           = 0x0000
            deviceType         = PHYSICAL_DEVICE_TYPE_CPU
            deviceName         = llvmpipe (LLVM 17.0.6, 256 bits)
            driverID           = DRIVER_ID_MESA_LLVMPIPE
            driverName         = llvmpipe
            driverInfo         = Mesa 24.0.9-0ubuntu0.3 (LLVM 17.0.6)
            conformanceVersion = 1.3.1.1
            deviceUUID         = 6d657361-3234-2e30-2e39-2d3075627500
            driverUUID         = 6c6c766d-7069-7065-5555-494400000000
    ```

### Visual Studio Code

Since the young artist wanted to use
[Visual Studio Code](https://code.visualstudio.com/)
to comfortably edit websites, I got around to try it out and
rather like how it uses more of the screen than the self-hosted
[Visual Studio Code Server](2023-05-29-running-vs-code-server-on-kubernetes.md),
and the screen on this PC is significantly smaller so
this should come in handy.

[Installation](https://code.visualstudio.com/docs/setup/linux) is fairly
simple, so much a single `.deb` file can be installed directly, but the
apt repository can also be installed manually with the following script:

``` console
# apt install apt-transport-https gpg wget -y
# wget -qO- https://packages.microsoft.com/keys/microsoft.asc \
  | gpg --dearmor > packages.microsoft.gpg
# install -D -o root -g root -m 644 packages.microsoft.gpg \
  /etc/apt/keyrings/packages.microsoft.gpg
# echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" \
  > /etc/apt/sources.list.d/vscode.list
# rm -f packages.microsoft.gpg
# apt update
# apt install code -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  code
0 upgraded, 1 newly installed, 0 to remove and 3 not upgraded.
Need to get 105 MB of archives.
After this operation, 424 MB of additional disk space will be used.
Get:1 https://packages.microsoft.com/repos/code stable/main amd64 code amd64 1.96.2-1734607745 [105 MB]
Fetched 105 MB in 4s (29.7 MB/s)
Preconfiguring packages ...
Selecting previously unselected package code.
(Reading database ... 426277 files and directories currently installed.)
Preparing to unpack .../code_1.96.2-1734607745_amd64.deb ...
Unpacking code (1.96.2-1734607745) ...
Setting up code (1.96.2-1734607745) ...
Warning in file "/usr/share/applications/displaycal-vrml-to-x3d-converter.desktop": usage of MIME type "x-world/x-vrml" is discouraged (the use of "x-world" as media type is strongly discouraged in favor of a subtype of the "application" media type)
Warning in file "/usr/share/applications/displaycal-vrml-to-x3d-converter.desktop": usage of MIME type "x-world/x-vrml" is discouraged (the use of "x-world" as media type is strongly discouraged in favor of a subtype of the "application" media type)
Processing triggers for shared-mime-info (2.4-4) ...
Processing triggers for desktop-file-utils (0.27-2build1) ...
```

### DisplayCal

[DisplayCAL](https://displaycal.net/) is no longer maintained, it was dropped from
Ubuntu 20.04 because it would not work with Python 3, but was still possible to build
with python2.7 packages. Later, that was no longer possible in Ubuntu 22.04, so a new
port to Python 3 was started: the
[DisplayCAL Python 3 Project](https://github.com/eoyilmaz/displaycal-py3/tree/develop?tab=readme-ov-file#displaycal-python-3-project).

Back in late 2022, the best method around was in (French)
[DisplayCAL en Python 3](https://ignace72.eu/displaycal-en-python-3.html)
and required only a few basic packages.

As or late 2024, the new project has its own
[Installation Instructions (Linux)](https://github.com/eoyilmaz/displaycal-py3/blob/develop/docs/install_instructions_linux.md)
but in Ubuntu Studio 24.04 none of this is necessary; `apt install displaycal` will do.

## System Configuration

The above having covered **installing** software, there are still
system configurations that need to be tweaked.

### Ubuntu Pro

When updating the system with `apt full-upgrade -y` a notice comes
up about additional security updates:

``` console
# apt full-upgrade -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
Get more security updates through Ubuntu Pro with 'esm-apps' enabled:
  libdcmtk17t64 python3-waitress libcjson1 libavdevice60 ffmpeg libpostproc57
  libavcodec60 libavutil58 libswscale7 libswresample4 libavformat60
  libavfilter9
Learn more about Ubuntu Pro at https://ubuntu.com/pro
The following upgrades have been deferred due to phasing:
  python3-distupgrade ubuntu-release-upgrader-core ubuntu-release-upgrader-qt
0 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
```

This being a new system, indeed it's not attached to an Ubuntu Pro
account (the old system was):

``` console
# pro security-status
installed:
     1645 packages from Ubuntu Main/Restricted repository
     1581 packages from Ubuntu Universe/Multiverse repository
     3 packages from third parties

To get more information about the packages, run
    pro security-status --help
for a list of available options.

This machine is receiving security patching for Ubuntu Main/Restricted
repository until 2029.
This machine is NOT attached to an Ubuntu Pro subscription.

Ubuntu Pro with 'esm-infra' enabled provides security updates for
Main/Restricted packages until 2034.

Ubuntu Pro with 'esm-apps' enabled provides security updates for
Universe/Multiverse packages until 2034. There are 12 pending security updates.

Try Ubuntu Pro with a free personal subscription on up to 5 machines.
Learn more at https://ubuntu.com/pro
```

After creating an Ubuntu account a token is available to use with
`pro attach`:

``` console
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
The current kernel (6.8.0-50-lowlatency, x86_64) is not covered by livepatch.
Covered kernels are listed here: https://ubuntu.com/security/livepatch/docs/kernels
Either switch to a covered kernel or `sudo pro disable livepatch` to dismiss this warning.

For a list of all Ubuntu Pro services and variants, run 'pro status --all'
Enable services with: pro enable <service>

    Account: ponder.stibbons@uu.am
Subscription: Ubuntu Pro - free personal subscription

```

Now the system can be updated *again* with `apt full-upgrade -y`
to receive those additional security updates:

??? terminal "`# apt full-upgrade -y`"

    ``` console
    # apt full-upgrade -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Calculating upgrade... Done
    The following upgrades have been deferred due to phasing:
    python3-distupgrade ubuntu-release-upgrader-core ubuntu-release-upgrader-qt
    The following packages will be upgraded:
    ffmpeg libavcodec60 libavdevice60 libavfilter9 libavformat60 libavutil58 libcjson1
    libdcmtk17t64 libpostproc57 libswresample4 libswscale7 python3-waitress
    12 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
    12 esm-apps security updates
    Need to get 19.3 MB of archives.
    After this operation, 6,144 B of additional disk space will be used.
    Preparing to unpack .../00-libswscale7_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libswscale7:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../01-libavdevice60_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libavdevice60:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../02-libavformat60_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libavformat60:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../03-libavfilter9_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libavfilter9:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../04-libavcodec60_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libavcodec60:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../05-libavutil58_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libavutil58:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../06-libpostproc57_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libpostproc57:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../07-libswresample4_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking libswresample4:amd64 (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../08-ffmpeg_7%3a6.1.1-3ubuntu5+esm2_amd64.deb ...
    Unpacking ffmpeg (7:6.1.1-3ubuntu5+esm2) over (7:6.1.1-3ubuntu5) ...
    Preparing to unpack .../09-libcjson1_1.7.17-1ubuntu0.1~esm2_amd64.deb ...
    Unpacking libcjson1:amd64 (1.7.17-1ubuntu0.1~esm2) over (1.7.17-1) ...
    Preparing to unpack .../10-libdcmtk17t64_3.6.7-9.1ubuntu0.1~esm1_amd64.deb ...
    Unpacking libdcmtk17t64:amd64 (3.6.7-9.1ubuntu0.1~esm1) over (3.6.7-9.1build4) ...
    Preparing to unpack .../11-python3-waitress_2.1.2-2ubuntu0.1~esm1_all.deb ...
    Unpacking python3-waitress (2.1.2-2ubuntu0.1~esm1) over (2.1.2-2) ...
    Setting up python3-waitress (2.1.2-2ubuntu0.1~esm1) ...
    Setting up libdcmtk17t64:amd64 (3.6.7-9.1ubuntu0.1~esm1) ...
    Setting up libavutil58:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libcjson1:amd64 (1.7.17-1ubuntu0.1~esm2) ...
    Setting up libswresample4:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libavcodec60:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libpostproc57:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libswscale7:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libavformat60:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libavfilter9:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up libavdevice60:amd64 (7:6.1.1-3ubuntu5+esm2) ...
    Setting up ffmpeg (7:6.1.1-3ubuntu5+esm2) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
    ```

### Make `dmesg` non-privileged

Since Ubuntu 22.04, `dmesg` has become a privileged operation
by default:

``` console
$ dmesg
dmesg: read kernel buffer failed: Operation not permitted
```

This is controlled by 

``` console
# sysctl kernel.dmesg_restrict
kernel.dmesg_restrict = 1
```

To revert this default, and make it permanent
([source](https://archived.forum.manjaro.org/t/why-did-dmesg-become-a-priveleged-operation-suddenly/86468/3)):

``` console
# echo 'kernel.dmesg_restrict=0' | tee -a /etc/sysctl.d/99-sysctl.conf
```

This is effective only after a reboot, but the next section requires
one too and is much more interesting.

### Make SDDM Look Good

Ubuntu Studio 24.04 uses 
[Simple Desktop Display Manager (SDDM)](https://wiki.archlinux.org/title/SDDM)
([sddm/sddm](https://github.com/sddm/sddm) in GitHub)
which is quite good looking out of the box, but I like to
customize this for each computer.

For most computers my favorite SDDM theme is
[Breeze-Noir-Dark](https://store.kde.org/p/1361460),
which I like to install system-wide.

``` console
# unzip -d /usr/share/sddm/themes Breeze-Noir-Dark.zip
```

!!! warning "Action icons won’t render if the directory name is changed."

    If needed, change the directory name in the `iconSource` fields
    in `Main.qml` to match final directory name so icons show. This is
    not the only thing that breaks when changing the directory name.

Other than installing this theme, all I really change in it
is the background image to use one that goes with the computer's name.
In this case,
[search for images of "raven wallpaper 4k"](https://www.google.com/search?sca_esv=70242eee9aad885d&sxsrf=ADLYWIIxnkJvJ7e4rr2bG4kHtRlDnQlMeQ:1735330282627&q=raven+wallpaper+4k&udm=2&fbs=AEQNm0CbCVgAZ5mWEJDg6aoPVcBgWizR0-0aFOH11Sb5tlNhd9L2WHVzAvhMQbvbuARNpGBcXTg9bjHozcoC1STf4cAK6TQt1QucotGleHY2pb6qE6LlBnOXI660eB8sIf84z99pS1_bZO1u4czxkXLQU14W-nkdOk4ao62bSo5vJ7qHvVbWvizUWq6xBgBQUFMcGKeI-huN&sa=X&ved=2ahUKEwi6gcGI4ciKAxVhUaQEHYgzKmwQtKgLegQIExAB&biw=2193&bih=845&dpr=1.1#vhid=1c7D-f48KGLBaM&vssid=mosaic)
led me to find this good-looking
[Raven Computer Wallpaper [2560x1600]](https://wallpapercave.com/w/wp6400810),
which needs to be cropped to fit the 2560x1080 screen resolution:

![Raven Computer Wallpaper [2560x1080]](../media/2024-12-27-ubuntu-studio-24-04-on-raven-gaming-pc-and-more/raven-2560-1080.jpg)

Once copied to `/usr/share/sddm/themes/Breeze-Noir-Dark/` the image
needs to be set up in *both* `theme.conf` and `theme.conf.user`:

``` console
# cp /jammy/usr/share/sddm/themes/Breeze-Noir-Dark/raven-2560-1080.jpg \
  /usr/share/sddm/themes/Breeze-Noir-Dark/
# cd /usr/share/sddm/themes/Breeze-Noir-Dark/

# vi theme.conf
[General]
type=image
color=#132e43
background=/usr/share/sddm/themes/Breeze-Noir-Dark/raven-2560-1080.jpg

# vi theme.conf.user
[General]
type=image
background=raven-2560-1080.jpg
```

**Additionally**, as this is new in Ubuntu 24.04, the theme has to
be selected by adding a `[Theme]` section in the system config
in `/usr/lib/sddm/sddm.conf.d/ubuntustudio.conf`

``` ini linenums="1" title="/usr/lib/sddm/sddm.conf.d/ubuntustudio.conf"
[General]
InputMethod=

[Theme]
Current="Breeze-Noir-Dark"
EnableAvatars=True
```

[Reportedly](https://superuser.com/questions/1720931/how-to-configure-sddm-in-kubuntu-22-04-where-is-config-file),
you have to **create** the `/etc/sddm.conf.d` directory to add
the *Local configuration* file that allows setting the theme:

``` console
# mkdir /etc/sddm.conf.d
# vi /etc/sddm.conf.d/ubuntustudio.conf
```

Besides setting the theme, it is also good to limit the range of
user ids so that only human users show up:

``` ini linenums="1" title="/etc/sddm.conf.d/ubuntustudio.conf"
[Theme]
Current=Breeze-Noir-Dark

[Users]
MaximumUid=1003
MinimumUid=1000
```

It seems no longer necessary to manually add Redshift to one's
desktop session. Previously, it would be necessary to launch
**Autostart** and **Add Application…** to add Redshift.

### Weekly btrfs scrub

To keep BTRFS file systems healthy, it is recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, I run
[the script](https://marc.merlins.org/linux/scripts/btrfs-scrub)
from crontab every Saturday night.

``` console
# wget -O /usr/local/bin/btrfs-scrub-all \
  http://marc.merlins.org/linux/scripts/btrfs-scrub

# crontab -e
...
# m h  dom mon dow   command
0 23 * * 6 /usr/local/bin/btrfs-scrub-all
```

[Marc MERLIN](https://marc.merlins.org/) keeps the script updated,
so each systme may benefit from a few modifications, e.g.
1. Remove tests for laptop battery status, when running on a PC.
2. Set the `BTRFS_SCRUB_SKIP` to filter out partitions to skip.

``` bash linenums="1" title="/usr/local/bin/btrfs-scrub-all"
#! /bin/bash

# By Marc MERLIN <marc_soft@merlins.org> 2014/03/20
# License: Apache-2.0
# http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html

which btrfs >/dev/null || exit 0
export PATH=/usr/local/bin:/sbin:$PATH

FILTER='(^Dumping|balancing, usage)'
BTRFS_SCRUB_SKIP="noskip"
source /etc/btrfs_config 2>/dev/null
test -n "$DEVS" || DEVS=$(grep '\<btrfs\>' /proc/mounts | awk '{ print $1 }' | sort -u | grep -v $BTRFS_SCRUB_SKIP)
for btrfs in $DEVS
do
    tail -n 0 -f /var/log/syslog | grep "BTRFS" | grep -Ev '(disk space caching is enabled|unlinked .* orphans|turning on discard|device label .* devid .* transid|enabling SSD mode|BTRFS: has skinny extents|BTRFS: device label|BTRFS info )' &
    mountpoint="$(grep "$btrfs" /proc/mounts | awk '{ print $2 }' | sort | head -1)"
    logger -s "Quick Metadata and Data Balance of $mountpoint ($btrfs)" >&2
    # Even in 4.3 kernels, you can still get in places where balance
    # won't work (no place left, until you run a -m0 one first)
    # I'm told that proactively rebalancing metadata may not be a good idea.
    #btrfs balance start -musage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # but a null rebalance should help corner cases:
    sleep 10
    btrfs balance start -musage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # After metadata, let's do data:
    sleep 10
    btrfs balance start -dusage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    sleep 10
    btrfs balance start -dusage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # And now we do scrub. Note that scrub can fail with "no space left
    # on device" if you're very out of balance.
    logger -s "Starting scrub of $mountpoint" >&2
    echo btrfs scrub start -Bd $mountpoint
    # -r is read only, but won't fix a redundant array.
    #ionice -c 3 nice -10 btrfs scrub start -Bdr $mountpoint
    time ionice -c 3 nice -10 btrfs scrub start -Bd $mountpoint
    pkill -f 'tail -n 0 -f /var/log/syslog'
    logger "Ended scrub of $mountpoint" >&2
done
```

The whole process takes about __ minutes for the 2TB NVMe SSD, then
something like _ hours for each of the 2TB RAID 5 of HDDs:

??? terminal "`# /usr/local/bin/btrfs-scrub-all`"

    ``` console
    # /usr/local/bin/btrfs-scrub-all
    <13>Dec 27 23:01:10 root: Quick Metadata and Data Balance of /home/raid (/dev/md0)
    Done, had to relocate 0 out of 1560 chunks
    Done, had to relocate 0 out of 1560 chunks
    Done, had to relocate 0 out of 1560 chunks
    <13>Dec 27 23:01:48 root: Starting scrub of /home/raid
    btrfs scrub start -Bd /home/raid
    Starting scrub on devid 1

    ```

![Disk I/O and SSD temperatures chart show btrfs scrub](../media/2024-12-27-ubuntu-studio-24-04-on-raven-gaming-pc-and-more/raven-btrfs-scrub-grafana.png)

Also, the weekly Btrfs scrub doesn't seem to really need `inn` or any
of its dependencies, so this installation step was **skipped**:

``` console
# apt install inn -y
```

### S.M.A.R.T. Monitoring

Install
[Smartmontools](https://help.ubuntu.com/community/Smartmontools)
to setup up
[S.M.A.R.T.](https://wiki.archlinux.org/index.php/S.M.A.R.T.)
monitoring:

``` console
# apt install smartmontools gsmartcontrol libnotify-bin nvme-cli -y
```

Create `/usr/local/bin/smartdnotify` to notify when errors are found:

``` bash linenums="1" title="/usr/local/bin/smartdnotify"
#!/bin/sh

# Ignore NVME "error" entries that are not errors.
elog=/root/smart-latest-error-log-entry
nvme error-log /dev/nvme0 | tail -16 > $elog
grep -iq 'status_field[[:blank:]]*: 0.SUCCESS' $elog && exit 0

# Otherwise, log and notify the error.
data=/root/smart-latest-error
echo "SMARTD_FAILTYPE=$SMARTD_FAILTYPE" >> $data
echo "SMARTD_MESSAGE=’$SMARTD_MESSAGE’" >> $data
sudo -u coder DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus notify-send "S.M.A.R.T Error ($SMARTD_FAILTYPE)" "$SMARTD_MESSAGE"  -i /usr/share/pixmaps/yoshimi.png
```

Configure `/etc/smartd.conf` to run this script:

``` bash
DEVICESCAN -d removable -n standby -m root -M exec /usr/local/bin/smartdnotify
```

Restart the `smartd` service:

``` console
# systemctl restart smartd.service

# systemctl status smartd.service
● smartmontools.service - Self Monitoring and Reporting Technology (SMART) Daemon
     Loaded: loaded (/usr/lib/systemd/system/smartmontools.service; enabled; preset: enabled)
     Active: active (running) since Fri 2024-12-27 23:47:21 CET; 3s ago
       Docs: man:smartd(8)
             man:smartd.conf(5)
   Main PID: 1363551 (smartd)
     Status: "Next check of 4 devices will start at 00:17:21"
      Tasks: 1 (limit: 38353)
     Memory: 1.9M (peak: 2.4M)
        CPU: 40ms
     CGroup: /system.slice/smartmontools.service
             └─1363551 /usr/sbin/smartd -n

Dec 27 23:47:20 raven smartd[1363551]: Device: /dev/nvme0, Samsung SSD 970 EVO Plus 2TB, S/N:S4J4NX0W427520B, FW:2B2QEXM7, 2.00 TB
Dec 27 23:47:20 raven smartd[1363551]: Device: /dev/nvme0, is SMART capable. Adding to "monitor" list.
Dec 27 23:47:20 raven smartd[1363551]: Device: /dev/nvme0, state read from /var/lib/smartmontools/smartd.Samsung_SSD_970_EVO_Plus_2TB-S4J4NX0W427520B.nvme.state
Dec 27 23:47:20 raven smartd[1363551]: Monitoring 3 ATA/SATA, 0 SCSI/SAS and 1 NVMe devices
Dec 27 23:47:20 raven smartd[1363551]: Device: /dev/sda [SAT], 3 Currently unreadable (pending) sectors
Dec 27 23:47:21 raven smartd[1363551]: Device: /dev/sda [SAT], state written to /var/lib/smartmontools/smartd.ST1000LM024_HN_M101MBB-S2R8J9DD902619.ata.state
Dec 27 23:47:21 raven smartd[1363551]: Device: /dev/sdb [SAT], state written to /var/lib/smartmontools/smartd.ST1000LM024_HN_M101MBB-S2R8J9DD902602.ata.state
Dec 27 23:47:21 raven smartd[1363551]: Device: /dev/sdc [SAT], state written to /var/lib/smartmontools/smartd.ST1000LM024_HN_M101MBB-S2R8J9DD902613.ata.state
Dec 27 23:47:21 raven smartd[1363551]: Device: /dev/nvme0, state written to /var/lib/smartmontools/smartd.Samsung_SSD_970_EVO_Plus_2TB-S4J4NX0W427520B.nvme.state
Dec 27 23:47:21 raven systemd[1]: Started smartmontools.service - Self Monitoring and Reporting Technology (SMART) Daemon.
```

Then add a scrip to re-notify: `/usr/local/bin/smartd-renotify`

``` bash linenums="1" title="/usr/local/bin/smartd-renotify"
#!/bin/sh

# Ignore NVME "error" entries that are not errors.
elog=/root/smart-latest-error-log-entry
nvme error-log /dev/nvme0 | tail -16 > $elog
grep -iq 'status_field[[:blank:]]*: 0.SUCCESS' $elog && exit 0

# Otherwise, re-notify.
latest=/root/smart-latest-error
if [ -f $latest ] ; then
  . $latest
  if [ -n "$SMARTD_FAILTYPE" ]
  then
    sudo -u coder DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus notify-send "S.M.A.R.T Error ($SMARTD_FAILTYPE)" "$SMARTD_MESSAGE"  -i /usr/share/pixmaps/yoshimi.png
  fi
fi
```

``` console
# chmod +x /usr/local/bin/smartdnotify /usr/local/bin/smartd-renotify
# crontab -e
*/5 * * * * /usr/local/bin/smartd-renotify
```

The above scripts have been updated to avoid spurious `ErrorCount`
errors, as discussed in 
[ErrorCount in NVME](2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#errorcount-in-nvme).

It is still always good to check that the disk **is** healthy:

??? terminal "`# smartctl -a /dev/nvme0`"

    ``` console
    # smartctl -a /dev/nvme0
    smartctl 7.4 2023-08-01 r5530 [x86_64-linux-6.8.0-50-lowlatency] (local build)
    Copyright (C) 2002-23, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF INFORMATION SECTION ===
    Model Number:                       Samsung SSD 970 EVO Plus 2TB
    Serial Number:                      S4J4NX0W427520B
    Firmware Version:                   2B2QEXM7
    PCI Vendor/Subsystem ID:            0x144d
    IEEE OUI Identifier:                0x002538
    Total NVM Capacity:                 2,000,398,934,016 [2.00 TB]
    Unallocated NVM Capacity:           0
    Controller ID:                      4
    NVMe Version:                       1.3
    Number of Namespaces:               1
    Namespace 1 Size/Capacity:          2,000,398,934,016 [2.00 TB]
    Namespace 1 Utilization:            1,304,029,368,320 [1.30 TB]
    Namespace 1 Formatted LBA Size:     512
    Namespace 1 IEEE EUI-64:            002538 5431b769e1
    Local Time is:                      Fri Dec 27 23:50:16 2024 CET
    Firmware Updates (0x16):            3 Slots, no Reset required
    Optional Admin Commands (0x0017):   Security Format Frmw_DL Self_Test
    Optional NVM Commands (0x005f):     Comp Wr_Unc DS_Mngmt Wr_Zero Sav/Sel_Feat Timestmp
    Log Page Attributes (0x03):         S/H_per_NS Cmd_Eff_Lg
    Maximum Data Transfer Size:         512 Pages
    Warning  Comp. Temp. Threshold:     85 Celsius
    Critical Comp. Temp. Threshold:     85 Celsius

    Supported Power States
    St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
     0 +     7.50W       -        -    0  0  0  0        0       0
     1 +     5.90W       -        -    1  1  1  1        0       0
     2 +     3.60W       -        -    2  2  2  2        0       0
     3 -   0.0700W       -        -    3  3  3  3      210    1200
     4 -   0.0050W       -        -    4  4  4  4     2000    8000

    Supported LBA Sizes (NSID 0x1)
    Id Fmt  Data  Metadt  Rel_Perf
     0 +     512       0         0

    === START OF SMART DATA SECTION ===
    SMART overall-health self-assessment test result: PASSED

    SMART/Health Information (NVMe Log 0x02)
    Critical Warning:                   0x00
    Temperature:                        54 Celsius
    Available Spare:                    100%
    Available Spare Threshold:          10%
    Percentage Used:                    0%
    Data Units Read:                    18,227,693 [9.33 TB]
    Data Units Written:                 16,663,786 [8.53 TB]
    Host Read Commands:                 420,175,269
    Host Write Commands:                178,467,400
    Controller Busy Time:               1,543
    Power Cycles:                       36
    Power On Hours:                     599
    Unsafe Shutdowns:                   3
    Media and Data Integrity Errors:    0
    Error Information Log Entries:      169
    Warning  Comp. Temperature Time:    0
    Critical Comp. Temperature Time:    0
    Temperature Sensor 1:               54 Celsius
    Temperature Sensor 2:               58 Celsius

    Error Information (NVMe Log 0x01, 16 of 64 entries)
    Num   ErrCount  SQId   CmdId  Status  PELoc          LBA  NSID    VS  Message
      0        169     0  0x0006  0x4004      -            0     0     -  Invalid Field in Command

    Self-test Log (NVMe Log 0x06)
    Self-test status: No self-test in progress
    No Self-tests Logged
    ```

Most of the entries are the same: not an error at all:

``` console
# nvme error-log /dev/nvme0 | grep status_field | sort | uniq -c
     63 status_field    : 0(Successful Completion: The command completed without error)
      1 status_field    : 0x2002(Invalid Field in Command: A reserved coded value or an unsupported value in a defined field)
```

### Bluetooth controller and devices

The system board
[MSI B450I Gaming Plus AC](https://es.msi.com/Motherboard/B450I-GAMING-PLUS-AC/Specification)
supports Bluetooth® 4.2, 4.1, BLE, 4.0, 3.0, 2.1, 2.1+EDR
but getting devices to pair, connect and work reliably is another story.

The bluetooth controller is detected and shows up in `dmesg`:

``` hl_lines="10 12"
[   12.847474] Bluetooth: Core ver 2.22
[   12.847825] NET: Registered PF_BLUETOOTH protocol family
[   12.847828] Bluetooth: HCI device and connection manager initialized
[   12.847835] Bluetooth: HCI socket layer initialized
[   12.847840] Bluetooth: L2CAP socket layer initialized
[   12.847848] Bluetooth: SCO socket layer initialized
...
[   12.924043] usbcore: registered new interface driver btusb
[   12.934914] Bluetooth: hci0: Legacy ROM 2.x revision 5.0 build 25 week 20 2015
[   12.936183] Bluetooth: hci0: Intel Bluetooth firmware file: intel/ibt-hw-37.8.10-fw-22.50.19.14.f.bseq
...
[   14.440042] Bluetooth: hci0: Intel BT fw patch 0x43 completed & activated
...
[   16.872207] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[   16.872215] Bluetooth: BNEP filters: protocol multicast
[   16.872223] Bluetooth: BNEP socket layer initialized
[   16.876155] Bluetooth: MGMT ver 1.22
...
[   22.338512] Bluetooth: RFCOMM TTY layer initialized
[   22.338525] Bluetooth: RFCOMM socket layer initialized
[   22.338532] Bluetooth: RFCOMM ver 1.11
```

Given the above, it may be possible to use the
[PlayStation Dual Shock 4 controller](2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#playstation-dual-shock-4-controller).
