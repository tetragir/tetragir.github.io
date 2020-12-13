---
layout:	post
title:	Installing Civilization V On Arch Linux
date: 2017-06-03 21:00:00
category: linux/other
tags: linux civilization archlinux games
type: it
platform: Arch Linux
---

I'm not really a gamer myself, but as it turned out, my wife is. She played with Civilization V many years ago on Windows, and wanted to re-play the game now. However, I have successfully eliminated it from everywhere in the household, we run Linux on the desktops and FreeBSD on the “servers”, so we seemed to have a slight problem: how to let the wife play without installing anything Windows?

* TOC
{:toc}

I remembered that Civilization V is available through Steam, so I checked and it turned out, that it runs on Linux! Currently we use Arch Linux on the main desktop PC: it's an i7 rig with an Nvidia graphics card. Would it work, my wife looked at me hopefully. I could only rise to the challenge.

Please note that this short guide is written on June 3rd in 2017 and it is accurate now. Over time, things change and this guide will be outdated. Also this guide is tailored to Arch Linux with a discrete Nvidia graphics card. Though the solution should work on other Linux systems, always consider the differences between distros!

## Installing Steam
The first step is to install Steam. If you run a 32bit system, you can just install it; on 64bit systems the multilib repo has to be enabled. To do that, uncomment the following lines in `/etc/pacman.conf`

```console
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Then just install the steam package: `pacman -S steam`

For me, Steam did not run immediately, I also had to install the steam-native-runtime package: `pacman -S steam-native-runtime`

Running on 64bit it is also important to install the 32bit version of the graphics driver. Install the appropriate one according to this page: https://wiki.archlinux.org/index.php/Xorg#Driver_installation
In my case it was the Nvidia driver: `pacman -S lib32-nvidia-utils`

After installing all of these, I was able to start Steam itself.

## Installing Civilization V
Though it didn't seem tricky at first, when I installed the game on an XFS file system it always complained that a file was not found, and after the intro video the game failed to start. (My wife was sad at this point.) The error message was: `Unable to load texture (LoadingBaseGame.dds)`

I found just one similar report and it was on macOS, but it made sense so I gave it a try. Someone installed a game on an HFS+ disk with case-sensitivity enabled. I threw in an old 250GB HDD to the machine and formatted it to FAT32. After installing the game to the new disk, it found the file and I was able to start the game.

So the solution is to use a file system without case sensitivity.

## Starting the game
From here there was only one problem to be solved. According to a few forum topics the new Nvidia driver changed a few things rendering the game unable to start. I got the following error in the terminal upon starting the game:

```console
GameAction [AppID 8930, ActionID 10] : LaunchApp changed task to Starting with ""
GameAction [AppID 8930, ActionID 10] : LaunchApp changed task to SynchronizingCloud with ""
GameAction [AppID 8930, ActionID 10] : LaunchApp changed task to CreatingProcess with ""
GameAction [AppID 8930, ActionID 10] : LaunchApp waiting for user response to CreatingProcess ""
GameAction[AppID 8930, ActionID 10] : LaunchApp continues with user response "CreatingProcess"
Game update: AppID 8930 "Sid Meier's Civilization V", ProcID 29664, IP 0.0.0.0:0
>>> Adding process 29664 for game ID 8930
GameAction [AppID 8930, ActionID 10] : LaunchApp changed task to WaitingGameWindow with ""
ERROR: ld.so: object '/home/user/.local/share/Steam/ubuntu12_32/gameoverlayrenderer.so' from LD_PRELOAD cannot be preloaded (wrong ELF class: ELFCLASS32): ignored.
ERROR: ld.so: object '/home/user/.local/share/Steam/ubuntu12_64/gameoverlayrenderer.so' from LD_PRELOAD cannot be preloaded (wrong ELF class: ELFCLASS64): ignored.
/mnt/397D-352A/Games/steamapps/common/Sid Meier's Civilization V/./Civ5XP: Symbol `_ZTVN10__cxxabiv120__si_class_type_infoE' has different size in shared object, consider re-linking
/mnt/397D-352A/Games/steamapps/common/Sid Meier's Civilization V/./Civ5XP: Symbol `_ZTVN10__cxxabiv117__class_type_infoE' has different size in shared object, consider re-linking
/mnt/397D-352A/Games/steamapps/common/Sid Meier's Civilization V/./Civ5XP: Symbol `_ZTVN10__cxxabiv121__vmi_class_type_infoE' has different size in shared object, consider re-linking
GameAction [AppID 8930, ActionID 10] : LaunchApp changed task to Completed with ""
>>> Adding process 29665 for game ID 8930
Game removed: AppID 8930 "Sid Meier's Civilization V", ProcID 29664 
No cached sticky mapping in ActivateActionSet.
```

Fortunately this can be solved with a startup parameter. In Steam, in the list of the games, right click on the game and choose properties.

![Civilization V Properties](/images/civ/properties.png)

Under launch options, paste the following: `LD_PRELOAD=/usr/lib/nvidia/libGL.so %command%`

After that, the game did start up and did not crash which is good.
Since then I lost access to the PC and I hadn't been able to regain it from my wife since. (But she is happy now.)
