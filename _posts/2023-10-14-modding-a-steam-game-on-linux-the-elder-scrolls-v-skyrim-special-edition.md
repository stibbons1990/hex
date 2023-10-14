---
title:  "Modding a Steam game on Linux: The Elder Scrolls V: Skyrim Special Edition"
date:   2023-10-14 06:10:14 +0200
categories: steam proton videogames bethesda skyrim mods
---

[The Elder Scrolls V: Skyrim Special Edition](https://store.steampowered.com/app/489830/The_Elder_Scrolls_V_Skyrim_Special_Edition/)
is admittedly one of my favorite games, having poured 200
hours and still wanted to go back and play it again.

## Vanilla

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

## Modding

There is only one `Data` folder to install mods in:

```
~/.local/share/Steam/steamapps/common/Skyrim Special Edition/Data
```


### Mod Manager

[One recommendation](https://www.protondb.com/app/489830#FBZCZuUlZn)
to get modding with this game is to

> Install mods using MO2 through steam tinker launch and make
> your skse data folder into a mod and install through MO2.
> Also having skse installed first will help it get auto
> detcted by MO2.

