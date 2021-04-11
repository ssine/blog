---
title: "Raspberry Pi for temperature and humidity reporting and infrared control"
date: 2021-04-07T00:48:32+08:00
---

__Note: This article is automatically translated, please turn to the Chinese version for more accurate expression if possible.__

For a while, I was tossing around collecting various data, one of which was the temperature and humidity of the room. I bought a humidifier at the beginning, but I don’t know how it works, so I bought a Xiaomi thermometer and hygrometer. Unexpectedly, exporting data from this thermometer and hygrometer is very troublesome. It needs to be sent to the mailbox with the mobile APP, so I wanted to create an automatic one. The device for collecting and reporting temperature and humidity, Raspberry Pi + sensor is the best choice.

Raspberry Pi is essentially a small host with GPIO, and operations other than GPIO are no different from ordinary ARM Linux. The key point is the gameplay of GPIO. Its existence allows the Raspberry Pi to be connected to many electronic components, such as sensors, screens, engines, and so on.

After you get the Raspberry Pi, you must install the system first, and follow[Official tutorial](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up) That's it. In order to facilitate the connection to the internal network, the Raspberry Pi is assigned a fixed IP, except for the first installation of the monitor behind the display, SSH operation is connected. It seems that people who play Raspberry Pi like root users directly, probably because many hardware-related programs require root permissions (I think embedded devices are not very safe).

## Temperature and humidity sensor DHT22

In order to detect the temperature and humidity, I bought a DHT22 sensor (by the way, there are thermometers and hygrometers on the market under Tucao. The accuracy of a few hundred yuan of hygrometers is only 5%RH, and the accuracy of a few dollars of DHT22 can reach 2.5%RH), using Python's CircuitPython Library to read.[Here](https://learn.adafruit.com/dht-humidity-sensing-on-raspberry-pi-with-gdocs-logging/python-setup) There is an official tutorial.

One point that is easy to confuse when getting started is how to wire the sensor to the pin and how to match the connected pin with the description in the code.

The first is how to wire it. Raspberry Pi comes with a very useful tool[Pinout](https://gpiozero.readthedocs.io/en/stable/cli_tools.html) , Just enter at the command line`pinout` And press Enter to see the layout of the board and the function of each pin:

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

A richer layout can also be found on the website https://pinout.xyz/. With the layout of the board, follow the instructions of the sensor to connect the wires.

Another problem is how to use the corresponding pins in the code. Different programs have different coding methods for the pins of Raspberry Pi, use`gpio readall` Instructions can view these codes:

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

The pins in the BCM column can be used`board.D<pin>` To indicate that, for example, the pin with BCM=2 is`board.D2` 。

Back to the topic of reading the temperature and humidity sensor, after connecting the sensor, confirm the pin corresponding to the data line, and you can start the software part. First install the relevant libraries:

```bash
pip3 install adafruit-circuitpython-dht
sudo apt-get install libgpiod2
```

Then you can test:

```python
import board
import adafruit_dht

dhtDevice = adafruit_dht.DHT22(board.D2)

temperature_c = dhtDevice.temperature
humidity = dhtDevice.humidity

print(temperature_c, humidity)

dhtDevice.exit()
```

## Infrared receiver and transmitter

The purpose of infrared receiving and transmitting is to automatically turn on and off the lights and to remotely control the air conditioner to save the battery waste of the air conditioner remote control.

The infrared receiver uses a module encapsulated by the 1838 infrared receiver. The transmitter is an ordinary infrared diode in series with a 10 ohm resistor. However, it is inconvenient to have an emission angle of only 30°. It is better to buy a larger angle.

After the line is connected, it is the software part. There is a python package that is easy to use (but not hot) for infrared receiving and sending: [ircodec](https://github.com/kentwait/ircodec) . It is very simple to use, first install (root is required):

```bash
pip3 install pigpio ircodec
pigpiod  # 启动服务
```

The following is divided into two steps, receiving and recording the remote control signal, and reading and transmitting the recorded signal. The first is to receive the signal and record:

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

Read and launch:

```python
controller = CommandSet.load('test.json')
controller.emit('volume_up')
```

. It is convenient to debug the received part using Jupyter Notebook.
