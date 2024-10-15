+++
title = "embedded doom part 1: analyzing gbadoom"
date = 2024-10-09

[taxonomies]
tags = ["doom", "software"]
+++

welcome back!

as foreshadowed, an article on doom. why? because i want to make an embedded doom port for a gag project and realized i have no idea how to do that. ^^;

plus, doom is essentially the [c. elegans](https://en.wikipedia.org/wiki/Caenorhabditis_elegans) of the software world: well-documented and simple enough to understand in parts, but complex and interesting enough to study in various ways. you can learn a lot about software development by studying the doom source.

so, we're going to study [gbadoom](https://github.com/doomhack/GBADoom), a port of doom to the [nintendo game boy advance](https://en.wikipedia.org/wiki/Game_Boy_Advance). (bear with me, i promise i'll get to why.)

### /// the challenge of embedded doom ///

doom was released by [id software](https://en.wikipedia.org/wiki/Id_Software) and had the following [minimum requirements](https://www.reddit.com/r/gaming/comments/a4yi5t/original_doom_system_requirements_from_my_25_year/):

- "IBM or compatible 386 or better"
- 4MB RAM
- VGA graphics card
- Hard disk drive

doom is still renowned today for being _insanely_ portable due to low system requirements and careful modular design. notably, because the i486's [fpu](https://en.wikipedia.org/wiki/Floating-point_unit) was too slow for use in doom, the codebase even includes a purpose-built fixed-point arithmetic library used for all of the rendering calculations, meaning it's particularly amenable to running on microcontrollers... 

..._except for that whole "4MB" of RAM thing._ that's what we need to work around. but how?

### /// existing ports ///

there have been several remarkable embedded ports of doom in just the last decade. here's a highlight of some:

| _port_ | _device_ | _processor_ | _ram_ | _writeup_ |
| --- | --- | --- | --- | --- |
| [stm32doom](https://github.com/floppes/stm32doom) | stm32f429i-disc1 | arm cortex-m4 | **256KB** | none |  
| [psion-doom](https://github.com/doomhack/PsionDoom) | epoc r5 pdas | cl-ps7110 | **8MB** | none |
| [gbadoom](https://github.com/doomhack/GBADoom) | gameboy advance   | arm7tdmi       | **32KB**  | none    | 
| [game-and-watch-doom](https://github.com/ghidraninja/game-and-watch-doom) | game and watch    | arm cortex-m7  | **1.4MB** | [video](https://www.youtube.com/watch?v=sNg_S9UM5ps) |
| [mg21doom](https://github.com/marciopocebon/MG21DOOM) | ikea trådfri lamp | arm cortex-m33 | **108KB** | none    | 
| [magiclantern-doom](https://github.com/turtiustrek/magiclantern_doom) | eos 200d camera   | arm946e-s      | ???   | none    |  
| [nrfdoom](https://github.com/NordicPlayground/nrf-doom) | nrf5340           | arm cortex-m33 | **512KB** | [blog](https://devzone.nordicsemi.com/nordic/nordic-blog/b/blog/posts/doom-on-nrf5340) |
| [nrf52840doom](https://github.com/next-hack/nRF52840Doom) | nrf52840          | arm cortex-m4  | **256KB** | [blog](https://next-hack.com/index.php/2021/11/13/porting-doom-to-an-nrf52840-based-usb-bluetooth-le-dongle/) |
| [rp2040-doom](https://kilograham.github.io/rp2040-doom/) | nrf52840          | arm cortex-m0+ | **264KB** | [article](https://kilograham.github.io/rp2040-doom/) |

you might note that there's a couple outliers in the ram column. 108KB? 32KB?? how do you get a game like doom down from 4MB to just 32KB?

### /// disecting gbadoom ///

the _really_ low-memory ports above all seem to share a common ancestry: they tend to be based on **gbadoom**, which in turn was based on [prBoom](https://prboom.sourceforge.net/), a well-loved port of doom emphasizing code quality and bug fixes while maintaining the original spirit of the game. 

to see what steps **gbadoom** took to go from a fully-featured desktop port of doom down to one that would fit on a gameboy, the best route is the git history. the general gist of what happens next is outlined in `README.md` from tag `0.1`:

> - Delete all the stuff that isn't needed.
> - Rewrite the IWAD handling to use const memory pointers so it can run from memory-mapped rom.
> - Remove as many non-const global vars as possible.
> - Move the remaining globals to a global struct which can be allocd in EWRAM to free up the 32kb of fast IWRAM.
> - Port the remaing mess to the GBA.
> - Move the hot-path code and data to IWRAM.
> - Play DOOM at about 7fps. Yay!

we'll cover these step by step. first, deleting stuff--this is pretty straightforward. 

### /// deleting stuff ///

the entire first chunk of the commit history is tossing out code. the dropped features include:

- **compatibility**: 
  
  prBoom has a [compatibility mode](https://doomwiki.org/wiki/PrBoom) setting to match the behavior of previous games--this saves a lot of memory off the bat

- **screen and audio adjustments**: 

  we don't need to allow various screen resolutions or numbers of audio channels, and the volume control is handled by the console

- **uncapped framerate support**

  a fixed framerate is fine, so all of the logic re: player movement interpolation and smoothing can be discarded

- **screenshots**, **netgame support**

  these can't really be done well on the gba anyway!
  
- **music**

  this is removed for space now, but is added back later in a more lightweight manner.

- **settings**, **console commands**

  every configurable setting was converted to a constant to save RAM, after which configuration loading was removed as well: 

  - key bindings
  - map, menu, and heads-up display (hud) colors
  - hud enable/disable
  - weapon preferences

- **wad (doom map file) loading**

  wads are now directly linked into the code, rather than loaded from a filesystem

according to the commit message following these changes, the memory footprint of prBoom was then **1.5MB**--still a far cry from 32KB.

# /// moving globals ///

the next step in porting was hoisting all of the globals into an 800-line `globals_t` struct with its own header file, `global_data.h`. 

from my understanding, the idea here is that now there's only a single memory address being passed around, and each function which recevies the struct can then take fixed offsets into it rather than having to pass around multiple addresses, meaning (ideally) less stack space is required. 

there's an interesting related discussion on stack overflow [here](https://stackoverflow.com/questions/6687088/performance-implications-of-global-variables-in-c) that's worth a peek.

# /// the interesting stuff /// 
