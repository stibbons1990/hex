---
title:  "Modding a Steam game on Linux: The Elder Scrolls V: Skyrim Special Edition"
date:   2023-10-14 06:10:14 +0200
categories: steam proton videogames bethesda skyrim mods
---

[The Elder Scrolls V: Skyrim Special Edition](https://store.steampowered.com/app/489830/The_Elder_Scrolls_V_Skyrim_Special_Edition/)
is admittedly one of my favorite games, having poured 200
hours and still wanted to go back and play it again.

## Vanilla game

According to
[Proton reports](https://www.protondb.com/app/489830)
this game runs *mostly* pretty well without tweaks, 
so the first time I run it there was no tweaking or
tinkering involved. The game spent 2 minutes compiling
Vulkan shaders, detected optimal video settings and run.

### Audio fix

However, a few Proto reports indicate that `xact_x64` is
required to fix the issue where voices are missing and
indeed I experienced this myself. To fix this, install
this with `winetricks`:

```
$ WINEARCH=win64 WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/489830/pfx winetricks xact
$ WINEARCH=win64 WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/489830/pfx winetricks xact_x64
```

*However*, after installing these the game no longer runs and
there is no clear error message even in `~/.xsession-errors`;
forcing the game to run on **Proton 7.0-6** did the trick and
the Skyrim theme (vocal music) plays as soon as it starts.

.local/share/Steam/steamapps/common/Skyrim\ Special\ Edition/Skyrim

### Screen resolution

The game is old and conservative, so it runs at a mere
1920x1080 resolution despite having an ultrawide 3440x1440
display. The game is *particularly* known to work poorly with
ultrawide screens, it is possible to set the game to run at
3440x1440 (full screen) but that will make UI elements at the
bottom and top not accessible, including controller button
hints and essential stats like gold and weight.

To set the change resolution, edit the `SkyrimPrefs.ini`
that is under your *personal* files, *not* the one that is
part of the game files:


```
$ cd; find . -name SkyrimPrefs.ini
./.local/share/Steam/steamapps/compatdata/489830/pfx/drive_c/users/steamuser/Documents/My Games/Skyrim Special Edition/SkyrimPrefs.ini
./.local/share/Steam/steamapps/common/Skyrim Special Edition/Skyrim/SkyrimPrefs.ini
```

Edit the file under the `.../steamuser/Documents/...` path,
otherwise the changes will not be effective. Change these
lines

```ini
bFull Screen=0
iSize H=1200
iSize W=1920
```

To run on 3440x1440 full screen:

```ini
bFull Screen=1
iSize H=1440
iSize W=3440
```

**Note:** see
[Guide:Skyrim Configuration Settings](https://stepmodifications.org/wiki/Guide:Skyrim_Configuration_Settings)
and the
[Guide:SkyrimPrefs INI](https://stepmodifications.org/wiki/Guide:SkyrimPrefs_INI)
for full details on how to configure Skyrim.

## Mods

There is only one `Data` folder to install mods in:

```
~/.local/share/Steam/steamapps/common/Skyrim Special Edition/Data
```

### Backup Skyrim Special Edition

Through the journey of installing mods and other tools, is is
recommended to make backups of the entire game folder
(`Skyrim Special Edition`) to rollback changes *when*
(not *if*) something breaks.

To make multiple backups, each with a timestamp:

```bash
$ cd ~/.local/share/Steam/steamapps/common/
$ rsync -ruta \
  "Skyrim Special Edition" \
  "Backup of Skyrim Special Edition on $(date +"%Y-%m-%d-%H-%M-%S")"
```

### Vortex mod manager (failed)

[This Proton report](https://www.protondb.com/app/489830#v5vFZx3xJY)
recommends using
[Vortex mod manager](https://www.nexusmods.com/about/vortex/) *installed through Lutris using the official install configuration.*  That sounds like it *should* be easy enough,
but even better there is a detailed walkthrough to 
[Manage Skyrim SE mods via Vortex Mod Manager in Linux](https://gist.github.com/attusan/a095d1d84a3b1aef98a3343e536b3045#manage-skyrim-se-mods-via-vortex-mod-manager-in-linux).

This walkthrough recommend installing an old version of Vortex
via a YML file breated by
[rockerbacon](https://github.com/rockerbacon)

```bash
$ wget -O ~/Downloads/vortex.yml \
  https://github.com/rockerbacon/modorganizer2-linux-installer/releases/download/1.9.3/vortex.yml
$ lutris -i ~/Downloads/vortex.yml
```

The installation is a bit *rocky*; after installing a few
components the installer seems to fail with *exit code 256*,
but hitting Back and then Continue/Install again it actually
works. It shows the instructrions to disable the Internet
connection and after that it installs and launches Vortex
just fine.

Add the Steam library under
**Settings => Games => Add Search Directory**
and enter the path based on **/** (root) as
`/home/coder/.local/share/Steam/steamapps/common`,
then do a full scan (**Games => Scan => Scan:Full**)
and that should show all your (installed) Steam games.

Add Skyrim Special Edition with the **`+`** sign on its
thumbnail, then go to **MODS** section and drop a file
in the **Drop File(s)** are to install mods manually.

**Warning:** **DO NOT** use the "SSE Engine Fixes" mod,
it will mess up your audio.

### SKSE

The [Skyrim Script Extender (SKSE)](https://www.nexusmods.com/skyrimspecialedition/mods/30379)
*is a tool used by many Skyrim mods that expands scripting*
*capabilities and adds additional functionality to the game.*
*Once installed, no additional steps are needed to launch*
*Skyrim with SKSE's added functionality. You can start the*
*game using SKSE from `skse64_loader.exe`.*

#### Using Vortex (failed)

Download the version
*Compatible with Skyrim Special Edition 1.6.640 from Steam* `Skyrim Script Extender (SKSE64)-30379-2-2-3-1665515370.7z`
and drop the file into Vortex, then click **Install**.

The whole Vortex window, or possibly a new, goes white and
does nothing. After a while, it tries to go full screen,
and still does nothing. No new files are installed.

#### Manual install (ok)

Extract and copy all SKSE files and folders to the Skyrim SE game folder, then rename executables:

```
$ 7z x Skyrim\ Script\ Extender\ \(SKSE64\)-30379-2-2-3-1665515370.7z 

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,16 CPUs AMD Ryzen 7 5800X 8-Core Processor              (A20F12),ASM,AES-NI)

Scanning the drive for archives:
1 file, 748635 bytes (732 KiB)

Extracting archive: Skyrim Script Extender (SKSE64)-30379-2-2-3-1665515370.7z
--
Path = Skyrim Script Extender (SKSE64)-30379-2-2-3-1665515370.7z
Type = 7z
Physical Size = 748635
Headers Size = 7306
Method = LZMA:5m BCJ2
Solid = +
Blocks = 2

Everything is Ok

Folders: 15
Files: 536
Size:       4282404
Compressed: 748635

$ cp -a \
  skse64_2_02_03/skse64_* \
  ~/.local/share/Steam/steamapps/common/Skyrim\ Special\ Edition/
$ cp -a \
  skse64_2_02_03/Data/Scripts/ \
  ~/.local/share/Steam/steamapps/common/Skyrim\ Special\ Edition/Data/
```

At this point there are 2 ways to launch Skyrim using the
SKSE loader.
[This Proto report](https://www.protondb.com/app/489830#v5vFZx3xJY)
recommends thesee launch options to launch SKSE without
having to rename executables (seems to breaks some mods):

```bash
$(echo %command% | sed -r "s/proton waitforexitandrun .*/proton waitforexitandrun/") "$STEAM_COMPAT_INSTALL_PATH/skse64_loader.exe"
```

However, these launch options require a custom proton version.

Also, in order to use the *very desirable* [Sky UI](#sky-ui)
mod, it is highly recommended to use
[Glorious Eggroll](#glorious-eggroll)
instead of the standard Proton release.

The simpler alterantive si to simply rename files:

```
$ mv SkyrimSELauncher.exe SkyrimSELauncher.orig.exe
$ mv skse64_loader.exe SkyrimSELauncher.exe
```

### Glorious Eggroll

Install
[Glorious Eggroll](https://github.com/GloriousEggroll/proton-ge-custom/releases)
using the
[Manual method for Native Steam](https://github.com/GloriousEggroll/proton-ge-custom#manual):

```bash
# make temp working directory
mkdir /tmp/proton-ge-custom
cd /tmp/proton-ge-custom

# download  tarball
curl -sLOJ "$(curl -s https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/latest | grep browser_download_url | cut -d\" -f4 | grep .tar.gz)"

# download checksum
curl -sLOJ "$(curl -s https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/latest | grep browser_download_url | cut -d\" -f4 | grep .sha512sum)"

# check tarball with checksum
sha512sum -c ./*.sha512sum
# if result is ok, continue

# make steam directory if it does not exist
mkdir -p ~/.steam/root/compatibilitytools.d

# extract proton tarball to steam directory
tar -xf GE-Proton*.tar.gz \
  -C ~/.steam/root/compatibilitytools.d/
```


### Sky UI

Download the updated UI mod
[Sky_UI](https://www.nexusmods.com/skyrimspecialedition/mods/12604) and install it manually (simple):

```
$ 7z x SkyUI_5_2_SE-12604-5-2SE.7z 

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,16 CPUs AMD Ryzen 7 5800X 8-Core Processor              (A20F12),ASM,AES-NI)

Scanning the drive for archives:
1 file, 2783417 bytes (2719 KiB)

Extracting archive: SkyUI_5_2_SE-12604-5-2SE.7z
--
Path = SkyUI_5_2_SE-12604-5-2SE.7z
Type = 7z
Physical Size = 2783417
Headers Size = 300
Method = LZMA2:3m
Solid = +
Blocks = 1

Everything is Ok

Folders: 1
Files: 6
Size:       2931171
Compressed: 2783417
$ cp -a SkyUI_* \
  ~/.local/share/Steam/steamapps/common/Skyrim\ Special\ Edition/Data/
```

At this point it becomes clear, not only from the game UI but
also from every single forum thread, that a Bethesda.net
account is required to even load mods in this game.

### Ultrawide UI

[Complete Widescreen Fix for Vanilla and SkyUI 2.2 and 5.2 SE](https://www.nexusmods.com/skyrimspecialedition/mods/1778)

```
$ $ unrar x Complete\ Widescreen\ Fix\ for\ SkyUI\ 5.2\ SE\ Alpha\ -\ 2560x1080-1778-2-0.rar 

UNRAR 6.11 beta 1 freeware      Copyright (c) 1993-2022 Alexander Roshal


Extracting from Complete Widescreen Fix for SkyUI 5.2 SE Alpha - 2560x1080-1778-2-0.rar

Creating    interface                                                 OK
Extracting  interface/bartermenu.swf                                  OK 
Extracting  interface/containermenu.swf                               OK 
Extracting  interface/craftingmenu.swf                                OK 
Extracting  interface/dialoguemenu.swf                                OK 
Extracting  interface/giftmenu.swf                                    OK 
Extracting  interface/inventorymenu.swf                               OK 
Extracting  interface/lockpickingmenu.swf                             OK 
Extracting  interface/magicmenu.swf                                   OK 
Extracting  interface/map.swf                                         OK 
Extracting  interface/messagebox.swf                                  OK 
Extracting  interface/quest_journal.swf                               OK 
Creating    interface/skyui                                           OK
Extracting  interface/skyui/bottombar.swf                             OK 
Extracting  interface/skyui/configpanel.swf                           OK 
Extracting  interface/sleepwaitmenu.swf                               OK 
Extracting  interface/statsmenu.swf                                   OK 
Extracting  interface/trainingmenu.swf                                OK 
Extracting  interface/tweenmenu.swf                                   OK 
Extracting  widescreen_skyui_fix.esp                                  OK 
Extracting  widescreen_skyui_fix.ini                                  OK 
Creating    interface/exported                                        OK
Extracting  interface/exported/racesex_menu.gfx                       OK 
All OK

$ cp -a interface \
  ~/.local/share/Steam/steamapps/common/Skyrim\ Special\ Edition/Data/
```

Then edit `SkyrimPrefs.ini` to [change screen resolution](#screen-resolution).

At this point it becomes clear, not only is *theoretically*
bad that a Bethesda.net account is required to even load mods,
it is **actually terrible in practice** when 10 out of 10
times trying to load mods all you get is the infamous
**Couldn’t connect to the Bethesda.net servers** error.

[accounts.bethesda.net/en/linked-accounts](https://accounts.bethesda.net/en/linked-accounts) shows the Steam account is
linked, yet the error persists and there is nothing you can
do but wait... forever?

### Mod Manager

[One recommendation](https://www.protondb.com/app/489830#FBZCZuUlZn)
to get modding with this game is to

> Install mods using MO2 through steam tinker launch and make
> your skse data folder into a mod and install through MO2.
> Also having skse installed first will help it get auto
> detected by MO2.
