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
* A couple of projects [[1](https://gamefaqs.gamespot.com/boards/915821-playstation-2/79956200)] [[2](https://www.ps2-home.com/forum/viewtopic.php?t=1573&start=10)] of people sharing screen captures of different icons. This was a noble cause but didn’t encourage collaboration, didn’t share the assets in a usable way, and often didn’t have animations.
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
|0|4|Magic Header "PS2D"|
|4|2|0|
|6|2|Offset of 2nd line in titlename|
|8|4|0|
|12|4|Background transparency when save is selected 0x00(transparent) to 0x80(opaque)|
|16|16|Background color upper left (RGB-) 0x00 – 0x80|
|32|16|Background color upper right (RGB-) 0x00 – 0x80|
|48|16|Background color lower left (RGB-) 0x00 – 0x80|
|64|16|Background color lower right (RGB-) 0x00 – 0x80|
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
* Traversing the filesystem.
* Picking apart the bytes to read the model, texture and animation data.
* How the models are displayed (OpenGL).
* The cross platform GUI framework in use (wxPython).

![A screenshot of the MYMC+ app from its repository.](ps2iodbdevblog1-mymcplusscreenshot.png)
_The *MYMC+* app as it stood before any modifications._

One of the ultimate goals of the project was to arhive **all** variants of icons, that is, different versions of the save icons that developers could optionally create, for when the PS2 BIOS UI tries to copy or delete a save file.

<video controls autoplay>
    <source src="/assets/vid/ps2iodbdevblog1-iconvariants.webm" type="video/webm">
</video>
<em>A video showing of the PS2 BIOS showcasing some games that have multiple icon variants.</em>

Since this was one of the ultimate goals it is important to test its feasability early in the project. We know from the above learnings that the icon.sys file in each directory ends with three strings, that contain the filenames of the idle, copy & delete icons. 

```python
_icon_sys_struct = struct.Struct(
    "<4sHHII"
    "4I4I4I4I"
    "4f4f4f"
    "4f4f4f4f"
    "68s"
    "64s64s64s"
    "512s"
)
```
{: file="ps2iconsys.py"}
_First encounter with Python's [Struct class](https://docs.python.org/3/library/struct.html)_

The code above was existing code in `ps2iconsys.py`, the strings allow to interpret a byte array as a series of values that the struct class will extract for us when we call the *unpack* method. Let's disect the meaning a bit:
* The starting `<` specifies little-endian ordering.
* The `s` indicates a character, and the `4` coming before it means "4 chars" (1 char = 1 byte).
* Capital `H` means unsigned short (2 bytes).
* Capital `I` means unsigned int (4 bytes). 
* `f` means float (4 bytes).

All of this is known from the two easy to read tables [[1](https://docs.python.org/3/library/struct.html#byte-order-size-and-alignment)] [[2](https://docs.python.org/3/library/struct.html#format-characters)] on the struct class' documentation. If we compare those python strings to the *icon.sys file format* table in this blog above, we can start to see the alignment, focusing on the key parts:
* `4s` A 4 character magic header equalling "PS2D".
* `4I4I4I4I` 4 groups of 4 unsigned integers, for background colors.
* `4f4f4f` 3 groups of 4 floats, for light directions.
* Skip a few...
* `68s` A 68 character string, the savegame's title.
* `64s64s64s` Three groups of 64 chracter strings, **the filenames for the idle, copy and delete icon files!**

Once we call `_icon_sys_struct.unpack()` we can see the results...
![The unpacked struct array in the VS Code debugger.](ps2iodbdevblog1-unpackedstructvalues.png)
_The unpacked struct array in the VS Code debugger._

Looks like we did something right! We can see the array starts with a 4 character string "PS2D" as we expect, all numbers look *sensible* (we will verify soon), and most importantly, we can see some legible text in elements 50, 51 & 52 at the end, that is the filenames for the idle, copy & delete icons. This data is for Crazy Taxi, but the icons are still called *slime_.ico*, which is a bit funny. Throughout this project I have noticed multiple quirks with the icon storage of many games. But they are stories for another  time.

Now let's try and verify that atleast some of the numbers are correct. From the table above we know that the 4 groups of 4 integers are supposed to indicate the background color for each corner when the icon is focused. Let's compare the unpacked data from Half-Life to what we see in the PS2 BIOS UI. It's a bit easier to explain this with an accompanying image...

![An image depicting selection of relevant values from the unpacked struct, converting them into hex colors, and comparing them against the display in the PS2 BIOS.](ps2iodbdevblog1-colorbytesmatching.png)
_Converting the unpacked bytes and comparing with the PS2 BIOS display._

* Take the integers from the points of interest, specified by the bytes layout table.
* Convert each integer to its hex representation (e.g. 155 = 0x9b).
* Take the 3 hex values and combine them, representing red, green and blue (e.g. #70789B).
* Compare to the display of the PS2 BIOS, confirming the values are in the correct locations we expect.

Now after learning all of this structure and how it's read, lets try and use it to access the other icon variants. Now we're diving into the GUI code of the Python app. This is all written with [wxPython](https://wxpython.org/) which is a wrapper of the [wxWidgets](https://wxwidgets.org/) C++ library. The great benefit of this for this PS2IODB project, is that the program will be able to be distributed for people on all platforms (Windows, MacOS & Linux) to use.

Diving into the GUI layer was a daunting thought but that was quickly put to ease once the code was explored a bit. The files are quite length (which seems to be a trend across Python projects) but the code that touches the GUI is simple and easy to understand, which seems to be one of the goals of wxPython.

It was within `icon_window.py` (meaning the 3D rendered window) that I found the code handling icon loading and the right-click context menu. After observing the pattern I was able to add some new items that act as radio buttons, allowing the user to select between the normal, copy or delete icon variants. After binding these new options to a new function, we're able to to load the icon variant's data by using the filename retrieved from the icon.sys file. After seeing how easy this was to do with the existing codebase, I was confident that extending this existing Python codebase was the right choice.

![An image depicting a before/after of the 3D icon view window. The after shows new context menu options that allow for viewing previously inaccessible icon variants (copy & delete).](ps2iodbdevblog1-contextmenuvariantsselection.png)
_After adding some new options to the 3D icon view window, the other variants can be freely viewed, while animated, for the first time (pictured: Ridge Racer V)._

After adding this new functionality, I was able to view a whole lot of new icons, which was entertaining for hours. This sense of exploration was one of the many things that made the PS2 special, and a key reason for starting this project. 

Thanks for reading along this far! In the next devblog we'll take a look at starting to export the asset data into a format that can be easily viewed and shared without any special software. Feel free to share your thoughts in the comments, or subscribe to the [rss](/feed.xml) or newsletter below!

<div id="mc_embed_shell">
      <link href="//cdn-images.mailchimp.com/embedcode/classic-061523.css" rel="stylesheet" type="text/css">
  <style type="text/css">
        #mc_embed_signup{ border-radius: 5px; background: #fff; clear:left; font:14px Helvetica,Arial,sans-serif; width: 600px; color: black !important; h2 { padding-top: 9px; color: black; } }
        /* Add your own Mailchimp form style overrides in your site stylesheet or in this style block.
           We recommend moving this block and the preceding CSS link to the HEAD of your HTML file. */
</style>
<div id="mc_embed_signup">
    <form action="https://issung.us22.list-manage.com/subscribe/post?u=0f999a7769794d0da6ebc4939&amp;id=28509ec383&amp;f_id=0028c5e1f0" method="post" id="mc-embedded-subscribe-form" name="mc-embedded-subscribe-form" class="validate" target="_blank">
        <div id="mc_embed_signup_scroll"><h2>Newsletter</h2>
            <div class="indicates-required"><span class="asterisk">*</span> indicates required</div>
            <div class="mc-field-group"><label for="mce-EMAIL">Email Address <span class="asterisk">*</span></label><input type="email" name="EMAIL" class="required email" id="mce-EMAIL" required="" value=""></div>
        <div id="mce-responses" class="clear foot">
            <div class="response" id="mce-error-response" style="display: none;"></div>
            <div class="response" id="mce-success-response" style="display: none;"></div>
        </div>
    <div aria-hidden="true" style="position: absolute; left: -5000px;">
        /* real people should not fill this in and expect good things - do not remove this or risk form bot signups */
        <input type="text" name="b_0f999a7769794d0da6ebc4939_28509ec383" tabindex="-1" value="">
    </div>
        <div class="optionalParent">
            <div class="clear foot">
                <input type="submit" name="subscribe" id="mc-embedded-subscribe" class="button" value="Subscribe">
                <p style="margin: 0px auto;"><a href="http://eepurl.com/iPsdEg" title="Mailchimp - email marketing made easy and fun"><span style="display: inline-block; background-color: transparent; border-radius: 4px;"><img class="refferal_badge" src="https://digitalasset.intuit.com/render/content/dam/intuit/mc-fe/en_us/images/intuit-mc-rewards-text-dark.svg" alt="Intuit Mailchimp" style="width: 220px; height: 40px; display: flex; padding: 2px 0px; justify-content: center; align-items: center;"></span></a></p>
            </div>
        </div>
    </div>
</form>
</div>
<script type="text/javascript" src="//s3.amazonaws.com/downloads.mailchimp.com/js/mc-validate.js"></script><script type="text/javascript">(function($) {window.fnames = new Array(); window.ftypes = new Array();fnames[0]='EMAIL';ftypes[0]='email';fnames[1]='FNAME';ftypes[1]='text';fnames[2]='LNAME';ftypes[2]='text';fnames[3]='ADDRESS';ftypes[3]='address';fnames[4]='PHONE';ftypes[4]='phone';fnames[5]='BIRTHDAY';ftypes[5]='birthday';}(jQuery));var $mcj = jQuery.noConflict(true);</script></div>

