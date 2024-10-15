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

to see what steps **gbadoom** took to go from a fully-featured desktop port of doom down to one that would fit on a gameboy, the best route is the git history. 

> _aside:_ reading the git history of projects is an _exceptionally_ underrated way of learning how to do new things. want to write an os? look at how linux started. want to write a compiler? look at what rustc grew from.

...
