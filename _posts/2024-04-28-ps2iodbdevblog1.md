---
title: PS2IODB Development Blog 1
date: 2024-04-28 3:00:00 +0000
categories: [PS2IODB]
tags: [engineering, ps2iodb, opensource]
img_path: /assets/img
---
Welcome to the first development blog for the PS2IODB project. If you're unfamiliar with the project, the goal is to create a means for a community to extract assets from PlayStation 2 save games and contribute them to an open archive.

The first objective is to expore what prior efforts there have been with this goal, or that can contribute to this goal.

A few things were found:
* A couple of projects [[1]](https://gamefaqs.gamespot.com/boards/915821-playstation-2/79956200) [[2]](https://www.ps2-home.com/forum/viewtopic.php?t=1573&start=10) of people sharing screen captures of different icons. This was a noble cause but didn’t encourage collaboration, didn’t share the assets in a usable way, and often didn’t have animations.
* Next was the [ps2iconsys project](https://www.ghulbus-inc.de/projects/ps2iconsys/index.html) by Ghulbus inc. This open source C++ project supported converting save icons to & from obj files. This project did not support animations, and had no GUI.
* Next was [mymc](http://www.csclub.uwaterloo.ca:11068/mymc/), an open source project by Ross Ridge, this Python tool has a GUI which allowed opening memory card files and importing new saves, with the ability to show icons rendered in the app, but not the ability to export.
    * From this I found [mymc+](https://sr.ht/~thestr4ng3r/mymcplus/), a fork by thestranger (Florian Märkl), this had an updated codebase, allowed for importing more file types, and ported the icon viewing window to OpenGL, which improves cross-platform usability.

A little documentation also existed:
* Ross Ridge’s [writings](http://www.csclub.uwaterloo.ca:11068/mymc/ps2mcfs.html).
* [ps2savetools.com](https://www.ps2savetools.com/) with many broken links, which were able to be fixed after getting in contact with the admin.
    * This led to access to [the PDF](https://www.ps2savetools.com/documents/ps2-icon-format-v05/) *PS2 Icon Format v0.5* by Martin Akesson.

After reading Ross' documentation and looking through the mymc codebase I began to understand the memory card was essentially a USB stick with a subset of the FAT filesystem. Each game has its own folder and files within. These folder/file names could be anything, the icon assets were their own files too! The icon.sys file must be present in each file for the ps2 bios to use. It contains data like the game title, background/lighting colours and the filenames for the idle, copy & delete icons.

| Offset | Length | Description |
|--- |--- |--- |
|0|4|"PS2D"|
|4|2|0|
|6|2|Offset of 2nd line in titlename|
|8|4|0|
|12|4|Background transparency when save is selected 0x00(transparent) to 0x80(opaque)|
|16|16|Background color upper left (RGB) 0x00 – 0x80|
|32|16|Background color upper right (RGB) 0x00 – 0x80|
|48|16|Background color lower left (RGB) 0x00 – 0x80|
|64|16|Background color lower right (RGB) 0x00 – 0x80|
|80|16|Light 1 direction|
|96|16|Light 2 direction|
|112|16|Light 3 direction|
|128|16|Light 1 RGB|
|144|16|Light 2 RGB|
|160|16|Light 3 RGB|
|176|16|Ambient light RGB|
|192|68|Title name of savegame (null terminated, S-JIS format)|
|260|64|Filename of normal icon (null terminated)|
|324|64|Filename of icon file displayed when copying (null terminated)|
|388|64|Filename of icon file displayed when deleting (null terminated)|
|452|512|0|

_icon.sys file format structure thanks to [ps2savetools.com](https://www.ps2savetools.com/documents/iconsys-format/)_

I was initially leaning towards using the learnings to writing a C# tool but realised that even though I’m not a Python export, so much groundwork had already been done in *MYMC* (and by extension its fork *MYMC+*) it would be silly to not utilise it (and would also be a great opportunity to learn some more Python!).

I cloned the mymc+ fork as this was the most advanced project and its licence permitted, it would be the best possible starting point. I started by familiarising myself with the codebase, particularly the areas of:
* Picking apart the binary structure to read the model, texture and animation data.
* How the data is displayed (OpenGL) .
* The cross platform GUI framework in use (wxPython).

![A screenshot of the MYMC+ app from its repository.](ps2iodbdevblog1-mymcplusscreenshot.png)
_The *MYMC+* app as it stood before any modifications._


