---
title: "Creating a 60Hz European copy of Ape Escape"
date: 2025-07-08
tags:
- gaming
- reverse
- ape-escape
- playstation
featuredImage: /images/ape_escape.png
---
## Introduction
**Ape Escape** is a famous PlayStation One game released in 1999 by **Sony Computer Entertainment Inc.**

Games released for PSX were created in three versions: NTSC, PAL and JP.

As most of you may know, back in 1999, **PAL games were about 17% slower than their NTSC/JP counterparts**, because our European televisions were slower (50Hz).
On top of that game creators based their FPS, physics and all that stuff on the CPU clock instead of some timer.
I actually don't know much about the MIPS processor (used in the PSX), but AFAIK **some games can be accelerated** just by simulating a NTSC console.

**This is not the case for Ape Escape**, as if you simulate NTSC, it simply increases the whole game speed, including audio and all movements, which is not something we want.

If you play the NTSC/JP version of this game you instantly notice that is way **smoother** and more **playable overall**, so a crazy man like me started to **reverse it to see if a PAL copy can run smoothly too**.

**Bad news**, a PAL copy can't do that, but **good news**, we can move localized content from a PAL copy to a NTSC copy so that **the NTSC code runs at 60Hz and shows PAL content!**

## Prerequisites
To create a 60Hz copy of Ape Escape we will simply replace some data from the PAL copy of your language to the NTSC one.

To do this we will use the following tools:
- `dumpsxiso` / https://github.com/Lameguy64/mkpsxiso
- `mkpsxiso` / https://github.com/Lameguy64/mkpsxiso
- `jPSXdec` / https://github.com/m35/jpsxdec
- `aemt` / https://github.com/deade1e/aemt

`dumpsxiso` is needed to **extract the iso content into a directory**.
`mkpsxiso` is needed to **re-pack the ISO content** after modification have been made.
`jPSXdec` is a tool that helps to **see inside the ISO file**. Also **decodes some audio / images**.
`aemt` is a small tool I created to **work with the KKIIDDZZ files**.

## Structure
The following is the **directory structure** of the Ape Escape's NTSC ISO:

```sh
.
├── DEMO
│   ├── A_HA_D1.STR
│   ├── A_HA_D2.STR
│   ├── A_NA_D1.STR
│   ├── A_NA_D2.STR
│   ├── A_RAI_D1.STR
│   ├── A_RAI_D2.STR
│   ├── A_SPE_D1.STR
│   ├── BUILDD1.STR
│   ├── BUILDD2.STR
│   ├── CASTLE.STR
│   ├── END1.ALL
│   ├── END1.STR
│   ├── END2.ALL
│   ├── END2.STR
│   ├── END3.ALL
│   ├── END3.STR
│   ├── FINAL_D1.STR
│   ├── INTER.ALL
│   ├── INTER.STR
│   ├── JUNGLE.ALL
│   ├── JUNGLE.STR
│   ├── OP.ALL
│   ├── OP.STR
│   ├── SFIN_D1.STR
│   ├── SFIN_D2.ALL
│   ├── SFIN_D2.STR
│   ├── SFIN_D3.STR
│   ├── TAKEOFF.ALL
│   ├── TAKEOFF.STR
│   ├── TITLE.ALL
│   └── TITLE.STR
├── KKIIDDZZ.BNS
├── KKIIDDZZ.DAT
├── KKIIDDZZ.HED
├── license_data.dat
├── MINIGAME
│   ├── MINI1
│   │   ├── CRS1.BIN
│   │   ├── CRS1.GRD
│   │   ├── CRS1V.BIN
│   │   ├── CRS2.BIN
│   │   ├── CRS2.GRD
│   │   ├── CRS2V.BIN
│   │   ├── CRS3.BIN
│   │   ├── CRS3.GRD
│   │   ├── CRS3V.BIN
│   │   ├── MINI1.EXE
│   │   ├── SKFIX.BIN
│   │   ├── SKLOAD.BIN
│   │   ├── SKREF.BIN
│   │   ├── SKSND.BIN
│   │   └── SKTIM.BIN
│   ├── MINI2
│   │   ├── BXFIX.BIN
│   │   ├── BXHEAD.BIN
│   │   ├── BXLOAD.BIN
│   │   ├── BXSARU.BIN
│   │   ├── BXSND.BIN
│   │   ├── BXSNDF.BIN
│   │   ├── BXTIM.BIN
│   │   └── MINI2.EXE
│   └── MINI3
│       ├── BANANA.DAT
│       └── BANANA.TMS
├── SCUS_944.23
├── STR
│   ├── END_TEST.ALL
│   ├── END_TEST.STR
│   ├── GOV1.STR
│   ├── GOV2.STR
│   ├── LAB.STR
│   ├── STEXP.STR
│   ├── TALK.STR
│   └── WEPGET.STR
└── SYSTEM.CNF

```

### DEMO
`DEMO` is a folder that contains lots of **cutscenes**.
They are stored inside the STR files, which is a file format that contains, among the things, ADPCM to be sent to the SPU.
For more info on the STR fromat check the excellent description linked in the references of this document.

We will, in fact, replace NTSC's STR files with our PAL's STR files and then adjust pointers/sizes in the HED, but this is work for later.

`.ALL` files contain images and other unknown stuff. Some of them can be replaced as they contain localized text.

### MINIGAME
Contains stuff  (images, also audio i think) about the minigames. Did not touch them as I wasn't interested, but a simple replace with other versions should work.

### STR
This folder also **contains cutscenes**, but these are continuously repeated, like at the end of each level or when you have a new gadget.
Developers decided to put them here, but they are of the same type of the `DEMO` folder.

### SCUS_944.23
**This is the "program" of the game**. It is a executable file that contains MIPS assembly.
Can be decompiled using Ghidra with a PSX plugin. More on that on another post.
In PAL versions you will find it under a different name.

### KKIIDDZZ.HED / DAT / BNS
The KKIIDDZZ files compose a big archive that contain other files.
And not only this. A portion of the HED file is dedicated to the pointing of some STR files, but let's start from the beginning.

#### 1. HED
The HED file is simply a **kind of "index"** composed by 32-bit numbers stored in little endian. The **first 20 bits are the file offset** and the **remaining 12 bits are the file size**. Offsets and sizes are expressed in CDROM data sectors (each data sector is 2048 bytes). You will notice that in PlayStation One games everything is aligned to CDROM sectors.

The offsets are related to the start of the `KKIIDDZZ.DAT` file. **HED also contains offsets/sizes of STR files** (cutscenes) inside the CDROM. **These offsets are absolute instead**.

#### 2. DAT / BNS
The DAT / BNS files are containers of other files. It's as simple as that. For example if you read the 6th 32-bit number from the HED file, you will get `Offset: 54` and `Size: 9`, this means that you will find a blob of data at the (54\*2048) position from the start of the DAT file and it will be (9\*2048) bytes long. This data can be of any type. Think of it like a .zip archive. If you are interested to go into detail please look at the references.

The BNS file is just the continuation of the DAT file. I don't know why developers did split them, but offsets that points in the BNS are always related to the start of the DAT.

I strongly suggest the usage `jPSXdec` to explore the ISO and its content before trying to modify stuff. It's useful also for debugging/understanding where the data lies.

To modify HED/DAT/BNS **you don't actually need to understand all of this**. You can just use `aemt`, which is a tool made for this specific purpose.

## Patch archives
As previously said, we must **patch the NTSC archive with some data from the PAL version**.

Some data will be **easily swapped as files**, some other **lies inside the KKIIDDZZ archive**.

We start from the "easy" one.

### Dump time!
First thing is to **dump all the ISO content** using `dumpsxiso`. Run the following commands:

Dump the PAL version of your choice (mine is italian):
`dumpsxiso -x dump_ita -s dump_ita/mkpsxiso.xml Ape\ Escape\ \(Italy\).bin`

Dump the NTSC version:
`dumpsxiso -x dump_ntsc -s dump_ntsc/mkpsxiso.xml Ape\ Escape.bin`

We now need to use `aemt` to patch the `NTSC KKIIDDZZ` archive with some data from the PAL one.

First off run `./aemt extracted_ntsc list --decimal`. It will yield this kind of output:

```sh
No.   Type  Offset     Length     Metadata  
0     DAT   0          1                    
1     DAT   1          17                   
2     DAT   18         16                   
3     DAT   34         8                    
4     DAT   42         12                   
5     DAT   54         9                    
6     DAT   63         9                    
7     DAT   72         17                   
8     DAT   89         98                   
9     DAT   187        37                   
10    DAT   224        128        ADPCM, Sound Pack
11    DAT   352        5          Sound Pack Pointers, SShd
...continues
```

This simply is the list of files contained in the KKIIDDZZ archive. **Notice that index starts from 0 (zero).**

### Patch time!
Simply run `./aemt dump_ita extract 10 ita_10.dat` to **extract the file at index 10** and save it to a file named `ita_10.dat`.

Now run `./aemt dump_ntsc patch 10 ita_10.dat` to **patch the file at index 10** with the previously extracted PAL version.

This is the list of the known files to be patched:
- 10: Sound pack of Spike doing stuff.
- 11: Pointers of this sound pack. Must be patched along with 10.
- 20: All the in-game mails.

There are others for sure, but at the time of writing they are unknown. **Please discover them and tell us!**

### Hedit time!
Yes you read correctly, `hedit`. This is the `aemt` command to **RAW edit the HED file**. We will use this command to **adjust the offsets and lengths of the STR files**.

This command does not modify any file but it is only used to adjust offset/length.

This is the STR portion of the output of `./aemt dump_ntsc list --decimal`:

```sh
No.   Type  Offset     Length     Metadata  
...
448   STR   32059      992        STR Pointer: LAB.STR
449   STR   33051      1328       STR Pointer: WEPGET.STR
450   STR   38475      1712       STR Pointer: GOV1.STR
451   STR   40187      1712       STR Pointer: GOV2.STR
452   STR   41899      992        STR Pointer: STEXP.STR
453   STR   51083      1008       STR Pointer: TALK.STR
454   STR   116577     722        STR Pointer: BUILDD1.STR
455   STR   121395     2794       STR Pointer: BUILDD2.STR
456   STR   124189     2846       STR Pointer: A_HA_D1.STR
457   STR   127035     626        STR Pointer: A_HA_D2.STR
458   STR   131757     2866       STR Pointer: A_NA_D1.STR
459   STR   134623     194        STR Pointer: A_NA_D2.STR
460   STR   138913     2194       STR Pointer: A_RAI_D1.STR
461   STR   141107     178        STR Pointer: A_RAI_D2.STR
462   STR   153573     2350       STR Pointer: A_SPE_D1.STR
463   STR   99755      2202       STR Pointer: SFIN_D1.STR
464   STR   113667     2910       STR Pointer: SFIN_D2.STR
465   STR   110149     3518       STR Pointer: SFIN_D3.STR
466   STR   155923     762        STR Pointer: CASTLE.STR
467   STR   160781     1202       STR Pointer: FINAL_D1.STR
```

This means that if we `hedit` the file at index 448 we are modifying its offset/length, in sectors, in the CDROM. This sounds complicated but thanks to **jPSXdec** we can easily see this info:

![Pasted image 20250706115407.png](/images/pi_20250706115407.png)

As you can see the output of the `list` command **perfectly matches what's under the "Sectors" column in jPSXdec**. For example the `TALK.STR` file is located at sector 51083 and it's 1008 sectors long. If you see under jPSXdec, the `TALK.STR` file starts at sector 51083 and ends at 52090. If we do the math 52090 - 51083 = 1007. We must always add 1 to this calculation as jPSXdec's range is inclusive.

As you may have noticed, **some STR files aren't mentioned** in the `aemt list` command and therefore aren't present in the HED file. I believe that the game program gets their offsets at run time via PsyQ, but that's food for another post.

Well, now that we know how to see the sectors each file lies on, **we must replace every .STR file inside `STR/` and `DEMO/` folder and then rebuild the ISO**. I personally did not replace `TITLE.STR` as it was different in size and added nothing in terms of locale.

Copy all STR files from your PAL dump to the NTSC one and then rebuild the ISO with the following command:

`mkpsxiso dump_ntsc/mkpsxiso.xml -o new_ape_escape.bin -c new_ape_escape.cue`

*.bin is equal to .iso*

Now if you open the `new_ape_escape.bin` file with jPSXdec, you will notice that sectors of STR files changed, and also their sizes. Time to `hedit`.. finally.

Run `./aemt dump_ntsc hedit <INDEX> <OFFSET> <LENGTH>` to edit these properties inside the HED file.

For example, after copying all Italian STR we discover that `TALK.STR` moved a bit back:

![Pasted image 20250706123539.png](/images/pi_20250706123539.png)

To fix it we issue `./aemt dump_ntsc hedit 453 49436 864`.
- `453` is the index of `TALK.STR` inside the HED file.
- `49436` is the new sector position.
- `864` is the new size. (In italian TALK is a bit smaller as you may notice).

**We must do this for all the files.**

*Remember that if you touch other files and rebuild the ISO, STR files positions may vary and you may need to adjust them again with `hedit`*.

After this re-run:

`mkpsxiso dump_ntsc/mkpsxiso.xml -o new_ape_escape.bin -c new_ape_escape.cue`

And you've got a working 60Hz copy of Ape Escape in your favorite language!

## Conclusion
This work is sourced from the deep passion I have about this game and from the help my brother Daddiu gave me during the research.

There is still something missing, especially regarding texts in the UI. I only cared about voices/cutscenes, hence this guide doesn't cover any text besides mails.

I sincerely hope that by creating AEMT some modding possibilities could open for the community.

Thank you for reading and do not hesitate to contact me if you have any problem!

## References
- https://github.com/m35/jpsxdec/blob/readme/jpsxdec/PlayStation1_STR_format.txt
- https://psx-spx.consoledev.net/soundprocessingunitspu/
- https://psx-spx.consoledev.net/cdromfileformats/#cdrom-file-archive-heddatbnsstr-ape-escape
- https://psx-spx.consoledev.net/cdromfileformats/#ape-escape-sound-archive-magdemo22kidzkkiiddzzheddat1bh-1dh49h-53h
- https://psx.arthus.net/sdk/Psy-Q/DOCS/FileFormat47.pdf
- https://psx.arthus.net/sdk/Psy-Q/DOCS/LibRef47.pdf
- https://psx.arthus.net/sdk/Psy-Q/DOCS/XATUT.pdf
