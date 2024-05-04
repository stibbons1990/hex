---
title:  "Getting started with Fallout 3 and 4 GOTY"
date:   2024-05-04 07:50:06 +0200
categories: steam proton videogames bethesda fallout
---

A few weeks ago
[Fallout 4: Game of the Year Edition](https://store.steampowered.com/sub/199943/)
was on sale (75% off) and so I finally pulled the trigger on it.
It had been in my wishlist for a few years, since I spent nearly 150 hours on
Fallout 3, but I had been reluctant on account of many people complaining
that its story was weaker and it was more focused on exploration and combat.
Well, people say the same (and I'd agree) about
[The Elder Scrolls V: Skyrim](https://store.steampowered.com/app/489830/)
and I spent already more than 200 hours on it and *still* want to go back.

## Fallout 4

As usualy, before purchasing the game I checked all recent
[Proton reports for Fallout 4](https://www.protondb.com/app/377160),
because *one does not simply* run Windows games on Linux.
At least, not *always* without a little tinkering.

Among all the recent recommendations, this one worked for me:

*  Switch to **GE-Proton8-16** (not experimental)
*  Limit framerate to 60: `DXVK_FRAME_RATE=60 %command%`

[Screen resolution](#screen-resolution) was limited to **1920x1200**,
but that was easy to change.

Other recommendations I found not (yet?) necessary,
but seemed likely to be useful (have not tried yet):

*  `gamemoderun ENABLE_VKBASALT=1 DXVK_ASYNC=1 WINEDLLOVERRIDES="xaudio2_7=n,b" %command%`
*  Install `faudio` from `protontricks`
*  [Emulate virtual desktop](#emulate-virtual-desktop)

### Screen Resolution

My only slight discontent with the initial setup was that
1920x1200 looks a little too small on a 3440x1440 screen.
I wouldn't *terribly* mind 2560x1440, but at least I'd
rather use all the screen's vertical space.

This is just because the game defaults to running windowed.
At least with the current (recent) version, disabling the
windows mode in the launcher (before launching the game)
will correctly default to glorious 3440x1440.

In window mode, updatding the values in `Fallout4Prefs.ini` under
`.local/share/Steam/steamapps/common/Fallout 4/Fallout4/`
had no effect:

```
bBorderless=1
bFull Screen=0
iSize H=1440
iSize W=2560
```

### Controller Support

Initially the game would not detect my DualShock controller,
but this wasn't the first game to show such problem
(I had seen the same with Dirt Rally when running on Proton)
so I searched a bit a found a simple workaround
[here](https://bbs.archlinux.org/viewtopic.php?id=268515#:~:text=Re%3A%20Proton%20games%20are%20not%20recognizing%20controller&text=Solved%20it%20by%20going%20to,%22Override%20for%22%2Doption.):

> Solved it by going to "Properties" on the game, "Controllers",
> then "Enable Steam input" under the "Override for"-option.

### Emulate virtual desktop

Use protontricks 377160 winecfg to get to winecfg,
then enable **Graphics > Emulate virtual desktop** and
set the size to 1920x1080 (or 2560x1440 in my case).
This fixes the infamous `Alt-Tab` bug.

### Mods

A user reports 

> The game played as expected under Proton.
> I was able to install F4SE and otherwise mod the game using Mod Organizer 2,
> which functioned similarly to Vortex (Vortex does not run in WINE).
> Performance was comparable to Windows, with no major boon or deficit.

Another user reports that *Mods need a little tweaks*:

> For mods i used the Vortex (mod manager), downloaded from the official website (also download dotnet 6.0 from there).

> 1.  Install dotnet6 and Vortex (in that order) on a Wine PREFIX outside the Proton prefix.
> 1.  Look over the proton PREFIX location of your game, because you need the folder "My Games/Fallout4".
> 1.  Link your `.../Documents/My Games/Fallout4` to your actual home:
>    ```
>    $ ln -sf  \
>      $path/SteamLibrary/steamapps/compatdata/377160/pfx/drive_c/users/steamuser/Documents/My\ Games/Fallout4 \
>      /home/$User/Documents/My\ Games/Fallout 4
>    ```
> 1.  Open Vortex, setup Fallout 4 and manually add the install location of the game.
> 1.  Vortex should look for your `/Documents/My Games/Fallout 4` that is linked, but you could need to set it manually.
> 1.  Everything should work now.

**Note:** Change `$path` and `$User` to your actual.

### May 2024 Update

Nothing appears to have broken, except perhaps my hope to experience the game as it used to be.

## Fallout 3

While working on this, I realize I never took note of similar tweaks I used to play Fallout 3.
I went back and updated its configuration, with what seems to work best (for me) now:

*  Switch to GE-Proton8-16 (not experimental)
*  `PROTON_USE_WINED3D=1 PROTON_FORCE_LARGE_ADDRESS_AWARE=1 DXVK_FRAME_RATE=60 %command%`

First attempt to run shows 

> You must use "Turn Windows features on of off"  
> in the Control Panel to install or configure  
> Microsoft .NET Framework 3.0 x64.

But then it just works. What is even better,
it actually works well at full screen on
**ultra-wide 3440x1440**.