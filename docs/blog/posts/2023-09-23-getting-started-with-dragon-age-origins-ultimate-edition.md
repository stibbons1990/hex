---
date: 2023-09-23
categories:
 - steam
 - proton
 - videogames
 - ea
 - bioware
title: "Getting started with Dragon Age: Origins - Ultimate Edition"
---

[Dragon Age: Origins - Ultimate Edition](https://store.steampowered.com/app/47810/Dragon_Age_Origins__Ultimate_Edition/)
was on sale since last weekend, and today I *fell for it*.

Before purchasing the game, I checked all recent
[Proton reports for Dragon Age: Origins - Ultimate Edition](https://www.protondb.com/app/47810),
because *one does not simply* run Windows games on Linux.
At least, not *always* without a little tinkering.

<!-- more --> 

### Launch flags

Among all the recent recommendations, this one worked for me:

``` console
__GLX_VENDOR_LIBRARY_NAME=nvidia PROTON_FORCE_LARGE_ADDRESS_AWARE=1 RADV_TEX_ANISO=16 PROTON_USE_D9VK=1 gamemoderun %command%
```

Adding `__NV_PRIME_RENDER_OFFLOAD=1` caused the game to get
stuck in the splash screen, so I had to remove that.

#### Update (2025-01-05)

At this point *more recent* Proton reports suggest that it i
best to stick with Proton **7.0-6**. This does seem to help,
although at this point the difference is only that the game
crashes right after character selection rather than not starting.

The same crash happens also with Proton version 6 and GE 8.16.
With the latter the crash also left the mouse cursor of the game
active and no other application reacted to (or received) click
events, at least for several seconds. After the crash it is still
necessary to ask Steam to stop the game, or even exit Steam.

Proton Experimental, Hotfix and verion 9 had the same crash.

### Install `physx`

Although I'm not sure it helped, I did try installing
NVidia's Physx library:

``` console
$ winetricks physx
```

The installation seems to have been made only for the default
`WINEPREFIX` in `$HOME` while the game uses the one in
`$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx` so
the correct command would be

``` console
$ WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx winetricks physx
```

#### Update (2025-01-05)

This worked out of the box in Ubuntu Studio 22.04, but later in
Ubuntu Studio 24.04 it failed with this error:

``` console
$ WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx winetricks physx
Executing cd /usr/bin
------------------------------------------------------
warning: Unknown file arch of /usr/bin/wine.
------------------------------------------------------
```

Searching around for that error did not lead anywhere useful,
but poking around did: `/usr/bin/wine` essentially symlinks to
`/usr/bin/wine-stable` which is just a shell script to decide
whether to launch `/usr/lib/wine/wine` or `/usr/lib/wine/wine64`
but only the latter is installed so this can be bypassed with
the `WINE` variable pointing directly to the binary:

??? terminal "`WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx winetricks physx`"

    ``` console
    $ WINE=/usr/lib/wine/wine64 WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx winetricks physx
    Executing cd /usr/bin
    ------------------------------------------------------
    warning: You are using a 64-bit WINEPREFIX. Note that many verbs only install 32-bit versions of packages. If you encounter problems, please retest in a clean 32-bit WINEPREFIX before reporting a bug.
    ------------------------------------------------------
    ------------------------------------------------------
    warning: You apppear to be using Wine's new wow64 mode. Note that this is EXPERIMENTAL and not yet fully supported. If reporting an issue, be sure to mention this.
    ------------------------------------------------------
    Using winetricks 20240105 - sha256sum: 17da748ce874adb2ee9fed79d2550c0c58e57d5969cc779a8779301350625c55 with wine-9.0 (Ubuntu 9.0~repack-4build3) and WINEARCH=win64
    Executing w_do_call physx
    ------------------------------------------------------
    warning: You are using a 64-bit WINEPREFIX. Note that many verbs only install 32-bit versions of packages. If you encounter problems, please retest in a clean 32-bit WINEPREFIX before reporting a bug.
    ------------------------------------------------------
    ------------------------------------------------------
    warning: You apppear to be using Wine's new wow64 mode. Note that this is EXPERIMENTAL and not yet fully supported. If reporting an issue, be sure to mention this.
    ------------------------------------------------------
    Executing load_physx 
    ------------------------------------------------------
    warning: /home/ponder/.local/share/Steam/steamapps/compatdata/47810/pfx/dosdevices/c:/Program Files (x86)/NVIDIA Corporation/PhysX/Engine/86C5F4F22ECD/APEX_Particles_x64.dll is not a regular file, not checking sha256sum
    ------------------------------------------------------
    Executing cd /home/ponder/.cache/winetricks/physx
    Downloading https://us.download.nvidia.com/Windows/9.21.0713/PhysX_9.21.0713_SystemSoftware.exe to /home/ponder/.cache/winetricks/physx
    --2025-01-05 23:25:47--  https://us.download.nvidia.com/Windows/9.21.0713/PhysX_9.21.0713_SystemSoftware.exe
    Resolving us.download.nvidia.com (us.download.nvidia.com)... 192.229.221.58, 2606:2800:233:ef6:15dd:1ece:1d50:1e1
    Connecting to us.download.nvidia.com (us.download.nvidia.com)|192.229.221.58|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 27152680 (26M) [application/octet-stream]
    Saving to: ‘PhysX_9.21.0713_SystemSoftware.exe’

    PhysX_9.21.0713_Sys 100%[=================>]  25.89M  55.4MB/s    in 0.5s    

    2025-01-05 23:25:48 (55.4 MB/s) - ‘PhysX_9.21.0713_SystemSoftware.exe’ saved [27152680/27152680]

    Executing cd /home/ponder
    Executing cd /home/ponder/.cache/winetricks/physx
    Executing /usr/lib/wine/wine64 PhysX_9.21.0713_SystemSoftware.exe
    0238:err:environ:init_peb starting L"Y:\\physx\\PhysX_9.21.0713_SystemSoftware.exe" in experimental wow64 mode
    wine: failed to load L"\\??\\C:\\windows\\syswow64\\ntdll.dll" error c0000135
    Application could not be started, or no application associated with the specif
    ied file.
    ShellExecuteEx failed: Internal error.

    ------------------------------------------------------
    warning: Note: command /usr/lib/wine/wine64 PhysX_9.21.0713_SystemSoftware.exe returned status 1. Aborting.
    ------------------------------------------------------
    ```

### Avoid broken launcher

Also, as reported by
[thehoagie](https://www.protondb.com/users/597403899),

> The game tries to use a launcher. Bypass it because it
> doesn't work. Change the line in this xml file:

``` console
$ vi "${HOME}/.local/share/Steam/steamapps/common/Dragon Age Ultimate Edition/data/DAOriginsLauncher.xml"
```

and change line **247** from

``` xml
<xmlbutton name="play" action="condition" value="FirstRunCheck">
```

to

``` xml
<xmlbutton name="play" action="execute" file="${BINARIES_DIR}\DAOrigins.exe" autoquit="true">
```

With that, the game finally launches. There is a pop-up dialog
about DLCs needing to be activated, which is one of those
things that EA has broken by not maintaining things working.

Years ago there was an option in Steam to obtain the "CD keys",
but it appears that option is no longer feasible and there is
no clear workaround that will work in 2023, except getting the
games from GOG:

*  [What does the "Dragon Age: Origins - Ultimate Edition DLC CD Key" do?](https://steamcommunity.com/app/47810/discussions/0/558752450420490163)
*  [How to get all the DLC](https://steamcommunity.com/app/47810/discussions/0/1698300679774459377/)
*  [Downloadable content (Origins)](https://dragonage.fandom.com/wiki/Downloadable_content_(Origins))

#### Update (2025-01-11)

In the off-chance that this game might have worked better with the
[Steam from snap](./2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#snap-package-removed),
tried installing it again and tried all the above on it.

To install `physx` the snap version of the plx directory was
`$HOME/snap/steam/common/.local/share/Steam/steamapps/compatdata/47810/pfx`
and the game was installed under
`$HOME/snap/steam/common/.local/share/Steam/steamapps/common/Dragon Age Ultimate Edition`.

Despite all the tweaks above and trying all versions of Proton **7.0-6**
and newer, the game would not even show the launcher at all, so that's even worse
than the [recent update with non-snap Steam](#update-2025-01-05).