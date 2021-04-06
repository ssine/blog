---
title: "用树莓派做温湿度上报与红外控制"
date: 2021-04-07T00:48:32+08:00
---

有段时间在折腾身边各种数据的采集，其中一个就是房间的温湿度。 一开始是买了个加湿器，但是不知道效果怎么样，就买了个小米的温湿度计，不料这个温湿度计导出数据十分麻烦，要连手机 APP 发到邮箱，所以就想搞一个自动采集并上报温湿度的装置，树莓派+传感器是最优选择了。

树莓派本质上是个带 GPIO 的小主机，GPIO 之外的操作和普通 ARM Linux 没什么区别。 重点就是 GPIO 的玩法，它的存在使得树莓派可以外接很多电子元器件，传感器、屏幕、发动机等等。

树莓派到手之后要先装好系统，跟随 [官方教程](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) 就可以了。 为了方便连接在内网给树莓派分配了固定 IP，除了第一次装机用显示器后面都连 SSH 操作了。 貌似玩树莓派的都喜欢 root 用户直接上，大概是因为很多硬件相关的程序都要 root 权限吧 ( 印象中嵌入式设备都不怎么安全呢 ) 。

## 温湿度传感器 DHT22

为了检测温湿度买了 DHT22 传感器 ( 顺便吐槽下市面上的温湿度计，几百块的湿度计精度也才 5%RH，几块钱的 DHT22 精度都能到 2.5%RH ) ，用 Python 的 CircuitPython 库来读取。 [这里](https://learn.adafruit.com/dht-humidity-sensing-on-raspberry-pi-with-gdocs-logging/python-setup) 有官方的教程。

有一个入门的时候比较容易迷惑的点，就是传感器如何接线到引脚，以及如何把接好的引脚跟代码里的描述对应起来。

首先是怎么接线。 树莓派自带了一个很好用的工具 [Pinout](https://gpiozero.readthedocs.io/en/stable/cli_tools.html) ，只要在命令行输入 `pinout` 并回车就能看到板子的布局和每个引脚的功能：

```text
,--------------------------------.
| oooooooooooooooooooo J8   +======
| 1ooooooooooooooooooo  PoE |   Net
|  Wi                    oo +======
|  Fi  Pi Model 4B  V1.4 oo      |
|        ,----.               +====
| |D|    |SoC |               |USB3
| |S|    |    |               +====
| |I|    `----'                  |
|                   |C|       +====
|                   |S|       |USB2
| pwr   |HD|   |HD| |I||A|    +====
`-| |---|MI|---|MI|----|V|-------'

Revision : b03114
SoC : BCM2711
RAM : 2048Mb
Storage : MicroSD
USB ports : 4 ( excluding power )
Ethernet ports : 1
Wi-fi : True
Bluetooth : True
Camera ports ( CSI ) : 1
Display ports ( DSI ) : 1

J8:
    3V3 ( 1 ) ( 2 ) 5V
  GPIO2 ( 3 ) ( 4 ) 5V
  GPIO3 ( 5 ) ( 6 ) GND
  GPIO4 ( 7 ) ( 8 ) GPIO14
    GND ( 9 ) ( 10 ) GPIO15
GPIO17 ( 11 ) ( 12 ) GPIO18
GPIO27 ( 13 ) ( 14 ) GND
GPIO22 ( 15 ) ( 16 ) GPIO23
   3V3 ( 17 ) ( 18 ) GPIO24
GPIO10 ( 19 ) ( 20 ) GND
 GPIO9 ( 21 ) ( 22 ) GPIO25
GPIO11 ( 23 ) ( 24 ) GPIO8
   GND ( 25 ) ( 26 ) GPIO7
 GPIO0 ( 27 ) ( 28 ) GPIO1
 GPIO5 ( 29 ) ( 30 ) GND
 GPIO6 ( 31 ) ( 32 ) GPIO12
GPIO13 ( 33 ) ( 34 ) GND
GPIO19 ( 35 ) ( 36 ) GPIO16
GPIO26 ( 37 ) ( 38 ) GPIO20
   GND ( 39 ) ( 40 ) GPIO21
```

样式更丰富的布局也可以在网站 https://pinout.xyz/ 上面找到。 有了板子的布局，按照传感器的说明接线即可。

另一个问题是如何在代码中使用对应的引脚。 不同的程序对树莓派的引脚有不同的编码方式，使用 `gpio readall` 指令可以查看这些编码：

```text
 +-----+-----+---------+------+---+---Pi 4B--+---+------+---------+-----+-----+
 | BCM | wPi |   Name  | Mode | V | Physical | V | Mode | Name    | wPi | BCM |
 +-----+-----+---------+------+---+----++----+---+------+---------+-----+-----+
 |     |     |    3.3v |      |   |  1 || 2  |   |      | 5v      |     |     |
 |   2 |   8 |   SDA.1 |   IN | 1 |  3 || 4  |   |      | 5v      |     |     |
 |   3 |   9 |   SCL.1 | ALT0 | 1 |  5 || 6  |   |      | 0v      |     |     |
 |   4 |   7 | GPIO. 7 |   IN | 1 |  7 || 8  | 1 | ALT5 | TxD     | 15  | 14  |
 |     |     |      0v |      |   |  9 || 10 | 1 | ALT5 | RxD     | 16  | 15  |
 |  17 |   0 | GPIO. 0 |   IN | 1 | 11 || 12 | 0 | OUT  | GPIO. 1 | 1   | 18  |
 |  27 |   2 | GPIO. 2 |   IN | 0 | 13 || 14 |   |      | 0v      |     |     |
 |  22 |   3 | GPIO. 3 |   IN | 0 | 15 || 16 | 0 | IN   | GPIO. 4 | 4   | 23  |
 |     |     |    3.3v |      |   | 17 || 18 | 0 | IN   | GPIO. 5 | 5   | 24  |
 |  10 |  12 |    MOSI | ALT0 | 0 | 19 || 20 |   |      | 0v      |     |     |
 |   9 |  13 |    MISO | ALT0 | 0 | 21 || 22 | 0 | IN   | GPIO. 6 | 6   | 25  |
 |  11 |  14 |    SCLK | ALT0 | 0 | 23 || 24 | 1 | OUT  | CE0     | 10  | 8   |
 |     |     |      0v |      |   | 25 || 26 | 1 | OUT  | CE1     | 11  | 7   |
 |   0 |  30 |   SDA.0 |   IN | 1 | 27 || 28 | 1 | IN   | SCL.0   | 31  | 1   |
 |   5 |  21 | GPIO.21 |   IN | 1 | 29 || 30 |   |      | 0v      |     |     |
 |   6 |  22 | GPIO.22 |   IN | 1 | 31 || 32 | 0 | IN   | GPIO.26 | 26  | 12  |
 |  13 |  23 | GPIO.23 |   IN | 0 | 33 || 34 |   |      | 0v      |     |     |
 |  19 |  24 | GPIO.24 |   IN | 0 | 35 || 36 | 0 | IN   | GPIO.27 | 27  | 16  |
 |  26 |  25 | GPIO.25 |   IN | 0 | 37 || 38 | 0 | IN   | GPIO.28 | 28  | 20  |
 |     |     |      0v |      |   | 39 || 40 | 0 | IN   | GPIO.29 | 29  | 21  |
 +-----+-----+---------+------+---+----++----+---+------+---------+-----+-----+
 | BCM | wPi |   Name  | Mode | V | Physical | V | Mode | Name    | wPi | BCM |
 +-----+-----+---------+------+---+---Pi 4B--+---+------+---------+-----+-----+
```

其中 BCM 列的引脚可以用 `board.D<pin>` 来表示，例如 BCM=2 的引脚就是 `board.D2` 。

回到读取温湿度传感器的主题，接好传感器之后，确认数据线对应的引脚，就可以搞软件部分了。 首先安装相关的库：

```bash
pip3 install adafruit-circuitpython-dht
sudo apt-get install libgpiod2
```

之后就可以测试了：

```python
import board
import adafruit_dht

dhtDevice = adafruit_dht.DHT22(board.D2)

temperature_c = dhtDevice.temperature
humidity = dhtDevice.humidity

print(temperature_c, humidity)

dhtDevice.exit()
```

## 红外接收器与发射器

搞红外接收与发射是为了自动开关电灯，以及遥控空调来节省空调遥控器的电池浪费。

红外接收器使用的是 1838 红外接收头封装的一个模块，发射器就是普通的红外二极管串联一个 10 欧级别的电阻，不过发射角只有 30° 挺不方便，最好还是买个角度大一点的。

连好线之后就是软件部分。 红外的接收有一个好用不火的 python 包 [ircodec](https://github.com/kentwait/ircodec) 。使用很简单，首先安装 ( 需要 root ) ：

```bash
pip3 install pigpio ircodec
pigpiod  # 启动服务
```

后面分成两步，接收遥控器信号并记录，以及读取记录的信号并发射。 首先是接收信号以及记录：

```python
from ircodec.command import CommandSet
controller = CommandSet(emitter_gpio=22, receiver_gpio=23, description='xxx')

# Add the volume up key
controller.add('volume_up')
# Connected to pigpio
# Detecting IR command...
# Received.

controller.save_as('test.json')
```

读取并发射：

```python
controller = CommandSet.load('another_testtv.json')
controller.emit('volume_up')
```

。接收的部分用 Jupyter Notebook 来做会很方便调试。
