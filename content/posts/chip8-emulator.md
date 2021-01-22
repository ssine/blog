---
title: 'CHIP-8 Emulator'
date: '2018-03-16'
---

A project from the design of computer composition principles.

<!-- more -->

# Introduction

## Why do it

The experimental class of computer composition principle should write a Virtual Machine in a high-level language. The first step is to hand over the design report. In the past few days, I have been thinking about what kind of VM to do after thinking about it. Software re-engineering the Proteus CPU of the last semester is too cold, and trying to write a Y86 simulation project is too large. When I was in class today, I saw my friend checking the NES simulator. I was inspired to make a game console simulator. It was very interesting, and I was able to run the game with a sense of accomplishment. Originally interested in early games, when the video games just started in the middle of the last century, those games were the originators. After that, the data collection began. According to history, the earliest game was played on the oscilloscope in the lab, followed by the PDP-1 machine, and then the first arcade appeared. However, these machines are not highly focused and have relatively few development documents. After that, the more popular one is the NES game console (Japan's FC). There is also a Gameboy that I am interested in. After deep understanding, I found that the workload to achieve them is a bit large, about a project of more than half a year, and the Final Project used to do one-half is stupid. (The Gameboy series has two upgraded versions of Gameboy Color and Gameboy Advanced. The GBA is directly on the ARM7 instruction set. Although it is very much wanted to do it, it is still too self-sufficient.) I am browsing the NES with fear of huge workload. When I found the information, I found a magical thing - Chip-8.

This is an interpreted programming language proposed in 1970 that runs on the dedicated **CHIP-8 Virtual Machine** to help people develop games more easily. Developer: Joseph Weisbecker. Of course, the point is that Chip-8 is probably the simplest actual simulation project I can currently implement. So just work on it, let's go.

## related resources

English tutorial [How to write an emulator (CHIP-8 interpreter)](http://www.multigesture.net/articles/how-to-write-an-emulator-chip-8-interpreter/)

Technical Doc [Cowgod’s Chip-8 Technical Reference v1.0](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)

Game ROM collection package [Chip-8 Games Pack](https://www.zophar.net/pdroms/chip8/chip-8-games-pack.html)

Github Project [Chip-8 Emulator](https://github.com/alexanderdickson/Chip-8-Emulator)

Chinese tutorial [手把手教你编写游戏模拟器 - Chip8篇(1)](http://blog.csdn.net/korekara88730/article/details/50987930)

JS Drawing [requestAnimationFrame for Smart Animating](https://www.paulirish.com/2011/requestanimationframe-for-smart-animating/)

Instructions for some games [David Winter's CHIP-8 emulation page](http://www.pong-story.com/chip8/)

Chip8 to Super Chip8 [SUPER-CHIP v1.1](http://devernay.free.fr/hacks/chip8/schip.txt)

# Machine overview

First, find out more about the Chip-8 VM. The following is organized from [Wikipedia](https://en.wikipedia.org/wiki/CHIP-8#Virtual_machine_description).

## RAM

CHIP-8 has 4K ( `0x1000`) memory, 8 bits long (the origin of chip 8). The CHIP-8 interpreter occupies 512 bytes ( `0x200`) before memory , and the highest 256 bytes ( `0xF00-0xFFF`) is reserved for display refresh. The 96 bytes ( `0xEA0-0xEFF`) below it are reserved for stack, internal use, and other variables.

In the current implementation, the interpreter does not need to be stored in memory as external code, but font data is typically stored in the lower 512 bytes.

## register

CHIP-8 has 16 8-bit registers, from `V0` to `VF` . `VF` Registers are used as flags for some instructions and should be avoided. (The carry flag in the addition, there is no borrow sign in the subtraction, the drawing is related to the pixel collision)

The name of the address register is called `I`, 16-bit word length and is related to several memory manipulation instructions.

## Stack

The stack stores the return address only when the subroutine is called. Today's implementations usually have a minimum of 16 layers.

## Timer

CHIP-8 has two timers. They count down at 60 Hz until 0.

* Delay Register: Used to time the event in the game, its value can be set and read.
* Sound Register: For sound effects, if its value is not 0, it beeps.

## Input

The input device is a 0 to F hex keyboard. Which `8462` is commonly used directional input. There are three instructions for detecting input. One is to skip an instruction when a particular key is pressed, and the other to skip an instruction when there is no button. The third waits for a keyboard input and saves it in a register.

## Graphics and sound

The native CHIP-8 display resolution is 64 x 32 pixels, monochrome. The image is rendered by drawing sprites that are 8 wide and 1 to 15 high. The Sprite pixel is XORed with the pixel on the corresponding position screen, so it is the pixel point at the corresponding position of the reverse screen. When a sprit is drawn, the `VF` register is set to 1 if there are pixels on the screen from light to dark , otherwise 0. This is used for collision detection.

The sound part is said in the sound register.

## Operation code table

CHIP-8 has 35 opcodes, all of which are 2 bytes long and are stored in big endian mode. Below is a list with the following flags:

* NNN: address
* NN: 8-bit constant
* N: 4-bit constant
* X and Y: 4-bit register identifier
* PC : Program Counter
* I : 16-bit register for memory addressing

see [Wikipedia](https://en.wikipedia.org/wiki/CHIP-8#Opcode_table) for more details.
