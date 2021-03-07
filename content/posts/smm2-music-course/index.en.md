---
title: 'Super Mario Maker 2: Music Level'
date: '2019-10-15'
---

I bought switch for _Super Mario Maker2_, and have been playing in multiplayer and durance mode after I've cleared story mode. The idea of making music levels strikes me and I've started investigation.

For thoese who doesn;t know music levels, see the video: https://www.bilibili.com/video/av65739781 .

# The Mechanism

First of all, explain the mechanism of the music map in the game, there are three points:

1. The system only loads blocks near the screen
2. When the object falls on the note, it will make a sound
3. The height at which notes are placed affects the pitch

Recommend a level that shows the related content: XRY-1KH-GPF

![基本块](building_block.jpg)

As shown in the figure above, put the chestnut on the note, the chestnut will fall to the note when the note and the chestnut are loaded, in order to prevent the chestnut from popping back when it sounds again, place a cloud brick to stop the chestnut, and Put a wall around to prevent the chestnut from moving.

The three rules are described in detail below.

## Block Loading

![Block Loading](block_loading.png)

As shown in the figure above, during the movement from left to right, the system loads the objects in the front 4 squares on the right side of the screen and removes the 8 objects behind. The notes and the objects above are placed in order, and when they are loaded, the object will fall to the notes and make a sound, thus forming a melody.

## Voice of Different Objects

Some sound effects currently known<sup>[1]</sup>：

![音效](sound_effect.jpg)

## Note Height and Pitch

A picture can explain the relationship between note height and pitch<sup>[1]</sup>：

![音调](tune.png)

Please refer to the [twelve averaging law](https://baike.baidu.com/item/%E5%8D%81%E4%BA%8C%E5%B9%B3%E5%9D%87%E5%BE%8B).

# Composition of Music Level

After understanding the principle of the music map, we can try to make a music map. The raw material of the music map is the score. All we have to do is to send the **corresponding tones** in the score at the **appropriate time** during the tour . First look at how to control the speed of music.

## Speed

The speed and beat of the music determine the speed of the level and the spacing of the notes. 𝅘𝅥=100 in the staff means playing 100 quarter notes per minute, there will be similar marks in the notation, converting the speed to the number of notes per second, and then selecting a suitable forward mode in the level. After a lot of experiments, the speed of different ways forward is as follows:

| kind | speed (unit / s) |
|:--:|:--:|
| walk | 5.64 |
| Conveyor belt | 3.75 |
| Ordinary scroll | 3.75 |
| Blue lava table | 9.3 |
| track | 2.81 |

Running at twice the speed of the walk, the slow and fast reels are 1 ⁄ 2 and 2 times the normal reel, respectively , and the conveyor belt is the same.

How to choose the moving speed according to the music beat? This is determined by the shortest key in the score. For example, the score speed is 60 quarter notes per minute, and the shortest note used is the eighth note, then the interval in the game is in eighth notes. 120 eighth notes per minute, 2 per second, if each eighth note occupies two squares, the required speed is 4 squares per second, you can use the normal scroll ( 3.75 ) or run on the reverse reel ( 3.78 ) to approximate.

## Pitch

Due to the height limitation of the horizontal map, the span of the sound is only two octaves, very limited. However, the vertical map cannot be made into a music map, because the pitch of the note is related to the height of the placement, but only two octaves are looped. There is also an additional limitation that it is necessary to emit a non-interfering sound that occupies at least four vertical spaces (notes, clouds, spaces, walls), so it is difficult to form chords. There is currently no good solution.

---

<sup>[1]</sup>["Super Mario Maker 2" music level production method](https://www.3dmgame.com/gl/3784030.html)