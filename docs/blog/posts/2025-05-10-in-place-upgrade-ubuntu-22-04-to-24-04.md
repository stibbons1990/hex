---
date: 2025-05-10
categories:
 - linux
 - ubuntu
 - upgrade
 - setup
title: In-place upgrade Ubuntu 22.04 to 24.04
---

Upgrading from Ubuntu 22.04 to 24.04 *in-place* can be very convenient and a lot faster
than installing Ubuntu 24.04 anew, but does it work well? That has been the question and
*doubt* keeping me from trying again ever since one such upgrade went bad years ago.

<!-- more -->

There is an old Intel [NUC11PAHi7](https://www.intel.com/content/dam/support/us/en/documents/intel-nuc/NUC11PAH_Kit_UserGuide.pdf)
(named `smart-computer` in an *implosion* of creative naming) kicking around since before
[this blog started](./2023-09-21-starting-a-blog-with-jekyll-on-github-pages.md),
running Ubuntu Studio 22.04 with a very specific setup that was not well documented.
Installing Ubuntu Studio 24.04 would have been a good change to document that process,
but parts of it are likely obsolete and there was time pressure to upgrade to 24.04
before 22.04 reached EOL, so an in-place upgrade seemed worth a try.

Starting the upgrade process is as simple as logging in with the *first* user and
hitting the **Upgrade** button when prompted to upgrade, entering the password, and
agreeing to proceed. A few warnings must be accepted (acknowledged) before proceeding:

1.  **Third party sources disabled**. This appears to disable just `google-chrome`.
1.  The upgrade process can take several hours to complete.
    1.  To ensure this is not interrupted, disable `crontab` entries to shut down.
1.  Screen locking is disabled during the process.
1.  A long list of packages will be removed (277), installed (899) and upgraded (2661).

After a while downloading packages (3882 MB) the installation becomes stuck when
installing Thunderbird snap package:

```
Unpacking thunderbird-locale-en (2:1snap1-0ubuntu3) over (1:115.18.0+build1-0ubuntu0.22.04.1)...
Preparing to unpack .../4-thunderbird_2%3a1snap1-0ubuntu3_amd64.deb ...
=> Installing the thunderbird snap
==> Checking connectivity with the snap store
===> Unable to contact the store, trying every minute for the next 30 minutes
```

There an issue preventing `snap` from reaching the store, despite the SNAP API
being reachable:

``` console
# snap refresh snap-store
error: cannot refresh "snap-store": Post "https://api.snapcraft.io/v2/snaps/refresh": dial tcp:
       lookup api.snapcraft.io on 127.0.0.53:53: read udp 127.0.0.1:52705->127.0.0.53:53: read:
       connection refused

# curl -I https://api.snapcraft.io/v2/snaps/refresh
HTTP/1.1 405 METHOD NOT ALLOWED
server: gunicorn
date: Sat, 10 May 2025 15:59:38 GMT
content-type: text/html; charset=utf-8
allow: OPTIONS, POST
content-length: 153
snap-store-version: 69
x-vcs-revision: 30d5a26fd6984c4f8f5b80e6c8a1d87dd1031179
x-request-id: 00000000000000000000FFFFD9A239409A5E00000000000000000000FFFF0A8325F101BB681F77EA4AF02E3B
```

From very few, *tangentially* related forum threds, it seemed the only option to get
snap *unstuck* as to restart the `snapd` service:

``` console
# systemctl restart snapd.service 

# snap info snap-store
name:    snap-store
summary: Snap Store is a graphical desktop application for discovering, installing and managing
  snaps on Linux.
publisher: Canonical✓
store-url: https://snapcraft.io/snap-store
contact:   https://bugs.launchpad.net/snap-store/
license:   GPL-2.0+
description: |
  Snap Store showcases featured and popular applications with useful descriptions, ratings, reviews
  and screenshots.
  
  Applications can be found either through browsing categories or by searching.
  
  Snap Store can also be used to switch channels, view and alter snap permissions and view and
  submit reviews and ratings.
  
  Snap Store is based on GNOME Software, optimized for the Snap experience.
snap-id: gjf3IPXoRiipCu9K0kVu52f0H56fIksg
channels:
  2/stable:          0+git.90575829   2025-04-02 (1270) 11MB -
  2/candidate:       ↑                                       
  2/beta:            ↑                                       
  2/edge:            0+git.506cb2c3   2025-04-24 (1279) 11MB -
  latest/stable:     41.3-72-g80e7130 2024-09-22 (1216) 12MB -
  latest/candidate:  ↑                                       
  latest/beta:       ↑                                       
  latest/edge:       0+git.506cb2c3   2025-04-24 (1279) 11MB -
  preview/stable:    –                                       
  preview/candidate: 0.2.7-alpha      2023-02-02  (864) 10MB -
  preview/beta:      ↑                                       
  preview/edge:      0.3.0-alpha      2023-08-14 (1017) 11MB -
  1/stable:          41.3-72-g80e7130 2024-09-22 (1216) 12MB -
  1/candidate:       ↑                                       
  1/beta:            ↑                                       
  1/edge:            41.3-72-g80e7130 2024-09-16 (1216) 12MB -
```

Once `snap` is able to reach the store again, the `thunderbird` app is finally upgraded
and the upgrade process moves forward. Yet another prompt asks whether to Remove or Keep
obsolete packages; I decided to keep obsolete packages; having already had quite a few
packages removed. After reboot, there is just one package to update:

??? terminal "`apt dist-upgrade -y`"

    ``` console
    # apt dist-upgrade -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    Calculating upgrade... Done
    The following packages were automatically installed and are no longer required:
      acpi-support acpid agordejo blender-data cyclist digikam-data encfs ffmpeg2theora fonts-kacst
      fonts-kacst-one fonts-khmeros-core fonts-lao fonts-lklug-sinhala fonts-sil-abyssinica fonts-sil-padauk
      fonts-thai-tlwg fonts-tibetan-machine fonts-tlwg-garuda fonts-tlwg-garuda-ttf fonts-tlwg-kinnari
      fonts-tlwg-kinnari-ttf fonts-tlwg-laksaman fonts-tlwg-laksaman-ttf fonts-tlwg-loma fonts-tlwg-loma-ttf
      fonts-tlwg-mono fonts-tlwg-mono-ttf fonts-tlwg-norasi fonts-tlwg-norasi-ttf fonts-tlwg-purisa
      fonts-tlwg-purisa-ttf fonts-tlwg-sawasdee fonts-tlwg-sawasdee-ttf fonts-tlwg-typewriter
      fonts-tlwg-typewriter-ttf fonts-tlwg-typist fonts-tlwg-typist-ttf fonts-tlwg-typo fonts-tlwg-typo-ttf
      fonts-tlwg-umpush fonts-tlwg-umpush-ttf fonts-tlwg-waree fonts-tlwg-waree-ttf foo-yc20 freeglut3 gamin
      gcc-12-base:i386 gimp-data haveged irqbalance kdenlive-data kross ksystemlog lib2geom1.1.0 libabsl20210324
      libamd2 libappimage0 libappstream4 libappstreamqt2 libarmadillo10 libart-2.0-2 libastro1 libatk1.0-data
      libavcodec58 libavdevice58 libavfilter7 libavformat58 libavif13 libavutil56 libbabl-0.1-0
      libblockdev-crypto2 libblockdev-fs2 libblockdev-loop2 libblockdev-part-err2 libblockdev-part2
      libblockdev-swap2 libblockdev-utils2 libblockdev2 libboost-filesystem1.74.0 libboost-iostreams1.74.0
      libboost-locale1.74.0 libboost-regex1.74.0 libboost-thread1.74.0 libbpf0 libcamd2 libcapi20-3t64
      libcbor0.8 libccolamd2 libcfitsio9 libchamplain-0.12-0 libchamplain-gtk-0.12-0 libcholmod3
      libclutter-1.0-0 libclutter-1.0-common libclutter-gtk-1.0-0 libcodec2-1.0 libcogl-common libcogl-pango20
      libcogl-path20 libcogl20 libcolamd2 libcrypto++8t64 libcupsfilters1 libdav1d5 libdcmtk16 libdcmtk17t64
      libdns-export1110 libdraco4 libdrm-nouveau2:i386 libembree4-4 libev4t64 libevent-pthreads-2.1-7t64
      libfaudio0 libflac++6v5 libflac8 libflac8:i386 libfltk1.1 libfontembed1 libgamin0 libgav1-0 libgcab-1.0-0
      libgegl-common libgeos3.10.2 libgit2-1.1 libglade2-0 libgnomecanvas2-0 libgnomecanvas2-common libgps28
      libgraphicsmagick++-q16-12t64 libgssdp-1.2-0 libgtkmm-2.4-1t64 libgupnp-1.2-1 libgupnp-igd-1.0-4
      libhavege2 libicu70 libicu70:i386 libilmbase25 libisc-export1105 libixml10 libjavascriptcoregtk-4.0-18
      libjemalloc2 libkdecorations2private9 libkdynamicwallpaper1 libkf5akonadi-data libkf5akonadicontact-data
      libkf5akonadicontact5abi1 libkf5akonadicore-bin libkf5akonadicore5abi2 libkf5akonadiprivate5abi2
      libkf5akonadiwidgets5abi1 libkf5baloowidgets-data libkf5calendarcore5abi2 libkf5contacteditor5
      libkf5grantleetheme-data libkf5grantleetheme-plugins libkf5grantleetheme5 libkf5jsapi5 libkf5krosscore5
      libkf5krossui5 libkf5libkleo5abi1 libkf5mime-data libkf5mime5abi2 libkf5pimtextedit-data
      libkf5pimtextedit5abi3 libkf5waylandserver5 libkpim5akonadi-data libkwaylandserver5 libkwinxrenderutils13
      liblcms2-2:i386 libldap-2.5-0:i386 libllvm13 libllvm15t64 libllvm15t64:i386 liblog4cplus-2.0.5t64
      liblua5.1-0 libmagick++-6.q16-8 libmagickcore-6.q16-6 libmagickcore-6.q16-6-extra libmagickwand-6.q16-6
      libmanette-0.2-0 libmarblewidget-qt5-28 libmetis5 libmicrohttpd12t64 libmpdec3 libmujs1 libnetpbm10
      libnetplan0 libnfs13 liboggkate1 libokular5core9 libopenal1:i386 libopencolorio1v5 libopencv-calib3d4.5d
      libopencv-core4.5d libopencv-dnn4.5d libopencv-features2d4.5d libopencv-flann4.5d libopencv-imgproc4.5d
      libopencv-ml4.5d libopencv-objdetect4.5d libopencv-video4.5d libopenexr25 libopenh264-6 libopenvdb10.0t64
      libopts25 liborcus-0.17-0 liborcus-parser-0.17-0 libosdcpu3.5.0t64 libosdgpu3.5.0t64 libosmesa6
      libparted-fs-resize0t64 libperl5.34 libphonon4qt5-data libplacebo192 libplist3 libpoppler118 libpostproc55
      libproj22 libprotobuf-lite23 libprotobuf23 libpython3.10 libpython3.10-dev libpython3.10-minimal
      libpython3.10-stdlib libqgpgme7 libqpdf28 libqt5networkauth5 libqtav1 libqtavwidgets1 libraw20 libre2-9
      libshp2 libshp4 libsmbios-c2 libsnapd-glib1 libsnapd-qt1 libsndio7.0:i386 libsoup-2.4-1:i386 libspnav0
      libsquish0 libsrt1.4-gnutls libstb0t64 libstk-4.6.1 libsuitesparseconfig5 libsuperlu5 libswresample3
      libswscale5 libtesseract4 libtexlua53 libtexluajit2 libtiff-tools libtiff5 libtiff5:i386 libtinyxml2-9
      libumfpack5 libunistring2 libunistring2:i386 libupnp13 libvkd3d-shader1 libvkd3d1 libvpx7 libvpx7:i386
      libwebkit2gtk-4.0-37 libwebsockets16 libwxbase3.0-0v5 libwxgtk3.0-gtk3-0v5 libx264-163 libxkbregistry0
      libxslt1.1:i386 libyaml-cpp0.7 libzxingcore1 linux-headers-5.15.0-138 linux-headers-5.15.0-138-generic
      linux-headers-5.15.0-139 linux-headers-5.15.0-139-generic linux-lowlatency-headers-5.15.0-136
      linux-lowlatency-headers-5.15.0-138 linuxaudio-new-session-manager marble-plugins marble-qt-data mediainfo
      midisnoop mlocate muon opencv-data p7zip perl-modules-5.34 petri-foo python3-alsaaudio python3-cffi
      python3-jack-client python3-lib2to3 python3-netifaces python3-pbr python3-pycparser python3-sip python3.10
      python3.10-dev python3.10-minimal qwinff rtmpdump sntp synfig-examples ubuntu-advantage-tools
      vkd3d-compiler vocproc xdg-dbus-proxy zita-njbridge
    Use 'apt autoremove' to remove them.
    The following packages will be REMOVED:
      linux-headers-5.15.0-136-lowlatency linux-headers-5.15.0-138-lowlatency linux-image-5.15.0-136-lowlatency
      linux-image-5.15.0-138-lowlatency linux-modules-5.15.0-136-lowlatency linux-modules-5.15.0-138-lowlatency
    The following NEW packages will be installed:
      libgdal34t64 libgeos-c1t64 libgeos3.12.1t64 libmlt++7 libmlt7 libopencv-contrib406t64
      libopencv-highgui406t64 libopencv-imgcodecs406t64 librttopo1 libspatialite8t64
    The following packages will be upgraded:
      krita
    1 upgraded, 10 newly installed, 6 to remove and 0 not upgraded.
    Need to get 39.4 MB of archives.
    After this operation, 940 MB disk space will be freed.
    Get:1 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgeos3.12.1t64 amd64 3.12.1-3build1 [849 kB]
    Get:2 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgeos-c1t64 amd64 3.12.1-3build1 [94.5 kB]
    Get:3 http://archive.ubuntu.com/ubuntu noble/universe amd64 librttopo1 amd64 1.1.0-3build2 [191 kB]
    Get:4 http://archive.ubuntu.com/ubuntu noble/universe amd64 libspatialite8t64 amd64 5.1.0-3build1 [1,919 kB]
    Get:5 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgdal34t64 amd64 3.8.4+dfsg-3ubuntu3 [8,346 kB]
    Get:6 http://archive.ubuntu.com/ubuntu noble/universe amd64 libopencv-imgcodecs406t64 amd64 4.6.0+dfsg-13.1ubuntu1 [128 kB]
    Get:7 http://archive.ubuntu.com/ubuntu noble/universe amd64 libopencv-highgui406t64 amd64 4.6.0+dfsg-13.1ubuntu1 [118 kB]
    Get:8 http://archive.ubuntu.com/ubuntu noble/universe amd64 libopencv-contrib406t64 amd64 4.6.0+dfsg-13.1ubuntu1 [3,968 kB]
    Get:9 http://archive.ubuntu.com/ubuntu noble/universe amd64 libmlt7 amd64 7.22.0-1build6 [1,603 kB]
    Get:10 http://archive.ubuntu.com/ubuntu noble/universe amd64 libmlt++7 amd64 7.22.0-1build6 [51.5 kB]
    Get:11 http://archive.ubuntu.com/ubuntu noble/universe amd64 krita amd64 1:5.2.2+dfsg-2build8 [22.1 MB]
    Fetched 39.4 MB in 3s (12.1 MB/s)     
    (Reading database ... 657253 files and directories currently installed.)
    Removing linux-headers-5.15.0-136-lowlatency (5.15.0-136.147) ...
    Removing linux-headers-5.15.0-138-lowlatency (5.15.0-138.148) ...
    Removing linux-modules-5.15.0-138-lowlatency (5.15.0-138.148) ...
    Removing linux-image-5.15.0-136-lowlatency (5.15.0-136.147) ...
    /etc/kernel/postrm.d/initramfs-tools:
    update-initramfs: Deleting /boot/initrd.img-5.15.0-136-lowlatency
    /etc/kernel/postrm.d/zz-update-grub:
    Sourcing file `/etc/default/grub'
    Sourcing file `/etc/default/grub.d/ubuntustudio.cfg'
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-6.8.0-59-lowlatency
    Found initrd image: /boot/initrd.img-6.8.0-59-lowlatency
    Found linux image: /boot/vmlinuz-5.15.0-139-lowlatency
    Found initrd image: /boot/initrd.img-5.15.0-139-lowlatency
    Found linux image: /boot/vmlinuz-5.15.0-138-lowlatency
    Found initrd image: /boot/initrd.img-5.15.0-138-lowlatency
    Found memtest86+ 64bit EFI image: /boot/memtest86+x64.efi
    Warning: os-prober will not be executed to detect other bootable partitions.
    Systems on them will not be added to the GRUB boot configuration.
    Check GRUB_DISABLE_OS_PROBER documentation entry.
    Adding boot menu entry for UEFI Firmware Settings ...
    done
    Removing linux-modules-5.15.0-136-lowlatency (5.15.0-136.147) ...
    Removing linux-image-5.15.0-138-lowlatency (5.15.0-138.148) ...
    /etc/kernel/postrm.d/initramfs-tools:
    update-initramfs: Deleting /boot/initrd.img-5.15.0-138-lowlatency
    /etc/kernel/postrm.d/zz-update-grub:
    Sourcing file `/etc/default/grub'
    Sourcing file `/etc/default/grub.d/ubuntustudio.cfg'
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-6.8.0-59-lowlatency
    Found initrd image: /boot/initrd.img-6.8.0-59-lowlatency
    Found linux image: /boot/vmlinuz-5.15.0-139-lowlatency
    Found initrd image: /boot/initrd.img-5.15.0-139-lowlatency
    Found memtest86+ 64bit EFI image: /boot/memtest86+x64.efi
    Warning: os-prober will not be executed to detect other bootable partitions.
    Systems on them will not be added to the GRUB boot configuration.
    Check GRUB_DISABLE_OS_PROBER documentation entry.
    Adding boot menu entry for UEFI Firmware Settings ...
    done
    Selecting previously unselected package libgeos3.12.1t64:amd64.
    (Reading database ... 623715 files and directories currently installed.)
    Preparing to unpack .../00-libgeos3.12.1t64_3.12.1-3build1_amd64.deb ...
    Unpacking libgeos3.12.1t64:amd64 (3.12.1-3build1) ...
    Selecting previously unselected package libgeos-c1t64:amd64.
    Preparing to unpack .../01-libgeos-c1t64_3.12.1-3build1_amd64.deb ...
    Unpacking libgeos-c1t64:amd64 (3.12.1-3build1) ...
    Selecting previously unselected package librttopo1:amd64.
    Preparing to unpack .../02-librttopo1_1.1.0-3build2_amd64.deb ...
    Unpacking librttopo1:amd64 (1.1.0-3build2) ...
    Selecting previously unselected package libspatialite8t64:amd64.
    Preparing to unpack .../03-libspatialite8t64_5.1.0-3build1_amd64.deb ...
    Unpacking libspatialite8t64:amd64 (5.1.0-3build1) ...
    Selecting previously unselected package libgdal34t64:amd64.
    Preparing to unpack .../04-libgdal34t64_3.8.4+dfsg-3ubuntu3_amd64.deb ...
    Unpacking libgdal34t64:amd64 (3.8.4+dfsg-3ubuntu3) ...
    Selecting previously unselected package libopencv-imgcodecs406t64:amd64.
    Preparing to unpack .../05-libopencv-imgcodecs406t64_4.6.0+dfsg-13.1ubuntu1_amd64.deb ...
    Unpacking libopencv-imgcodecs406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Selecting previously unselected package libopencv-highgui406t64:amd64.
    Preparing to unpack .../06-libopencv-highgui406t64_4.6.0+dfsg-13.1ubuntu1_amd64.deb ...
    Unpacking libopencv-highgui406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Selecting previously unselected package libopencv-contrib406t64:amd64.
    Preparing to unpack .../07-libopencv-contrib406t64_4.6.0+dfsg-13.1ubuntu1_amd64.deb ...
    Unpacking libopencv-contrib406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Selecting previously unselected package libmlt7:amd64.
    Preparing to unpack .../08-libmlt7_7.22.0-1build6_amd64.deb ...
    Unpacking libmlt7:amd64 (7.22.0-1build6) ...
    Selecting previously unselected package libmlt++7:amd64.
    Preparing to unpack .../09-libmlt++7_7.22.0-1build6_amd64.deb ...
    Unpacking libmlt++7:amd64 (7.22.0-1build6) ...
    Preparing to unpack .../10-krita_1%3a5.2.2+dfsg-2build8_amd64.deb ...
    Unpacking krita (1:5.2.2+dfsg-2build8) over (1:5.0.2+dfsg-1build1) ...
    Setting up libgeos3.12.1t64:amd64 (3.12.1-3build1) ...
    Setting up libgeos-c1t64:amd64 (3.12.1-3build1) ...
    Setting up librttopo1:amd64 (1.1.0-3build2) ...
    Setting up libspatialite8t64:amd64 (5.1.0-3build1) ...
    Setting up libgdal34t64:amd64 (3.8.4+dfsg-3ubuntu3) ...
    Setting up libopencv-imgcodecs406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Setting up libopencv-highgui406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Setting up libopencv-contrib406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Setting up libmlt7:amd64 (7.22.0-1build6) ...
    Setting up libmlt++7:amd64 (7.22.0-1build6) ...
    Setting up krita (1:5.2.2+dfsg-2build8) ...
    Processing triggers for mailcap (3.70+nmu1ubuntu1) ...
    Processing triggers for desktop-file-utils (0.27-2build1) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.4) ...
    ```

After this, install packages from the list of Ubuntu Studio 24.04
[essential packages](https://hex.very-very-dark-gray.top/blog/2024/09/14/ubuntu-studio-2404-on-computer-for-a-young-artist/#install-essential-packages):

??? terminal "`apt install ... -y`"

    ``` console
    # apt install gdebi-core wget gkrellm vim curl gkrellm-leds \
      gkrellm-xkb gkrellm-cpufreq geeqie playonlinux exfat-fuse \
      clementine id3v2 htop vnstat neofetch tigervnc-viewer sox \
      scummvm wine gamemode python-is-python3 exiv2 rename scrot \
      speedtest-cli xcalib python3-pip netcat-openbsd jstest-gtk \
      etherwake python3-selenium lm-sensors sysstat tor unrar \
      ttf-mscorefonts-installer winetricks icc-profiles ffmpeg \
      iotop-c xdotool redshift-gtk inxi vainfo vdpauinfo mpv \
      tigervnc-tools screen -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    gdebi-core is already the newest version (0.9.5.7+nmu7).
    wget is already the newest version (1.21.4-1ubuntu4.1).
    gkrellm is already the newest version (2.3.11-2build2).
    vim is already the newest version (2:9.1.0016-1ubuntu7.8).
    curl is already the newest version (8.5.0-2ubuntu10.6).
    gkrellm-xkb is already the newest version (1.05-5.1build2).
    gkrellm-cpufreq is already the newest version (0.6.4-4).
    geeqie is already the newest version (1:2.2-2ubuntu0.1).
    playonlinux is already the newest version (4.3.4-3).
    exfat-fuse is already the newest version (1.4.0-2).
    clementine is already the newest version (1.4.0~rc1+git867-g9ef681b0e+dfsg-1ubuntu4).
    id3v2 is already the newest version (0.1.12+dfsg-7).
    htop is already the newest version (3.3.0-4build1).
    vnstat is already the newest version (2.12-1).
    neofetch is already the newest version (7.1.0-4).
    tigervnc-viewer is already the newest version (1.13.1+dfsg-2build2).
    sox is already the newest version (14.4.2+git20190427-4build4).
    scummvm is already the newest version (2.8.0+dfsg-1build5).
    wine is already the newest version (9.0~repack-4build3).
    gamemode is already the newest version (1.8.1-2build1).
    python-is-python3 is already the newest version (3.11.4-1).
    exiv2 is already the newest version (0.27.6-1build1).
    rename is already the newest version (2.02-1).
    scrot is already the newest version (1.10-1build2).
    speedtest-cli is already the newest version (2.1.3-2).
    xcalib is already the newest version (0.8.dfsg1-3).
    python3-pip is already the newest version (24.0+dfsg-1ubuntu1.1).
    netcat-openbsd is already the newest version (1.226-1ubuntu2).
    netcat-openbsd set to manually installed.
    jstest-gtk is already the newest version (0.1.1~git20180602-2build2).
    etherwake is already the newest version (1.09-4build1).
    python3-selenium is already the newest version (4.18.1+dfsg-1).
    lm-sensors is already the newest version (1:3.6.0-9build1).
    sysstat is already the newest version (12.6.1-2).
    tor is already the newest version (0.4.8.10-1build2).
    unrar is already the newest version (1:7.0.7-1build1).
    ttf-mscorefonts-installer is already the newest version (3.8.1ubuntu1).
    winetricks is already the newest version (20240105-2).
    icc-profiles is already the newest version (2.1-2).
    ffmpeg is already the newest version (7:6.1.1-3ubuntu5+esm2).
    iotop-c is already the newest version (1.26-1).
    xdotool is already the newest version (1:3.20160805.1-5build1).
    redshift-gtk is already the newest version (1.12-4.2ubuntu4).
    inxi is already the newest version (3.3.34-1-1).
    vainfo is already the newest version (2.12.0+ds1-1).
    vdpauinfo is already the newest version (1.5-2).
    mpv is already the newest version (0.37.0-1ubuntu4).
    tigervnc-tools is already the newest version (1.13.1+dfsg-2build2).
    The following packages were automatically installed and are no longer required:
      acpi-support acpid agordejo blender-data cyclist digikam-data encfs ffmpeg2theora fonts-kacst
      fonts-kacst-one fonts-khmeros-core fonts-lao fonts-lklug-sinhala fonts-sil-abyssinica fonts-sil-padauk
      fonts-thai-tlwg fonts-tibetan-machine fonts-tlwg-garuda fonts-tlwg-garuda-ttf fonts-tlwg-kinnari
      fonts-tlwg-kinnari-ttf fonts-tlwg-laksaman fonts-tlwg-laksaman-ttf fonts-tlwg-loma fonts-tlwg-loma-ttf
      fonts-tlwg-mono fonts-tlwg-mono-ttf fonts-tlwg-norasi fonts-tlwg-norasi-ttf fonts-tlwg-purisa
      fonts-tlwg-purisa-ttf fonts-tlwg-sawasdee fonts-tlwg-sawasdee-ttf fonts-tlwg-typewriter
      fonts-tlwg-typewriter-ttf fonts-tlwg-typist fonts-tlwg-typist-ttf fonts-tlwg-typo fonts-tlwg-typo-ttf
      fonts-tlwg-umpush fonts-tlwg-umpush-ttf fonts-tlwg-waree fonts-tlwg-waree-ttf foo-yc20 freeglut3 gamin
      gcc-12-base:i386 gimp-data haveged irqbalance kdenlive-data kross ksystemlog lib2geom1.1.0 libabsl20210324
      libamd2 libappimage0 libappstream4 libappstreamqt2 libarmadillo10 libart-2.0-2 libastro1 libatk1.0-data
      libavcodec58 libavdevice58 libavfilter7 libavformat58 libavif13 libavutil56 libbabl-0.1-0
      libblockdev-crypto2 libblockdev-fs2 libblockdev-loop2 libblockdev-part-err2 libblockdev-part2
      libblockdev-swap2 libblockdev-utils2 libblockdev2 libboost-filesystem1.74.0 libboost-iostreams1.74.0
      libboost-locale1.74.0 libboost-regex1.74.0 libboost-thread1.74.0 libbpf0 libcamd2 libcapi20-3t64
      libcbor0.8 libccolamd2 libcfitsio9 libchamplain-0.12-0 libchamplain-gtk-0.12-0 libcholmod3
      libclutter-1.0-0 libclutter-1.0-common libclutter-gtk-1.0-0 libcodec2-1.0 libcogl-common libcogl-pango20
      libcogl-path20 libcogl20 libcolamd2 libcrypto++8t64 libcupsfilters1 libdav1d5 libdcmtk16 libdcmtk17t64
      libdns-export1110 libdraco4 libdrm-nouveau2:i386 libembree4-4 libev4t64 libevent-pthreads-2.1-7t64
      libfaudio0 libflac++6v5 libflac8 libflac8:i386 libfltk1.1 libfontembed1 libgamin0 libgav1-0 libgcab-1.0-0
      libgegl-common libgeos3.10.2 libgit2-1.1 libglade2-0 libgnomecanvas2-0 libgnomecanvas2-common libgps28
      libgraphicsmagick++-q16-12t64 libgssdp-1.2-0 libgtkmm-2.4-1t64 libgupnp-1.2-1 libgupnp-igd-1.0-4
      libhavege2 libicu70 libicu70:i386 libilmbase25 libisc-export1105 libixml10 libjavascriptcoregtk-4.0-18
      libjemalloc2 libkdecorations2private9 libkdynamicwallpaper1 libkf5akonadi-data libkf5akonadicontact-data
      libkf5akonadicontact5abi1 libkf5akonadicore-bin libkf5akonadicore5abi2 libkf5akonadiprivate5abi2
      libkf5akonadiwidgets5abi1 libkf5baloowidgets-data libkf5calendarcore5abi2 libkf5contacteditor5
      libkf5grantleetheme-data libkf5grantleetheme-plugins libkf5grantleetheme5 libkf5jsapi5 libkf5krosscore5
      libkf5krossui5 libkf5libkleo5abi1 libkf5mime-data libkf5mime5abi2 libkf5pimtextedit-data
      libkf5pimtextedit5abi3 libkf5waylandserver5 libkpim5akonadi-data libkwaylandserver5 libkwinxrenderutils13
      liblcms2-2:i386 libldap-2.5-0:i386 libllvm13 libllvm15t64 libllvm15t64:i386 liblog4cplus-2.0.5t64
      liblua5.1-0 libmagick++-6.q16-8 libmagickcore-6.q16-6 libmagickcore-6.q16-6-extra libmagickwand-6.q16-6
      libmanette-0.2-0 libmarblewidget-qt5-28 libmetis5 libmicrohttpd12t64 libmpdec3 libmujs1 libnetpbm10
      libnetplan0 libnfs13 liboggkate1 libokular5core9 libopenal1:i386 libopencolorio1v5 libopencv-calib3d4.5d
      libopencv-core4.5d libopencv-dnn4.5d libopencv-features2d4.5d libopencv-flann4.5d libopencv-imgproc4.5d
      libopencv-ml4.5d libopencv-objdetect4.5d libopencv-video4.5d libopenexr25 libopenh264-6 libopenvdb10.0t64
      libopts25 liborcus-0.17-0 liborcus-parser-0.17-0 libosdcpu3.5.0t64 libosdgpu3.5.0t64 libosmesa6
      libparted-fs-resize0t64 libperl5.34 libphonon4qt5-data libplacebo192 libplist3 libpoppler118 libpostproc55
      libproj22 libprotobuf-lite23 libprotobuf23 libpython3.10 libpython3.10-dev libpython3.10-minimal
      libpython3.10-stdlib libqgpgme7 libqpdf28 libqt5networkauth5 libqtav1 libqtavwidgets1 libraw20 libre2-9
      libshp2 libshp4 libsmbios-c2 libsnapd-glib1 libsnapd-qt1 libsndio7.0:i386 libsoup-2.4-1:i386 libspnav0
      libsquish0 libsrt1.4-gnutls libstb0t64 libstk-4.6.1 libsuitesparseconfig5 libsuperlu5 libswresample3
      libswscale5 libtesseract4 libtexlua53 libtexluajit2 libtiff-tools libtiff5 libtiff5:i386 libtinyxml2-9
      libumfpack5 libunistring2 libunistring2:i386 libupnp13 libvkd3d-shader1 libvkd3d1 libvpx7 libvpx7:i386
      libwebkit2gtk-4.0-37 libwebsockets16 libwxbase3.0-0v5 libwxgtk3.0-gtk3-0v5 libx264-163 libxkbregistry0
      libxslt1.1:i386 libyaml-cpp0.7 libzxingcore1 linux-headers-5.15.0-138 linux-headers-5.15.0-138-generic
      linux-headers-5.15.0-139 linux-headers-5.15.0-139-generic linux-lowlatency-headers-5.15.0-136
      linux-lowlatency-headers-5.15.0-138 linuxaudio-new-session-manager marble-plugins marble-qt-data mediainfo
      midisnoop mlocate muon opencv-data p7zip perl-modules-5.34 petri-foo python3-alsaaudio python3-cffi
      python3-jack-client python3-lib2to3 python3-netifaces python3-pbr python3-pycparser python3-sip python3.10
      python3.10-dev python3.10-minimal qwinff rtmpdump sntp synfig-examples ubuntu-advantage-tools
      vkd3d-compiler vocproc xdg-dbus-proxy zita-njbridge
    Use 'apt autoremove' to remove them.
    Suggested packages:
      byobu | screenie | iselect
    The following NEW packages will be installed:
      gkrellm-leds screen
    0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
    Need to get 671 kB of archives.
    After this operation, 1,080 kB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 screen amd64 4.9.1-1build1 [655 kB]
    Get:2 http://archive.ubuntu.com/ubuntu noble/universe amd64 gkrellm-leds amd64 0.8.0-2build2 [16.0 kB]
    Fetched 671 kB in 1s (705 kB/s)         
    Selecting previously unselected package screen.
    (Reading database ... 623901 files and directories currently installed.)
    Preparing to unpack .../screen_4.9.1-1build1_amd64.deb ...
    Unpacking screen (4.9.1-1build1) ...
    Selecting previously unselected package gkrellm-leds.
    Preparing to unpack .../gkrellm-leds_0.8.0-2build2_amd64.deb ...
    Unpacking gkrellm-leds (0.8.0-2build2) ...
    Setting up screen (4.9.1-1build1) ...
    Setting up gkrellm-leds (0.8.0-2build2) ...
    Processing triggers for debianutils (5.17build1) ...
    Processing triggers for install-info (7.1-3build2) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    ```

Then reinstall GIMP since it was removed during the upgrade process:

??? terminal "`apt install gimp ... -y`"

    ``` console
    # apt install gimp gimp-gap gimp-gmic gimp-gutenprint \
          gimp-cbmplugs gimp-plugin-registry gimp-texturize -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following packages were automatically installed and are no longer required:
      acpi-support acpid agordejo blender-data cyclist digikam-data encfs ffmpeg2theora fonts-kacst
      fonts-kacst-one fonts-khmeros-core fonts-lao fonts-lklug-sinhala fonts-sil-abyssinica fonts-sil-padauk
      fonts-thai-tlwg fonts-tibetan-machine fonts-tlwg-garuda fonts-tlwg-garuda-ttf fonts-tlwg-kinnari
      fonts-tlwg-kinnari-ttf fonts-tlwg-laksaman fonts-tlwg-laksaman-ttf fonts-tlwg-loma fonts-tlwg-loma-ttf
      fonts-tlwg-mono fonts-tlwg-mono-ttf fonts-tlwg-norasi fonts-tlwg-norasi-ttf fonts-tlwg-purisa
      fonts-tlwg-purisa-ttf fonts-tlwg-sawasdee fonts-tlwg-sawasdee-ttf fonts-tlwg-typewriter
      fonts-tlwg-typewriter-ttf fonts-tlwg-typist fonts-tlwg-typist-ttf fonts-tlwg-typo fonts-tlwg-typo-ttf
      fonts-tlwg-umpush fonts-tlwg-umpush-ttf fonts-tlwg-waree fonts-tlwg-waree-ttf foo-yc20 freeglut3 gamin
      gcc-12-base:i386 haveged irqbalance kdenlive-data kross ksystemlog lib2geom1.1.0 libabsl20210324 libamd2
      libappimage0 libappstream4 libappstreamqt2 libarmadillo10 libart-2.0-2 libastro1 libatk1.0-data
      libavcodec58 libavdevice58 libavfilter7 libavformat58 libavif13 libavutil56 libblockdev-crypto2
      libblockdev-fs2 libblockdev-loop2 libblockdev-part-err2 libblockdev-part2 libblockdev-swap2
      libblockdev-utils2 libblockdev2 libboost-filesystem1.74.0 libboost-iostreams1.74.0 libboost-locale1.74.0
      libboost-regex1.74.0 libboost-thread1.74.0 libbpf0 libcamd2 libcapi20-3t64 libcbor0.8 libccolamd2
      libcfitsio9 libchamplain-0.12-0 libchamplain-gtk-0.12-0 libcholmod3 libclutter-1.0-0 libclutter-1.0-common
      libclutter-gtk-1.0-0 libcodec2-1.0 libcogl-common libcogl-pango20 libcogl-path20 libcogl20 libcolamd2
      libcrypto++8t64 libcupsfilters1 libdav1d5 libdcmtk16 libdcmtk17t64 libdns-export1110 libdraco4
      libdrm-nouveau2:i386 libembree4-4 libev4t64 libevent-pthreads-2.1-7t64 libfaudio0 libflac++6v5 libflac8
      libflac8:i386 libfltk1.1 libfontembed1 libgamin0 libgav1-0 libgcab-1.0-0 libgeos3.10.2 libgit2-1.1
      libglade2-0 libgnomecanvas2-0 libgnomecanvas2-common libgps28 libgssdp-1.2-0 libgtkmm-2.4-1t64
      libgupnp-1.2-1 libgupnp-igd-1.0-4 libhavege2 libicu70 libicu70:i386 libilmbase25 libisc-export1105
      libixml10 libjavascriptcoregtk-4.0-18 libjemalloc2 libkdecorations2private9 libkdynamicwallpaper1
      libkf5akonadi-data libkf5akonadicontact-data libkf5akonadicontact5abi1 libkf5akonadicore-bin
      libkf5akonadicore5abi2 libkf5akonadiprivate5abi2 libkf5akonadiwidgets5abi1 libkf5baloowidgets-data
      libkf5calendarcore5abi2 libkf5contacteditor5 libkf5grantleetheme-data libkf5grantleetheme-plugins
      libkf5grantleetheme5 libkf5jsapi5 libkf5krosscore5 libkf5krossui5 libkf5libkleo5abi1 libkf5mime-data
      libkf5mime5abi2 libkf5pimtextedit-data libkf5pimtextedit5abi3 libkf5waylandserver5 libkpim5akonadi-data
      libkwaylandserver5 libkwinxrenderutils13 liblcms2-2:i386 libldap-2.5-0:i386 libllvm13 libllvm15t64
      libllvm15t64:i386 liblog4cplus-2.0.5t64 liblua5.1-0 libmagick++-6.q16-8 libmagickcore-6.q16-6
      libmagickcore-6.q16-6-extra libmagickwand-6.q16-6 libmanette-0.2-0 libmarblewidget-qt5-28 libmetis5
      libmicrohttpd12t64 libmpdec3 libmujs1 libnetpbm10 libnetplan0 libnfs13 liboggkate1 libokular5core9
      libopenal1:i386 libopencolorio1v5 libopencv-calib3d4.5d libopencv-core4.5d libopencv-dnn4.5d
      libopencv-features2d4.5d libopencv-flann4.5d libopencv-imgproc4.5d libopencv-ml4.5d
      libopencv-objdetect4.5d libopencv-video4.5d libopenexr25 libopenh264-6 libopenvdb10.0t64 libopts25
      liborcus-0.17-0 liborcus-parser-0.17-0 libosdcpu3.5.0t64 libosdgpu3.5.0t64 libosmesa6
      libparted-fs-resize0t64 libperl5.34 libphonon4qt5-data libplacebo192 libplist3 libpoppler118 libpostproc55
      libproj22 libprotobuf-lite23 libprotobuf23 libpython3.10 libpython3.10-dev libpython3.10-minimal
      libpython3.10-stdlib libqgpgme7 libqpdf28 libqt5networkauth5 libqtav1 libqtavwidgets1 libraw20 libre2-9
      libshp2 libshp4 libsmbios-c2 libsnapd-glib1 libsnapd-qt1 libsndio7.0:i386 libsoup-2.4-1:i386 libspnav0
      libsquish0 libsrt1.4-gnutls libstb0t64 libstk-4.6.1 libsuitesparseconfig5 libsuperlu5 libswresample3
      libswscale5 libtesseract4 libtexlua53 libtexluajit2 libtiff5 libtiff5:i386 libtinyxml2-9 libumfpack5
      libunistring2 libunistring2:i386 libupnp13 libvkd3d-shader1 libvkd3d1 libvpx7 libvpx7:i386
      libwebkit2gtk-4.0-37 libwebsockets16 libwxbase3.0-0v5 libwxgtk3.0-gtk3-0v5 libx264-163 libxkbregistry0
      libxslt1.1:i386 libyaml-cpp0.7 libzxingcore1 linux-headers-5.15.0-138 linux-headers-5.15.0-138-generic
      linux-headers-5.15.0-139 linux-headers-5.15.0-139-generic linux-lowlatency-headers-5.15.0-136
      linux-lowlatency-headers-5.15.0-138 linuxaudio-new-session-manager marble-plugins marble-qt-data mediainfo
      midisnoop mlocate muon opencv-data p7zip perl-modules-5.34 petri-foo python3-alsaaudio python3-cffi
      python3-jack-client python3-lib2to3 python3-netifaces python3-pbr python3-pycparser python3-sip python3.10
      python3.10-dev python3.10-minimal qwinff rtmpdump sntp synfig-examples ubuntu-advantage-tools
      vkd3d-compiler vocproc xdg-dbus-proxy zita-njbridge
    Use 'apt autoremove' to remove them.
    The following additional packages will be installed:
      gutenprint-locales libamd3 libcamd3 libccolamd3 libcholmod5 libgegl-0.4-0t64 libgimp2.0t64 libgmic1
      libgutenprint-common libgutenprint9 libgutenprintui2-2 libopencv-videoio406t64 libumfpack6
    Suggested packages:
      gmic gutenprint-doc
    The following NEW packages will be installed:
      gimp gimp-cbmplugs gimp-gap gimp-gmic gimp-gutenprint gimp-plugin-registry gimp-texturize
      gutenprint-locales libamd3 libcamd3 libccolamd3 libcholmod5 libgegl-0.4-0t64 libgimp2.0t64 libgmic1
      libgutenprint-common libgutenprint9 libgutenprintui2-2 libopencv-videoio406t64 libumfpack6
    0 upgraded, 20 newly installed, 0 to remove and 0 not upgraded.
    Need to get 17.5 MB of archives.
    After this operation, 74.9 MB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu noble/universe amd64 libamd3 amd64 1:7.6.1+dfsg-1build1 [27.2 kB]
    Get:2 http://archive.ubuntu.com/ubuntu noble/universe amd64 libcamd3 amd64 1:7.6.1+dfsg-1build1 [23.8 kB]
    Get:3 http://archive.ubuntu.com/ubuntu noble/universe amd64 libccolamd3 amd64 1:7.6.1+dfsg-1build1 [25.9 kB]
    Get:4 http://archive.ubuntu.com/ubuntu noble/universe amd64 libcholmod5 amd64 1:7.6.1+dfsg-1build1 [667 kB]
    Get:5 http://archive.ubuntu.com/ubuntu noble/universe amd64 libumfpack6 amd64 1:7.6.1+dfsg-1build1 [268 kB]
    Get:6 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgegl-0.4-0t64 amd64 1:0.4.48-2.4build2 [1,983 kB]
    Get:7 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 libgimp2.0t64 amd64 2.10.36-3ubuntu0.24.04.1 [898 kB]
    Get:8 http://archive.ubuntu.com/ubuntu noble-updates/universe amd64 gimp amd64 2.10.36-3ubuntu0.24.04.1 [4,680 kB]
    Get:9 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-cbmplugs amd64 1.2.2-1.2build2 [31.5 kB]
    Get:10 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-gap amd64 2.6.0+dfsg-7ubuntu3 [1,985 kB]
    Get:11 http://archive.ubuntu.com/ubuntu noble/universe amd64 libopencv-videoio406t64 amd64 4.6.0+dfsg-13.1ubuntu1 [199 kB]
    Get:12 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgmic1 amd64 2.9.4-4build11 [3,617 kB]
    Get:13 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-gmic amd64 2.9.4-4build11 [712 kB]
    Get:14 http://archive.ubuntu.com/ubuntu noble/main amd64 libgutenprint-common all 5.3.4.20220624T01008808d602-1build4 [618 kB]
    Get:15 http://archive.ubuntu.com/ubuntu noble/main amd64 libgutenprint9 amd64 5.3.4.20220624T01008808d602-1build4 [553 kB]
    Get:16 http://archive.ubuntu.com/ubuntu noble/universe amd64 libgutenprintui2-2 amd64 5.3.4.20220624T01008808d602-1build4 [91.4 kB]
    Get:17 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-gutenprint amd64 5.3.4.20220624T01008808d602-1build4 [49.9 kB]
    Get:18 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-plugin-registry amd64 9.20240404 [1,004 kB]
    Get:19 http://archive.ubuntu.com/ubuntu noble/universe amd64 gimp-texturize amd64 2.2-4build2 [19.1 kB]
    Get:20 http://archive.ubuntu.com/ubuntu noble/universe amd64 gutenprint-locales all 5.3.4.20220624T01008808d602-1build4 [7,308 B]
    Fetched 17.5 MB in 3s (6,401 kB/s)              
    Selecting previously unselected package libamd3:amd64.
    (Reading database ... 624405 files and directories currently installed.)
    Preparing to unpack .../00-libamd3_1%3a7.6.1+dfsg-1build1_amd64.deb ...
    Unpacking libamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Selecting previously unselected package libcamd3:amd64.
    Preparing to unpack .../01-libcamd3_1%3a7.6.1+dfsg-1build1_amd64.deb ...
    Unpacking libcamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Selecting previously unselected package libccolamd3:amd64.
    Preparing to unpack .../02-libccolamd3_1%3a7.6.1+dfsg-1build1_amd64.deb ...
    Unpacking libccolamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Selecting previously unselected package libcholmod5:amd64.
    Preparing to unpack .../03-libcholmod5_1%3a7.6.1+dfsg-1build1_amd64.deb ...
    Unpacking libcholmod5:amd64 (1:7.6.1+dfsg-1build1) ...
    Selecting previously unselected package libumfpack6:amd64.
    Preparing to unpack .../04-libumfpack6_1%3a7.6.1+dfsg-1build1_amd64.deb ...
    Unpacking libumfpack6:amd64 (1:7.6.1+dfsg-1build1) ...
    Selecting previously unselected package libgegl-0.4-0t64:amd64.
    Preparing to unpack .../05-libgegl-0.4-0t64_1%3a0.4.48-2.4build2_amd64.deb ...
    Unpacking libgegl-0.4-0t64:amd64 (1:0.4.48-2.4build2) ...
    Selecting previously unselected package libgimp2.0t64:amd64.
    Preparing to unpack .../06-libgimp2.0t64_2.10.36-3ubuntu0.24.04.1_amd64.deb ...
    Unpacking libgimp2.0t64:amd64 (2.10.36-3ubuntu0.24.04.1) ...
    Selecting previously unselected package gimp.
    Preparing to unpack .../07-gimp_2.10.36-3ubuntu0.24.04.1_amd64.deb ...
    Unpacking gimp (2.10.36-3ubuntu0.24.04.1) ...
    Selecting previously unselected package gimp-cbmplugs.
    Preparing to unpack .../08-gimp-cbmplugs_1.2.2-1.2build2_amd64.deb ...
    Unpacking gimp-cbmplugs (1.2.2-1.2build2) ...
    Selecting previously unselected package gimp-gap.
    Preparing to unpack .../09-gimp-gap_2.6.0+dfsg-7ubuntu3_amd64.deb ...
    Unpacking gimp-gap (2.6.0+dfsg-7ubuntu3) ...
    Selecting previously unselected package libopencv-videoio406t64:amd64.
    Preparing to unpack .../10-libopencv-videoio406t64_4.6.0+dfsg-13.1ubuntu1_amd64.deb ...
    Unpacking libopencv-videoio406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Selecting previously unselected package libgmic1:amd64.
    Preparing to unpack .../11-libgmic1_2.9.4-4build11_amd64.deb ...
    Unpacking libgmic1:amd64 (2.9.4-4build11) ...
    Selecting previously unselected package gimp-gmic.
    Preparing to unpack .../12-gimp-gmic_2.9.4-4build11_amd64.deb ...
    Unpacking gimp-gmic (2.9.4-4build11) ...
    Selecting previously unselected package libgutenprint-common.
    Preparing to unpack .../13-libgutenprint-common_5.3.4.20220624T01008808d602-1build4_all.deb ...
    Unpacking libgutenprint-common (5.3.4.20220624T01008808d602-1build4) ...
    Selecting previously unselected package libgutenprint9.
    Preparing to unpack .../14-libgutenprint9_5.3.4.20220624T01008808d602-1build4_amd64.deb ...
    Unpacking libgutenprint9 (5.3.4.20220624T01008808d602-1build4) ...
    Selecting previously unselected package libgutenprintui2-2.
    Preparing to unpack .../15-libgutenprintui2-2_5.3.4.20220624T01008808d602-1build4_amd64.deb ...
    Unpacking libgutenprintui2-2 (5.3.4.20220624T01008808d602-1build4) ...
    Selecting previously unselected package gimp-gutenprint.
    Preparing to unpack .../16-gimp-gutenprint_5.3.4.20220624T01008808d602-1build4_amd64.deb ...
    Unpacking gimp-gutenprint (5.3.4.20220624T01008808d602-1build4) ...
    Selecting previously unselected package gimp-plugin-registry.
    Preparing to unpack .../17-gimp-plugin-registry_9.20240404_amd64.deb ...
    Unpacking gimp-plugin-registry (9.20240404) ...
    Selecting previously unselected package gimp-texturize.
    Preparing to unpack .../18-gimp-texturize_2.2-4build2_amd64.deb ...
    Unpacking gimp-texturize (2.2-4build2) ...
    Selecting previously unselected package gutenprint-locales.
    Preparing to unpack .../19-gutenprint-locales_5.3.4.20220624T01008808d602-1build4_all.deb ...
    Unpacking gutenprint-locales (5.3.4.20220624T01008808d602-1build4) ...
    Setting up libamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Setting up libgutenprint-common (5.3.4.20220624T01008808d602-1build4) ...
    Setting up libcamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Setting up gutenprint-locales (5.3.4.20220624T01008808d602-1build4) ...
    Setting up libopencv-videoio406t64:amd64 (4.6.0+dfsg-13.1ubuntu1) ...
    Setting up libccolamd3:amd64 (1:7.6.1+dfsg-1build1) ...
    Setting up libgutenprint9 (5.3.4.20220624T01008808d602-1build4) ...
    Setting up libgmic1:amd64 (2.9.4-4build11) ...
    Setting up libcholmod5:amd64 (1:7.6.1+dfsg-1build1) ...
    Setting up libumfpack6:amd64 (1:7.6.1+dfsg-1build1) ...
    Setting up libgutenprintui2-2 (5.3.4.20220624T01008808d602-1build4) ...
    Setting up libgegl-0.4-0t64:amd64 (1:0.4.48-2.4build2) ...
    Setting up libgimp2.0t64:amd64 (2.10.36-3ubuntu0.24.04.1) ...
    Setting up gimp (2.10.36-3ubuntu0.24.04.1) ...
    Setting up gimp-gap (2.6.0+dfsg-7ubuntu3) ...
    Setting up gimp-gmic (2.9.4-4build11) ...
    Setting up gimp-cbmplugs (1.2.2-1.2build2) ...
    Setting up gimp-gutenprint (5.3.4.20220624T01008808d602-1build4) ...
    Setting up gimp-plugin-registry (9.20240404) ...
    Setting up gimp-texturize (2.2-4build2) ...
    Processing triggers for desktop-file-utils (0.27-2build1) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.4) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for mailcap (3.70+nmu1ubuntu1) ...
    ```

Also reinstall KDEnlive and Krita, which were also not included in the upgrade:

??? terminal "`apt install kdenlive krita krita-gmic -y`"

     ``` console
    # apt install kdenlive krita krita-gmic -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    krita is already the newest version (1:5.2.2+dfsg-2build8).
    krita set to manually installed.
    The following packages were automatically installed and are no longer required:
      acpi-support acpid agordejo blender-data cyclist digikam-data encfs ffmpeg2theora fonts-kacst
      fonts-kacst-one fonts-khmeros-core fonts-lao fonts-lklug-sinhala fonts-sil-abyssinica fonts-sil-padauk
      fonts-thai-tlwg fonts-tibetan-machine fonts-tlwg-garuda fonts-tlwg-garuda-ttf fonts-tlwg-kinnari
      fonts-tlwg-kinnari-ttf fonts-tlwg-laksaman fonts-tlwg-laksaman-ttf fonts-tlwg-loma fonts-tlwg-loma-ttf
      fonts-tlwg-mono fonts-tlwg-mono-ttf fonts-tlwg-norasi fonts-tlwg-norasi-ttf fonts-tlwg-purisa
      fonts-tlwg-purisa-ttf fonts-tlwg-sawasdee fonts-tlwg-sawasdee-ttf fonts-tlwg-typewriter
      fonts-tlwg-typewriter-ttf fonts-tlwg-typist fonts-tlwg-typist-ttf fonts-tlwg-typo fonts-tlwg-typo-ttf
      fonts-tlwg-umpush fonts-tlwg-umpush-ttf fonts-tlwg-waree fonts-tlwg-waree-ttf foo-yc20 freeglut3 gamin
      gcc-12-base:i386 haveged irqbalance kross ksystemlog lib2geom1.1.0 libabsl20210324 libamd2 libappimage0
      libappstream4 libappstreamqt2 libarmadillo10 libart-2.0-2 libastro1 libatk1.0-data libavcodec58
      libavdevice58 libavfilter7 libavformat58 libavif13 libavutil56 libblockdev-crypto2 libblockdev-fs2
      libblockdev-loop2 libblockdev-part-err2 libblockdev-part2 libblockdev-swap2 libblockdev-utils2
      libblockdev2 libboost-filesystem1.74.0 libboost-iostreams1.74.0 libboost-locale1.74.0 libboost-regex1.74.0
      libboost-thread1.74.0 libbpf0 libcamd2 libcapi20-3t64 libcbor0.8 libccolamd2 libcfitsio9
      libchamplain-0.12-0 libchamplain-gtk-0.12-0 libcholmod3 libclutter-1.0-0 libclutter-1.0-common
      libclutter-gtk-1.0-0 libcodec2-1.0 libcogl-common libcogl-pango20 libcogl-path20 libcogl20 libcolamd2
      libcrypto++8t64 libcupsfilters1 libdav1d5 libdcmtk16 libdcmtk17t64 libdns-export1110 libdraco4
      libdrm-nouveau2:i386 libembree4-4 libev4t64 libevent-pthreads-2.1-7t64 libfaudio0 libflac++6v5 libflac8
      libflac8:i386 libfltk1.1 libfontembed1 libgamin0 libgav1-0 libgcab-1.0-0 libgeos3.10.2 libgit2-1.1
      libglade2-0 libgnomecanvas2-0 libgnomecanvas2-common libgps28 libgssdp-1.2-0 libgtkmm-2.4-1t64
      libgupnp-1.2-1 libgupnp-igd-1.0-4 libhavege2 libicu70 libicu70:i386 libilmbase25 libisc-export1105
      libixml10 libjavascriptcoregtk-4.0-18 libjemalloc2 libkdecorations2private9 libkdynamicwallpaper1
      libkf5akonadi-data libkf5akonadicontact-data libkf5akonadicontact5abi1 libkf5akonadicore-bin
      libkf5akonadicore5abi2 libkf5akonadiprivate5abi2 libkf5akonadiwidgets5abi1 libkf5baloowidgets-data
      libkf5calendarcore5abi2 libkf5contacteditor5 libkf5grantleetheme-data libkf5grantleetheme-plugins
      libkf5grantleetheme5 libkf5jsapi5 libkf5krosscore5 libkf5krossui5 libkf5libkleo5abi1 libkf5mime-data
      libkf5mime5abi2 libkf5pimtextedit-data libkf5pimtextedit5abi3 libkf5waylandserver5 libkpim5akonadi-data
      libkwaylandserver5 libkwinxrenderutils13 liblcms2-2:i386 libldap-2.5-0:i386 libllvm13 libllvm15t64
      libllvm15t64:i386 liblog4cplus-2.0.5t64 liblua5.1-0 libmagick++-6.q16-8 libmagickcore-6.q16-6
      libmagickcore-6.q16-6-extra libmagickwand-6.q16-6 libmanette-0.2-0 libmarblewidget-qt5-28 libmetis5
      libmicrohttpd12t64 libmpdec3 libmujs1 libnetpbm10 libnetplan0 libnfs13 liboggkate1 libokular5core9
      libopenal1:i386 libopencolorio1v5 libopencv-calib3d4.5d libopencv-core4.5d libopencv-dnn4.5d
      libopencv-features2d4.5d libopencv-flann4.5d libopencv-imgproc4.5d libopencv-ml4.5d
      libopencv-objdetect4.5d libopencv-video4.5d libopenexr25 libopenh264-6 libopenvdb10.0t64 libopts25
      liborcus-0.17-0 liborcus-parser-0.17-0 libosdcpu3.5.0t64 libosdgpu3.5.0t64 libosmesa6
      libparted-fs-resize0t64 libperl5.34 libphonon4qt5-data libplacebo192 libplist3 libpoppler118 libpostproc55
      libproj22 libprotobuf-lite23 libprotobuf23 libpython3.10 libpython3.10-dev libpython3.10-minimal
      libpython3.10-stdlib libqgpgme7 libqpdf28 libqtav1 libqtavwidgets1 libraw20 libre2-9 libshp2 libshp4
      libsmbios-c2 libsnapd-glib1 libsnapd-qt1 libsndio7.0:i386 libsoup-2.4-1:i386 libspnav0 libsquish0
      libsrt1.4-gnutls libstb0t64 libstk-4.6.1 libsuitesparseconfig5 libsuperlu5 libswresample3 libswscale5
      libtesseract4 libtexlua53 libtexluajit2 libtiff5 libtiff5:i386 libtinyxml2-9 libumfpack5 libunistring2
      libunistring2:i386 libupnp13 libvkd3d-shader1 libvkd3d1 libvpx7 libvpx7:i386 libwebkit2gtk-4.0-37
      libwebsockets16 libwxbase3.0-0v5 libwxgtk3.0-gtk3-0v5 libx264-163 libxkbregistry0 libxslt1.1:i386
      libyaml-cpp0.7 libzxingcore1 linux-headers-5.15.0-138 linux-headers-5.15.0-138-generic
      linux-headers-5.15.0-139 linux-headers-5.15.0-139-generic linux-lowlatency-headers-5.15.0-136
      linux-lowlatency-headers-5.15.0-138 linuxaudio-new-session-manager marble-plugins marble-qt-data midisnoop
      mlocate muon opencv-data p7zip perl-modules-5.34 petri-foo python3-alsaaudio python3-cffi
      python3-jack-client python3-lib2to3 python3-netifaces python3-pbr python3-pycparser python3-sip python3.10
      python3.10-dev python3.10-minimal qwinff rtmpdump sntp synfig-examples ubuntu-advantage-tools
      vkd3d-compiler vocproc xdg-dbus-proxy zita-njbridge
    Use 'apt autoremove' to remove them.
    The following NEW packages will be installed:
      kdenlive krita-gmic melt
    0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
    Need to get 0 B/3,769 kB of archives.
    After this operation, 11.3 MB of additional disk space will be used.
    Selecting previously unselected package melt.
    (Reading database ... 625685 files and directories currently installed.)
    Preparing to unpack .../melt_7.22.0-1build6_amd64.deb ...
    Unpacking melt (7.22.0-1build6) ...
    Selecting previously unselected package kdenlive.
    Preparing to unpack .../kdenlive_4%3a23.08.5-0ubuntu4_amd64.deb ...
    Unpacking kdenlive (4:23.08.5-0ubuntu4) ...
    Selecting previously unselected package krita-gmic.
    Preparing to unpack .../krita-gmic_2.9.4-4build11_amd64.deb ...
    Unpacking krita-gmic (2.9.4-4build11) ...
    Setting up melt (7.22.0-1build6) ...
    Setting up krita-gmic (2.9.4-4build11) ...
    Setting up kdenlive (4:23.08.5-0ubuntu4) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for mailcap (3.70+nmu1ubuntu1) ...
    Processing triggers for desktop-file-utils (0.27-2build1) ...
    ```

There may be more packages to install later, but so far the above seems to cover what
was not automatically preserved during the upgrade process.
